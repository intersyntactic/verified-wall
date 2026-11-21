# Brief: one peer-identity package that fails closed

Nick asked for this 2026-11-21 18:32 AEDT. Context: the verified-wall forge
(https://github.com/intersyntactic/verified-wall, `docs/crucible/verified-wall/`) found that six
intersyntactic packages each carry their own copy of `_peerUid(Socket)`:

| package | file | copy |
|---|---|---|
| chat_gpt_driver | lib/service.dart:181 | A |
| claude_code_driver | lib/service.dart:193 | A |
| logbook | lib/service.dart:896 | A |
| program_driver | lib/src/program_driver.dart:219 | A |
| file_driver | lib/src/file_driver.dart:242 | B (drifted) |
| web_request_driver | lib/service.dart:164 | C (drifted) |

Each reads `SO_PEERCRED` (Linux) or `LOCAL_PEERCRED` (macOS) and returns the uid, or `null`
where it can't tell. RESEARCH.md in that bundle checked the constants: `SOL_SOCKET 1`,
`SO_PEERCRED 17`, a 12-byte `ucred` with uid at offset 4 are right on riscv64, x86-64 and arm64
(wrong only on powerpc).

## The job

1. Diff the three variants and say what drifted and whether any drift is a bug.
2. Read how every caller handles `null`. Report any path where an unknown peer is let in.
3. Make one package, in a repo of its own under the intersyntactic org (modules live in their
   own repos), that answers "may this socket's peer in?" so that **an unknown peer is denied by
   construction**: the API should not hand callers a nullable uid to mishandle. Name it by its
   role, per AGENTS.md. Also refuse the overflow uid 65534 (what the kernel reports for a uid
   unmapped in the reader's user namespace), so it can never appear in an allow list.
4. Move each of the six packages onto it. **Each repo's change goes through that repo's own
   session where one is running** (see ListAgents; e.g. file-driver, web-request-driver,
   claude-code-driver sessions); brief it rather than editing its repo yourself. For a repo
   with no running session, tell Nick and ask.
5. Follow `~/.intersyntactic/AGENTS.md`: Dart, is_lint_rules loaded at the top level of
   analysis_options.yaml, `dart analyze --fatal-infos` and `dart test` clean, tests with a fake
   for the socket read. PRs, not direct pushes; give Nick every PR URL.
