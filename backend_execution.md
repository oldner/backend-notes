### Backend Execution Patterns Introduction

Backend execution patterns are fundamental approaches to handling incoming connections and processing requests in server applications. Think of it like a restaurant system - you need to manage how customers (requests) come in, who serves them (threads/processes), and how their orders (data) are processed.

### Processes and Threads

Let's dive deep into what these are:

**Process:**
- A process is like a completely independent restaurant. It has its own:
  - Memory space (like having its own kitchen)
  - Resources (like having its own staff, equipment)
  - State (like having its own order system)

Here's a practical example in Node.js:
```javascript
// main.js
const { fork } = require('child_process');

// Create a new process
const childProcess = fork('worker.js');

// Send message to child process
childProcess.send({ task: 'processOrder', orderId: 123 });

// Receive message from child process
childProcess.on('message', (message) => {
    console.log('Order processed:', message);
});
```

```javascript
// worker.js
process.on('message', (message) => {
    // Process the order in separate process
    const result = doSomeHeavyWork(message.orderId);
    process.send({ status: 'completed', result });
});
```

**Thread:**
- A thread is like a worker within the restaurant. Multiple threads share:
  - Same memory space (same kitchen)
  - Same resources (same equipment)
  - But can execute independently (different tasks simultaneously)

Here's an example in Rust showing thread usage:
```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        // This runs in a separate thread
        for i in 1..10 {
            println!("Thread: number {}", i);
            thread::sleep(std::time::Duration::from_millis(1));
        }
    });

    // Main thread continues running
    for i in 1..5 {
        println!("Main: number {}", i);
        thread::sleep(std::time::Duration::from_millis(1));
    }

    // Wait for spawned thread to finish
    handle.join().unwrap();
}
```

### How Backend Accepts Connections

This is like having a host at a restaurant who manages incoming customers. Here's how it works:

1. **Socket Creation**: First, the server creates a socket (like opening the restaurant door)
2. **Binding**: The socket is bound to a specific address and port (like having a specific restaurant location)
3. **Listening**: The server starts listening for connections (like having a host ready to greet customers)
4. **Accepting**: When a connection comes in, it's accepted and handled (like seating customers)

Here's an example in Go:
```go
package main

import (
    "fmt"
    "net"
)

func main() {
    // Create and bind socket to port 8080
    listener, err := net.Listen("tcp", ":8080")
    if err != nil {
        fmt.Println("Failed to create listener:", err)
        return
    }
    defer listener.Close()

    fmt.Println("Server listening on port 8080")

    for {
        // Accept incoming connection
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println("Failed to accept connection:", err)
            continue
        }

        // Handle connection in a new goroutine
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()
    
    buffer := make([]byte, 1024)
    for {
        // Read incoming data
        n, err := conn.Read(buffer)
        if err != nil {
            return
        }

        // Echo back
        conn.Write(buffer[:n])
    }
}
```

### Reading and Sending Socket Data

This is where the actual data transfer happens. Let's understand with two main patterns:

**1. Blocking I/O:**
```go
// Blocking read example
func readData(conn net.Conn) {
    buffer := make([]byte, 1024)
    for {
        n, err := conn.Read(buffer) // This blocks until data is available
        if err != nil {
            return
        }
        processData(buffer[:n])
    }
}
```

**2. Non-blocking I/O with event loop:**
```javascript
// Node.js example (naturally non-blocking)
const server = net.createServer((socket) => {
    socket.on('data', (data) => {
        // Process data when it arrives
        processData(data);
    });
});
```

### Different Execution Patterns

Let's look at various patterns with real-world analogies:

1. **Single Listener, Acceptor, and Reader:**
   ```go
   func main() {
       listener, _ := net.Listen("tcp", ":8080")
       for {
           conn, _ := listener.Accept()
           handleConnection(conn)  // Everything in one thread
       }
   }
   ```

2. **Single Listener, Acceptor with Multiple Readers:**
   ```go
   func main() {
       listener, _ := net.Listen("tcp", ":8080")
       // Create worker pool
       for i := 0; i < 4; i++ {
           go worker(i)
       }
       for {
           conn, _ := listener.Accept()
           // Distribute to workers
           connectionQueue <- conn
       }
   }
   ```

### Backend Idempotency

Idempotency means that making the same request multiple times should have the same effect as making it once. Example:

```go
type PaymentRequest struct {
    RequestID string  // Unique ID for idempotency
    Amount    float64
}

func processPayment(req PaymentRequest) error {
    // Check if request was already processed
    if hasProcessed(req.RequestID) {
        return nil // Already processed, return success
    }
    
    // Process payment
    err := performPayment(req.Amount)
    if err != nil {
        return err
    }
    
    // Mark as processed
    markAsProcessed(req.RequestID)
    return nil
}
```

### Nagle's Algorithm

Nagle's algorithm is about optimizing network traffic by combining small packets. Here's how to control it:

```go
conn, _ := net.Dial("tcp", "localhost:8080")
// Disable Nagle's algorithm
tcpConn := conn.(*net.TCPConn)
tcpConn.SetNoDelay(true)
```

Each of these patterns has its own use cases. For example:
- Single thread pattern: Good for simple applications with low concurrency
- Multiple readers: Better for high-concurrency applications
- Socket sharding: Excellent for very high-performance requirements