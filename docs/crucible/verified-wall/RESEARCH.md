# RESEARCH.md: Heat pass for the verified wall

Read 2026-11-21. Sources pinned: LPS = `theorem-labs/rocq-linux-process-semantics` @ `d3f3a701d99afeee638963228562f4b5ff6e96ba`; OSV = `theorem-labs/oss-verify` @ `dc8238d020fcadf623c65ef3b3418a76b01eff68`. Links below use those commits.

## Verdict on falsifier 1: NOT COVERED

1. The model has the syscall *numbers* (socket 198 through recvmsg 212, accept4 242) in its table, but answers almost every AF_UNIX path the gatekeeper needs with `unmodeled`.
2. `socket(AF_UNIX, SOCK_STREAM, 0)` → `unmodeled`. Only AF_INET TCP streams, AF_UNIX/AF_INET6 *datagram* sockets (on which every stateful call declines), and in-process `socketpair` exist.
3. `getsockopt` → `unmodeled` for every option except `SO_SNDBUF`/`SO_RCVBUF` on a socketpair end. `SO_PEERCRED` appears nowhere in the model.
4. `sendmsg` on any AF_UNIX socket → `unmodeled`, "so no descriptor travels by `SCM_RIGHTS`". The model's invariants *depend on* fds never being in flight.
5. `recvmsg` answers only the errors before the fd lookup (EBADF/ENOTSOCK). On a real socket it returns no data.
6. The environment is one process. Peers are a coinductive queue of TCP connections carrying an address and a byte script, with **no uid**. The only uid in the input is the process's own (`in_uid`).
7. Signals: only SIGABRT and SIGPIPE, both fatal. No handlers, and "nothing outside the process can send one".
8. The proof rule is that an execution reaching `unmodeled` makes the statement false (Coverage), so a gatekeeper binary cannot even reach an unproved state; it is outside every statement.
9. OSS Verify proves crash-freedom of fuzz targets that "use no network". It is not a service for daemons or custom properties.
10. Platform mismatch, separate from the syscalls: the model is Linux v6.18.46 with Sv48 address spaces. The VisionFive 2 is (unverified) Sv39 on a Debian 13 kernel that is not 6.18.46.

## Falsifier 1 evidence

### What the model is

> "This repository contains a model, in the Rocq Prover, of a Linux process that runs a 64-bit RISC-V (RV64GC) binary... The model is not complete. It models part of Linux... Where the model cannot be sound, it says `unmodeled`."
> https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/README.md

> "The model describes **one system**: stable Linux v6.18.46 (commit `1f99e9ab748f…`) on riscv64."
> (same README)

**How `unmodeled` is treated** (README, "The two conditions"):

> "1. **Coverage.** No execution of the source gets to a configuration that the model does not interpret (`unmodeled`)... A system call that the model does not interpret does not give this alternative: it makes condition 1 false."

So an unmodelled syscall is not an oracle that returns arbitrary values. It is a wall, and any proof must show the program never reaches it. Where the kernel's answer is genuinely open, the model uses nondeterminism instead (`choose P`, `alt`): "A system call specification gives a set of permitted outcomes. It does not choose one outcome" (README, "System calls"). Errno branches are kept live on purpose: "the failure has to stay LIVE, or a target could delete the source's error handling" ([Socket.v:24-34](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/theories/LinuxProcessSem/Sys/Net/Socket.v#L24)).

### Syscall by syscall (all in `theories/LinuxProcessSem/Sys/Net/Socket.v` unless noted)

