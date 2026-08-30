# Design: userspace conntrack SHM logging for Open vSwitch

## Problem

Userspace OVS conntrack updates connection status (`CS_*`, exported as
`CT_DPIF_STATUS_*`) and L4 state machines (TCP: `SYN_SENT`, `SYN_RECV`,
`ESTABLISHED`, `FIN_WAIT_1`/`FIN_WAIT_2`, `TIME_WAIT`, `CLOSED`, …; SCTP
similarly) inside `lib/conntrack.c` and `lib/conntrack-tcp.c`.  None of
those transitions are logged.  The only operational views are:

- `ovs-dpctl dump-conntrack` / `ct_dpif_dump_*`, a point-in-time snapshot
  of `struct ct_dpif_entry`
- a few administrator `VLOG`s that are not per-flow or per-state

There is no way to reconstruct the lifetime of a connection — when it
became `ESTABLISHED`, when NAT flags flipped, when it entered
`TIME_WAIT`, when it expired — without continuously polling dump, which
misses short-lived flows and is too expensive on a PMD.

This work is an independent, upstreamable feature.  It is motivated by
a production NAT datapath; the design is written against upstream OVS
types only.

## Goals

- Emit one fixed-size record per conntrack event (new, update, expire,
  invalid) from the userspace datapath.
- Keep the writer path **wait-free** and **atomic-free**: never take a
  lock, never block, never allocate, never VLOG, never use atomics.
  The producer always writes.  The only ordering is a memory fence
  after the payload stores.
- Isolate writers: **one SPSC ring per writer thread**, not per PMD
  only.  Non-PMD threads sometimes create CT records.  Deletes/expires
  are usually done by the `ct_clean` thread — that thread **must** have
  its own ring and emit `EXPIRE` there.  PMD rings emit `NEW` / `UPDATE`
  / `INVALID` from the execute path.  No cross-thread atomics on the
  record payload.
- Let a **separate process** consume events by mapping the same file in
  `OVS_RUNDIR`.  The reader is not in the OVS address space and can be
  written from the 4K header alone.
- Make the record a faithful, packed copy of `struct ct_dpif_entry`
  dump fields plus a few fields dump does not have.
- Stay portable to Linux and FreeBSD via a POSIX shared mapping of a
  named file.  No Linux-only primitives in the ABI.

## Non-goals

- Kernel datapath (`nf_conntrack`) logging.
- Guaranteed delivery (a slow reader can overrun; it resyncs).
- Storing ALG helper state beyond a fixed-size name.
- A stable cross-host / cross-endian capture format (SHM is same-host).

## Architecture

```
  PMD 0          PMD 1         ct_clean        other non-PMD
  ROLE_PMD       ROLE_PMD      ROLE_CT_CLEAN   ROLE_OTHER
    |              |                |               |
    v              v                v               v
  ring 0         ring 1          ring 2           ring 3     wait-free SPSC
    \              |                |               /
     \             |                |              /
      +---- $OVS_RUNDIR/ovs-ct-shm (file-backed mmap) ----+
                            |
                            v
                     reader process (mmap)
```

The feature is **off by default**.  When disabled, no SHM file is
created and the hot path is a single predictable false branch.  When
enabled at datapath init the writer creates `$OVS_RUNDIR/ovs-ct-shm`
(typically `/var/run/openvswitch/ovs-ct-shm`) with `open` + `ftruncate`
+ `mmap(MAP_SHARED)`, and fills `struct ovs_ct_shm_hdr` (exactly 4K).
Each writer thread is assigned a ring and stamps `thread_id` +
`thread_role`.

A reader opens the same path `O_RDONLY`, maps the first page, checks
`magic` / `version`, then uses `hdr_size`, `page_size`, `record_size`,
`n_rings`, `ring_capacity`, `ring_stride`, `rings_offset`,
`ring_ctrl_size` from the header (never hardcodes sizes except as
defaults when creating a segment).  It then maps the rest and consumes
each ring independently.

