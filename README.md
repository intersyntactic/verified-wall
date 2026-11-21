# verified-wall

A gatekeeper for intersyntactic's Unix sockets: one small static RISC-V program that accepts
each connection, asks the kernel which OS user is calling, and passes the connection on only to
the service allowed to receive that caller. The aim is a machine-checked proof, in Rocq against
a model of RISC-V Linux, that it never passes on a caller who isn't allowed.

Nothing is built yet. The design is being worked out in `docs/crucible/verified-wall/`.
