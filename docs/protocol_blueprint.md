# Tic-Tac-Toe Application Protocol Blueprint

| | |
|---|---|
| **Project** | CS 457 Networked Tic-Tac-Toe |
| **Author** | Evan Lira |
| **Protocol version** | 1.2 |
| **Last updated** | 2026-10-04 |
| **Transport** | TCP (one long-lived connection per client) |
| **Serialization** | JSON, UTF-8 |
| **Framing** | Option A: newline-delimited JSON (`\n`, byte `0x0A`) |
| **SOW reference** | §2 Application-Layer Messaging Protocol Blueprint (Sprint 1) |
| **Companion documents** | [`fsm_specification.md`](fsm_specification.md) (server state machine) · [`ai_prompts.md`](ai_prompts.md) (AI prompting & constraint strategy) |

---

## Contents

1. [Protocol Overview](#1-protocol-overview)
2. [Transport Layer & Packet Framing Mechanism](#2-transport-layer--packet-framing-mechanism)
3. [Common Message Envelope & Data Types](#3-common-message-envelope--data-types)
4. [Message Structures](#4-message-structures)
5. [Error Codes & Validation Order](#5-error-codes--validation-order)
6. [Game State Machine (summary)](#6-game-state-machine-summary)
7. [Disconnect & Forfeit Management](#7-disconnect--forfeit-management)
8. [Wire Examples](#8-wire-examples)
9. [AI Prompting & Constraint Strategy (summary)](#9-ai-prompting--constraint-strategy-summary)
10. [Revision History](#10-revision-history)

### Sprint 1 Deliverables

| # | Deliverable | File |
|---|---|---|
| 1 | Application Protocol Blueprint | `docs/protocol_blueprint.md` (this file) |
| 2 | Game State Machine (FSM) Specification | [`docs/fsm_specification.md`](fsm_specification.md) |
| 3 | AI Prompting & Constraint Strategy | [`docs/ai_prompts.md`](ai_prompts.md) |

### Sprint 1 Requirements Traceability

| Sprint 1 requirement | Where it is met |
|---|---|
| Design the protocol and FSM before writing application code | This document and [`fsm_specification.md`](fsm_specification.md) |
| **2.1** Transport protocol: TCP | [§2.1](#21-transport--serialization) |
| **2.1** Serialization format: structured JSON | [§2.1](#21-transport--serialization), [§3](#3-common-message-envelope--data-types) |
| **2.1** Deterministic framing rule that handles coalescing and fragmentation | [§2.2](#22-the-framing-rule)–[§2.4](#24-receiver-rules); worked cases in [§8.4](#84-coalescing--fragmentation-how-the-receiver-buffer-evolves) |
| **2.1** Wire stream example (continuous stream) | [§8.2](#82-continuous-stream-client--server), byte-level in [§8.3](#83-byte-level-view-xxd) |
| **2.1** JSON schema / structure specification (`MOVE`) | [§3.1](#31-envelope), [§4.4](#44-move) |
| **2.2** Exact message structures for all eight message types | [§4](#4-message-structures) |
| Explicit message schemas | [§3](#3-common-message-envelope--data-types), [§4](#4-message-structures) |
| Complete field specifications and data types for all message types, including `DISCONNECT` / forfeit management | [§3](#3-common-message-envelope--data-types), [§4](#4-message-structures), [§7](#7-disconnect--forfeit-management) |
| **2.3** Server-side FSM in Mermaid `stateDiagram-v2`: valid moves, invalid moves, disconnects | [`fsm_specification.md`](fsm_specification.md) (summary in [§6](#6-game-state-machine-summary)) |
| **2.4** Connection termination & socket lifecycle | [§7](#7-disconnect--forfeit-management) (protocol behavior), [`fsm_specification.md` §5](fsm_specification.md#5-connection-termination--socket-lifecycle) (detection and handling) |
| Show how AI assistants are prompted and constrained to follow this blueprint | [`ai_prompts.md`](ai_prompts.md) |

---

## 1. Protocol Overview

- **Architecture:** client–server. One server hosts a single game room with exactly two player slots. Clients never talk to each other.
- **Server-authoritative:** the server owns the board, turn order, scores, and win/draw detection. Clients only send intents (`CONNECT`, `MOVE`, `DISCONNECT`) and render whatever the server sends in `STATE_UPDATE` and `GAME_OVER`. A client never updates its local board on its own.
- **Roles:** the first player to connect is **Player 1** (`PLAYER_1`, plays `X`). The second is **Player 2** (`PLAYER_2`, plays `O`). Roles stay the same for the whole match.
- **Match vs. game:** a *match* is the series of games played by the same two players. Scores belong to the match and reset when a new opponent joins. A *game* is one board, from empty to win, draw, or forfeit.
- **Turn order (SOW §1.2):** who moves first in game 1 is random. After that it alternates every game.
- **Reset or quit (SOW §1.2):** after a win or draw, the server starts the next game automatically after `NEXT_GAME_DELAY_S`. A player who wants to stop sends `DISCONNECT`.

### 1.1 Protocol Constants

| Constant | Value | Notes |
|---|---|---|
| `DEFAULT_PORT` | `5457/tcp` | Can be overridden on the command line; client connects to `server.lira.edu:5457` |
| `ENCODING` | UTF-8 | All bytes on the wire |
| `DELIMITER` | `0x0A` (`\n`) | Terminates every message |
| `MAX_FRAME_BYTES` | `4096` | Includes the trailing `\n`. The largest real message is 275 bytes. |
| `PLAYER_ID_PATTERN` | `^[A-Za-z0-9_-]{1,16}$` | Player aliases |
| `SERVER_ID` | `"SERVER"` | Reserved `player_id` used on every server→client message |
| `TURN_TIMEOUT_S` | `60` | Active player must move within this window (§7.4) |
| `NEXT_GAME_DELAY_S` | `5` | Pause between `GAME_OVER` and the next `GAME_START` (§7.4) |

---

## 2. Transport Layer & Packet Framing Mechanism

### 2.1 Transport & Serialization

| | |
|---|---|
| **Transport protocol** | TCP. The server listens on `5457/tcp`. Each client opens one connection and keeps it for the whole session. |
| **Serialization format** | Structured JSON, UTF-8 encoded. One JSON object per message (envelope in §3). |
| **Framing mechanism** | **Option A: newline-delimited JSON (`\n` framing).** Each message is terminated by byte `0x0A`. |

**Why Option A instead of Option B (length prefix) or Option C (pipe-delimited text):**

- **Readable on the wire.** Every frame is plain text, so it can be read directly in Wireshark's *Follow TCP Stream* (Sprint 5) and tested by hand with `nc server.lira.edu 5457`. A binary length prefix would show up as unreadable bytes.
- **No delimiter collisions.** The usual risk with delimiter framing is a payload that contains the delimiter. Compact JSON escapes every newline inside a string, so a raw `0x0A` can never appear inside a message (§2.5). That removes the main advantage of Option B.
- **Bounded memory.** Messages are under 300 bytes, and `MAX_FRAME_BYTES` = 4096 caps the receive buffer. That gives the same bounded allocation that a length prefix provides.
- **Structured nested data.** The board, scores, role map, and winning line are nested. JSON represents them directly, where Option C would need a custom grammar with secondary delimiters for each message.
- **Easy debugging.** A 3×3 board is a JSON array of arrays, so printing a raw frame shows the grid as it is: `"board":[["X","-","-"],["O","X","-"],["-","-","-"]]`. If an `X` lands in the wrong cell, the printed message shows it right away.

The one weakness of newline framing for this project is user-typed text (player aliases). It is handled explicitly in §2.5.

### 2.2 The Framing Rule

> Every message is one JSON object, UTF-8 encoded, and terminated by exactly one newline character `\n` (`0x0A`). The receiver adds incoming bytes to a per-connection stream buffer until it sees a `\n`. It then takes out the complete line and deserializes it as one JSON object.

```
<frame>  ::= <json-object-utf8> 0x0A
<stream> ::= <frame>*
```

TCP is a continuous byte stream with no built-in message boundaries. Two things follow, and the framing rule handles both:

- **Coalescing:** messages sent back-to-back can arrive in a single `recv()` chunk. The receiver extracts **every** complete `\n`-terminated frame in the buffer, not just the first one.
- **Fragmentation:** a single message can be split across several `recv()` chunks. The receiver keeps the incomplete tail in the buffer until the rest arrives, and never parses a frame before its `\n`.

Both can happen in the same chunk (the end of one message plus the start of the next). The `\n` delimiter is the only message boundary. Message boundaries are **never** inferred from `recv()` call boundaries. This makes the rule deterministic: the same byte stream produces the same messages however TCP splits it (worked examples in §8.4).

### 2.3 Sender Rules

1. Serialize with **compact** JSON on a single line: `json.dumps(msg, separators=(",", ":"))`.
   **Never** use `indent=`. Pretty-printing inserts raw newlines inside the object and breaks framing.
2. Encode to UTF-8 and append exactly one `b"\n"`.
3. If the resulting frame is larger than `MAX_FRAME_BYTES`, do not send it (this is a programming error).
4. Write with `sock.sendall(frame)`, not `send()`. `send()` may write only part of the buffer.
5. One message per frame. Never put two JSON objects on the same line or wrap several messages in an array.

### 2.4 Receiver Rules

1. Keep **one byte buffer per connection**. Never share a buffer between sockets.
2. Append every `recv()` chunk to the buffer.
3. While the buffer contains `0x0A`, split at the **first** `0x0A`. The bytes before it are one frame. The bytes after it stay in the buffer.
4. Strip any trailing `\r` from the frame (this tolerates CRLF from `nc`/`telnet` testing) and silently skip empty lines.
5. Decode the frame as UTF-8 and parse it as JSON. The result must be a JSON **object**. Otherwise the server answers `ERROR MALFORMED_JSON` (not fatal: the next `\n` still marks a valid boundary, so the stream stays in sync).
6. If a frame, or the unterminated data left in the buffer, reaches `MAX_FRAME_BYTES` without a `\n`, the receiver gives up on the stream. The server sends `ERROR FRAME_TOO_LARGE` (fatal) and closes the connection.
7. `recv()` returning `b""` means the peer closed the connection (EOF). Any bytes still in the buffer are a **truncated frame**. They are discarded and never processed.

### 2.5 Why `\n` Framing Is Safe

- **JSON escapes newlines inside strings.** The JSON grammar does not allow raw control characters (U+0000–U+001F, including LF) inside strings, so they must be escaped. A line break inside a `reason` string goes on the wire as the two characters `\` `n` (bytes `0x5C 0x6E`), never as a raw `0x0A`. A compact encoder therefore never emits a raw `0x0A` inside a message.
- **UTF-8 never hides `0x0A` inside a multi-byte character.** Every byte of a multi-byte UTF-8 sequence is `≥ 0x80`, so a `0x0A` byte always means LF. Splitting on raw bytes *before* decoding is safe.
- **No whitespace between tokens.** Compact separators mean the only `0x0A` in a correctly formed stream is the delimiter.

**Caveat: user-typed text (player aliases).** Players choose their own alias, and an alias that carries a raw `\n` onto the wire would end the frame early and split one message into two broken ones. Three independent layers prevent this:

1. **Client input check.** The client calls `.strip()` on the typed alias and checks it against `PLAYER_ID_PATTERN` (`^[A-Za-z0-9_-]{1,16}$`) **before** sending `CONNECT`. Newlines, spaces, tabs, and other control characters fail the pattern, so the client asks again and sends nothing.
2. **Serializer escaping.** Every message is built with `json.dumps()`, which turns a newline inside a string into the two characters `\` `n`. Messages are **never** assembled with f-strings or string concatenation (e.g. `'{"player_id":"' + name + '"}'`), because that would put the user's raw newline onto the wire.
3. **Server validation.** The server checks the alias against the same pattern and rejects anything else with `ERROR INVALID_NAME`. A newline that reaches the server inside a correctly escaped string is therefore rejected, not stored, and can never be echoed to the other player.

The only other free-text field a client sends is `DISCONNECT.reason`. It is protected by layer 2, and the server only logs it, never forwards it. §8.6 shows both the correct and the broken behavior on the wire.

### 2.6 Reference Implementation (Python)

```python
import json
import socket

DELIMITER = b"\n"           # 0x0A
MAX_FRAME_BYTES = 4096      # includes the trailing delimiter


class FrameTooLarge(Exception):
    pass


def encode_frame(msg: dict) -> bytes:
    """Serialize one message dict into exactly one wire frame."""
    frame = json.dumps(msg, separators=(",", ":")).encode("utf-8") + DELIMITER
    if len(frame) > MAX_FRAME_BYTES:
        raise FrameTooLarge(len(frame))
    return frame


def send_msg(sock: socket.socket, msg: dict) -> None:
    sock.sendall(encode_frame(msg))  # sendall: never assume one send() == one frame


class FrameReader:
    """Accumulates recv() bytes and returns every complete line received so far."""

    def __init__(self):
        self.buffer = b""

    def feed(self, data: bytes) -> list[bytes]:
        self.buffer += data
        lines = []
        while DELIMITER in self.buffer:
            line, _, self.buffer = self.buffer.partition(DELIMITER)
            if len(line) + 1 > MAX_FRAME_BYTES:
                raise FrameTooLarge(len(line) + 1)
            line = line.rstrip(b"\r")    # tolerate CRLF from nc/telnet testing
            if line:                     # skip blank lines
                lines.append(line)
        if len(self.buffer) >= MAX_FRAME_BYTES:  # no delimiter within the limit
            raise FrameTooLarge(len(self.buffer))
        return lines
```

Typical receive loop:

```python
reader = FrameReader()
while True:
    data = sock.recv(4096)
    if not data:                      # EOF: peer closed (see §7)
        break
    for line in reader.feed(data):    # may yield 0, 1, or many frames
        msg = json.loads(line.decode("utf-8"))
        handle(msg)
```

---

## 3. Common Message Envelope & Data Types

### 3.1 Envelope

Every message, in both directions, is a JSON object with this top-level shape:

```json
{
  "msg_type": "MOVE",
  "player_id": "Player_1",
  "payload": {
    "row": 0,
    "col": 2
  },
  "timestamp": 1727000000
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `msg_type` | `string` | **Yes** | One of the eight types in §4. Uppercase and case-sensitive. |
| `player_id` | `PlayerId` or `"SERVER"` | **Yes** | Sender identity. Clients use their registered alias. The server always uses `"SERVER"`. |
| `payload` | `object` | Conditional | Required when the message type defines required payload fields. It may be left out, or sent as `{}`, for types with no required payload fields (`CONNECT`, `DISCONNECT`). A missing payload is treated as `{}`. |
| `timestamp` | `integer` | **Yes** | Sender's Unix epoch time in whole seconds (UTC), e.g. `int(time.time())`. Informational and for logs only. It is **not** used for ordering (TCP already guarantees order) and it is **not** validated against the receiver's clock. |

### 3.2 Data Types

| Type | JSON encoding | Constraints |
|---|---|---|
| `string` | JSON string | UTF-8. Free-text fields (`message`, `reason`) are at most 128 characters. |
| `integer` | JSON number with no fraction or exponent | Python: validate with `type(v) is int`. `isinstance(v, int)` wrongly accepts `True`/`False`. |
| `boolean` | `true` / `false` | — |
| `null` | `null` | Used only where a field is listed as nullable |
| `PlayerId` | string | The player's alias. Matches `^[A-Za-z0-9_-]{1,16}$`: letters, digits, `_`, and `-` only, so no spaces, `\n`, or other control characters (§2.5). Must not equal `SERVER` (case-insensitive). Unique within the room. |
| `Role` | string | `"PLAYER_1"` or `"PLAYER_2"` |
| `Symbol` | string | `"X"` or `"O"`. `PLAYER_1` always plays `X` and `PLAYER_2` always plays `O`. |
| `Cell` | string | `"X"`, `"O"`, or `"-"` (empty). `"-"` keeps a printed board readable, e.g. `[["X","-","-"],["O","X","-"],["-","-","-"]]`. |
| `Board` | array of 3 arrays of 3 `Cell` | Indexed as `board[row][col]` |
| `Coord` | array of 2 integers | `[row, col]`, each 0–2 |
| `Scores` | object | `PlayerId → integer ≥ 0`. Always contains exactly the two players in the match. |

### 3.3 Board Coordinates

Row 0 is the top and column 0 is the left:

```
            col 0    col 1    col 2
          +--------+--------+--------+
  row 0   | [0][0] | [0][1] | [0][2] |
          +--------+--------+--------+
  row 1   | [1][0] | [1][1] | [1][2] |
          +--------+--------+--------+
  row 2   | [2][0] | [2][1] | [2][2] |
          +--------+--------+--------+
```

### 3.4 General Rules

- **Unknown fields are ignored**, at the top level and inside `payload`. This allows forward-compatible additions.
- **Field order is not significant.** JSON objects are unordered. The examples use `msg_type, player_id, payload, timestamp` for readability.
- **Identity comes from the socket, not the message.** At `CONNECT` the server binds a `player_id` to the socket. Every later message on that socket must carry the same `player_id`, otherwise the server replies `ERROR PLAYER_ID_MISMATCH` and ignores the message. A client cannot act as the other player by changing the field.
- **Clients never send `ERROR`.** If a client receives a line it cannot parse, or a `msg_type` it does not know, it logs it and ignores it.

---

## 4. Message Structures

This section defines the exact structure of every message exchanged between the clients and the server. There are exactly eight message types:

| # | Message Type | Direction | Purpose & Description | Delivery | Payload fields | Spec |
|---|---|---|---|---|---|---|
| 1 | `CONNECT` | Client → Server | Client requests to join the game room with a player alias. | — | *(none; the alias is the envelope `player_id`)* | [§4.1](#41-connect) |
| 2 | `LOBBY_WAIT` | Server → Client | Server notifies Client 1 that it is waiting for Player 2 to connect. | Unicast | `players_connected`, `players_required`, `message` | [§4.2](#42-lobby_wait) |
| 3 | `GAME_START` | Server → Clients | Server notifies both clients that the game has started and assigns roles (Player 1 / Player 2). | Unicast to each player | `game_number`, `your_role`, `players`, `first_turn`, `turn_timeout_s` | [§4.3](#43-game_start) |
| 4 | `MOVE` | Client → Server | Active player submits move coordinates. | — | `row`, `col` | [§4.4](#44-move) |
| 5 | `STATE_UPDATE` | Server → Clients | Server broadcasts the updated board state, scores, and active player turn. | Broadcast (identical bytes to both) | `board`, `current_turn`, `move_number`, `last_move`, `scores`, `draws` | [§4.5](#45-state_update) |
| 6 | `ERROR` | Server → Client | Server notifies the client of an out-of-turn move, invalid coordinates, or a malformed message. | Unicast to the offender | `code`, `message`, `ref_msg_type`, `fatal` | [§4.6](#46-error) |
| 7 | `DISCONNECT` | Client → Server | Client notifies the server of an intentional departure/quit. | — | `reason` *(optional)* | [§4.7](#47-disconnect) |
| 8 | `GAME_OVER` | Server → Clients | Server broadcasts the final game outcome (Winner / Draw / Forfeit) and final scores. | Broadcast to players still connected | `result`, `winner`, `winning_line`, `forfeit_reason`, `scores`, `draws`, `next_game_in_s` | [§4.8](#48-game_over) |

Each subsection below gives the envelope values, every payload field (type, whether it is required, and its allowed values), an exact example, and the same message in its single-line wire form. The pretty-printed examples are for readability. **On the wire, every message is compact JSON on one line followed by `\n`** (see §2 and §8).

---

### 4.1 `CONNECT`

**Direction:** Client → Server  ·  **Sent:** once, as the first message after the TCP connection opens

| Envelope field | Value |
|---|---|
| `msg_type` | `"CONNECT"` |
| `player_id` | The alias the client wants to use (`PlayerId`) |
| `payload` | Leave it out or send `{}`. There are no payload fields. |

**Exact structure:**

```json
{
  "msg_type": "CONNECT",
  "player_id": "Alice",
  "timestamp": 1727000000
}
```

**Wire form:**

```
{"msg_type":"CONNECT","player_id":"Alice","timestamp":1727000000}\n
```

**Client behavior before sending:** strip the typed alias and check it against `PLAYER_ID_PATTERN`. If it fails (for example it contains a space or newline), ask the user again without sending anything (§2.5).

**Server response:**

| Condition | Response |
|---|---|
| First player in an empty room | `LOBBY_WAIT` to the sender. The sender becomes `PLAYER_1`. |
| Second player | The sender becomes `PLAYER_2`. `GAME_START` goes to each player, followed by `STATE_UPDATE` to both. |
| Alias fails `PLAYER_ID_PATTERN` or is `SERVER` | `ERROR INVALID_NAME` (not fatal; the client may send `CONNECT` again) |
| Alias already used by the other player | `ERROR NAME_TAKEN` (not fatal; the client may send `CONNECT` again) |
| Two players already registered | `ERROR ROOM_FULL` (**fatal**; the connection is closed) |

---

### 4.2 `LOBBY_WAIT`

**Direction:** Server → Client (unicast)  ·  **Sent:** (a) after the first player's successful `CONNECT`, and (b) to the remaining player after their opponent leaves (§7.2)

| Envelope field | Value |
|---|---|
| `msg_type` | `"LOBBY_WAIT"` |
| `player_id` | `"SERVER"` |

| Payload field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `players_connected` | `integer` | Yes | `1` | Registered players in the room, including the recipient |
| `players_required` | `integer` | Yes | `2` | Players needed to start |
| `message` | `string` | Yes | ≤ 128 chars | Status text to show the user |

**Exact structure:**

```json
{
  "msg_type": "LOBBY_WAIT",
  "player_id": "SERVER",
  "payload": {
    "players_connected": 1,
    "players_required": 2,
    "message": "Waiting for an opponent..."
  },
  "timestamp": 1727000000
}
```

**Wire form:**

```
{"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"players_required":2,"message":"Waiting for an opponent..."},"timestamp":1727000000}\n
```

**Client behavior:** show `message` and keep waiting. The next message will be `GAME_START`.

---

### 4.3 `GAME_START`

**Direction:** Server → Clients (one copy to each player)  ·  **Sent:** when the second player registers (`game_number` = 1), and `NEXT_GAME_DELAY_S` after each `WIN`/`DRAW` (`game_number` + 1)

The two copies differ only in `your_role`, so each player gets its own message. The server **always** follows `GAME_START` immediately with a `STATE_UPDATE` (`move_number` = 0). That `STATE_UPDATE` carries the starting board, the scores, and whose turn it is, so clients have one rendering path.

| Envelope field | Value |
|---|---|
| `msg_type` | `"GAME_START"` |
| `player_id` | `"SERVER"` |

| Payload field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `game_number` | `integer` | Yes | ≥ 1 | 1 for the first game of a match, +1 for each later game |
| `your_role` | `Role` | Yes | `"PLAYER_1"` or `"PLAYER_2"` | **The role assigned to the recipient** |
| `players` | `object` | Yes | Exactly the keys `PLAYER_1` and `PLAYER_2` | Role assignment for both players, fixed for the match |
| `players.<role>.player_id` | `PlayerId` | Yes | — | Alias of the player who holds that role |
| `players.<role>.symbol` | `Symbol` | Yes | `PLAYER_1` → `"X"`, `PLAYER_2` → `"O"` | Symbol that player places on the board |
| `first_turn` | `PlayerId` | Yes | One of the two players | Who moves first this game. Random for game 1, then alternates. |
| `turn_timeout_s` | `integer` | Yes | ≥ 0 (`0` = disabled) | Seconds allowed per turn (§7.4) |

**Exact structure** (Alice's copy; Bob's copy is identical except `"your_role": "PLAYER_2"`):

```json
{
  "msg_type": "GAME_START",
  "player_id": "SERVER",
  "payload": {
    "game_number": 1,
    "your_role": "PLAYER_1",
    "players": {
      "PLAYER_1": {"player_id": "Alice", "symbol": "X"},
      "PLAYER_2": {"player_id": "Bob", "symbol": "O"}
    },
    "first_turn": "Alice",
    "turn_timeout_s": 60
  },
  "timestamp": 1727000010
}
```

**Wire form:**

```
{"msg_type":"GAME_START","player_id":"SERVER","payload":{"game_number":1,"your_role":"PLAYER_1","players":{"PLAYER_1":{"player_id":"Alice","symbol":"X"},"PLAYER_2":{"player_id":"Bob","symbol":"O"}},"first_turn":"Alice","turn_timeout_s":60},"timestamp":1727000010}\n
```

**Client behavior:** save `your_role` and `players` so the UI can show "You are Player 1 (X) vs. Bob (O)". Then wait for the `STATE_UPDATE` that follows.

---

### 4.4 `MOVE`

**Direction:** Client → Server  ·  **Sent:** by the active player only (the `current_turn` in the latest `STATE_UPDATE`) while a game is in progress

| Envelope field | Value |
|---|---|
| `msg_type` | `"MOVE"` |
| `player_id` | The sender's registered alias |

| Payload field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `row` | `integer` | Yes | 0–2 | Board row (0 = top) |
| `col` | `integer` | Yes | 0–2 | Board column (0 = left) |

**Exact structure:**

```json
{
  "msg_type": "MOVE",
  "player_id": "Player_1",
  "payload": {
    "row": 0,
    "col": 2
  },
  "timestamp": 1727000000
}
```

**Wire form:**

```
{"msg_type":"MOVE","player_id":"Player_1","payload":{"row":0,"col":2},"timestamp":1727000000}\n
```

**Server response:**
- **Accepted:** a `STATE_UPDATE` broadcast to both players. If the move ends the game, a `GAME_OVER` broadcast follows.
- **Rejected:** an `ERROR` to the sender only (validation order in §5.2). The board and turn do not change, and it is still the sender's turn.

---

### 4.5 `STATE_UPDATE`

**Direction:** Server → Clients (broadcast; both players receive identical bytes)  ·  **Sent:** right after every `GAME_START`, and after every accepted `MOVE`

Each `STATE_UPDATE` describes the whole game screen on its own: board, scores, and whose turn it is. A client only needs the most recent one to draw its display.

| Envelope field | Value |
|---|---|
| `msg_type` | `"STATE_UPDATE"` |
| `player_id` | `"SERVER"` |

| Payload field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `board` | `Board` | Yes | 3×3 of `"X"`, `"O"`, `"-"` | Full board, sent every time (never a diff) |
| `current_turn` | `PlayerId` or `null` | Yes | — | **Active player:** who must move next. `null` after the game-ending move. |
| `move_number` | `integer` | Yes | 0–9 | Moves played so far in this game |
| `last_move` | `object` or `null` | Yes | — | The move that produced this state. `null` when `move_number` = 0. |
| `last_move.player_id` | `PlayerId` | Yes* | — | Who moved |
| `last_move.symbol` | `Symbol` | Yes* | — | Symbol placed |
| `last_move.row` | `integer` | Yes* | 0–2 | — |
| `last_move.col` | `integer` | Yes* | 0–2 | — |
| `scores` | `Scores` | Yes | — | **Match wins** for each player. The game-ending `STATE_UPDATE` already counts the game it ends. |
| `draws` | `integer` | Yes | ≥ 0 | Match draws, counted the same way |

<sub>* Required when `last_move` is not `null`.</sub>

**Exact structure:**

```json
{
  "msg_type": "STATE_UPDATE",
  "player_id": "SERVER",
  "payload": {
    "board": [["X", "O", "-"],
              ["-", "X", "-"],
              ["-", "-", "-"]],
    "current_turn": "Bob",
    "move_number": 3,
    "last_move": {"player_id": "Alice", "symbol": "X", "row": 0, "col": 0},
    "scores": {"Alice": 0, "Bob": 0},
    "draws": 0
  },
  "timestamp": 1727000025
}
```

**Wire form:**

```
{"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["X","O","-"],["-","X","-"],["-","-","-"]],"current_turn":"Bob","move_number":3,"last_move":{"player_id":"Alice","symbol":"X","row":0,"col":0},"scores":{"Alice":0,"Bob":0},"draws":0},"timestamp":1727000025}\n
```

**Client behavior:** redraw the board and scoreboard. If `current_turn` is this client's alias, ask the user for a move. Otherwise show "Waiting for <opponent>...".

---

### 4.6 `ERROR`

**Direction:** Server → Client (unicast to the offending client only; never broadcast)  ·  **Sent:** when a client message is rejected, or right before the server closes a connection

| Envelope field | Value |
|---|---|
| `msg_type` | `"ERROR"` |
| `player_id` | `"SERVER"` |

| Payload field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `code` | `string` | Yes | One of the codes in §5.1 | Machine-readable reason |
| `message` | `string` | Yes | ≤ 128 chars | Text to show the user |
| `ref_msg_type` | `string` or `null` | Yes | — | `msg_type` of the rejected message, or `null` if it could not be parsed or the error was not caused by a message |
| `fatal` | `boolean` | Yes | — | `true` means the server closes this connection right after sending |

The three cases in the purpose description map to these codes. §5.1 has the full list.

| Case | `code` |
|---|---|
| Out-of-turn move | `NOT_YOUR_TURN` |
| Invalid coordinates | `OUT_OF_BOUNDS`, `CELL_OCCUPIED` |
| Malformed message | `MALFORMED_JSON`, `INVALID_FIELD`, `UNKNOWN_MSG_TYPE`, `FRAME_TOO_LARGE` |

**Exact structure:**

```json
{
  "msg_type": "ERROR",
  "player_id": "SERVER",
  "payload": {
    "code": "NOT_YOUR_TURN",
    "message": "It is Alice's turn.",
    "ref_msg_type": "MOVE",
    "fatal": false
  },
  "timestamp": 1727000012
}
```

**Wire form:**

```
{"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"NOT_YOUR_TURN","message":"It is Alice's turn.","ref_msg_type":"MOVE","fatal":false},"timestamp":1727000012}\n
```

**Client behavior:** show `message`. If `fatal` is `false`, carry on; after a rejected `MOVE`, ask the user to try again. If `fatal` is `true`, expect EOF, close the socket, and exit.

---

### 4.7 `DISCONNECT`

**Direction:** Client → Server  ·  **Sent:** when the user deliberately quits (e.g. types `quit` or presses Ctrl-C)

| Envelope field | Value |
|---|---|
| `msg_type` | `"DISCONNECT"` |
| `player_id` | The sender's registered alias |

| Payload field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `reason` | `string` | No | ≤ 128 chars | Readable explanation, used for server logs |

**Exact structure:**

```json
{
  "msg_type": "DISCONNECT",
  "player_id": "Bob",
  "payload": {
    "reason": "Player quit"
  },
  "timestamp": 1727000032
}
```

**Wire form:**

```
{"msg_type":"DISCONNECT","player_id":"Bob","payload":{"reason":"Player quit"},"timestamp":1727000032}\n
```

**Server response:** the server sends **nothing back** to the departing client and closes its socket. What happens to the opponent depends on the room state (§7.2):
- **During a game:** the departure is a **forfeit**. The opponent receives `GAME_OVER` (`FORFEIT`) and then `LOBBY_WAIT`.
- **Between games:** the next game is canceled. The opponent receives `LOBBY_WAIT`.
- **In the lobby:** the player is removed. No one else is notified.

The server never sends `DISCONNECT`. When the server itself ends a connection (turn timeout, shutdown, protocol violation), it sends an `ERROR` with `fatal: true` instead (§7.3).

---

### 4.8 `GAME_OVER`

**Direction:** Server → Clients (broadcast to every player still connected)  ·  **Sent:** when a game ends

- **Win or draw:** sent to both players **right after** the game-ending `STATE_UPDATE` (the one with `current_turn: null`).
- **Forfeit:** sent only to the remaining player. The board is not resent.

| Envelope field | Value |
|---|---|
| `msg_type` | `"GAME_OVER"` |
| `player_id` | `"SERVER"` |

| Payload field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `result` | `string` | Yes | `"WIN"`, `"DRAW"`, `"FORFEIT"` | How the game ended |
| `winner` | `PlayerId` or `null` | Yes | `null` only for `DRAW` | Winner. For `FORFEIT`, the player who stayed. |
| `winning_line` | array of 3 `Coord`, or `null` | Yes | Non-null only for `WIN` | The three winning cells, ordered by row and then column |
| `forfeit_reason` | `string` or `null` | Yes | Non-null only for `FORFEIT`: `"DISCONNECT"`, `"CONNECTION_LOST"`, `"TIMEOUT"` | Why the opponent forfeited (§7.1) |
| `scores` | `Scores` | Yes | — | **Final** match wins, including this game |
| `draws` | `integer` | Yes | ≥ 0 | Final match draws, including this game |
| `next_game_in_s` | `integer` or `null` | Yes | `NEXT_GAME_DELAY_S` for `WIN`/`DRAW`, `null` for `FORFEIT` | Seconds until the server sends `GAME_START` for the next game. `null` means there is no next game: the opponent has left and this player is going back to the lobby. |

**Scoring:** `WIN`: winner +1. `DRAW`: `draws` +1. `FORFEIT`: the remaining player +1.
**Determinism:** if one move completes two lines at once, the server reports the first one found in this check order: rows 0→2, columns 0→2, main diagonal, anti-diagonal.

**Exact structure:**

```json
{
  "msg_type": "GAME_OVER",
  "player_id": "SERVER",
  "payload": {
    "result": "WIN",
    "winner": "Alice",
    "winning_line": [[0, 0], [1, 1], [2, 2]],
    "forfeit_reason": null,
    "scores": {"Alice": 1, "Bob": 0},
    "draws": 0,
    "next_game_in_s": 5
  },
  "timestamp": 1727000035
}
```

**Wire forms for each outcome:**

```
{"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"result":"WIN","winner":"Alice","winning_line":[[0,0],[1,1],[2,2]],"forfeit_reason":null,"scores":{"Alice":1,"Bob":0},"draws":0,"next_game_in_s":5},"timestamp":1727000035}\n
{"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"result":"DRAW","winner":null,"winning_line":null,"forfeit_reason":null,"scores":{"Alice":1,"Bob":0},"draws":1,"next_game_in_s":5},"timestamp":1727000090}\n
{"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"result":"FORFEIT","winner":"Alice","winning_line":null,"forfeit_reason":"DISCONNECT","scores":{"Alice":1,"Bob":0},"draws":0,"next_game_in_s":null},"timestamp":1727000032}\n
```

**Client behavior:** show the result and the final scores. If `next_game_in_s` is a number, show a countdown ("Next game in 5 s, type `quit` to leave"). If it is `null`, expect a `LOBBY_WAIT` next.

---

## 5. Error Codes & Validation Order

### 5.1 Error Codes

| Category | `code` | Fatal | Trigger |
|---|---|---|---|
| Out-of-turn move | `NOT_YOUR_TURN` | No | `MOVE` from the player who is not `current_turn` |
| Invalid coordinates | `OUT_OF_BOUNDS` | No | `row` or `col` is not in 0–2 |
| Invalid coordinates | `CELL_OCCUPIED` | No | Target cell is not `"-"` |
| Malformed message | `MALFORMED_JSON` | No | Line is not valid UTF-8 or JSON, or is not a JSON object |
| Malformed message | `INVALID_FIELD` | No | A required field is missing or has the wrong type (e.g. `"row": "1"`) |
| Malformed message | `UNKNOWN_MSG_TYPE` | No | `msg_type` is not one of the eight types, or is a server→client type sent by a client |
| Malformed message | `FRAME_TOO_LARGE` | **Yes** | `MAX_FRAME_BYTES` reached without a `\n` |
| Identity / registration | `PLAYER_ID_MISMATCH` | No | Envelope `player_id` is not the alias registered to this socket |
| Identity / registration | `INVALID_NAME` | No | `CONNECT` alias fails `PLAYER_ID_PATTERN` or is `SERVER` |
| Identity / registration | `NAME_TAKEN` | No | `CONNECT` alias is already used by the other player |
| Identity / registration | `ROOM_FULL` | **Yes** | `CONNECT` when two players are already registered |
| Wrong state | `UNEXPECTED_MESSAGE` | No | Valid message that is not allowed in the current state (see §6.2) |
| Server-initiated close | `TURN_TIMEOUT` | **Yes** | Active player sent no accepted `MOVE` within `TURN_TIMEOUT_S` (§7.4) |
| Server-initiated close | `SERVER_SHUTDOWN` | **Yes** | Server is shutting down |

After a fatal error the server closes the socket. If that player was in a game, this counts as a forfeit (`TIMEOUT` for `TURN_TIMEOUT`, otherwise `CONNECTION_LOST`; see §7.1).

### 5.2 Validation Order for an Incoming Message

The server applies these checks in order and stops at the first failure:

1. Frame decodes as a JSON object → `MALFORMED_JSON`
2. `msg_type` is a known client→server type (`CONNECT`, `MOVE`, `DISCONNECT`) → `UNKNOWN_MSG_TYPE`
3. Envelope fields are present and correctly typed → `INVALID_FIELD`
4. `player_id` matches the socket's registered alias (skipped before `CONNECT`) → `PLAYER_ID_MISMATCH`
5. Message is allowed in the current state (§6.2) → `UNEXPECTED_MESSAGE`
6. Payload fields are present and correctly typed → `INVALID_FIELD`
7. *(MOVE only)* Sender is `current_turn` → `NOT_YOUR_TURN`
8. *(MOVE only)* `0 ≤ row ≤ 2` and `0 ≤ col ≤ 2` → `OUT_OF_BOUNDS`
9. *(MOVE only)* `board[row][col] == "-"` → `CELL_OCCUPIED`
10. Apply the move, check for a win or draw, then broadcast `STATE_UPDATE` (and `GAME_OVER` if the game ended)

---

## 6. Game State Machine (summary)

The server-side state engine is specified in full in **[`fsm_specification.md`](fsm_specification.md)**. That file has the Mermaid `stateDiagram-v2` diagrams, the T1–T21 transition table, per-state handling logic, valid and invalid move handling, and connection termination. This section keeps only the protocol-level rules a client can observe.

### 6.1 States at a Glance

`INIT` → `WAITING_FOR_PLAYERS` → `GAME_START` → `PLAYER_TURN` ⇄ `EVALUATE_MOVE` → `CHECK_WIN_DRAW` → `GAME_OVER` → `GAME_START` (next game) or `CLEANUP` → `WAITING_FOR_PLAYERS`.

Events are only received in `WAITING_FOR_PLAYERS`, `PLAYER_TURN`, and `GAME_OVER`. The other states are transient steps inside a single event ([FSM §1.1](fsm_specification.md#11-states)).

### 6.2 Allowed Client Messages per State

| Client sends | Socket not yet registered | `WAITING_FOR_PLAYERS` (in lobby) | `PLAYER_TURN` (game running) | `GAME_OVER` (between games) |
|---|---|---|---|---|
| `CONNECT` | Register, or `INVALID_NAME` / `NAME_TAKEN` / `ROOM_FULL` | `UNEXPECTED_MESSAGE` | `UNEXPECTED_MESSAGE` | `UNEXPECTED_MESSAGE` |
| `MOVE` | `UNEXPECTED_MESSAGE` | `UNEXPECTED_MESSAGE` | Validate (§5.2) | `UNEXPECTED_MESSAGE` |
| `DISCONNECT` | Close socket | Remove player; room is empty | **Forfeit** (§7) | Remove player; cancel the next game; opponent goes back to the lobby |

### 6.3 Server Send-Order Guarantees

1. `GAME_START` is always followed immediately by `STATE_UPDATE` with `move_number: 0`.
2. A game-ending move produces `STATE_UPDATE` (`current_turn: null`) and then `GAME_OVER`, in that order.
3. After a `WIN`/`DRAW` `GAME_OVER`, the next message from the server is `GAME_START`, sent `NEXT_GAME_DELAY_S` later. The only exception is when the opponent leaves first, in which case it is `LOBBY_WAIT`.
4. A forfeit produces `GAME_OVER` (`FORFEIT`) and then `LOBBY_WAIT`, sent to the remaining player.
5. Each event runs to completion before the next one is handled ([FSM §6](fsm_specification.md#6-run-to-completion-rules--race-conditions)), so both players see the same sequence of states.

---

## 7. Disconnect & Forfeit Management

### 7.1 How a Departure Is Detected

| # | Trigger | How the server sees it | `forfeit_reason` |
|---|---|---|---|
| 1 | Graceful quit | A `DISCONNECT` message arrives | `"DISCONNECT"` |
| 2 | Orderly TCP close (process exits, socket closed) | `recv()` returns `b""` (FIN) | `"CONNECTION_LOST"` |
| 3 | Abrupt TCP failure | `recv()`/`sendall()` raises `ConnectionResetError`, `BrokenPipeError`, or another `OSError` (RST) | `"CONNECTION_LOST"` |
| 4 | Fatal protocol error | Server sends `ERROR` with `fatal: true` (e.g. `FRAME_TOO_LARGE`) and closes the socket | `"CONNECTION_LOST"` |
| 5 | Silent peer (CML node powered off, link down, client hung) | `TURN_TIMEOUT_S` expires with no accepted `MOVE` from the active player. The server sends `ERROR TURN_TIMEOUT` (fatal). | `"TIMEOUT"` |

Trigger 5 is needed because TCP sends nothing when a host disappears without sending FIN or RST. Without an application-level timer, the server would wait forever for that player's move.

**Graceful vs. abrupt termination.** A *graceful* exit is an application-layer `DISCONNECT` followed by `close()`, which starts the TCP FIN 4-way teardown (trigger 1). A process that exits without sending `DISCONNECT` still produces a FIN (trigger 2). An *abrupt* termination is a crash with unread data, which produces RST (trigger 3), or a network drop such as a powered-off CML node or a cut router link, which produces no packets at all (trigger 5).

**The TCP 0-byte EOF rule.** When the peer closes cleanly, `recv()` does not raise an exception. It returns `b""`. Every receive loop must check `if not data: break`. Without that check the loop spins forever at 100% CPU, because every later `recv()` returns `b""` immediately. On EOF, any partial frame left in the buffer is discarded (§2.4 rule 7).

**Socket exceptions.** `ConnectionResetError` (RST from the peer), `BrokenPipeError` (writing to a socket whose peer has closed), `ConnectionAbortedError`, and `TimeoutError` are all caught in the receive loop and in the send helper. Each one turns into a player departure (forfeit if a game is running) instead of crashing the server:

```python
try:
    data = sock.recv(4096)
    if not data:                                  # 0-byte EOF: peer sent FIN
        handle_departure(player, "CONNECTION_LOST")
        return
    ...                                           # feed FrameReader, dispatch frames
except (ConnectionResetError, BrokenPipeError, ConnectionAbortedError, TimeoutError):
    handle_departure(player, "CONNECTION_LOST")   # RST or network drop
```

The complete, tested session loop, the socket lifecycle diagram, and the full exception table are in [FSM §5](fsm_specification.md#5-connection-termination--socket-lifecycle).

### 7.2 Server Action by State

When player **P** leaves (for any reason in §7.1) and **Q** is the opponent:

| Room state when P leaves | Messages to Q | Scores | Next room state |
|---|---|---|---|
| Socket not yet registered | none | — | unchanged |
| `WAITING_FOR_PLAYERS` (P alone in lobby) | none | — | `WAITING_FOR_PLAYERS` (empty) |
| `PLAYER_TURN`: game running, **either player's turn** | `GAME_OVER` `{result: FORFEIT, winner: Q, forfeit_reason, next_game_in_s: null}` and then `LOBBY_WAIT` | Q +1 (shown in that `GAME_OVER`) | `CLEANUP` → `WAITING_FOR_PLAYERS`, with Q as `PLAYER_1` for the next match |
| `GAME_OVER` (between games) | `LOBBY_WAIT` only, and the pending next game is canceled. This is **not** a forfeit because the game already ended. | unchanged | `CLEANUP` → `WAITING_FOR_PLAYERS`, with Q as `PLAYER_1` |
| Server shutdown (any state) | `ERROR SERVER_SHUTDOWN` (fatal) to every registered player, then close | — | terminated |

In every case the server stops sending to P, closes P's socket, and frees P's alias. When a new opponent joins, a new match starts with `game_number` 1 and scores at 0.

### 7.3 Graceful Exit Procedure

**Client quitting (e.g. user types `quit` or presses Ctrl-C):**
1. Send `DISCONNECT` (optionally with a `reason`).
2. Close the socket. Do not wait for a reply. The server sends nothing back to a client that has disconnected.

**Server ending a connection (timeout, shutdown, protocol violation):**
1. Send `ERROR` with `fatal: true` and the matching `code`, using `sendall()`.
2. Call `sock.shutdown(socket.SHUT_RDWR)`. This sends FIN after the final message, so the client can read it before it sees EOF. It also wakes the server's own blocked `recv()` on that socket, so the session loop closes the socket through its normal cleanup ([FSM §5.4](fsm_specification.md#54-socket-lifecycle-diagram)).

**Rule:** the server never closes a registered player's socket without first sending a fatal `ERROR` explaining why, unless that player has already disconnected.

### 7.4 Timeouts

| Timer | Starts | Reset / canceled by | On expiry |
|---|---|---|---|
| `TURN_TIMEOUT_S` (60 s) | When a `STATE_UPDATE` names a player in `current_turn` | Reset only by an **accepted** `MOVE`. Rejected moves do not reset it. | Server sends `ERROR TURN_TIMEOUT` (fatal) to the active player and closes their socket. The opponent gets `GAME_OVER FORFEIT (TIMEOUT)` and then `LOBBY_WAIT`. |
| `NEXT_GAME_DELAY_S` (5 s) | When a `WIN`/`DRAW` `GAME_OVER` is sent | Canceled if either player leaves | Server sends `GAME_START` (`game_number` + 1, `first_turn` switched to the other player) to each player, then `STATE_UPDATE` to both. |

`turn_timeout_s` is sent in `GAME_START`, and `next_game_in_s` in `GAME_OVER`, so clients can show countdowns. A `turn_timeout_s` of `0` turns the turn timer off (useful for debugging). A player who walks away between games is handled by the turn timer once the next game starts, so no separate idle timer is needed.

### 7.5 Concurrency & Races

Events are processed one at a time, to completion. Races such as a winning move arriving together with a disconnect, or a disconnect arriving as the next-game timer fires, are resolved in [FSM §6](fsm_specification.md#6-run-to-completion-rules--race-conditions). One protocol-level consequence: if P's process dies in the middle of a `MOVE` (some bytes received, no `\n`), the partial frame is discarded (§2.4 rule 7). Only complete frames are ever acted on.

### 7.6 Client-Side Handling of Server Loss

If the client's `recv()` returns `b""` or raises `ConnectionResetError`, the client tells the user "Connection to server lost" and exits cleanly. If a fatal `ERROR` came first, it shows that message instead. It does not attempt any game logic locally. Clients need no timers of their own.

---

## 8. Wire Examples

### 8.1 Notation

- In the wire streams below, `\n` stands for the **single byte `0x0A`**, not the two characters `\` and `n`.
- `C1` = Alice's client, `C2` = Bob's client, `S` = server. `S → C1,C2` means the same bytes are sent to both sockets.
- Transcripts list each frame on its own line for readability. The leading `C1 → S` label is **not** on the wire, and each line is followed by `0x0A`.

### 8.2 Continuous Stream (Client → Server)

Two messages written to one socket, exactly as they appear on the wire:

```
{"msg_type":"CONNECT","player_id":"Alice","timestamp":1727000000}\n{"msg_type":"MOVE","player_id":"Alice","payload":{"row":0,"col":2},"timestamp":1727000005}\n
```

Frame 1 is 66 bytes (65 bytes of JSON + `\n`). Frame 2 is 91 bytes (90 bytes of JSON + `\n`). The stream is 157 bytes in total.

### 8.3 Byte-Level View (`xxd`)

The same 157 bytes. The two `0a` bytes, at offsets `0x41` and `0x9c`, are the only frame boundaries:

```
00000000: 7b22 6d73 675f 7479 7065 223a 2243 4f4e  {"msg_type":"CON
00000010: 4e45 4354 222c 2270 6c61 7965 725f 6964  NECT","player_id
00000020: 223a 2241 6c69 6365 222c 2274 696d 6573  ":"Alice","times
00000030: 7461 6d70 223a 3137 3237 3030 3030 3030  tamp":1727000000
00000040: 7d0a 7b22 6d73 675f 7479 7065 223a 224d  }.{"msg_type":"M
            ^^ 0x41 = 0x0A  ← end of frame 1
00000050: 4f56 4522 2c22 706c 6179 6572 5f69 6422  OVE","player_id"
00000060: 3a22 416c 6963 6522 2c22 7061 796c 6f61  :"Alice","payloa
00000070: 6422 3a7b 2272 6f77 223a 302c 2263 6f6c  d":{"row":0,"col
00000080: 223a 327d 2c22 7469 6d65 7374 616d 7022  ":2},"timestamp"
00000090: 3a31 3732 3730 3030 3030 357d 0a         :1727000005}.
                                        ^^ 0x9c = 0x0A  ← end of frame 2
```

### 8.4 Coalescing & Fragmentation: How the Receiver Buffer Evolves

The same 157-byte stream can reach the server split up in different ways. The `FrameReader` (§2.6) produces the same two messages every time.

**Case A: coalescing (both frames arrive in one `recv()`)**

| `recv()` | Bytes received | Frames extracted | Buffer after |
|---|---|---|---|
| #1 | all 157 bytes | `CONNECT`, `MOVE` | *(empty)* |

**Case B: fragmentation (one frame split across two `recv()` calls)**

| `recv()` | Bytes received | Frames extracted | Buffer after |
|---|---|---|---|
| #1 | `{"msg_type":"CONNECT","player_id":"Alice` (40 B) | none (no `\n` yet) | `{"msg_type":"CONNECT","player_id":"Alice` |
| #2 | `","timestamp":1727000000}\n{"msg_type":"MOVE",…}\n` (117 B) | `CONNECT`, `MOVE` | *(empty)* |

**Case C: coalescing and fragmentation together (one and a half frames)**

| `recv()` | Bytes received | Frames extracted | Buffer after |
|---|---|---|---|
| #1 | `{"msg_type":"CONNECT",…,"timestamp":1727000000}\n{"msg_type":"MOVE","player_id"` (96 B) | `CONNECT` | `{"msg_type":"MOVE","player_id"` |
| #2 | `:"Alice","payload":{"row":0,"col":2},"timestamp":1727000005}\n` (61 B) | `MOVE` | *(empty)* |

**Case D: maximum fragmentation (one byte per `recv()`)**: 157 calls. Calls 1–65 and 67–156 extract nothing. Call 66 extracts `CONNECT` and call 157 extracts `MOVE`.

### 8.5 Full Session Transcript

Alice connects and becomes Player 1, and Bob joins as Player 2. Alice wins game 1 on the diagonal. Five seconds later the server starts game 2 automatically, with Bob moving first.

```
C1 → S     {"msg_type":"CONNECT","player_id":"Alice","timestamp":1727000000}
S → C1     {"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"players_required":2,"message":"Waiting for an opponent..."},"timestamp":1727000000}
C2 → S     {"msg_type":"CONNECT","player_id":"Bob","timestamp":1727000010}
S → C1     {"msg_type":"GAME_START","player_id":"SERVER","payload":{"game_number":1,"your_role":"PLAYER_1","players":{"PLAYER_1":{"player_id":"Alice","symbol":"X"},"PLAYER_2":{"player_id":"Bob","symbol":"O"}},"first_turn":"Alice","turn_timeout_s":60},"timestamp":1727000010}
S → C2     {"msg_type":"GAME_START","player_id":"SERVER","payload":{"game_number":1,"your_role":"PLAYER_2","players":{"PLAYER_1":{"player_id":"Alice","symbol":"X"},"PLAYER_2":{"player_id":"Bob","symbol":"O"}},"first_turn":"Alice","turn_timeout_s":60},"timestamp":1727000010}
S → C1,C2  {"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["-","-","-"],["-","-","-"],["-","-","-"]],"current_turn":"Alice","move_number":0,"last_move":null,"scores":{"Alice":0,"Bob":0},"draws":0},"timestamp":1727000010}
C1 → S     {"msg_type":"MOVE","player_id":"Alice","payload":{"row":1,"col":1},"timestamp":1727000015}
S → C1,C2  {"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["-","-","-"],["-","X","-"],["-","-","-"]],"current_turn":"Bob","move_number":1,"last_move":{"player_id":"Alice","symbol":"X","row":1,"col":1},"scores":{"Alice":0,"Bob":0},"draws":0},"timestamp":1727000015}
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":0,"col":1},"timestamp":1727000020}
S → C1,C2  {"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["-","O","-"],["-","X","-"],["-","-","-"]],"current_turn":"Alice","move_number":2,"last_move":{"player_id":"Bob","symbol":"O","row":0,"col":1},"scores":{"Alice":0,"Bob":0},"draws":0},"timestamp":1727000020}
C1 → S     {"msg_type":"MOVE","player_id":"Alice","payload":{"row":0,"col":0},"timestamp":1727000025}
S → C1,C2  {"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["X","O","-"],["-","X","-"],["-","-","-"]],"current_turn":"Bob","move_number":3,"last_move":{"player_id":"Alice","symbol":"X","row":0,"col":0},"scores":{"Alice":0,"Bob":0},"draws":0},"timestamp":1727000025}
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":0,"col":2},"timestamp":1727000030}
S → C1,C2  {"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["X","O","O"],["-","X","-"],["-","-","-"]],"current_turn":"Alice","move_number":4,"last_move":{"player_id":"Bob","symbol":"O","row":0,"col":2},"scores":{"Alice":0,"Bob":0},"draws":0},"timestamp":1727000030}
C1 → S     {"msg_type":"MOVE","player_id":"Alice","payload":{"row":2,"col":2},"timestamp":1727000035}
S → C1,C2  {"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["X","O","O"],["-","X","-"],["-","-","X"]],"current_turn":null,"move_number":5,"last_move":{"player_id":"Alice","symbol":"X","row":2,"col":2},"scores":{"Alice":1,"Bob":0},"draws":0},"timestamp":1727000035}
S → C1,C2  {"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"result":"WIN","winner":"Alice","winning_line":[[0,0],[1,1],[2,2]],"forfeit_reason":null,"scores":{"Alice":1,"Bob":0},"draws":0,"next_game_in_s":5},"timestamp":1727000035}
           [5 s pause: NEXT_GAME_DELAY_S]
S → C1     {"msg_type":"GAME_START","player_id":"SERVER","payload":{"game_number":2,"your_role":"PLAYER_1","players":{"PLAYER_1":{"player_id":"Alice","symbol":"X"},"PLAYER_2":{"player_id":"Bob","symbol":"O"}},"first_turn":"Bob","turn_timeout_s":60},"timestamp":1727000040}
S → C2     {"msg_type":"GAME_START","player_id":"SERVER","payload":{"game_number":2,"your_role":"PLAYER_2","players":{"PLAYER_1":{"player_id":"Alice","symbol":"X"},"PLAYER_2":{"player_id":"Bob","symbol":"O"}},"first_turn":"Bob","turn_timeout_s":60},"timestamp":1727000040}
S → C1,C2  {"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["-","-","-"],["-","-","-"],["-","-","-"]],"current_turn":"Bob","move_number":0,"last_move":null,"scores":{"Alice":1,"Bob":0},"draws":0},"timestamp":1727000040}
```

**What Alice's socket actually receives** between `t=1727000000` and `t=1727000010`, as one continuous server→client stream. `GAME_START` and `STATE_UPDATE` are sent back-to-back, so they often arrive together in a single `recv()`:

```
{"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"players_required":2,"message":"Waiting for an opponent..."},"timestamp":1727000000}\n{"msg_type":"GAME_START","player_id":"SERVER","payload":{"game_number":1,"your_role":"PLAYER_1","players":{"PLAYER_1":{"player_id":"Alice","symbol":"X"},"PLAYER_2":{"player_id":"Bob","symbol":"O"}},"first_turn":"Alice","turn_timeout_s":60},"timestamp":1727000010}\n{"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"board":[["-","-","-"],["-","-","-"],["-","-","-"]],"current_turn":"Alice","move_number":0,"last_move":null,"scores":{"Alice":0,"Bob":0},"draws":0},"timestamp":1727000010}\n
```

### 8.6 Error Exchanges

**Alias collision at `CONNECT` (not fatal, client retries):**

```
C2 → S     {"msg_type":"CONNECT","player_id":"Alice","timestamp":1727000008}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"NAME_TAKEN","message":"player_id 'Alice' is already in use.","ref_msg_type":"CONNECT","fatal":false},"timestamp":1727000008}
C2 → S     {"msg_type":"CONNECT","player_id":"Bob","timestamp":1727000010}
```

**Newline in an alias, done correctly (`json.dumps`).** A client that skipped the input check sends the alias `Al⏎ice`. The serializer escapes the newline, so framing holds. In this line `\n` is the **two-byte escape `0x5C 0x6E`**, not a delimiter, so this is still **one** frame, and the server rejects the alias:

```
C2 → S     {"msg_type":"CONNECT","player_id":"Al\nice","timestamp":1727000008}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"INVALID_NAME","message":"Alias must be 1-16 letters, digits, _ or -.","ref_msg_type":"CONNECT","fatal":false},"timestamp":1727000008}
```

**Newline in an alias, done wrong (string concatenation, which this protocol forbids).** The raw `0x0A` from the user's input reaches the wire and splits one message into two broken frames:

```
C2 → S     {"msg_type":"CONNECT","player_id":"Al                ← frame 1 ends at the raw 0x0A
C2 → S     ice","timestamp":1727000008}                         ← frame 2
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"MALFORMED_JSON","message":"Line is not a valid JSON object.","ref_msg_type":null,"fatal":false},"timestamp":1727000008}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"MALFORMED_JSON","message":"Line is not a valid JSON object.","ref_msg_type":null,"fatal":false},"timestamp":1727000008}
```

**Out-of-turn move** (it is Alice's turn, `move_number` 0):

```
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":0,"col":0},"timestamp":1727000012}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"NOT_YOUR_TURN","message":"It is Alice's turn.","ref_msg_type":"MOVE","fatal":false},"timestamp":1727000012}
```

**Invalid coordinates and wrong field type** (Bob's turn, center already taken):

```
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":1,"col":1},"timestamp":1727000017}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"CELL_OCCUPIED","message":"Cell (1,1) is already taken.","ref_msg_type":"MOVE","fatal":false},"timestamp":1727000017}
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":3,"col":0},"timestamp":1727000018}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"OUT_OF_BOUNDS","message":"row and col must be 0, 1, or 2.","ref_msg_type":"MOVE","fatal":false},"timestamp":1727000018}
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":"0","col":1},"timestamp":1727000019}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"INVALID_FIELD","message":"payload.row must be an integer.","ref_msg_type":"MOVE","fatal":false},"timestamp":1727000019}
```

**Impersonation attempt** (Bob's socket claims to be Alice):

```
C2 → S     {"msg_type":"MOVE","player_id":"Alice","payload":{"row":2,"col":0},"timestamp":1727000019}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"PLAYER_ID_MISMATCH","message":"This connection is registered as 'Bob'.","ref_msg_type":"MOVE","fatal":false},"timestamp":1727000019}
```

**Malformed message.** The line is terminated, so framing stays in sync and the next frame is processed normally:

```
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":0,
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"MALFORMED_JSON","message":"Line is not a valid JSON object.","ref_msg_type":null,"fatal":false},"timestamp":1727000019}
C2 → S     {"msg_type":"MOVE","player_id":"Bob","payload":{"row":0,"col":1},"timestamp":1727000020}
```

**Third client while a game is running (fatal):**

```
C3 → S     {"msg_type":"CONNECT","player_id":"Carol","timestamp":1727000022}
S → C3     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"ROOM_FULL","message":"A game is already in progress.","ref_msg_type":"CONNECT","fatal":true},"timestamp":1727000022}
           [server: shutdown(SHUT_WR), close C3 socket]
```

**Oversized frame (fatal).** The client sends 4096+ bytes with no `\n`:

```
C2 → S     xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx… (4096 bytes, no 0x0A)
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"FRAME_TOO_LARGE","message":"Frame exceeds 4096 bytes.","ref_msg_type":null,"fatal":true},"timestamp":1727000024}
           [server: close C2 socket. If a game is running, Alice wins by FORFEIT (CONNECTION_LOST), see §8.7]
```

### 8.7 Disconnect & Forfeit Exchanges

All scenarios below happen during **game 1** of the Alice-vs-Bob match (scores 0–0 before the forfeit).

**A. Graceful quit in the middle of a game (`forfeit_reason: DISCONNECT`):**

```
C2 → S     {"msg_type":"DISCONNECT","player_id":"Bob","payload":{"reason":"Player quit"},"timestamp":1727000032}
           [C2 closes its socket; server closes C2 socket and sends nothing back to C2]
S → C1     {"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"result":"FORFEIT","winner":"Alice","winning_line":null,"forfeit_reason":"DISCONNECT","scores":{"Alice":1,"Bob":0},"draws":0,"next_game_in_s":null},"timestamp":1727000032}
S → C1     {"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"players_required":2,"message":"Bob left the match. Waiting for a new opponent..."},"timestamp":1727000032}
```

**B. Abrupt connection loss (`forfeit_reason: CONNECTION_LOST`).** It is Bob's turn (after Alice's move 3), and Bob's client crashes in the middle of sending a `MOVE`:

```
C2 → S     {"msg_type":"MOVE","player_id":"Bob","pay          ← partial frame, no 0x0A
           [TCP FIN or RST from C2: server recv() returns b"" or raises ConnectionResetError]
           [server discards the 41 buffered bytes, which are never parsed]
S → C1     {"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"result":"FORFEIT","winner":"Alice","winning_line":null,"forfeit_reason":"CONNECTION_LOST","scores":{"Alice":1,"Bob":0},"draws":0,"next_game_in_s":null},"timestamp":1727000027}
S → C1     {"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"players_required":2,"message":"Bob left the match. Waiting for a new opponent..."},"timestamp":1727000027}
```

**C. Turn timeout (`forfeit_reason: TIMEOUT`).** Bob's turn started at `1727000025` (after Alice's move 3), and Bob's CML node goes silent:

```
           [t=1727000025 … 1727000085: no bytes from C2]
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"TURN_TIMEOUT","message":"No move received within 60 s. You forfeit.","ref_msg_type":null,"fatal":true},"timestamp":1727000085}
           [server: shutdown(SHUT_WR), close C2 socket]
S → C1     {"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"result":"FORFEIT","winner":"Alice","winning_line":null,"forfeit_reason":"TIMEOUT","scores":{"Alice":1,"Bob":0},"draws":0,"next_game_in_s":null},"timestamp":1727000085}
S → C1     {"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"players_required":2,"message":"Bob left the match. Waiting for a new opponent..."},"timestamp":1727000085}
```

**D. Quitting between games (not a forfeit).** Game 1 ended with Alice's win at `1727000035`, and game 2 was due to start at `1727000040`:

```
C2 → S     {"msg_type":"DISCONNECT","player_id":"Bob","payload":{"reason":"Done playing"},"timestamp":1727000038}
           [server cancels the next-game timer and closes C2 socket]
S → C1     {"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"players_required":2,"message":"Bob left the match. Waiting for a new opponent..."},"timestamp":1727000038}
```

No `GAME_OVER` is sent and the scores do not change, because game 1 had already ended and game 2 never started.

**E. Server shutdown:**

```
S → C1     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"SERVER_SHUTDOWN","message":"Server is shutting down.","ref_msg_type":null,"fatal":true},"timestamp":1727000099}
S → C2     {"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"SERVER_SHUTDOWN","message":"Server is shutting down.","ref_msg_type":null,"fatal":true},"timestamp":1727000099}
           [server closes both sockets and the listening socket]
```

---

## 9. AI Prompting & Constraint Strategy (summary)

How AI coding tools are prompted and constrained to implement this blueprint exactly is documented in **[`ai_prompts.md`](ai_prompts.md)**. It contains:

- the **system prompt**, including a schema card of every message's exact fields, types, and key order;
- **task prompts** with fixed function signatures for the parser and serialization functions (`protocol.py`), tests, server, and client;
- a correction prompt, a review checklist, and acceptance tests built from the wire examples in §4 and §8;
- a prompt log.

---

## 10. Revision History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-04 | Initial blueprint: newline-delimited JSON framing, message catalog, forfeit management, wire examples. |
| 1.1 | 2026-10-04 | Aligned the message set with the required eight message types. `GAME_START` now assigns roles (`PLAYER_1` / `PLAYER_2`). `STATE_UPDATE` now carries `scores` and `draws`. `DISCONNECT` is Client → Server only; server-initiated closes use fatal `ERROR` codes (`TURN_TIMEOUT`, `SERVER_SHUTDOWN`). `REMATCH` was removed: the next game starts automatically after `NEXT_GAME_DELAY_S`, and `GAME_OVER.rematch_allowed` was replaced by `next_game_in_s`. |
| 1.2 | 2026-10-04 | Mapped the document to the Sprint 1 instructions with a requirements traceability table. §2 renamed to *Transport Layer & Packet Framing Mechanism*, with a transport/serialization summary, the reasons for choosing Option A, and explicit coalescing and fragmentation handling. **Wire change:** empty board cells are now `"-"` instead of `""`. Added newline-in-alias safeguards (§2.5) with wire examples (§8.6), the FSM transition table (§6.2), and AI implementation constraints (§9). |
| 1.2 | 2026-10-04 | Documentation split into the three Sprint 1 deliverables (protocol unchanged). FSM diagrams, transition table, and termination handling moved to `fsm_specification.md`; §6 is now a summary that keeps the client-observable rules. AI strategy moved to `ai_prompts.md`; §9 is now a summary. §7.3 now uses `shutdown(SHUT_RDWR)`. |
