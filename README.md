# Task-Per-Connection Design Model (Tokio + Hyper)

---

## 1. How Task-Per-Connection works?

A **Tokio task** is a lightweight async unit of work, and in the server here
(**Hyper Server**), a **new task is created for every incoming TCP connection**.

**Here's the intende flow to be built**

1. **Tokio runtime** starts the server.
2. Hyper **listens for connections** (like an OS event loop).
3. Each time a client connects:
   - Hyper **spawns a new task** using `tokio::spawn`.
   - That task owns and manages the lifecycle of the connection.
4. The task:
   - Reads from the socket
   - Parses HTTP requests.
   - Invokes your handler (e.g, `service_fn`).
   - Sends responses.
   - Repeats if keep-alive is enabled.

The entire interaction for a given connection - from handshake to socket closure - is contained within **one Tokio task**.

---

## 2. Where does this happen in the code?

This logic sits inside **Hyper's server internals**, but from your perspective as the developer:

- You interact via:

```rust
Server::bind(addr).serve(make_service)

```

### Under the hood...

- `serve` sets up a listener loop. For each incoming connection, Hyper calls:

```
tokio::spawn(handle_connection(socket));
```

- Your `make_service` or `service_fn` handles each request within that connection.

- If your service does I/O (e.g., database access), that may trigger additional tasks or awaits.

---

## 3. Why Task-Per-Connection?

- Because it's highly scalable and efficient for this system design for asyncI/O, as well as;

    - `Non-blocking`: Tasks only use CPU when `.await` is ready, so thousands
       can run concurrently.
    - `Isolation`: A slow connection or long response doesn't block others.
    - `Simplicity`: Logical separation of concerns, one task = one connection.
    - `Fine Control`: You can cancel tasks, timeout them, or supervise them individually.

- This allows us to scale each instance of the server to ~100k+ concurrent clients.

---

## 4. Who creates and manages tasks?

- In the case of this server, there are afew entities responsible for different aspects of tasks;

    `Tokio runtime`: Schedules and runs all tasks co-operatively.
    `Hyper`: Accepts connections and spawns per-connection tasks using `tokio::spawn`.
    `You(the developer)`: Provide the `make_service`/`service_fn` handler, and decide what awaits or spawns internally.
    `The code in the handler`: May spawn additional tasks -e.g., DB queries, logging, background work.

- You won't see most per-connection tasks because Hyper handles them, your participation is defining what runs inside them.

---

## 5. Components Summary

- `Hyper`: Accepts connections, spawns per-connection tasks.
- `Tokio::spawn`: Used to launch tasks.
- `Task-per-connection`: Each client connection handles its own async task.
- `Task-per-operation`: Pattern where you spawn additional tasks inside a connection task(e.g., DB, logging).

---

## 6. Cafe Waiters Analogy(Your reasoning while coding)

- Imaging this server is a busy cafe:
    - Clients are customers.
    - Each connection is a table.
    - A tokio task is a waiter.
    - Each waiter only does something when needed(co-op async).
    - You can have thousands of tables and just a few waiters - because they       pause when idle.
