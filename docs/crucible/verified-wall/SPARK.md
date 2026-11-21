# SPARK.md: the out-breath for the verified wall (re-scoped, option C)

Thrown 2026-11-21 18:32 AEDT, blind and in parallel, one spark per family. Every seat got the
same prompt: the seed, `CRUCIBLE.md`, and `RESEARCH.md` up to (not including) its "Solution
space" section, so no seat saw the prior art. No angles were assigned. Maxwell is a fresh Claude
agent with no context from the session that ran the forge. Sparks are verbatim.

## Maxwell (Claude)

**Spark: the export port, where a capability leaving the process counts as output**

Stop thinking of `SCM_RIGHTS` as a descriptor *in flight*. That framing is what runs into State.v's "no fd in flight" invariant. From inside one process, a descriptor sent to the outside world doesn't fly anywhere. It leaves. The model already has a word for things that leave the process: output.

So the extension is three small changes, each shaped like something Theorem already wrote.

1. **Uid as quantified input.** `NConn` gets a `peer_uid : bv 32` field beside `PeerAddr`, quantified over every value the same way. The kernel's answer to `SO_PEERCRED` stops being an oracle and becomes part of the input tape. It stays consistent per connection, and the proof covers all of them.

2. **A new fd kind, `FdExport`.** It is an AF_UNIX stream that connects to a path outside the process and only sends. `recvmsg` on it declines any control data, so nothing ever comes back in. No loop ever closes inside the process, and the pipe-liveness invariants never see a descriptor that comes back. They stay true because the new fd kind can't break them.

3. **Handoff as a byte event.** `sendmsg(export, [1 byte], SCM_RIGHTS{conn})` writes a record on the existing observable channel: `HANDOFF <conn-index> <export-index>`. The model already observes bytes on channels. The capability's departure becomes an event in the output alphabet. So the safety property needs no new trusted statement over `Run.star`, and fits the published shape: every `HANDOFF k j` in the trace has `peer_uid(k) ∈ allow(j)`.

For liveness, every allowed connection eventually appears in a `HANDOFF` or in a named refusal. That rules out the vacuous winner, the gatekeeper that never passes anything.

Then the gatekeeper changes shape to match. It becomes a **Mealy machine**: a machine whose output depends only on its input and its current state. It reads uids off the input tape and writes handoffs onto the output tape, with no config file (the allowlist is compiled in as numbers) and no handlers. The allowlist never contains 65534, and uid 0 is never a default. The whole binary is maybe 150 lines of C against musl. It is *smaller* than the six Dart copies put together, which answers falsifier 2 before anyone raises it.

The upstream PR is small enough for an outside contributor: one record field, one fd constructor, one event encoding, plus a SCOPE.md row: "descriptors may leave; none may arrive." Theorem gets fd passing in the only direction their invariants can absorb today. It is also the direction seL4 people would recognise: a send-only endpoint that grants a capability and keeps no record of it. The receivers stay outside the model. That is the honest boundary, and the `HANDOFF` trace is exactly what each Dart service could check against on arrival.

The thrill is that the proof never has to know what a file descriptor *is* once it's gone. It only has to know the process said its name on the way out.

Oh, of course: the kernel is input, the capability is output, and a gatekeeper is a transducer between two tapes the model already knows how to read.

## Noether (DeepSeek V4.1 Flash, effort high)

Take Theorem's `rocq-linux-process-semantics` and bolt on a ghost capability layer, not an AF_UNIX stack. Call it the Badge. Every accepted connection in `NetQueue` gains one field: `pa_uid : bv 32`, universally quantified over every uid the kernel can report. The trace gains one event: `Pass(conn, token, dest)`, where `token` is a linear ghost token standing for an `SCM_RIGHTS` fd in flight. It never enters the fd table, so the model's existing "no fd in flight" invariants stay true.

The gatekeeper is a single static RISC-V C binary, one thread, musl, blocking with `poll`. It binds one Unix socket per service. On `accept`, it calls `getsockopt(SO_PEERCRED)`, reads the uid at byte 4, looks up a compiled-in allowlist, and if allowed `sendmsg`s one byte plus `SCM_RIGHTS` to the service's socket; otherwise it closes. The proof has two halves. Safety: no `Pass` event ever carries a token whose `pa_uid` is not in the allowlist. Liveness: for every allowed uid, a client that connects and sends the magic byte causes exactly one `Pass` to the right service. That kills the vacuous "never passes anything" reward hack.

