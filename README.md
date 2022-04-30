FIPE is FP-style approach to C++ implementation of CSP/Unix pipes paradigm. Multiple "processes" can be combined into a single pipe, where each process communicates only with preceding and following processes via "stdin" and "stdout" channels.


# Design principles

Pipes:
- Each process receives messages from its "stdin" and sends messages to its "stdout"
- Process type is templated by the types of input and output messages: Process<X,Y>
- Two processes can be combined into a single process by pipe creation operation:
  - `auto p = p1 | p2;` combines Process<X,Y> and Process<Y,Z> into Process<X,Z>
- Since result of piping operation is also a process, a pipe containg arbitrary number of processes can be created, even in a loop
- You can also use back channel to return messages in opposite direction by replacing Y message type on both sides of a pipe with withBackChannel<Y,Yback>:
  - f.e. a pipe can combine Process<X, withBackChannel<Y,Yback> > with Process< withBackChannel<Y,Yback>, Z>

Channels:
- There are many kinds of channels:
  - rendez-vous (immediate)
  - fixed capacity
  - unlimited capacity (usually OK for back channels and when channel volume limited f.e. by number of buffers allocated)
- Main channel and back channel between two processes may have different kinds

Execution:
- "Process" can be represented by OS process, OS thread, fiber, coroutine. One just needs to provide an API for process creation and channel send/recv operations
- `run(p)` runs Process<void,void> and waits till it finished
- `RunningProcess rp = runAsync(p)` starts process p in background and allows to communicate with its stdin and stdout

Error propagation:
- Unhandled exception or just mere finishing of process sends signals requesting cancellation to preceding and following processes in the pipe
- This signals raises exception on the next send/receive operation performed by the process being signalled
  - ... except for sending data to back channel?