| call | status | evidence |
|---|---|---|
| `socket(AF_UNIX, SOCK_STREAM)` | **unmodeled** | L284 `else if negb (Z.eqb fam AF_INET) then unmodeled`. The comment at L159-161 says SOCK_STREAM and SOCK_SEQPACKET are "connection sockets this model holds as nothing yet" |
| `socket(AF_UNIX, SOCK_DGRAM)` | creates `FdDgram`, which every stateful call declines | [SCOPE.md:319](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/SCOPE.md?plain=1#L319): "every call that would read or change socket state... is `unmodeled`" |
| `bind`/`listen`/`accept4` | modelled **for AF_INET TCP only**: `INADDR_ANY, the only bind address the model accepts` (L139). `accept4` declines `SOCK_NONBLOCK` and a non-NULL address | L795ff |
| `getsockopt` | AF_INET with a known level → `unmodeled` ("the option table"). Socketpair end → only `SO_SNDBUF`/`SO_RCVBUF` (L1430). Dgram → `unmodeled` (L1812) | L1792-1813 |
| `SO_PEERCRED` | **absent**. A repo-wide grep finds no `PEERCRED`, `SCM_CREDENTIALS` or `ucred` | grep of the pinned tree |
| `sendmsg` + `SCM_RIGHTS` | **unmodeled** on every AF_UNIX kind. On TCP, a non-zero `msg_control` is declined | L2513: "A STREAM-PAIR END DECLINES, WHOLE, and this decline is LOAD-BEARING elsewhere. An AF_UNIX [sendmsg] carries SCM_RIGHTS..." |
| `recvmsg` | only EBADF/ENOTSOCK. "The socket arm here is [unmodeled] when it lands" | L3236 and the comment above it |
| `poll`/`ppoll`/`epoll` | exist (`Sys/Fs/Select.v`, `Sys/Fs/EventPoll.v`, syscall arms 20-22 and 73), but only over the fd kinds above | [SyscallTable.v:44-67](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/theories/LinuxProcessSem/Sys/SyscallTable.v#L44) |
| `close`, `getuid` | modelled | SyscallTable.v:101-102 |
| signals | the [SCOPE.md:309](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/SCOPE.md?plain=1#L309) row: "The model does not run signal handlers. It has two signals and both are fatal... nothing outside the process can send one." `rt_sigaction` declines installing a handler address | |
| fd passed in at start (systemd `LISTEN_FDS`, inetd) | **not expressible** | SCOPE.md:313: "a proof says nothing about a program that is handed a connected socket on fd 0, as an inetd-style server is." |

The SCM_RIGHTS exclusion is structural, not an oversight ([SCOPE.md:441-446](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/SCOPE.md?plain=1#L441)):

> "An in-process stream pair... is used by this one thread only... `sendmsg` on one is declined, so no descriptor travels by `SCM_RIGHTS`."

[State.v ~L5905-5925](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/theories/LinuxProcessSem/State.v#L5905) adds that pipe-liveness reasoning reads the fd table, and that the table is "a WRONG" count once an fd is in flight. Adding SCM_RIGHTS means revising those invariants, not adding an arm.

### How peers and the kernel's values are modelled

Peers are input, universally quantified ([State.v:2591-2617](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/theories/LinuxProcessSem/State.v#L2591)):

> "Record PeerAddr := mkPeer { pa_addr : bv 32; pa_port : bv 16 }" ... "NOTHING IN THIS MODEL READS IT YET."
> "CoInductive NetQueue : Type := | NConn (peer : PeerAddr) (script : NetStream) (rest : NetQueue)." ... "INFINITE — there is no nil... THAT IS AN UNDER-APPROXIMATION: it excludes the environment in which [accept] blocks forever."

So **could a proof say "passed only if the kernel-returned uid is in the list"?** In principle, yes. *If* `SO_PEERCRED` were modelled, the natural design (matching `PeerAddr`) would be a per-connection uid field in `NetQueue`, quantified over all values. A proof over all inputs would then cover every uid the kernel could return, which is exactly the property we want. The kernel is not an arbitrary oracle per call, though. It is a quantified, *consistent* environment: the same connection gives the same peer. Two obstacles remain even then:

- **The observable is wrong.** The events are "input and output on a channel, one byte at a time, normal exit..., termination by a modeled fatal signal" (README, "What the model observes"). Passing an fd is not an event, so none of the published statements (`strong_universal_T`, `halts_allowed_T`, `must_exit_with`, the classifier statements) can express "fd X went to socket Y". We would have to write a new statement over `Run.star` ourselves, and that statement becomes trusted base that nobody else has reviewed.
- **The receiver is outside the model.** "For `fork`-like calls, the model describes the parent only" (SCOPE.md:308), and only one process exists. A claim about *which service* receives can only be phrased as "sendmsg was issued on the fd that was bound/connected to path P". The model has no path-addressed AF_UNIX at all.

### Platform fit

- "The system call specifications use Sv48. Thus you cannot prove the Sv39 and Sv57 statements for an image that can run" (SCOPE.md:310). The SiFive U74 implements Sv39 *(from memory, unverified: a search did not return the U74 manual's satp table; check https://www.starfivetech.com/uploads/u74_core_complex_manual_21G1.pdf)*. If that holds, no proof in today's model applies to the board as it boots.
- The kernel is pinned to v6.18.46. "Other stable kernels (v4.17 and later) are a goal. They are not supported now" ([SCOPE.md:20](https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/SCOPE.md?plain=1#L20)). Debian 13 ships 6.12 *(unverified, from memory)*, and the VisionFive 2 often runs a StarFive vendor kernel.
- Only threads: "The model has one thread" (SCOPE.md:308).

### Toolchain, sizes, cost, offer

- Languages with statements today: "`c` ... `c++` ... `rust` ... statements today"; `python` paused; Go builds but has no statements; Swift is "not possible" ([docs/project-yaml.md, language](https://github.com/theorem-labs/oss-verify/blob/dc8238d020fcadf623c65ef3b3418a76b01eff68/docs/project-yaml.md#language)). Dart is not listed.
- Build: "clang 20.1.8 for `riscv64-linux-musl`, and lld"; "Each binary is a static riscv64 Linux program for RV64G (`-march=rv64g -mabi=lp64d`), linked with lld against musl"; dynamic binaries are refused ([docs/build-contract.md:22-29](https://github.com/theorem-labs/oss-verify/blob/dc8238d020fcadf623c65ef3b3418a76b01eff68/docs/build-contract.md)).
- Size: "we are most likely to respond quickly to a project in C whose plain binaries... have less than 1 MB of code" ([README](https://github.com/theorem-labs/oss-verify/blob/dc8238d020fcadf623c65ef3b3418a76b01eff68/README.md)). Anthropic's page: median binary of 18,000 instructions, a typical proof of about 600,000 lines, "hundreds of millions of tokens", a 5M-token benchmark budget, and no dollar figure (https://www.anthropic.com/research/formal-verification). theorem.dev: "No cost" for qualifying projects; "These proofs can take multiple days to produce" (https://theorem.dev/oss-verify/).
- What is proved: "For each fuzz target, you will receive a proof (in Rocq) that the target does not crash regardless of input" (https://theorem.dev/oss-verify/).
- The skill forbids our shape: "### 3.6 Depend only on the input — ... a process has one thread and the input is its only data. Start no thread, read no clock, take each random number from the input, **use no network** or environment, and open no file but the input and temporary files" ([SKILL.md:49-51](https://github.com/theorem-labs/oss-verify/blob/dc8238d020fcadf623c65ef3b3418a76b01eff68/.claude/skills/add-project/SKILL.md)).
- Eligibility: "an established open-source project", enrolled by "a core maintainer", needing "at least one fuzz target"; "We will prioritize widely used critical infrastructure." A new 300-line intersyntactic daemon does not fit the offer. Only its *pure decision core*, packaged as a fuzz target, could in principle.

## Constraints

**Constants on riscv64** (riscv has no arch `uapi/asm/socket.h`, since fetching `arch/riscv/include/uapi/asm/socket.h` at v6.12 returns 404, so asm-generic applies):
- `SOL_SOCKET 1`, `SO_PASSCRED 16`, `SO_PEERCRED 17`, `SO_PASSPIDFD 76`, `SO_PEERPIDFD 77` (https://github.com/torvalds/linux/blob/v6.12/include/uapi/asm-generic/socket.h). The `#ifndef SO_PASSCRED /* powerpc only differs in these */` guard means 16/17 differ only on powerpc.
- `struct ucred { __u32 pid; __u32 uid; __u32 gid; }`: 12 bytes, uid at offset 4 (https://github.com/torvalds/linux/blob/v6.12/include/linux/socket.h). `SCM_CREDENTIALS 0x02`, `SCM_PIDFD 0x04`, and `SCM_RIGHTS 0x01` (that one from memory; my grep missed its line).
- **So the Dart magic `RawSocketOption(1, 17, Uint8List(12))`, uid at byte 4, is correct on riscv64, x86-64 and arm64.** It would be wrong only on powerpc. The bytes are little-endian on all three.
- `SCM_MAX_FD 253` (https://github.com/torvalds/linux/blob/v6.12/include/net/scm.h).

**SO_PEERCRED semantics.** "The returned credentials are those that were in effect at the time of the call to connect(2), listen(2), or socketpair(2)" (https://man7.org/linux/man-pages/man7/unix.7.html). Consequences:
- The client's uid is captured at `connect()`, not at accept. A process that connects as uid A and then `setuid`s, or passes its connected fd to a uid B process (SCM_RIGHTS again), keeps showing A. Credentials name *who connected*, not *who is talking now*.
- The pid in `ucred` can be reused after the client exits. `SO_PEERPIDFD` (6.5+) fixes this for pid. The uid has no reuse problem.
- The uid is translated into the *reader's* user namespace. An unmapped uid shows as the overflow uid 65534 *(from memory: `cred_to_ucred` → `from_kuid_munged`, net/core/sock.c; unverified this session)*. An allowlist must never contain 65534/nobody.
- `SO_PASSCRED`/`SCM_CREDENTIALS` gives per-message credentials, and the sender may specify its own (privileged senders can claim other pids/uids): "The returned credentials are those specified by the sender using SCM_CREDENTIALS, or a default..." (unix(7)). For authentication, `SO_PEERCRED` is the stronger one.

**Socket file permissions.** "On Linux, connecting to a stream socket object requires write permission on that socket... Portable programs should not rely on this feature for security" (unix(7)). This gives a second, independent wall (directory mode plus socket mode) that costs nothing. Abstract-namespace sockets have no permissions at all *(from memory)*.

**SCM_RIGHTS mechanics** (unix(7)):
- "what is being passed is a reference to an open file description... equivalent to duplicating (dup(2))".
- "If the buffer used to receive the ancillary data... is too small (or is absent), then the ancillary data is truncated (or discarded) and the excess file descriptors are automatically closed"; then "the MSG_CTRUNC flag will be set".
- Excess fds beyond the receiver's `RLIMIT_NOFILE` are silently closed.
- `ETOOMANYREFS` when the number of "in-flight" fds exceeds the sender's `RLIMIT_NOFILE` (since 4.5). The gatekeeper is the sender, so a slow receiver can make it fail.
- Ancillary data rides with the first byte of a stream message. The receiver must `recvmsg` with a control buffer, at least one data byte must be sent, and messages with differing ancillary data are not merged.
- The receiver should use `MSG_CMSG_CLOEXEC`.

**systemd.** `Accept=yes` already does the accept, then spawns one service instance per connection with the connected socket as fd 3 via `LISTEN_FDS` *(from memory; see https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html)*. With `SocketUser=`/`SocketMode=`, systemd sets the socket file's permissions. The LPS model cannot describe a binary handed an fd at launch (SCOPE.md:313).

**"Single-threaded static, fixed memory" forbids:** threads; signal handlers (so SIGTERM is a fatal default, and SIGPIPE needs `MSG_NOSIGNAL` or a SIG_IGN launch); `SOCK_NONBLOCK`/`O_NONBLOCK` (declined in `socket_spec` and `accept4_spec`); `MAP_SHARED`; dynamic linking (glibc NSS for name lookup, so uids must be numbers, not names); reading a config file outside the modelled file system ("no full directory tree, no symbolic links... no permissions", SCOPE.md row "Files").

**Trusted base and reward hacking.** "developers must trust only the formal models of RISC-V and Linux processes, the exact theorem statements, and the Rocq checker", plus footnote 5 "the hardware, OS, OCaml compiler", and footnote 6: "this leaves open the door to reward hacking... Claude could make any theorem provable by rewriting the program into one that trivially never crashes" (https://www.anthropic.com/research/formal-verification). For our property, the trivial winner is a gatekeeper that never passes anything: "never passes a disallowed uid" holds vacuously. A liveness or "passes every allowed one" half is needed.

## Failure modes others hit

- **unix_gc races (SCM_RIGHTS in-flight cycles).** CVE-2021-0920, a use-after-free in the SCM_RIGHTS garbage collector, "exploited in the wild until it was fixed in November 2021" (https://projectzero.google/2022/08/the-quantum-state-of-linux-kernel.html, https://access.redhat.com/security/cve/CVE-2021-0920). CVE-2024-26923, "garbage collector racing against connect()", introduced in 2.6.23 (https://nvd.nist.gov/vuln/detail/CVE-2024-26923). Oracle's write-up covers io_uring interplay (https://blogs.oracle.com/linux/post/unix-garbage-collection-and-iouring). *The "2019 unix_gc CVE" the brief mentions: I did not find or confirm a specific 2019 one.* A proof against the LPS model would not see any of these: "Bugs in Linux do not count" (README).
- **Credential confusion after fd passing.** A connected socket handed on keeps the original connector's `SO_PEERCRED`. This is benign here (the gatekeeper passes the *accepted* fd, whose peer creds still name the client), but a downstream service that rechecks creds gets the client's uid, not the gatekeeper's. That is good, and worth noting as a cross-check.
- **Lost fds on MSG_CTRUNC.** A receiver with a small `msg_control` silently loses the connection (closed for it). A receiver that reads with plain `read()` discards it entirely.
- **Gatekeeper blocking.** A blocking `sendmsg` to a receiver that never reads stalls a single-threaded gatekeeper for *all* services (head-of-line). Without `O_NONBLOCK` (unmodelled), there is no way out other than poll with timeouts.
- **Dart side.** The six copies return `null` when unsure (CRUCIBLE.md). The analogous gatekeeper bug is treating `getsockopt` failure or `optlen != 12` as uid 0. Uid 0 must never be a default.

## Open questions

1. Is a proof of only the **pure decision core** (bytes → allow/deny, as a crash-on-violation fuzz target) worth the Spark? It fits OSS-Verify's statement shape today, but proves nothing about the syscall shell.
2. Would Theorem accept an upstream LPS extension (AF_UNIX stream, SO_PEERCRED as a per-connection quantified uid, SCM_RIGHTS with in-flight accounting)? Their SCOPE says priority 3 is "no `unmodeled` cases". Scale unknown, and it touches State.v invariants.
3. Is the U74 Sv39-only? (Unverified; if yes, a second blocker independent of syscalls.)
4. Which kernel does the VisionFive 2 actually run (`uname -r` on the box)?
5. Is "the gatekeeper is no smaller than what it guards" (falsifier 2) already true? The Dart check is ~42 lines; a C gatekeeper with musl is plausibly 300-600 lines of C plus libc.

---

## Solution space (withheld from Spark)

**Peer-credential gatekeepers and brokers**
- djb UCSPI: `unixserver` (ucspi-unix, Bruce Guenter) accepts, reads peer creds, sets `UNIXREMOTEEUID`/`UNIXREMOTEEGID` env vars, then execs the handler with the socket on fds 0/1 *(from memory; http://untroubled.org/ucspi-unix/)*. tcpserver is its TCP twin, with `-x` cdb rules files.
- skarnet s6: `s6-ipcserver` plus `s6-ipcserver-access` (rules keyed on uid/gid, per-uid directories) and `s6-sudod`/`s6-sudoc`, which pass the client's fds 0/1/2 over the socket to a server-side process *(from memory; https://skarnet.org/software/s6/s6-ipcserver-access.html, https://skarnet.org/software/s6/s6-sudod.html)*. This is the closest existing shape: rules by uid, then exec.
- OpenBSD privsep with `imsg` (`imsg_compose` carries an fd): a privileged parent passes opened fds to unprivileged children *(from memory; https://man.openbsd.org/imsg_init.3)*.
- systemd `Accept=yes` plus `SocketUser`/`SocketMode`, giving one service instance per connection; the service sees creds on fd 3. Lennart's "socket activation" posts.
- inetd/xinetd: accept plus fork/exec.
- Capsicum (FreeBSD): capability-mode processes with fd rights; Casper services broker fds. CloudABI (defunct) built on it. (https://www.cl.cam.ac.uk/research/security/capsicum/)
- seL4 badged endpoints: a minted badge on an endpoint capability identifies the sender unforgeably. In Microkit, a channel between two protection domains is declared in the system XML, and `notified(ch)`/`protected(ch, msginfo)` receive the channel id as the identity *(from memory; https://docs.sel4.systems/projects/microkit/manual/latest/)*. CapDL statically describes the whole capability distribution, verified to be initialised correctly (https://docs.sel4.systems/projects/capdl/).

**Verified-systems prior art for a tiny daemon**
- CompCert: a verified C compiler that removes the compiler from the trust base for a C source proof (https://compcert.org).
- VST: separation-logic proofs of C, linked through CompCert. Theorem has a fork (https://github.com/theorem-labs/VST).
- Verus: SMT-backed Rust verification; it has been used for systems code (https://github.com/verus-lang/verus).
- Low*/HACL*: F* subset extracted to C, verified crypto shipped in Firefox and Linux (https://github.com/hacl-star/hacl-star).
- `rocq-of-rust` (Theorem fork: https://github.com/theorem-labs/rocq-of-rust) is a source-level path for Rust.
- LPS's own `Classifier.v` statements (`sound_fo`, `classifier_total3_T_fo`) fit "a binary that reads bytes and exits 0/1/2". The decision core could be such a classifier: input is (uid, allowlist, service id), exit code is the verdict. That is provable today in the published statement vocabulary, with the syscall shell left trusted (https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/README.md).
- **Existing formally verified fd-passing or IPC broker:** I found none on Linux. The nearest are seL4 IPC itself (verified kernel) and CapDL-initialised systems. *(Not searched exhaustively.)*
