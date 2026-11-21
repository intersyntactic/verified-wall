# PROPOSAL.md: what the verified wall asks of Theorem's model

Nick's call, 2026-11-21 18:50 AEDT: after RESEARCH-2.md, the forge's output is an upstream
proposal, not a gatekeeper design. Three pieces, in order:

1. **A first PR: `SO_PEERCRED` on the socket kinds the model already has** (E0 in
   RESEARCH-2.md). Touches no invariant; tests the path. Being drafted locally from
   `docs/THEOREM-PR-BRIEF.md`; Nick sends it.
2. **Issue A: Unix listening sockets with a peer credential per connection** (E1), drafted
   below.
3. **Issue B: Sv39 and the 6.12 kernel**, drafted below.

The issue texts are Claude's drafts for Nick to send under his own name, so they should be read
and rewritten in his voice first. Every line number is from LPS at `d3f3a701` and should be
re-checked against current main before sending.

---

## Issue A (draft): AF_UNIX listening sockets with a per-connection peer credential

**Title:** AF_UNIX stream listeners: a quantified peer credential per connection, and SO_PEERCRED

We'd like to prove a small Unix-socket service on riscv64 against LPS: one that accepts
connections, reads each caller's credential with `SO_PEERCRED`, and answers or refuses by uid.
Today it falls outside every statement: `socket(AF_UNIX, SOCK_STREAM)` is unmodeled
(`Sys/Net/Socket.v:284`), and `SO_PEERCRED` appears nowhere.

A shape that seems to leave the "no fd in flight" invariants alone (I1-I5 and I8 as we
understood them from `State.v:5860-5998`, `6164-6194`, `2826-2846` and
`Sys/Fs/ReadWrite.v:405-417`):

- A new launch input, a coinductive queue of AF_UNIX connections, each carrying
  `(pid, euid, egid)` and a `NetStream` script, quantified over every value the way `in_net`
  and `PeerAddr` are. `accept4` on an AF_UNIX listener draws from it; reads and writes reuse the
  `SkConnected` arms; `SO_PEERCRED` returns the entry's credential (12 bytes; a short buffer
  truncates, as in v6.18 `sock.c`).
- `sendmsg` keeps declining control data, so no descriptor travels (`Sys/Net/Socket.v:2513`).
- To avoid path-addressed binds (the model's `Fs` has no inode kinds), either bind only an
  abstract address, or accept a listening socket **inherited at launch** (systemd `Accept=no`,
  fd 3), which is the gap SCOPE.md:313 names.
- `NTimedOut` would need excluding on unix connections, since an AF_UNIX peer doesn't time out.

The property we'd state is a new one, so we'd want your view on its shape: every `WriteEv` on
connection n's channel implies the euid of the nth queued connection is in an allowed set, plus a
converse (an allowed connection with a well-formed request gets a reply), so a service that
never answers anyone doesn't satisfy it trivially.

We expect this is the kind of change you'd rather make yourselves (issue #6), so this is an
offer of a worked use case and help, not a request to merge our patch. We've started with a
small separate PR for `SO_PEERCRED` on socketpair ends and TCP sockets.

## Issue B (draft): Sv39 and the Debian 13 kernel

**Title:** Proofs that apply to Sv39 hardware (SiFive U74) and Linux 6.12

Our target board is a StarFive VisionFive 2 (JH7110). Its SiFive U74 cores support only Sv39
(U74 Core Complex Manual 21G3.02.00, §3.7, `satp.MODE` lists only Bare and Sv39), and Debian 13
runs 6.12. SCOPE.md:310 says Sv39 statements can't be proved for a runnable image yet, because
spec arms compare against the Sv48 `TASK_SIZE`, and SCOPE.md:20 lists other stable kernels as a
goal. Is threading the declared `VaSize` through the range checks on your roadmap, and is 6.12
among the kernels you'd consider? Many common RISC-V boards are Sv39-only, so this would decide
whether proofs apply to them at all.

---

## Parked: a proven logbook writer

With Issue A's extension, the smallest real gatekeeper answers connections itself instead of
handing them on (E1' in RESEARCH-2.md). In intersyntactic the natural such service is the
logbook writer (https://github.com/intersyntactic/logbook), whose README says "nothing checks it
as it runs, so a person reads it instead". A small static C writer with a proof that only
allowed callers' bytes are ever recorded is the client to come back to once Issues A and B have
answers. Open questions for then: the file-system side (appending objects and entries, which
LPS's `Fs` models only partly), crash consistency, and whether the doorkeeper-per-service shape
from SPARK.md still has a use.

## Forge disposition

Ore → Heat (two passes) → Spark → **stopped before Cast** by Nick's call. Not tempered: the
design was re-pointed at the upstream proposal because every gatekeeper shape depends on
extensions and platform support only Theorem can land. If the forge resumes, it resumes at Cast
with the logbook writer as the service, and its first falsifier is Issue B's answer.
