On the ARM and RISCV64 platforms, there is a further proof that the binary code which executes on the hardware is a correct translation of the C code. This means that the compiler does not have to be trusted, and extends the functional correctness property to the binary.

---

##### Message passing
In seL4, IPC is via ‘‘Endpoint Objects’’. An Endpoint can be considered a mailbox through which the sender and receiver exchange the message through a handshake. Anyone holding a Send capability can send a message through an Endpoint, and anyone holding a Receive cap can receive a message. This means that there can be any number of sender and receivers for each Endpoint. In particular, a specific message is only delivered to one receiver (the first in the queue), no matter how many threads are trying to receive from the Endpoint.