# Chapter 48

## Distributed Systems

- A set of machines to construct a system
- The major challenge to focus on is **failure**
  - If one machine fails, the system can still operate
  - There are many aspects of a system to fail
    1.  machines
    2.  disks
    3.  networks
    4.  software
- Other challenges are there like performance, security, etc.

## Communication

- The fundamental of the networking communication is that the packets are often lost
- There are two main approachs (unreliable vs. reliable)
  1.  we don't deal with it at all (e.g. UDP/IP)
      1.  UDP still checks if the packet is corrupted or not
  2.  we deal with
      1. losing messages from sender to receiver
      2. losing acks from receiver to sender

### Reliable Communication

- The sender first sends the message to the receiver
  - If the receiver receives it, the receiver acks
  - If the sender doesn't receive the ack for a timeout, the sender will resend the message
- The receive acks to the sender
  - If the send receives it, the sender will stop sending
  - If the sender sends the message again, the receiver will detect the duplicate
  - and it still asks but does **NOT** pass the data to the application

#### Detect Duplicate

- Using an unique ID for each message sent
- However, tracking all unique IDs are expensive
- A simple approach is to use a sequence counter
- The counter will check if the ID of the received packet matches the counter it has to detect duplicates

## Abstractions

- OS abstraction
  - Give the applications the abstraction that the entire system is just a regular OS
  - Downside is too hard to provide regular OS memory access speed with memory distributed over many machines
  - Another big downside is that failures are really complicated to handle
- Programming Language abstraction
  - Remote Procedure Call (more details below)

## Remote Procedure Call (RPC)

- The main idea is to execute code on a remote machine
- From the client POV, a procedure call is made and the results are returned some time later
- From the server POV, it exports some routines for clients to call
- The rest of the heavylifting is handled by the RPC system
  1.  Stub generator (protocol compiler)
  2.  Run-time library

### Stub Generator (Protocol Compiler)

- The job is to pack the functions arguments and results into messages
- So the client and the server code does not repeat this automatable process to avoid mistakes
- The input to a stub generator is a set of calls a server wishes to export
- The output will be a client stub and a server stub
- The steps of each function in the stubs are below

#### Client Stub

1. Create a message buffer
2. Pack information to the buffer
   1. procedure identifier
   2. arguments
   3. **serialize** all the information
3. Send the message to the RPC server
4. Wait for the reply
5. Unpack returned results
   1. **deserialize** all the information from the server
6. Return to the caller

#### Server Stub

1. Unpack the message
   1. deserialize the information from the client
2. Call into the actual function
3. Package the results
   1. **serialize** all the results
4. Send the reply back to the client

#### Issues

1. Complex arguments (pointers)
2. Server organization
   1. one simple loop for one request at a time
   2. a thread pool with multiple workers to perform requests simultaneously

### Run-Time Library

- Challenge #1: locating the remote service (naming)
  - An example is to use hostname/IP and port number to identify
- Challenge #2: choosing communication layer (reliable or unreliable)
  - reliable solution might be inefficient
  - unreliable solution requires more responsibility to handle failures
- Other Challenges
  - the procedure call takes a while to complete
  - the procedure call takes large arguments
    - sender-side fragmentation (break into small packets)
    - receiver-side reassembly (merge packets into a complete one)
  - server and client have different byte ordering (big/little endian)
  - asynchronous support

## Keywords

1. acks: acknowledgement from the receiver to sender to stop sender from sending the same packet again
