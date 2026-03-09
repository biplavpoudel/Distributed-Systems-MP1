# MP1 Specification Document

## Important Dates

- **Released:** Week 1 of C3 Part 1  
- **Due:** End of Week 5 C3 Part 1  

---

# 1. Overview: What Is This MP About?

This MP (Machine Programming assignment) is about implementing a **membership protocol** similar to one discussed in class.

Since it is infeasible to run a thousand cluster nodes (peers) over a real network, we provide an implementation of an **emulated network layer (EmulNet)**. Your membership protocol implementation will sit above EmulNet in a **peer-to-peer (P2P) layer**, but below an **Application layer**.

Conceptually this is a **three-layer protocol stack**:

```
Application Layer
P2P Layer
EmulNet Layer
```

Your protocol must satisfy:

- **Completeness at all times:**  
  Every non-faulty process must detect every node **join**, **failure**, and **leave**.

- **Accuracy of failure detection:**  
  When there are **no message losses** and **message delays are small**, failure detection must be accurate.

- **Robustness under message loss:**  
  When messages are lost, completeness must still hold and accuracy should remain high.

- The protocol must function correctly even under **simultaneous multiple failures**.

You may implement any of the membership protocols discussed in class:

- All-to-all heartbeating
- Gossip-style heartbeating
- SWIM-style membership

**Recommendation:** Implement **gossip** or **SWIM**, as they provide deeper learning.

The template code is written in **C++**, one of the most commonly used languages in systems programming. You will need **C++11 or later (gcc ≥ 4.7)**.

Template code download:

```
https://spark-public.s3.amazonaws.com/cloudcomputing/assignments/mp1/mp1.zip
```

Autograder scripts (unit tests) will also be provided to test your program.

### Academic Integrity

All work must be **individual**.

You may discuss:

- The MP specification
- General concepts

You may **not**:

- Discuss solutions
- Share code
- Copy code

Your code may be checked for structural similarity.

---

# 2. The Three Layers

The framework allows running **multiple peers inside a single process** using a **single-threaded simulation engine**.

## 2.1 Emulated Network: EmulNet

EmulNet provides the following functions that your membership protocol should use:

```cpp
void *ENinit(Address *myaddr, short port);
int ENsend(Address *myaddr, struct address *addr, string data);
int ENrecv(Address *myaddr, int (* enqueue)(void *, char *, int), struct timeval *t, int times, void *queue);
int ENcleanup();
```

### Function Descriptions

**ENinit**

- Called once by each node
- Initializes its address (`myaddr`)

**ENsend**

- Sends a message to another peer

**ENrecv**

- Receives messages waiting for a node
- Enqueues them using the function pointer `enqueue()`

**ENcleanup**

- Cleans up the EmulNet implementation at the end of the simulation

Notes:

- Parameters `t` and `times` in `ENrecv` are currently unused.
- `ENsend` and `ENrecv` can be assumed **reliable when the network has no message losses**.

⚠️ Do **not modify** the files:

```
EmulNet.cpp
EmulNet.h
```

These files will be replaced during testing. Access the network layer **only through the provided functions**.

---

## 2.2 Application Layer

This layer drives the simulation.

Files:

```
Application.cpp
Application.h
```

Look at the `main()` function.

The simulation runs in **synchronous periods**, controlled by the variable:

```
globaltime
```

During each period:

- Some peers may **start**
- Some peers may **crash-stop**

For every peer that is alive, the function:

```
nodeLoop()
```

is called.

`nodeLoop()` is implemented in the **P2P layer (`MP1Node.cpp`)**.

Its responsibilities include:

- Receiving messages sent during the previous period
- Checking whether the application has new requests

⚠️ Do **not modify** the files:

```
Application.cpp
Application.h
```

---

## 2.3 P2P Layer

Files:

```
MP1Node.cpp
MP1Node.h
```

This layer implements the **membership protocol**. All your implementation code should go here.

This layer could eventually support distributed system operations such as:

- File insertion
- File lookup
- File removal

---

# 3. What Does the Code Do Currently?

Currently the code prints debugging messages into:

```
dbg.log
```

Format:

```
node_address [globaltime] message
```

Debugging can be enabled or disabled by commenting/uncommenting:

```cpp
#define DEBUGLOG
```

in:

```
stdincludes.h
```

Two message types currently exist in the P2P layer:

- `JOINREQ`
- `JOINREP`

### Current Behavior

