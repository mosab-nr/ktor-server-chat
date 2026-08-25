# Ktor Chat Server

A real-time chat backend built with **Kotlin** and **Ktor**, using WebSockets for live messaging, MongoDB for persistence, and Koin for dependency injection.

> **Status:** Working server. Handles multi-user rooms over WebSockets with message history persisted to MongoDB.

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [API](#api)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)

## Features

- **Real-time messaging** over WebSockets, broadcast to all connected members
- **Message persistence** — every message is written to MongoDB before broadcast, so history survives restarts
- **Session-based identity** — usernames are bound to a cookie session with a server-generated nonce
- **Duplicate-username protection** — a second connection under an active username is rejected with `409 Conflict`
- **Message history endpoint** — full history retrievable over HTTP, newest first
- **Clean shutdown handling** — sockets are closed and members removed when a connection drops

## Tech Stack

| Component | Choice |
|---|---|
| Language | Kotlin 2.0 |
| Framework | Ktor 2.3.12 |
| Engine | Netty |
| Real-time transport | Ktor WebSockets |
| Database | MongoDB via KMongo (coroutine driver) |
| Dependency injection | Koin 3.5.6 |
| Serialisation | kotlinx.serialization (JSON) |
| Logging | Logback + Ktor call logging |

## Architecture

The server is organised into four layers, each with a single responsibility:

```
Routes  ──▶  RoomController  ──▶  MessageDataSource  ──▶  MongoDB
  │                │
  │                └── in-memory member registry (ConcurrentHashMap)
  │
  └── ChatSession (cookie-backed identity)
```

### Connection lifecycle

1. An interceptor assigns a `ChatSession` on first contact, reading `username` from the query string and generating a session nonce
2. The client opens a WebSocket at `/chat-socket`; a request without a session is closed with `VIOLATED_POLICY`
3. `RoomController.onJoin` registers the member, or throws `MemberAlreadyExistsException` if the username is taken
4. Incoming text frames are persisted and then broadcast to every connected member
5. On disconnect — clean or otherwise — the `finally` block removes the member and closes the socket

### Concurrency

Connected members are held in a `ConcurrentHashMap<String, Member>` rather than a plain map, since WebSocket handlers run concurrently across coroutines and the registry is mutated on every join and disconnect.

### Persistence

`MessageDataSource` is an interface; `MessageDataSourceImpl` wraps a KMongo `CoroutineDatabase`. Keeping the interface separate means the storage layer can be swapped — for an in-memory implementation in tests, for instance — without touching `RoomController`.

### Dependency injection

Koin wires the Mongo client, the data source, and the room controller as singletons in `di/MainModule.kt`, and routes obtain the controller through `by inject()` instead of constructing it.

## API

### WebSocket

**`GET /chat-socket`** — upgrade to a WebSocket connection.

Establish a session first by passing a username as a query parameter:

```
ws://localhost:8080/chat-socket?username=mosab
```

Send plain text frames; receive JSON frames:

```json
{
  "text": "Hello",
  "username": "mosab",
  "timestamp": 1719830400000,
  "id": "66a1f0c2e4b0a1b2c3d4e5f6"
}
```

**Close codes**

| Code | Meaning |
|---|---|
| `VIOLATED_POLICY` | No session established before connecting |

### HTTP

**`GET /messages`** — returns the full message history as a JSON array, sorted newest first.

```bash
curl http://localhost:8080/messages
```

## Getting Started

### Prerequisites

- **JDK 17** or newer
- **MongoDB** running locally on the default port (`27017`)
- Gradle (the wrapper is included)

### Run

```bash
git clone https://github.com/mosabmoammar/ktor-server-chat.git
cd ktor-server-chat
./gradlew run
```

The server starts on **port 8080**.

### Quick test

```bash
# History (empty on first run)
curl http://localhost:8080/messages

# WebSocket — using websocat
websocat "ws://localhost:8080/chat-socket?username=alice"
```

Open a second client with a different username and messages will appear in both.

### Tests

```bash
./gradlew test
```

## Configuration

| Setting | Where | Default |
|---|---|---|
| Port | `application.conf`, or the `PORT` environment variable | `8080` |
| Database name | `di/MainModule.kt` | `message_db_yt` |
| WebSocket ping period | `plugins/Sockets.kt` | 15 s |
| WebSocket timeout | `plugins/Sockets.kt` | 15 s |

To deploy behind a host that assigns a port:

```bash
PORT=3000 ./gradlew run
```

## Project Structure

```
src/main/kotlin/example/com/
├── Application.kt              # Entry point, plugin installation
├── data/
│   ├── MessageDataSource.kt    # Persistence interface
│   ├── MessageDataSourceImpl.kt# KMongo implementation
│   └── model/Message.kt        # Serialisable message entity
├── di/MainModule.kt            # Koin module
├── plugins/
│   ├── Sockets.kt              # WebSocket configuration
│   ├── Routing.kt              # Route registration
│   ├── Security.kt             # Session handling
│   ├── Serialization.kt        # JSON content negotiation
│   └── Monitoring.kt           # Call logging
├── room/
│   ├── RoomController.kt       # Member registry and broadcast logic
│   ├── Member.kt               # Connected member
│   └── MemberAlreadyExistsException.kt
├── routes/ChatRoutes.kt        # WebSocket and history endpoints
└── session/ChatSession.kt      # Session model
```

## Roadmap

- [ ] Multiple named rooms rather than a single global room
- [ ] Authentication beyond the username-based session
- [ ] Pagination for the message history endpoint
- [ ] Typing indicators and presence
- [ ] Docker Compose setup bundling the server and MongoDB

## License

Not currently licensed for reuse. Contact the author regarding use of this code.