## User knobs (other_config / similar)

All off/default unless set.  Suggested keys:

| Knob | Default | Meaning |
| --- | --- | --- |
| `other_config:ct-shm-enable` | `false` | Create SHM and emit events.  Off ⇒ no file, one false branch. |
| `other_config:ct-shm-n-rings` | cover PMDs + `ct_clean` + other non-PMD writers | Rings must cover every thread that can emit. |
| `other_config:ct-shm-ring-capacity` | `65536` | Slots per ring; **power of two**. |
| `other_config:ct-shm-event-mask` | all (`0xf`) | Which of NEW / UPDATE / EXPIRE / INVALID to emit. |

The live `event_mask` is also stored in the 4K header so a reader can
see which types the writer promised.

## Writer protocol (no atomics)

Capacity `C` is a power of two.  `write_idx` is a **single-writer**
`uint64_t`.  The slot is `write_idx & (C - 1)`.  The ring does not
store a reader cursor.

Producer (one writer thread per ring):

1. Fill `records[write_idx & (C - 1)]` with **ordinary stores**.
2. **Memory fence** (`ovs_ct_shm_payload_fence`) so the payload is
   visible before the index.  On x86 TSO a compiler barrier is enough;
   on aarch64 `dmb ishst`.
3. **Plain store** of `write_idx + 1`.

Always write.  Old slots are overwritten when the reader is behind.

Each reader keeps its own cursor (not in SHM).  Independent readers
are fine.

1. Load `write_idx` (reader-side fence as needed).
2. If `write_idx - my_cursor > C`, the reader overran: resync to
   `write_idx - C` (or `write_idx`) and continue.
3. Consume `[my_cursor, write_idx)` and advance the private cursor.

## Performance

Negligible by construction:

- **Disabled (default):** no SHM mapping exists.  The emit site is
  `if (OVS_UNLIKELY(ct_shm_enabled)) emit…;` — one predictable false
  branch on the PMD / `ct_clean` path.
- **Enabled:** fill a pre-mapped record, one fence, one plain store
  of `write_idx`.  Never blocks, never allocates, never `VLOG`s on
  the write path.

## Why not VLOG / Netlink / a Unix socket?

Those paths allocate, format, take mutexes, or can block in `send`.
Any of that on the PMD is a latency and packet-loss bug.  A pre-mapped
ring with ordinary stores plus one fence is the smallest thing that
still lets an out-of-process tool observe state.

## Mapping onto `struct ct_dpif_entry`

The dump ABI is `lib/ct-dpif.h`.  The SHM record copies every
connection-tracking field that dump exposes, with one exception:

| `ct_dpif_entry` | SHM record | Notes |
| --- | --- | --- |
| `tuple_orig` | `tuple_orig` | `l3_type`, `ip_proto`, `src`/`dst`, ports or ICMP |
| `tuple_reply` | `tuple_reply` | same |
| `tuple_parent` | `tuple_parent` | related/expected parent 5-tuple |
| `helper` (`char *name`) | `helper_name[32]` | **see below** |
| `id`, `zone`, `mark` | same | |
| `labels` (`ovs_u128`) | `labels_lo`, `labels_hi` | `.u64.lo` / `.u64.hi` |
| `status` | `status` | `CT_DPIF_STATUS_*` / `CS_*` dump form |
| `timeout` | `timeout` | seconds remaining |
| `counters_orig/reply` | same | `packets`, `bytes` |
| `timestamp.start/stop` | `start_ns`, `stop_ns` | ns since Unix epoch |
| `protoinfo` | `protoinfo` | TCP states/wscale/flags; SCTP state/vtags |
| `have_labels` | `rec_flags` bit | dump-only bool |
| `bkt` | omitted | implementation bucket; not a CT attribute |

Fields dump does **not** have, added on the record:

- `event` — `NEW` / `UPDATE` / `EXPIRE` / `INVALID`
- `thread_id` + `thread_role` — writer identity (`PMD` / `CT_CLEAN` /
  `OTHER`).  Replaces v1 `core_id`.
- `time_ns` — `CLOCK_MONOTONIC` nanoseconds, or TSC if flagged
- `magic` + `abi_version` — so a snapshot of a slot is self-describing

### Helper name

`struct ct_dpif_helper { char *name; }` is a pointer into the writer
heap.  Storing that pointer in SHM would be meaningless (and dangerous)
in the reader.  The ABI therefore copies at most 31 characters plus NUL
into `helper_name`.  An implementation that does not want helper names
in the hot path may leave the array zeroed and must document that; the
slot remains in the record so the size stays stable.

## Shared-memory layout (ABI v2)

```
offset 0
  struct ovs_ct_shm_hdr          /* exactly 4096 bytes (one page):
                                    magic, version, hdr_size, page_size,
                                    cache_line, record_size, n_rings,
                                    ring_capacity, ring_stride, rings_offset,
                                    event_mask, flags, pid, ring_ctrl_size,
                                    created_ns, reserved pad to 4096 */

offset hdr.rings_offset          /* page-aligned; typically 4096 */
  for r in 0 .. n_rings-1:       /* each hdr.ring_stride, page-aligned */
      struct ovs_ct_shm_ring     /* 128 bytes, 2 cache lines:
                                    write_idx | thread id/role */
      struct ovs_ct_shm_record[ring_capacity]   /* 320 bytes each, 64-aligned */
      pad to next page

total mapping rounded up to a page
```

Helpers `ovs_ct_shm_ring_offset()`, `ovs_ct_shm_record_offset()`,
`ovs_ct_shm_compute_ring_stride()`, and `ovs_ct_shm_map_size()` in the
header compute these offsets from the live `hdr` fields.  Incompatible
changes bump `version`.

v2 vs v1: header grew from 64 to 4096, record from 280 packed to 320
(5×64) cache-line aligned, rings are page-aligned, writer identity is
`thread_id` + `thread_role`.  That is why `OVS_CT_SHM_VERSION` is 2.

## ABI (matches `include/ovs-ct-shm.h`)