The binary-level proof is trace refinement: the RISC-V machine code's syscall trace is decoded into the ghost layer, and every unmodeled AF_UNIX path is unreachable because the gatekeeper uses only the seven calls the extension interprets. The allowlist is compiled as immediates, so the proof is parametric in the table and rechecked per deployment. Upstream gets a reusable pattern: a capability event, not a kernel reimplementation. On the VisionFive 2, this is seL4 discipline on Linux: a connection is a capability, `SO_PEERCRED` is its badge, `SCM_RIGHTS` is the grant, and Rocq says no unbadged grant ever leaves.

Oh, of course: the smallest extension isn't AF_UNIX—it's a ghost token in the trace, and the gatekeeper only has to be proven to stamp it with the kernel's uid.
## Carnot (GPT 6.1, effort medium)

Build **the authority loom**: a gatekeeper whose machine-code proof follows the life of a connection as an unbroken thread—from the kernel’s credential snapshot to the service that receives its descriptor.

The wild move is to make **authority transfer a first-class observable in the Linux model**. Beside bytes and exits, Rocq can see an open-file-description reference crossing a socket boundary. Descriptor numbers are temporary handles; the underlying object has an identity that survives duplication, queuing, receipt, and closing. Every handoff has a history.

Extend the single-process model with an external world that owns references, connects sockets, and consumes messages. It can delay, disconnect, exhaust buffers, and pass references onward. Credentials attach to the connection at the kernel’s snapshot point. In-flight descriptors become explicit roots in the object graph. The existing process model gains a surrounding sea of capability ownership without needing to execute every peer’s instructions.

Then give the gatekeeper a striking physical shape: **one tiny binary per service, each guarding one doorway**. Each instance has one numeric admission policy and one established delivery channel to its service. It accepts a connection, reads its credentials, and transfers that exact connection through that exact channel. The six Dart services receive admitted sockets through a common adapter.

The theorem becomes a story with an ending: every delivered connection traces back to an allowed connector; every successful transfer preserves the connection’s identity; and, when the kernel and receiver keep making progress, every admitted connection reaches the service. Ship a concrete successful execution alongside the universal proof so admission is visibly part of the artifact.

On the VisionFive, show the loom running: connections illuminate threads between OS users and services, with each transfer linked to the binary hash and checked theorem. The diagram is a rendering of the events the proof actually names.

The upstream contribution is a Linux semantics that can finally speak about authority moving. Intersyntactic becomes its first living exhibit.

Oh, of course: prove the journey of the capability, and the gatekeeper becomes the smallest machine that can complete that journey.

## Tesla (Grok, effort medium)

The model's file-descriptor table becomes a lie the moment a descriptor is in flight. That lie is the thing we prove.

`State` grows a second table, the wire: a finite map from an output socket to the capabilities sitting between this process and the next. An `SCM_RIGHTS` `sendmsg` is a move. The accepted descriptor leaves the process table and appears on the wire, and the sum of the two counts stays equal to the number of live open-file descriptions. Every lemma already in `State.v` is the special case where the wire is empty. One lemma, `wire_empty`, instantiates them. The patch goes upstream as a parameter. Old proofs keep their world, the one in which nothing ever travels.

Peers grow a credential frozen at connect, beside the `PeerAddr` the queue already stores and the comment admits nothing reads. `SO_PEERCRED` is the first reader of that field. The queue stays coinductive and universally quantified, so a proof ranges over every uid the kernel might stamp, and one connection can never acquire a second identity. The man-page rule — credentials name who called `connect`, not who holds the descriptor now — becomes the type.

