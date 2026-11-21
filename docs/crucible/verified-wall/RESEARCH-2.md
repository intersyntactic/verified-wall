# RESEARCH-2.md: what extending LPS would take

Read 2026-11-21. LPS is pinned at `d3f3a701` and the Procs branch at `procstate-migration` `7baf9c25` (shallow clone). Citations of the form `path:line` are relative to https://github.com/theorem-labs/rocq-linux-process-semantics/blob/d3f3a701d99afeee638963228562f4b5ff6e96ba/theories/LinuxProcessSem/ unless they name another file. **R** marks something I read; *I* marks my inference.

## Summary: extension directions, smallest first

1. **SO_PEERCRED on the fd kinds that already exist.** About one getsockopt arm. It returns the process's own creds on a socketpair end, and pid 0 with uid/gid 0xFFFFFFFF on TCP (R: `net/core/sock.c` v6.18 L1691-1701). It touches no invariant, and it gives a gatekeeper nothing.
2. **An AF_UNIX listener built like TCP, plus a uid.** A new `in_*` queue carries a credential for each connection, accept4 draws from it, and SO_PEERCRED returns that credential. It leaves every "no fd in flight" invariant alone as long as sendmsg still declines control data. Cost is in the range of the `FdDgram` kind: 128 mentions across 33 files (R, grep).
3. **That listener, with the gatekeeper serving or refusing the connection itself.** It needs no new event: `conn_channel n = n+3` (`Sys/Net/Socket.v:87`) already ties each written byte to connection n and therefore to its uid. *I:* this is the smallest real gatekeeper.
4. **A byte-relaying gatekeeper.** Item 2, plus an outbound `connect`, which LPS declines today because "its only source here is [m_net]" (`Sys/Net/Socket.v` connect comment, ~L971). That means a second environment queue and a new statement about data flow.
5. **SCM_RIGHTS, send-only, with the receiver outside the model.** Items 2 and 4, plus control data on sendmsg, a new `event` constructor (`KillEv` reaches 16 files), and a rewrite of the argument at `State.v:5897-5959` that is load-bearing for pipe EOF.
6. **A second process.** Theorem is already building it: Procs.v on `procstate-migration`, with open PR #62 at +119,964 lines (https://github.com/theorem-labs/rocq-linux-process-semantics/pull/62). The clone gate still declines a process that holds a socket (`Sys/Kernel/Fork.v:615-618`), and there is no execve. No outsider could land this.
7. **Upstream caveat.** Outside PRs do land (Trail of Bits, #34). But Theorem lands model changes that ripple into proofs itself, because adding `FdStream` broke 48 of 89 published proof trees (`State.v:2841-2846`).
8. **Platform blockers.** The U74 is Sv39-only (R, manual §3.7). Debian 13 ships 6.12 (R). LPS is Sv48 and 6.18.46, and `TASK_SIZE` is read at 468 sites. These block every item above.

---

## 1. How the model is built (R unless marked)

**The four artefacts per syscall.** Each call has:
- a relational spec in `Sys/<area>/*.v` (trusted);
- an executable checker with its proofs in `Checks/Sys/...`;
- a witness in `Checks/Witness/...`;
- worked cases in `Checks/Cases/...`.

`Checks/Sys/Net/Socket.v:1-12` says "The checkers here are executable, and their proofs relate them to the relational specs." So an extension writes all four.

**Sizes.**

| file | lines |
|---|---|
| `State.v` | 10,888 |
| `Run.v` | 6,827 |
| `Sys/Net/Socket.v` | 3,241 |
| `Checks/Sys/Net/Socket.v` | 3,683 |
| `Checks/Cases/Net/Socket.v` | 2,774 |
| `Checks/Witness/Net/Socket.v` | 1,085 |
| StreamPair and StreamReset cases plus `Checks/StreamReset.v` | 1,900 |

Everything about networking comes to 12,956 lines. The tree has 351 `.v` files and 284k lines.

**The fd table.** `FdState := { fdst_table : list (word * FdEntry); fdst_pipes }` (`State.v:3591`). `FdEntry` has 14 constructors (`State.v:3257-3271`): `FdChannel`, four `FdFile*` kinds, `FdDir`, `FdSocket (sk : SockState)`, `FdPipe`, `FdDevNull`, `FdUrandom`, `FdStream id side st`, `FdEventfd`, `FdDgram family`, `FdAnon`.

`SockState` (`State.v:2699-2704`) is `SkFresh | SkBound | SkListening | SkConnected peer chan rx | SkReset peer chan`. The per-connection state lives in the constructor, and the socket lives in the fd entry, so a close disposes of both.

**The environment is input, quantified once** (`State.v:7340ff`, `ProcInput`):
- `in_net : NetQueue`;
- `in_uid`, `in_gid` and `in_groups`, which are the process's own;
- the rest of the launch values.

The queue is defined as follows:
- `CoInductive NetQueue := NConn (peer : PeerAddr) (script : NetStream) (rest : NetQueue)` (`State.v:2616`). It is infinite by design, so that accept stays receptive. The comment calls that "AN UNDER-APPROXIMATION: it excludes the environment in which [accept] blocks forever" (`State.v:2603-2614`).
- `NetStream := NFin | NRst | NTimedOut | NCons b rest` (`State.v:2400`).
- `PeerAddr` is `{pa_addr : bv 32; pa_port : bv 16}`, and "NOTHING IN THIS MODEL READS IT YET" (`State.v:2591`).

Classifier inputs carry `fin_net` across unchanged, and a weight function "may read NONE of [fin_uid] ... [fin_net]" (`EquivalenceDefinition/FinInput.v:416-439`).

**Worked example: accept4 on a TCP listener** (`Sys/Net/Socket.v:795-904`):
1. `fd_lookup` returns None, so the answer is `-EBADF`. An unknown flag bit gives `-EINVAL`. `SOCK_NONBLOCK` gives `unmodeled`.
2. On `FdSocket`, a non-NULL `addrp` gives `unmodeled`, because "SyscallEffect cannot express the [move_addr_to_user] write".
3. On `SkListening` the result is `alt (errno_from [ENOMEM; EMFILE; ENFILE]) (success)`. This is the **live errno branch**: "Without it a source that handles an accept failure and a target with that handling deleted would have identical trace sets" (L740-748, and `socket_errnos` at L24-36).
4. Success takes `lowest_fd`, then calls `with_accepted` (`State.v:5605`). That one step inserts `FdSocket (SkConnected (nq_peer net) (conn_channel ix) (nq_script net))`, advances `m_net` to `nq_rest`, and adds 1 to `m_conn_ix`.
5. When the counter is exhausted the success side is `choose (fun _ => False)`, which leaves only the errno.
6. A socket that is not listening gets `alt EINVAL (allocation errnos)`. `FdStream` and `FdDgram` give `unmodeled`.

The read that follows is `Sys/Fs/ReadWrite.v:694-770`:
- On `SkConnected _ ch (NCons ..)` the spec writes `choose` over a non-empty prefix of the script, 1 to count bytes, which is a real short read.
- It writes the remainder back into the entry and does `emit (map (ReadEv ch) bs)`, "on the CONNECTION's channel — not stdin's".
- `NFin` answers 0 and stays there. `NRst` and `NTimedOut` answer the pending error once and move the socket to `SkReset`.

**What is observed.** The type is `event := ReadEv ch b | WriteEv ch b | ExitEv code fs | KillEv sig fs` (`State.v:5617`). A run is built like this:
- `behavior := {bh_traces : list event -> Prop; bh_errors; bh_all_errors; bh_unmodeled}` (`Run.v:81`);
- `star : config -> list event -> config -> Prop` (`Run.v:817`) accumulates the events;
- an `Unmodeled` spec result is an `SError` step (`Run.v:810-815`).

The published statements:
- `strong_universal_T` (`EquivalenceDefinition/Equivalence.v:783`): for all entries and every `valid_input inp`, the source is `modeled` and the two runs are `equiv`.
- `halts_allowed_T S img` (`HaltsAllowed.v:109`): every valid input gives `must_exit_in S`.
- `sound_fo C spec` (`Classifier.v:1961`): exit-accept implies `spec inp`, and `spec inp` implies the run is not exit-reject.

*I:* none of them is a safety property over `ReadEv`/`WriteEv` that refers to `in_net`. A gatekeeper property would be a new statement, and a new statement means a new trusted base. Its shape is simple: `forall inp tr, bh_traces (run ..) tr -> WriteEv (conn_channel n) _ ∈ tr -> allowed (uid_of (nth n (in_net inp)))`.

## 2. What "no fd in flight" and "one process" protect (R, then I)

These are the invariants that depend on there being no reference outside this process's fd table, or on a single process:

| # | invariant | where | what keeps it true today |
|---|---|---|---|
| I1 | "NO REFERENCE TO A PIPE EXISTS OUTSIDE THIS TABLE". Pipe EOF is computed from the table by `pipe_end_live`; with an end in flight the table is "a WRONG" count, and the error fails open ("answering a modeled 0 where Linux blocks") | `State.v:5860-5998` | `dup_modeled_entry` (`Sys/Fs/File.v:81`), fcntl's closing `unmodeled` for F_DUPFD (called "the weak one"), the clone gate (`Fork.v:615`), and sendmsg declining `FdStream`/`FdDgram` (`Sys/Net/Socket.v:2513-2530`) |
| I2 | A stream end is unique; "peer closed" ⇔ the peer is absent from the table | `stream_end`/`stream_peer`, `State.v:6164-6194` ("A missed end would make [stream_peer] answer 'closed' for a live peer") | the same routes |
| I3 | `se_reset` has one holder, so ECONNRESET is reported once | `State.v:2826-2833`; `Checks/Cases/Net/StreamReset.v:331-345` lists "[sendmsg] (no SCM_RIGHTS)" | the same routes |
| I4 | close is kind-blind, and the FileSim `tfo_close` lemmas say a close only removes its entry. "78 of the 89 published proof trees vendor them" | `State.v:2835-2846` | a design choice |
| I5 | `anon_agree`: the replicas of an unnamed file agree | `State.v:6147` | dup and clone decline |
| I6 | One process: one global accept queue (the Procs.v comment calls it "AN EXISTING APPROXIMATION"); bind accepts only INADDR_ANY port 0 because "port-table state a one-process model does not have" (`State.v:2663`); connect is declined; locks (`Sys/Fs/Locks.v:218`); blocking arms decline because "No SECOND THREAD" (`State.v` StreamEnd comment) | as cited | structural |
| I7 | `conn_channel` is injective and disjoint from 0-2 | `Sys/Net/Socket.v:87-111` | the `conn_ix_exhausted` guard |
| I8 | A stream read never stops between skbs, because "SO_PASSCRED ... or SCM_RIGHTS ... neither of which any modeled call can set up" | `Sys/Fs/ReadWrite.v:405-417` | the setsockopt/sendmsg declines |

**(a) AF_UNIX stream with a quantified peer uid per connection, plus SO_PEERCRED.** *I:*
- I1-I5 stay as they are if sendmsg keeps declining non-empty control data, as the AF_INET arm already does (`if negb (w_is0 ctllen) then unmodeled`).
- The new kind has to be added to the deliberately exhaustive matches in `pipe_end_live`, `stream_end` and `anon_agree`. That work is mechanical, and those matches exist so the build forces it.
- I6 bites at `bind`, because a filesystem path needs a socket inode and `Fs.Fs` is `list (path * bytes)` with no inode kinds. Two ways around it: bind only an **abstract** address, which never touches `Fs`, with `EADDRINUSE` as an environment errno exactly as TCP's `bind_errnos` handles it; or add a launch value for an **inherited listening fd**, which is what systemd `Accept=no` does with fd 3. That second option is itself a gap SCOPE.md:313 names.
- I8 holds as long as SO_PASSCRED stays declined.
- The `NetStream` terminals do not carry over one-for-one. An AF_UNIX peer never times out, and its close gives ECONNRESET only when it left unread data. So a validity clause has to exclude `NTimedOut` on unix connections.

**(b) SCM_RIGHTS sendmsg, sending only, with the receiver outside the model.** *I:*
- I1, I2, I3 and I5 survive only if the payload is restricted to `FdSocket (SkConnected ..)`. Sending a pipe end, a stream end or an anon file breaks them.
- Even then, the sent connection's `rx` script is read by both the sender and an outside reader, so the sender's entry has to stop answering. That needs a new `SockState` such as `SkGiven` on which only close answers (`SkReset` reaches 70 mentions in 10 files).
- I4 is unaffected.
- A new `SendFdEv via conn` event reaches every statement file.
- The prose at `State.v:5917-5929` prescribes exactly this choice: a change answering sendmsg "must decline its control data or give this model a notion of a file held by nothing in the fd table".
- It also needs a connected AF_UNIX socket to the receiver, which is outbound connect again (I6).

**(c) A second process.** *I:* everything in I6 changes. I1 needs real per-`struct file` counts, which is what `Procs.v`'s `Ofd.ofd_refs` adds ("It is what makes a pipe's EOF expressible", `procstate-migration:theories/LinuxProcessSem/Procs.v` ~L141-160). The clone gate declines `FdSocket` (`Fork.v:615-618`). There is no execve in `SyscallTable.v`.

## 3. Smaller extensions, judged by what they leave untouched (I, grounded in the counts)

- **E0: SO_PEERCRED on today's kinds.** R from Linux v6.18:
  - `cred_to_ucred` sets `uid = gid = -1`, then, if a cred exists, `from_kuid_munged(current_ns, cred->euid)`. So **SO_PEERCRED returns the euid, not the uid** (this corrects RESEARCH.md).
  - A TCP socket has no `sk_peer_cred`, so it gives uid/gid 0xFFFFFFFF and pid 0.
  - `len < 12` truncates silently (`sock.c` L1890-1903).
  - On a socketpair end, both ends carry the creator's creds (`af_unix.c` `init_peercred`, L1851-1852).

  In the model this is one arm in `getsockopt_spec`'s `SoPair` case (`Sys/Net/Socket.v:1427-1453` declines every option except SNDBUF/RCVBUF) and one in the `SoInet` case. It touches no invariant. It is useful as the template for the 12-byte write, and as a first PR to test the upstream path.
- **E1: AF_UNIX listener with a uid** (direction 2 in the summary).
  - A new `ProcInput` field carries a coinductive queue whose entries are `(cred : pid*euid*egid, script : NetStream)`. Adding launch fields is routine upstream: PR #126 added "ProcInput field 27".
  - Either `FdSocket` grows a family, or a new `FdUnix` kind follows the `FdDgram` precedent (128 mentions, 33 files).
  - accept4 is a copy of the TCP arm. Reads and writes reuse the `SkConnected` arms. SO_PEERCRED reads the credential out of the entry.
  - I1-I5 and I8 stay as they are. I6 is avoided by an abstract-address bind or an inherited fd.
  - For scale: in-process AF_UNIX pairs, which also added eventfd, dgram probes and SIGPIPE, were +14,386/-1,383 lines in 75 files (PR #57, https://github.com/theorem-labs/rocq-linux-process-semantics/pull/57). SEQPACKET pairs alone were +3,492/-537 in 72 files (PR #112, https://github.com/theorem-labs/rocq-linux-process-semantics/pull/112).
  - *I:* E1 is about #112-sized, plus a new input type.
- **E1': a gatekeeper that serves or refuses the connection itself.** The model needs nothing beyond E1.
  - The statement: every `WriteEv (n+3) _` in a trace implies that the euid of the nth queued connection is allowed.
  - To rule out the trivial answer that never serves anyone, add a converse: for an allowed connection whose script is a well-formed request, a reply is emitted.
  - *I:* this is the smallest extension that still serves a real gatekeeper. It changes what the gatekeeper does, from handing a connection on to answering it, which CRUCIBLE.md explicitly allows.
- **E2: byte relay.** E1 plus outbound connect, which needs an environment queue of outbound peers. The connect comment's objection is that taking outbound peers from `m_net` "is a wrong answer"; a *separate* queue answers it. The relay also has head-of-line blocking, and without `O_NONBLOCK` (declined in accept4/socket) it needs ppoll.
- **E3: SCM_RIGHTS send-only.** E1, E2's outbound socket (or an inherited one), control data, `SkGiven`, a new event, and an edit to the I1 argument. It is the first item on this list that changes trusted prose an invariant rests on.

## 4. Upstream path (R)

- **Licence.** MIT, "Copyright (c) 2026 Theorem Labs, PBC" (`LICENSE`).
- **Contributing.** There is no CONTRIBUTING file: GitHub's community profile shows `contributing: null`, and the README says nothing about contributions. The sibling oss-verify-proofs does have one: outside PRs "may touch only a harness's `proofs/` and `counterexamples/` folders ... `docs/`", under MIT with a DCO sign-off (`oss-verify-proofs/CONTRIBUTING.md`).
- **How outside work has been handled:**
  - PR #34, Kevin Valerio ("senior security engineer @trailofbits"), adding `clock_getres` and `set_robust_list`: held ("We hold it while we complete the verification work ... It is not rejected"), then merged 2026-11-02 into main and on into the revision PR #38. https://github.com/theorem-labs/rocq-linux-process-semantics/pull/34
  - Robert Joseph (Caltech) had PRs #2, #3 and #4 merged.
  - 0xalpharush (bio @tempoxyz) had #1, #23, #29 and #31 merged.
  - Issue #6: "Thank you for the offer to help. We make this change ourselves, because it also changes the proofs that are built on the model." https://github.com/theorem-labs/rocq-linux-process-semantics/issues/6
  - Issues #8 and #9 from Vaibhav Tulsyan (Google) were closed after fixes.
- **Screening.** Outside PRs and issues are screened for prompt injection, then reviewed by Claude against the Linux source (`.github/workflows/review.yml:1-60`).
- **Roadmap.** The README's priorities: "1. **Sound (required).**" "2. **Precise (goal).** The model includes no behavior that Linux v6.18.46 does not give." "3. **Covered (goal).** The model has no `unmodeled` cases." SCOPE.md:20: "Other stable kernels (v4.17 and later) are a goal. They are not supported now." Sv39 is not on the roadmap; it exists only as a declaration whose statements are refused (`State.v:455-466`).
- **Maintainers.** The opam maintainer is "Jason Gross <jason@theorem.dev>". Commit counts: sjawhar 244 and sjawhar-agent[bot] 75; 0xalpharush 4; JasonGross 3; Robertboy18 3; kevin-valerio 1; legion-implementer[bot] 1.
- **Activity.**
  - The repo was created 2026-10-19. The newest commit on main is 2026-11-20 (#155), after bursts of 11-26 commits a day on Nov 3-8.
  - Public main lags an internal tree: "Public main holds an older revision of the Rocq model (OSSV-811)" (PR #153, +104,531 lines, open). Specs cite "main LPS (theorem-labs/linux-process-semantics) #1536", and that repo returns 404 to me.
  - The multi-process work goes through the integration branch `procstate-migration` (issue #125).
- **Is CI re-checking proofs?** Yes:
  - `ci.yml` runs `dune build` on Rocq 9.2, 9.3 and dev, then `.github/check_assumptions.py`.
  - PR #1 added "kernel-check final proof closures with rocqchk".
  - oss-verify-proofs runs `oft check`, which refuses any assumption outside an allowed list (its README, L168).

*I:* the evidence on the slag test is mixed. A small arm (E0, or E1 split into pieces) is the kind of PR they have merged. A new `FdEntry` kind is the kind they do themselves (issue #6, and the 48/89 tree-breakage note). So the realistic upstream path is a SCOPE-gap issue with a design note, plus a draft PR for the spec arms, with Theorem landing it.

## 5. Platform (R)

- **The U74 is Sv39-only.** Confirmed. *SiFive U74 Core Complex Manual* 21G3.02.00, §3.7: "MMU supports the Sv39 address mode"; Table 9, `satp.MODE`: "0x0 - Bare mode" and "0x8 - Page-based ... (Sv39)", with no other modes (https://www.scs.stanford.edu/~zyedidia/docs/sifive/sifive-u74.pdf). StarFive's JH7100 page agrees: "Sv39 Virtual Memory support with 38 Physical Address bits" (https://doc-en.rvspace.org/JH7100/Datasheet/JH7100_DS/sifive_u74_dual_core.html).
- **Debian 13 kernel.** The 6.12 series (https://www.debian.org/releases/trixie/release-notes/whats-new.en.html). The trixie source package is at 6.12.74-2 (https://packages.debian.org/hu/source/trixie/kernel/linux), and backports have 6.16 (https://packages-pkgmirror-csail.debian.org/tr/trixie-backports/linux-image-6.16.3+deb13-riscv64). None of these is 6.18.46.
- **What covering Sv39 would take.** `VaSize := Sv39 | Sv48 | Sv57` already exists (`State.v:474`), as do `valid_input_sv39` (`Run.v:5139`) and Sv39 band proofs (`Checks.Cases.MmapBase`). But "the spec arms ... still compare against [TASK_SIZE], the Sv48 constant, because that number is read at hundreds of sites" (`State.v:455-466`): R, 468 uses of `TASK_SIZE` across 53 files and 757 of `access_okb`. SCOPE.md:310 says the same: "you cannot prove the Sv39 and Sv57 statements for an image that can run".
  - *I:* the work is to put the declared size into the model and thread it through every range test, then re-prove the checkers. That is a 53-file change Theorem would do itself, and it is a prerequisite for any proof that applies to this board.
  - On the kernel side, many arms already carry version windows ("unchanged in behaviour across v6.1, v6.6, v6.12 and mainline", connect_spec). But no statement covers 6.12, and SCOPE.md:20 makes that a goal only.

## 6. Cost (R)

- **Building LPS.** "A full build uses about 45 CPU-minutes. Each compile job can use up to 3 GB of memory" (README:106-107). On CI, the lanes on main took 20-41 minutes of wall time (run 44667474688: 9.2 took 20 min, 9.3 took 26, dev took 40), with a 300-minute timeout.
  - Rocq 9.2 once ran out of memory at the default `-j` on `ProcsWait4Sigchld.v`, which peaked at 1.42 GB per file (PR #141, https://github.com/theorem-labs/rocq-linux-process-semantics/pull/141).
- **Checking one OSS proof.** "On 16 cores, one proof check needs up to 20 GiB of memory" (oss-verify-proofs README:159).
- **Existing proofs over sockets.**
  - Inside LPS, the networking model plus its checks and cases come to 12,956 lines.
  - In the public oss-verify-proofs (14 GB on disk), the curl entries are `fuzz_bufq` and `fuzz_fnmatch`, which use no sockets. The cURL protocol targets that drove PR #57 appear only in internal trackers ("census of the 19 wave-3 cURL targets", `State.v:2856-2860`).
  - *I:* I found no published proof that exercises accept or socketpair. A grep over the 248k `.v` files timed out, so this is unconfirmed.

Corrections to RESEARCH.md:
- SO_PEERCRED reports the **euid/egid**, not the uid.
- On a non-AF_UNIX socket it reports -1/-1 and pid 0.
- A short `optlen` truncates the result without an error, so a gatekeeper must check that the returned length is 12.