```c
#define OVS_CT_SHM_MAGIC            0x43544c47u  /* "CTLG" */
#define OVS_CT_SHM_REC_MAGIC        0x43545245u  /* "CTRE" */
#define OVS_CT_SHM_VERSION          2u
#define OVS_CT_SHM_HDR_SIZE         4096u
#define OVS_CT_SHM_PAGE_SIZE        4096u
#define OVS_CT_SHM_CACHE_LINE       64u
#define OVS_CT_SHM_RECORD_SIZE      320u
#define OVS_CT_SHM_RING_CTRL_SIZE   128u
#define OVS_CT_SHM_DEFAULT_BASENAME "ovs-ct-shm"
#define OVS_CT_SHM_DEFAULT_DIR      "/var/run/openvswitch"
#define OVS_CT_SHM_HELPER_NAME_LEN  32

enum ovs_ct_shm_event {
    OVS_CT_SHM_EV_NEW     = 1,
    OVS_CT_SHM_EV_UPDATE  = 2,
    OVS_CT_SHM_EV_EXPIRE  = 3,
    OVS_CT_SHM_EV_INVALID = 4
};

enum ovs_ct_shm_thread_role {
    OVS_CT_SHM_ROLE_PMD      = 1,
    OVS_CT_SHM_ROLE_CT_CLEAN = 2,
    OVS_CT_SHM_ROLE_OTHER    = 3
};

struct ovs_ct_shm_tuple {            /* 40 bytes, packed */
    uint16_t l3_type;                /* AF_INET / AF_INET6 */
    uint8_t  ip_proto;
    uint8_t  pad;
    uint8_t  src[16];                /* network order; IPv4 in [0..3] */
    uint8_t  dst[16];
    union { uint16_t src_port; uint16_t icmp_id; };
    union {
        uint16_t dst_port;
        struct { uint8_t icmp_type; uint8_t icmp_code; };
    };
};

struct ovs_ct_shm_counters {         /* 16 bytes */
    uint64_t packets, bytes;
};

struct ovs_ct_shm_timestamp {        /* 16 bytes; ns since epoch */
    uint64_t start_ns, stop_ns;
};

struct ovs_ct_shm_protoinfo {        /* 16 bytes */
    uint16_t proto;                  /* IPPROTO_* */
    uint8_t  pad[2];
    union {
        struct {                     /* CT_DPIF_TCPS_* / CT_DPIF_TCPF_* */
            uint8_t state_orig, state_reply;
            uint8_t wscale_orig, wscale_reply;
            uint8_t flags_orig, flags_reply;
            uint8_t pad_tcp[2];
        } tcp;
        struct {                     /* CT_DPIF_SCTP_STATE_* */
            uint8_t  state;
            uint8_t  pad_sctp[3];
            uint32_t vtag_orig, vtag_reply;
        } sctp;
    };
};

struct ovs_ct_shm_record {           /* 320 bytes, packed, 64-byte aligned */
    uint32_t magic;                  /* OVS_CT_SHM_REC_MAGIC */
    uint16_t abi_version;            /* OVS_CT_SHM_VERSION */
    uint16_t event;                  /* enum ovs_ct_shm_event */
    uint32_t thread_id;
    uint16_t thread_role;            /* enum ovs_ct_shm_thread_role */
    uint16_t rec_flags;
    uint64_t time_ns;                /* monotonic ns, or TSC if flagged */

    uint32_t id;
    uint16_t zone;
    uint16_t pad1;
    uint32_t mark;
    uint32_t status;                 /* CT_DPIF_STATUS_* */
    uint32_t timeout;
    uint32_t pad2;
    uint64_t labels_lo, labels_hi;   /* ovs_u128 */

    struct ovs_ct_shm_tuple     tuple_orig, tuple_reply, tuple_parent;
    struct ovs_ct_shm_counters  counters_orig, counters_reply;
    struct ovs_ct_shm_timestamp timestamp;
    struct ovs_ct_shm_protoinfo protoinfo;
    char helper_name[32];            /* not a pointer; see above */
    uint8_t pad_cl[40];              /* 280 -> 320 (5 cache lines) */
};

struct ovs_ct_shm_ring {             /* 128 bytes, 64-aligned */
    uint64_t write_idx; uint8_t pad_prod[56];
    uint32_t thread_id;
    uint16_t thread_role, reserved;
    uint8_t  pad_tail[56];
};

struct ovs_ct_shm_hdr {              /* 4096 bytes */
    uint32_t magic;                  /* OVS_CT_SHM_MAGIC */
    uint32_t version;                /* OVS_CT_SHM_VERSION */
    uint32_t hdr_size;               /* 4096 */
    uint32_t page_size;              /* 4096 */
    uint32_t cache_line;             /* 64 */
    uint32_t record_size;            /* 320 */
    uint32_t n_rings;
    uint32_t ring_capacity;          /* power of two */
    uint32_t ring_stride;            /* page-aligned bytes per ring */
    uint32_t rings_offset;           /* page-aligned */
    uint32_t event_mask;
    uint32_t flags;
    uint32_t pid;
    uint32_t ring_ctrl_size;         /* 128 */
    uint64_t created_ns;
    uint8_t  reserved[4032];
};
```

Record size is 320 bytes (`_Static_assert`ed, `% 64 == 0`, align 64).
Header size is 4096.  Ring control size `% 64 == 0`.  All pads are
written zero so memcmp/fuzzers do not see uninitialized holes.

## Suggested hook points in OVS