- `JOINREQ` messages are received by the **introducer**.
- The **introducer** is the **first peer to join the system**.

For Linux systems this is typically:

```
1.0.0.0:0
```

due to **big-endianness**.

A good starting point is to implement **JOINREP responses** from the introducer.

---

# 4. What Do I Implement?

All code must go inside:

```
MP1Node.cpp
MP1Node.h
```

Do **not modify any other files**, since they will be replaced during grading.

You must implement:

- `nodeLoopOps()`
- `recvCallBack()`

These functions are invoked by `nodeLoop()` to perform periodic protocol operations.

---

## Required Functionality

### 1. Introduction

Each new peer contacts a **well-known peer (the introducer)**.

This uses:

- `JOINREQ`
- `JOINREP`

Current behavior:

- `JOINREQ` reaches the introducer
- `JOINREP` is **not implemented**

You must implement `JOINREP` so that it includes the **cluster membership list**.

The introducer **does not need a full list of all peers**. A **partial list of fixed size** is acceptable.

---

### 2. Membership Protocol

Your protocol must satisfy:

- **Completeness for joins and failures**
- **Accuracy when there are no delays or message losses**
- **High accuracy when losses or delays occur**

Recommended approaches:

- **Gossip-style heartbeating**
- **SWIM membership**

You will likely need to modify:

```cpp
struct member
enum MsgTypes
```

in:

```
Mp1Node.h
```

New message types should be processed using functions similar to those used for `JOINREQ` and `JOINREP`.

---

# 4.1 Logging

Logging is **critical** because the grader uses the logs to evaluate correctness.

Files:

```
Log.cpp
Log.h
```

The logging system includes:

- `LOG()` → prints node status to `dbg.log`
- `logNodeAdd()` → logs a node being added
- `logNodeRemove()` → logs a node being removed

Whenever a node **adds or removes a member**, call the appropriate function.

Function parameters:

1. Address of the **recording process**
2. Address of the **node being added or removed**

Example:

```
logNodeAdd(self_address, added_node_address)
logNodeRemove(self_address, removed_node_address)
```

The grader will check these entries.

---

# 5. What Are These Other Files?

### Params.cpp / Params.h

Contains:

```
setparams()
```

This initializes parameters such as:

- Number of peers (`EN_GPSZ`)
- Global time (`globaltime`)

### Member.cpp / Member.h

Contains necessary definitions and declarations.

⚠️ Avoid modifying these files.

---

## Why Is the Code Structure So Involved?

Two main reasons:

### 1. Easy Transition to Real Networks

The `EN*()` functions can be replaced by implementations using **TCP sockets**.

With minor changes such as:

- Replacing periodic calls with threads
- Modifying `nodeLoop()` scheduling

the system can run on a **real distributed network**.

### 2. Easier Debugging and Performance Measurement

Running many distributed processes on a real network is hard to debug.

This simulation framework allows:

- Debugging hundreds of peers
- Measuring protocol behavior
- Running everything on **one host machine**

Once the protocol works in simulation, it can be migrated to real networks.

---

# 6. How Do I Test My Code?

## 6.1 Testing

Compile the code:

```
make
```

Run the program:

```
./Application testcases/<test_name>.conf
```

Configuration files contain parameters such as:

```
MAX_NNB: val
SINGLE_FAILURE: val
DROP_MSG: val
MSG_DROP_PROB: val
```

Parameter meanings:

- **MAX_NNB** – Maximum number of neighbors
- **SINGLE_FAILURE** – Single vs multiple failure scenario
- **MSG_DROP** – Whether message dropping is enabled
- **MSG_DROP_PROB** – Probability of message drop (0–1)

---

## Grader Script

Script:

```
Grader.sh
```

It evaluates three scenarios:

1. Single node failure
2. Multiple node failures
3. Single node failure under a lossy network

Metrics tested:

1. All nodes join correctly
2. Failed nodes are detected (**completeness**)
3. Correct node is identified as failed (**accuracy**)

Test configurations are stored in the:

```
testcases/
```

directory.

---

## 6.2 Grading

Total points:

```
90 points
```

The grader reports how many points your implementation receives.

Your goal is to **pass all tests on the Coursera grading platform**.

---

## 6.3 Running the Grader Locally

Make the script executable:

```
chmod +x Grader.sh
```

Run the grader:

```
./Grader.sh
```

Ensure you are using **C++11 or newer (gcc ≥ 4.7)**.

---