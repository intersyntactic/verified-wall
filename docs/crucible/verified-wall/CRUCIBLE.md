# CRUCIBLE.md: the verified wall

Ore selected 2026-11-21 18:12 AEDT; Nick gave consent and chose this repo as its home at 18:17.

## The seed

Nick: "We could formally verify each part of Intersyntactic (see
https://www.anthropic.com/research/formal-verification)", after a conversation about seL4 and
the VisionFive 2 (StarFive JH7110, RISC-V) box that will run intersyntactic's components.

## What the linked research says (read, not assumed)

Anthropic and Theorem, 2026-10-22: Claude wrote Rocq proofs that 239 entry points across 123
OSS-Fuzz libraries never crash on any input up to 64 MB. The proofs are about the compiled
RISC-V machine code, checked against Theorem's `rocq-linux-process-semantics`, for "a
single-threaded static binary with a fixed memory budget on a RISC-V CPU using Linux OS".
Median binary 18,000 instructions; a typical proof about 600,000 lines of Rocq; proofs can
take hundreds of millions of tokens. Stated limits: verified is not bug-free; the trusted base
is the Rocq checker, the RISC-V and Linux models, and the hardware; Claude can reward-hack by
rewriting the program into one that trivially satisfies the spec.

## The pick

Every wall in intersyntactic on Linux rests on one question: which OS user is on the other
end of this Unix socket? Six packages answer it with their own `_peerUid(Socket)`, reading
`SO_PEERCRED` through `RawSocketOption(1, 17, Uint8List(12))` and taking the uid at byte 4:

| package | file | copy |
|---|---|---|
| chat_gpt_driver | lib/service.dart:181 | A |
| claude_code_driver | lib/service.dart:193 | A |
| logbook | lib/service.dart:896 | A |
| program_driver | lib/src/program_driver.dart:219 | A |
| file_driver | lib/src/file_driver.dart:242 | B (drifted) |
| web_request_driver | lib/service.dart:164 | C (drifted) |

(Copy letters from a hash of the 42 lines from the comment down, 2026-11-21. Four match.)

Each returns `null` where it can't tell. One wrong constant, or one caller that lets `null`
through, opens every wall at once and nothing says so.

The candidate: move that question out of Dart into one small static RISC-V program, a
gatekeeper that accepts each connection, asks the kernel who is calling, and passes the
connection on (by `SCM_RIGHTS` file-descriptor passing) only to the service allowed to receive
that caller. Then prove in Rocq, against Theorem's RISC-V Linux model, that it never passes a
connection whose kernel-reported uid is not allowed.

## Scores

| # | candidate | aliveness | impact | product |
|---|---|---|---|---|
| 1 | the verified wall | 3: the proof pipeline targets this box's exact platform; logbook's README says "nothing checks it as it runs, so a person reads it instead" | 2: every driver's "answers only the airlock's user" rests on these lines, six copies, three variants | 6 |
| 2 | Rocq/Lean model of the airlock protocol + differential tests against Dart | 2 | 2 | 4 |
| 3 | start-from-nothing systemd units + containment tests on the box | 1 | 3 | 3 |
| 5 | binary-level proof of the logbook | 3 | 2, hinges on file-system crash semantics | deferred |
| 4 | port to seL4 | 3 | 0 now: no H-extension on the U74, Dart runtime needs POSIX | slag |

## Why it thrills

`SCM_RIGHTS` is Linux's own capability: an open descriptor passed over a socket carries its
authority and cannot be forged, as an seL4 capability is. A proven gatekeeper that hands out
connections that way is the first piece of seL4 discipline on Linux with a proof behind it, on
a cheap RISC-V board, with a pipeline published a month ago. It replaces "a person reads it
instead" with "Rocq checked it" for the lines every other wall depends on.

## Falsifiers

1. Theorem's Linux process model doesn't cover the calls the gatekeeper needs (`socket`,
   `bind`, `listen`, `accept`, `getsockopt(SO_PEERCRED)`, `sendmsg` with `SCM_RIGHTS`). Then
   there is nothing to prove against, and this drops to candidate 2. Heat checks this first.
2. The gatekeeper can't be a single-threaded static binary small enough for the pipeline, or
   ends up no smaller than what it guards.

## Scout memory

`enspyrco/intersyntactic/docs/crucible/harness-capability/TEMPER.md` (unanimous RECAST,
2026-09) prescribed where enforcement belongs: "the kernel, a dedicated user, a VM, or
not-our-process — not another 80 lines of `classify`." This candidate is not-our-process
enforcement, so it follows that shape rather than conflicting with it.
`context-as-a-fold` is unrelated.
