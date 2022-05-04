FIPE is FP-style approach to C++ implementation of CSP/Unix pipes paradigm. It allows to combine multiple data-crunching "processes" into a linear pipeline where each process communicates only with neighboring ones via "stdin" and "stdout" typed data streams.

NOTES: THERE IS NO IMPLEMENTATION CODE YET. The design closely follows [my older Haskell module for CSP](https://gist.github.com/Bulat-Ziganshin/a593b11bc6febd2ec201a6bef095bc25).


# Design principles

Pipes:
- Each process receives messages from its "stdin" and sends messages to its "stdout"
- Process type is templated by the types of input and output messages: Process<X,Y>
- Two processes can be combined into a single process by pipe creation operation:
  - `auto p = p1 | p2;` combines Process<X,Y> and Process<Y,Z> into Process<X,Z>
- Since result of piping operation is also a process, a pipe containg arbitrary number of processes can be created, even in a loop
- You can also use back channel to return messages in opposite direction by replacing Y message type on both sides of a pipe with withBackChannel<Y,Yback>

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
- This signals throws exception on the next send/receive operation performed by the process being signalled
  - ... except for sending data to back channel?


# Design

Pipeline:
- Single process can be made from anything callable with Pipe<X,Y> - function, functor, lambda...
- Currently, we use `std::function< void (Pipe<X,Y>) >` to store these values
- Piping operation (either `|` or pipe()) should return Pipeline<X,Y,Z>, combining pair of Pipeline/Process objects templated with <X,Y> and <Y,Z>
- Both SingleProcess<X,Z> and Pipeline<X,Y,Z> should inherit from Process<X,Z>
- `p1 | p2` specifies unlimited channels (forward and backward) between the two processes
- pipe() allows to specify kind and capacity of the channels:
  - pipe(p1, p2, N [,M]) specifies N entries for forward channel and M entries for backward channel

Pipe and RunningProcess supports the same set of operations:
- p >> x, x = recv(p): receive value from stdin
- p << x, send(p,x): send value to stdout
- x = recvBack(p): receive value from stdout's backchannel
- sendBack(p,x): send value to stdin's backchannel
- EOF(p) returns true when there will be no more data in stdin (a preceding process was finished), false when value became available, waits otherwise
- heartbeat(p) is empty operation, it just throws the termination exception when required
- we may need extra "protected" operations to avoid handling of termination exceptions

Termination propagation:
- We suppose that the sole purpose of an asynchronous pipeline is to process data for its owner. This means that when the owner was finished, processes in the pipeline should be finished ASAP
- Similarly, the sole purpose of a process in a pipeline is to support values for the next process in the pipeline
- We may explore protection of pipeline parts with protect(p), allowing them to continue reading data (from stdin and stdout.back) until they last and write data to stdin.back till the channel capacity. All other operations (and all operations on a Pipe in unprotected processes) should throw an exception
- Pipeline termination may be started by:
  - Pipe::cancel(), which just throws CancelException
  - RunningProcess::cancel()
  - unhandled exception in SingleProcess
  - normal exit of SingleProcess
  - RunningProcess destructor (i.e. termination of its owner, either normal exit or by exception), which just calls RunningProcess::cancel()
- When a process normally exits, we should throw:
  - CancelException in all preceding processes in the pipeline (on the next call to any Pipe function)
  - ? EofException in the immediately following process when it tries to read past last value from stdin
- When a RunningProcess is terminated by cancel() or destructor, all processes in the pipeline should be finished ASAP (by throwing exception on their next Pipe operation)
- When a process in a pipeline terminates by unhandled exception, all processes in this pipeline should be finished ASAP and the original exception should be rethrown either by run() itself or next operation on RunningProcess


# Example: producer | transformer | consumer

First process generates messages, second one transforms them, and third one prints the messages it received.

```C++
void producer( Pipe<void,int> p) {
  for (i=1; i<=10; i++)
    p << i;
}

void transformer( Pipe<int,double> p) {
  while (p.ready())
    p << (p.recv() * 3.14);
}

void consumer( Pipe<double,void> p) {
  while (p.ready())
    cout << p.recv();
}

main() {
  run( producer | transformer | consumer);
}
```

# Example: async pipeline

```C++
main() {
  auto p = [] (Pipeline<int,int> p) {
    while (p.ready())
      p << (p.recv() * 2);
  };
  RunningProcess<int,int> rp = runAsync( p | p | p );
  for (int i=1; i<=5; i++) {
    rp << i;
    cout << recv(rp);  // prints 8 16 24 32 40
  }
}
```

# Example: using backward channel to send buffers back after use

```C++
void producer( Pipe<void, withBackChannel<char*,char*> > p) {
  for (i=1; i<=5; i++) {
    p.sendOwnBack(new char[10]);  // push values into our own stdout backward channel
  }
  for (i=1; i<=100; i++) {
    auto buf = p.recvBack();
    sprintf(buf, "%d\n", i);
    p.send(buf);
  }
  // to do: release the buffers
}

void consumer( Pipe< withBackChannel<char*,char*>, void> p) {
  while (p.ready())
    auto buf = p.recv();
    cout << buf;
    p.sendBack(buf);
  }
}

main() {
  run( producer | consumer);
}
```
