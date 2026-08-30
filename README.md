# ovs-ct-shm-log

Upstream-facing design and packed ABI for **per-state logging of Open vSwitch
userspace connection tracking**, delivered over a file-backed POSIX shared
mapping in `OVS_RUNDIR`.

OVS userspace conntrack (`lib/conntrack.c`, `lib/conntrack-tcp.c`) already
maintains `CS_*` flags and TCP states (`SYN_SENT`, `ESTABLISHED`,
`FIN_WAIT_2`, `TIME_WAIT`, `CLOSED`, …) but has no per-transition log.
Operators can only snapshot the table with `ovs-dpctl dump-conntrack` or
read a handful of admin `VLOG`s.  This project specifies a wait-free,
per-writer-thread ring so a separate reader process can consume every
new / update / expire / invalid event without stalling a PMD.

The ABI in [`include/ovs-ct-shm.h`](include/ovs-ct-shm.h) is version **2**:
a standardized **4K header** (so independent readers can be written from
the header alone), **320-byte** cache-line-aligned records, page-aligned
rings, and writer identity as `thread_id` + `thread_role` (`PMD` /
`CT_CLEAN` / `OTHER`).  It mirrors `struct ct_dpif_entry` from
`lib/ct-dpif.h` so an implementation can be submitted to OVS as a
self-contained patch series rather than an out-of-tree hook.  Motivated
by a production NAT datapath.

## Properties

- **Off by default.**  User knobs (`other_config:ct-shm-enable`,
  `ct-shm-n-rings`, `ct-shm-ring-capacity`, `ct-shm-event-mask`).  When
  disabled: no SHM, hot path is one predictable false branch.
- **Negligible cost when enabled.**  Wait-free SPSC: ordinary stores to
  fill the record, one fence, plain store of `write_idx`.
  No atomics on the write path.  Never blocks, never allocates, never
  `VLOG`s while writing.
- **One ring per writer thread**, not per PMD only.  The `ct_clean`
  thread has its own ring and emits `EXPIRE` there.  PMD rings emit
  `NEW` / `UPDATE` / `INVALID` from the execute path.  Other non-PMD
  threads that create CT records get `ROLE_OTHER` rings.
- **Backing file in `OVS_RUNDIR`** (`$OVS_RUNDIR/ovs-ct-shm`, typically
  `/var/run/openvswitch/ovs-ct-shm`).  Not a nameless `/dev/shm` object.
  Left in place after `ovs-vswitchd` crashes for postmortem readers.

## How this becomes an OVS patch series

1. Land this ABI as `include/openvswitch/ct-shm.h` (or `lib/ct-shm.h`)
   with the `_Static_assert`s intact (`sizeof hdr == 4096`,
   `sizeof record == 320`, record and ring control `% 64 == 0`).
2. Wire wait-free producers at the existing conntrack mutation points
   (insert/update/invalid on PMD or other writer threads; expire on
   `ct_clean`).  One SPSC ring per writer thread.
3. Create the file-backed mapping at datapath init only when enabled
   (`open` + `mmap` of `$OVS_RUNDIR/ovs-ct-shm`); do not unlink on
   crash.  Linux and FreeBSD need a POSIX shared file mapping — no
   Linux-only `memfd` or eventfd.
4. Ship a small reader (`ovs-ct-log`) that maps the 4K header first,
   then the rings, and prints records.  Tests can drive conntrack and
   assert events.
5. Post the series to `ovs-dev` with the design from
   [`DESIGN.md`](DESIGN.md).

This repository is the design and ABI only.  It is not a clone of
Open vSwitch and does not contain OVS sources.

## Layout

| Path | What |
| --- | --- |
| [`DESIGN.md`](DESIGN.md) | Problem, ring protocol, 4K header, OVS_RUNDIR, knobs, patch plan |
| [`include/ovs-ct-shm.h`](include/ovs-ct-shm.h) | C ABI v2 (4K header, ring, 320-byte record, event enum) |
| [`LICENSE`](LICENSE) | Apache-2.0, same as Open vSwitch |

## License

Apache License 2.0.