These are conceptual; the first implementation patch should attach at
the existing mutation sites rather than sprinkling logs through TCP:

- **NEW** — successful insert of a `struct conn` (the path that today
  allocates the conn and sets initial `CS_*` / TCP `SYN_SENT`).  PMD
  execute path, or `ROLE_OTHER` if a non-PMD thread inserts.
- **UPDATE** — after a TCP/SCTP state change or a `CS_*` / mark / label
  change that dump would show.  Counter-only updates may be rate-limited
  or omitted; state changes must not be.  PMD execute path.
- **EXPIRE** — `ct_clean` timeout/sweep/flush deletion, with
  `timestamp.stop_ns` filled.  **Must** be emitted on the `ct_clean`
  thread's own ring (`ROLE_CT_CLEAN`), not on a PMD ring.
- **INVALID** — packet rejected before a conn is created (bad flags,
  out-of-window, etc.).  Tuples are from the packet; `id` may be zero.
  PMD execute path.

The producer must be ordinary stores, one fence, and one plain store of
the index; formatting is the reader's job.

## POSIX shared mapping in OVS_RUNDIR

The backing file lives in `OVS_RUNDIR`, e.g.
`$OVS_RUNDIR/ovs-ct-shm`, typically `/var/run/openvswitch/ovs-ct-shm`.
This is a named file-backed POSIX `mmap(MAP_SHARED)`, **not** a
nameless `/dev/shm` object and not `memfd_create`.

The writer:

1. `open(path, O_RDWR | O_CREAT, 0600)` where
   `path = $OVS_RUNDIR/ovs-ct-shm`.
2. `ftruncate(fd, map_size)` with `map_size` page-rounded.
3. `mmap(NULL, map_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0)`.
4. Initialize header and rings, then publish `magic` last.

The reader uses `O_RDONLY` / `PROT_READ`, maps 4K first, then the rest
from header sizes.

**Do not `unlink` / `shm_unlink` on crash.**  The file must remain
reachable after `ovs-vswitchd` dies so postmortem readers can still
map it.  A clean restart may reuse or replace the file; a crash must
leave it.

No `memfd_create`, `eventfd`, `io_uring`, or kqueue is required for the
ABI.  A reader may sleep on a timer; optional platform wakeups are out
of band and must not be assumed by the writer.

## Path to an upstream patch series

1. **Patch 1** — `ct-shm.h` ABI + documentation (this design, trimmed
   to the OVS tree's doc style).
2. **Patch 2** — SHM segment lifetime in the userspace datapath, gated
   by `other_config:ct-shm-enable` (off by default), with knobs for
   `n-rings`, `ring-capacity`, and `event-mask`.  File in `OVS_RUNDIR`.
3. **Patch 3** — wait-free producer helpers (plain stores + fence) and
   calls from conntrack insert/update (`ROLE_PMD` / `ROLE_OTHER`) and
   expire (`ROLE_CT_CLEAN`).
4. **Patch 4** — `ovs-ct-log` reader + unit tests that do not require a
   live PMD.  Reader parses the 4K header first.
5. **Patch 5** — NEWS / man page.

Keep the series free of out-of-tree types.  The record uses
`uintN_t` only; OVS code copies from `ct_dpif_entry` / `struct conn`
into that ABI at the hook.

## Testing the ABI in this repo

```sh
printf '%s\n' '#include "ovs-ct-shm.h"' 'int main(void) { return (int)sizeof(struct ovs_ct_shm_hdr) + (int)sizeof(struct ovs_ct_shm_record); }' \
  | cc -std=c11 -Wall -Wextra -Werror -I include -x c - -o /tmp/ovs-ct-shm-check && /tmp/ovs-ct-shm-check; echo exit:$?
```

`sizeof(struct ovs_ct_shm_hdr)` must be 4096.
`sizeof(struct ovs_ct_shm_record)` must be 320.
`main` returns 4416.