The gatekeeper is one static, single-threaded RV64 binary, and it never looks at twelve bytes. The `ucred` layout is a lemma inside the model. The program receives the uid in a register, a projection `peer_uid` of the descriptor it just accepted. Its text holds a perfect hash of the allowlist as immediates: no file, no NSS, no name. The initial descriptor table, which the model has refused to parameterize, becomes an argument: one listening Unix stream, plus one already-connected output socket per service — the systemd shape, seen at last. The hash picks the output. On a hit, one `sendmsg` moves the accepted descriptor onto that service's wire, carrying the single data byte the kernel demands, and the process table loses it in the same step. On a miss, `close`, and the wire is untouched. There is no default. Offset 4 does not exist in the program. Uid 0 is simply a uid the hash misses.

Two statements over `Run.star`, new and therefore named as trusted base. Safety: a capability appears on a service wire only when its frozen uid sits in that service's set. Liveness: every accepted connection whose uid is in the set produces exactly one such appearance, so a gatekeeper that never moves anything does not satisfy it. The published crash-freedom theorems stay, discharged by the empty wire.

Oh, of course: the in-flight descriptor was the capability, and the proof says it changes tables only when the kernel's frozen uid lets it.

---

## Fusion

**Axis check.** The sparks did not collapse onto one axis, though two came close. Maxwell and
Noether, blind to each other, found the same move: an outgoing `SCM_RIGHTS` is an *event on the
output*, not a descriptor in flight, so the fd table and its invariants never change. Tesla
changes the state instead (a second table, "the wire", with a conservation law, and every
existing lemma as the special case where the wire is empty), and alone attacks the *launch*:
the gatekeeper opens nothing and is handed its sockets. Carnot goes widest (a full ownership
graph of open file descriptions, and a proof that runs to the receiver) and alone splits the
gatekeeper into **one instance per service**. Three seats raised the liveness half unprompted.

**Decomposed to sockets (function, input, output):**

| spark | what it adds to the model | what it changes in the gatekeeper | what it proves |
|---|---|---|---|
| Maxwell | uid field on each queued connection; send-only `FdExport` kind; `HANDOFF` as output bytes | a Mealy machine, allowlist compiled in | each `HANDOFF` uid is allowed; each allowed one is handed off or refused by name |
| Noether | uid field; a ghost `Pass` token that never enters the fd table | one binary, `poll`, one socket per service | safety + liveness over `Pass` events |
| Carnot | an outside world owning references; descriptions with identity | one binary per service, one channel each | delivered ⇒ allowed; identity preserved; admitted ⇒ arrives |
| Tesla | a wire table, conserved count; the initial fd table as a parameter | handed its sockets at launch; perfect hash; no `ucred` parsing in the program | safety + liveness over the wire; old proofs via `wire_empty` |

**The shared anchor:** all four put the peer's uid on the connection in the input, frozen at
connect and quantified over every value. That is the one extension nobody disputes.

**The third object: the doorkeeper.** One tiny gatekeeper *per service* (Carnot), started by
systemd with its whole world already open (Tesla): one listening Unix stream socket and one
connected socket to its service. It calls nothing that creates a socket, binds a path or
connects anywhere. It loops: `accept`, read the frozen uid, and on a hit `sendmsg` the accepted
descriptor with one byte to its service, which appears as an output event (Maxwell, Noether);
on a miss, `close`. Each instance has one allow set, so there is no routing table and no hash.

The fusion is more than the best spark with decoration, because two properties come only from
the seam:
- **Per-service instances dissolve a failure mode from RESEARCH.md.** A single gatekeeper whose
  `sendmsg` blocks on one slow service stalls every service. One doorkeeper per service means a
  stuck receiver stalls only its own door.
- **Handed-in sockets shrink the extension.** With sockets handed in at launch and handoff
  modelled as an output event, the model needs three things: an initial descriptor table given
  as a parameter, a uid on each queued AF_UNIX connection with `SO_PEERCRED` returning it, and
  `sendmsg` with `SCM_RIGHTS` on a handed-in socket as an observable event. It needs no
  path-addressed AF_UNIX, no `connect`, no second process, and no change to the "no fd in
  flight" invariants. Tesla's wire table stays available as the general upstream form, if
  Theorem wants descriptors that can also arrive.

**Oh, of course:** systemd opens the doors, each door gets its own keeper, and the proof covers
only the moment a keeper lets someone through.
