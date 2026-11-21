# Brief: draft a first PR for Theorem's Linux process model (SO_PEERCRED, E0)

Nick asked for this 2026-11-21 18:50 AEDT. **Draft only: do not push, fork, open a PR or comment
anywhere on GitHub. Nick sends it himself.**

## Background

The verified-wall forge (https://github.com/intersyntactic/verified-wall,
`docs/crucible/verified-wall/`) wants Theorem's `rocq-linux-process-semantics` (LPS,
https://github.com/theorem-labs/rocq-linux-process-semantics, MIT) to grow toward Unix listening
sockets with a peer credential per connection. Read `RESEARCH.md` and especially `RESEARCH-2.md`
in that bundle first: they cite the exact files and lines.

The first step is the smallest PR that tests the upstream path and touches no invariant: **E0**,
`getsockopt(SOL_SOCKET, SO_PEERCRED)` on the socket kinds LPS already has.
- Today `getsockopt_spec` declines every option except `SO_SNDBUF`/`SO_RCVBUF` on a socketpair
  end (`theories/LinuxProcessSem/Sys/Net/Socket.v:1427-1453`) and declines on AF_INET.
- Linux v6.18 behaviour (`net/core/sock.c` ~L1691-1701 and ~L1890-1903, `net/unix/af_unix.c`
  `init_peercred` ~L1851): a socketpair end reports the creating process's pid, **euid**, egid;
  a TCP socket reports pid 0, uid and gid 0xFFFFFFFF; a buffer shorter than 12 bytes is
  truncated without error. Verify every line number against v6.18.46 yourself.

## The job

1. Clone LPS fresh to `~/git/orgs/theorem-labs/rocq-linux-process-semantics` (create the
   parent dir). Work on a local branch `so-peercred`.
2. Install the toolchain the repo's README / opam file names (Homebrew `opam`, then a switch
   with the Rocq version CI uses, 9.2 or 9.3, and the repo's opam dependencies). Build main
   once first to prove the toolchain (README: ~45 CPU-minutes, up to 3 GB per job; the machine
   has 10 cores and 64 GB, so keep `-j` moderate).
3. Implement E0 the way the repo does every syscall arm: the relational spec arm, the executable
   checker and its proof, a witness, and worked cases (`Checks/Sys/...`, `Checks/Witness/...`,
   `Checks/Cases/...`). Follow the house style of neighbouring arms exactly: live errno
   branches, `unmodeled` for anything not covered (e.g. AF_INET options other than SO_PEERCRED
   stay declined), and comments citing the Linux source the way existing arms do. The process's
   own credentials come from `in_uid`/`in_gid`; check whether the model has an euid distinct
   from uid, and if not, say so in the PR text rather than inventing one.
4. Build clean, and run `.github/check_assumptions.py` as CI does.
5. Write `PR.md` next to the clone (not inside it) with a title, a body in the repo's PR style
   (look at merged PRs #34 and #112 for tone), the diff stat, and anything you were unsure of.
6. Report to Nick: the branch, `PR.md`'s path, build time, and open questions. Do not push.
