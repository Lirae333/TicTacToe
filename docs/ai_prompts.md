# AI Prompting & Constraint Strategy

| | |
|---|---|
| **Project** | CS 457 Networked Tic-Tac-Toe |
| **Author** | Evan Lira |
| **Version** | 1.0 (targets protocol version 1.2) |
| **Last updated** | 2026-10-04 |
| **Companion documents** | [`protocol_blueprint.md`](protocol_blueprint.md) · [`fsm_specification.md`](fsm_specification.md) |

AI coding tools (e.g. Claude, ChatGPT, GitHub Copilot) left unconstrained produce generic socket boilerplate: an echo server, `recv(1024)` treated as one message, ad-hoc strings. That does not implement this protocol. This document records the **system prompt** and **task prompts** used to force AI tools to generate code, above all the **parser and serialization functions**, that matches the exact schema in the protocol blueprint. It also defines how AI output is checked before it is accepted.

Section references written as **B§n** point to [`protocol_blueprint.md`](protocol_blueprint.md), and **F§n** to [`fsm_specification.md`](fsm_specification.md).

---

## Contents

1. [Strategy](#1-strategy)
2. [System Prompt](#2-system-prompt)
3. [Task Prompts](#3-task-prompts)
4. [Correction Prompt](#4-correction-prompt)
5. [Review Checklist](#5-review-checklist)
6. [Acceptance Tests](#6-acceptance-tests)
7. [Prompt Log](#7-prompt-log)
8. [Revision History](#8-revision-history)

---

## 1. Strategy

1. **The specs are the source of truth.** The protocol blueprint and the FSM specification are attached to every session. If AI output disagrees with them, the code is fixed, not the spec. A protocol change is made in the blueprint first, with a revision-history entry, before any code changes.
2. **Two prompt layers.** A fixed **system prompt** (§2) carries the rules that never change, plus a **schema card**: a compact copy of every message's exact field names, types, and allowed values. The model therefore cannot invent or rename fields. A short **task prompt** (§3) then asks for one module at a time.
3. **I fix the function signatures; the AI writes the bodies.** Each task prompt names the exact functions, parameters, and return types to write. This stops the model from inventing its own architecture, which is where generic boilerplate creeps in.
4. **Small modules that map to spec sections.**

   | Module | Prompt | Implements |
   |---|---|---|
   | `protocol.py`: serialization (message builders) | §3.1 | B§3.1, B§4 |
   | `protocol.py`: parsing and validation | §3.2 | B§2.4, B§3.2, B§3.4, B§5 |
   | `framing.py` | *(not generated)* | Copied verbatim from B§2.6 |
   | `tests/test_protocol.py` | §3.3 | §6 (this document) |
   | `server.py`: game engine and sessions | §3.4 | F§1–F§6, B§7 |
   | `client.py` | §3.5 | B§4 "Client behavior", B§2.5, B§7.3, B§7.6 |

5. **Tests decide acceptance.** Generated code is accepted only after it passes the acceptance tests in §6. Those tests are built from the wire examples already in the blueprint (B§4, B§8), so "matches the schema" means "produces the same bytes".
6. **Ask, don't guess.** Every prompt tells the model to stop and ask when the spec does not cover a case, instead of guessing.
7. **Every correction is logged** in §7 as evidence of the process.

---

## 2. System Prompt

Used as the system message, custom instructions, or project instructions in whichever tool is used. (For GitHub Copilot, the same text goes in `.github/copilot-instructions.md`.) Both spec files are attached alongside it.

```text
ROLE
You are a code generator for a CS 457 networked Tic-Tac-Toe project. You implement an
existing, fixed application-layer protocol. You do not design or change protocols.

SOURCES OF TRUTH (attached)
- protocol_blueprint.md : framing, envelope, message schemas, error codes, wire examples
- fsm_specification.md  : server state machine (states, transitions T1-T21), disconnects
If your code and these documents disagree, the documents are right and your code is wrong.

NON-NEGOTIABLE RULES
1. Python 3 standard library only.
2. Wire format: one compact JSON object per line, UTF-8, terminated by exactly one b"\n".
   Serialize ONLY with json.dumps(obj, separators=(",", ":")).
   NEVER use indent=, f-strings, %-formatting, or "+" concatenation to build JSON.
3. Framing: one recv() is NEVER one message. Read through FrameReader (blueprint §2.6),
   which handles coalescing and fragmentation. recv() returning b"" means EOF: stop reading.
4. Exactly eight message types exist. Use the SCHEMA CARD below for every field name, type,
   allowed value, and key order. Do not add, rename, reorder, or drop fields. Do not invent
   message types, error codes, or states.
5. Validation: integers with type(v) is int (bool is rejected); strings with type(v) is str.
   Unknown extra fields are ignored, never rejected. A missing payload is treated as {}.
6. Errors use only the codes in the SCHEMA CARD, with the fatal flag listed there.
7. Identity comes from the socket a message arrived on, never from its player_id field.
8. Put a comment above every function naming the spec section it implements,
   e.g.  # blueprint §5.2  or  # fsm T9.
9. If anything is unspecified or ambiguous, STOP and ask me. Do not guess.

SCHEMA CARD (blueprint §3 to blueprint §5)
Envelope, key order: msg_type:str, player_id:str, payload:object, timestamp:int
  player_id = sender's alias (^[A-Za-z0-9_-]{1,16}$, never "SERVER") or "SERVER" from the server
  payload   = omitted for CONNECT, and for DISCONNECT without a reason
  timestamp = int(time.time())
Types: Cell = "X" | "O" | "-"      Board = list of 3 rows, each a list of 3 Cells
       Role = "PLAYER_1" | "PLAYER_2"   Coord = [row, col]   Scores = {alias: int}

CONNECT       C->S  (no payload; the alias is the envelope player_id)
LOBBY_WAIT    S->C  players_connected:int (=1), players_required:int (=2), message:str (<=128)
GAME_START    S->C  game_number:int (>=1), your_role:Role,
                    players:{"PLAYER_1":{"player_id":str,"symbol":"X"},
                             "PLAYER_2":{"player_id":str,"symbol":"O"}},
                    first_turn:str, turn_timeout_s:int (>=0)
MOVE          C->S  row:int (0-2), col:int (0-2)
STATE_UPDATE  S->C  board:Board, current_turn:str|null, move_number:int (0-9),
                    last_move:{"player_id":str,"symbol":"X"|"O","row":int,"col":int}|null,
                    scores:Scores, draws:int
ERROR         S->C  code:str, message:str (<=128), ref_msg_type:str|null, fatal:bool
DISCONNECT    C->S  reason:str (<=128, optional)
GAME_OVER     S->C  result:"WIN"|"DRAW"|"FORFEIT", winner:str|null,
                    winning_line:[Coord,Coord,Coord]|null,
                    forfeit_reason:"DISCONNECT"|"CONNECTION_LOST"|"TIMEOUT"|null,
                    scores:Scores, draws:int, next_game_in_s:int|null

ERROR codes (* = fatal): NOT_YOUR_TURN, OUT_OF_BOUNDS, CELL_OCCUPIED, MALFORMED_JSON,
  INVALID_FIELD, UNKNOWN_MSG_TYPE, FRAME_TOO_LARGE*, PLAYER_ID_MISMATCH, INVALID_NAME,
  NAME_TAKEN, ROOM_FULL*, UNEXPECTED_MESSAGE, TURN_TIMEOUT*, SERVER_SHUTDOWN*
```

---

## 3. Task Prompts

Each task prompt is sent after the system prompt, one module per conversation.

### 3.1 Serialization: Message Builders

```text
TASK: Write the serialization half of protocol.py.
IMPLEMENTS: blueprint §3.1 (envelope) and blueprint §4.1 to §4.8 (one builder per
            message type).

Write exactly these functions with exactly these signatures. Every builder also takes a
keyword-only argument  ts: int | None = None  (None means int(time.time())) so tests can
pin the timestamp.

  def make_envelope(msg_type: str, player_id: str, payload: dict | None, *, ts=None) -> dict
  def build_connect(player_id: str, *, ts=None) -> dict
  def build_lobby_wait(message: str, *, ts=None) -> dict
  def build_game_start(game_number: int, your_role: str, player_1: str, player_2: str,
                       first_turn: str, turn_timeout_s: int, *, ts=None) -> dict
  def build_move(player_id: str, row: int, col: int, *, ts=None) -> dict
  def build_state_update(board: list, current_turn: str | None, move_number: int,
                         last_move: dict | None, scores: dict, draws: int, *, ts=None) -> dict
  def build_error(code: str, message: str, ref_msg_type: str | None, fatal: bool,
                  *, ts=None) -> dict
  def build_disconnect(player_id: str, reason: str | None = None, *, ts=None) -> dict
  def build_game_over(result: str, winner: str | None, winning_line: list | None,
                      forfeit_reason: str | None, scores: dict, draws: int,
                      next_game_in_s: int | None, *, ts=None) -> dict

Rules:
- Each builder returns a plain dict whose keys appear in exactly the SCHEMA CARD order,
  so encode_frame() output is byte-identical to the "Wire form" examples in blueprint §4.
- Server-side builders always use player_id "SERVER". LOBBY_WAIT always has
  players_connected=1 and players_required=2. GAME_START always maps PLAYER_1 to "X" and
  PLAYER_2 to "O".
- build_connect never includes "payload". build_disconnect omits "payload" when reason is None.
- Builders only build dicts: no sockets, no sending, no printing, no validation.
- Do not write encode_frame; import it from framing.py (blueprint §2.6).
- After the code, show encode_frame(builder(...)) for the example values in each blueprint §4
  subsection, so I can diff it against the "Wire form" lines.
```

### 3.2 Parsing & Validation

```text
TASK: Write the parsing and validation half of protocol.py.
IMPLEMENTS: blueprint §2.4 rule 5, blueprint §3.2 (data types), blueprint §3.4
            (general rules), blueprint §5.1 and §5.2.

Write exactly:

  class ProtocolError(Exception):
      def __init__(self, code: str, message: str,
                   ref_msg_type: str | None = None, fatal: bool = False): ...

  def parse_frame(line: bytes) -> dict
      # UTF-8 decode + json.loads. Anything that fails, or is not a dict,
      # raises ProtocolError("MALFORMED_JSON", ...).

  def validate_envelope(msg: dict, registered_alias: str | None) -> str
      # Server side, blueprint §5.2 steps 2-4. Returns msg_type.

  def validate_payload(msg: dict) -> dict
      # Server side, blueprint §5.2 step 6. Returns the payload dict ({} if missing).

  def validate_server_message(msg: dict) -> dict
      # Client side: checks LOBBY_WAIT, GAME_START, STATE_UPDATE, ERROR, GAME_OVER
      # against the SCHEMA CARD. Returns the payload dict.

Rules:
- The server calls these in this order, with its own state check in between:
  parse_frame -> validate_envelope -> (FSM: UNEXPECTED_MESSAGE check, blueprint §5.2
  step 5) -> validate_payload -> (FSM: turn, bounds, cell checks, blueprint §5.2 steps 7-9).
  So these functions must NOT check state, turn, bounds, or board contents.
- validate_envelope:
  * msg_type must be CONNECT, MOVE, or DISCONNECT; anything else (including the five
    server->client types) raises UNKNOWN_MSG_TYPE.
  * msg_type and player_id must be str, timestamp must be int, payload (if present) must be
    a dict; otherwise INVALID_FIELD.
  * For CONNECT: player_id must match ^[A-Za-z0-9_-]{1,16}$ and must not equal "SERVER"
    in any letter case; otherwise INVALID_NAME.
  * For MOVE and DISCONNECT: player_id must equal registered_alias; otherwise
    PLAYER_ID_MISMATCH.
- validate_payload:
  * MOVE: row and col present and type(v) is int; otherwise INVALID_FIELD.
    (A value outside 0-2 is NOT an error here; the FSM reports OUT_OF_BOUNDS.)
  * DISCONNECT: reason, if present, must be a str of at most 128 chars; otherwise INVALID_FIELD.
  * CONNECT: no payload fields.
- Every ProtocolError carries ref_msg_type = msg["msg_type"] when it is a str, else None,
  and fatal=False (the fatal codes are raised by framing and the FSM, not here).
- Unknown extra fields are ignored everywhere.
- Never use isinstance(v, int) for integers.
```

### 3.3 Acceptance Tests

```text
TASK: Write tests/test_protocol.py with unittest (standard library only).
IMPLEMENTS: ai_prompts.md §6.

Copy every expected byte string from the blueprint exactly as written there. Do not
regenerate expected values by calling the code under test.

1. Framing: feed the 157-byte stream from blueprint §8.2 into FrameReader using the splits
   of blueprint §8.4: all at once; 40 + 117 bytes; 96 + 61 bytes; one byte at a time.
   Each must yield exactly two frames that parse to CONNECT then MOVE.
2. Oversize: 4096 bytes with no b"\n" raises FrameTooLarge.
3. Encoding: for every "Wire form" in blueprint §4, build the same message with the
   protocol.py builder (pin ts to the example's timestamp) and assert
   encode_frame(msg) == the wire form bytes.
4. Alias safety: encode_frame(build_connect("Al\nice")) contains exactly one 0x0A byte (the
   last one), and validate_envelope(parse_frame(...), None) raises INVALID_NAME.
5. Parser errors from blueprint §8.6: the truncated MOVE line -> MALFORMED_JSON;
   {"row":"0"} -> INVALID_FIELD; Bob's socket sending player_id "Alice" -> PLAYER_ID_MISMATCH;
   a STATE_UPDATE sent by a client -> UNKNOWN_MSG_TYPE.
```

### 3.4 Server Engine

```text
TASK: Write server.py: the game engine and client sessions.
IMPLEMENTS: fsm_specification.md §1 to §6 (states, transitions T1-T21, session loop
            in fsm §5.5)
            and blueprint §7 (disconnect and forfeit management).

Rules:
- States are an Enum with exactly the eight states of fsm §1.1. Room data is exactly fsm §1.3.
- Room data changes ONLY inside transitions T1-T21. Each transition is one method or branch,
  commented with its T-number. No other transitions exist.
- Every connection runs client_session() from fsm §5.5 unchanged.
- Concurrency model: <FILL IN FROM SPRINT 2: threading + one threading.Lock, or selectors>.
  Either way, each event runs to completion before the next one (fsm §6).
- Timers only post TURN_TIMEOUT / NEXT_GAME_TIMER events; they never change room data.
- All sending goes through one send helper that catches OSError and queues
  CLIENT_DISCONNECTED for that player (fsm §5.3). Server-initiated closes send a fatal ERROR,
  then sock.shutdown(socket.SHUT_RDWR) (fsm §5.4).
- Everything on the wire goes through protocol.py builders/validators and framing.py.
  No json.dumps calls anywhere else in this file.
```

### 3.5 Client

```text
TASK: Write client.py.
IMPLEMENTS: blueprint §4 "Client behavior" paragraphs, blueprint §2.5 (alias check),
            blueprint §7.3 and blueprint §7.6.

Rules:
- Usage: python3 client.py [host] [port]   (defaults: server.lira.edu 5457).
- Ask for an alias and check it against ^[A-Za-z0-9_-]{1,16}$ before sending CONNECT.
  On failure, ask again without sending anything.
- One thread reads frames (FrameReader + EOF rule) and renders LOBBY_WAIT, GAME_START,
  STATE_UPDATE, ERROR, and GAME_OVER using validate_server_message. The main thread reads input.
- The board shown is ALWAYS the board from the latest STATE_UPDATE. The client never
  changes its own board.
- Input "row col" sends MOVE (only when current_turn is my alias); "quit" sends DISCONNECT,
  then closes the socket without waiting for a reply.
- On EOF, ConnectionResetError, or an ERROR with fatal=true: print the reason and exit cleanly.
```

---

## 4. Correction Prompt

When generated code fails a checklist item (§5) or a test (§6), it is sent back with the violated rule quoted. The model is not asked to "fix bugs" in general:

```text
Your code violates the spec. Change only what is needed to fix this.

VIOLATION: <what the code does, with the function name and line>
SPEC:      <file and section> says: "<exact quote>"
FIX:       Rewrite only <function> so it follows that section. Show the diff,
           then show the failing test from ai_prompts.md §6 passing.
```

---

## 5. Review Checklist

Generated code is rejected and corrected with the §4 prompt if it does any of the following:

| Reject if the code... | Violates |
|---|---|
| Treats one `recv()` result as one message, or calls `json.loads` on raw `recv()` data | B§2.2, B§2.4 |
| Ignores `recv()` returning `b""` (loops forever on EOF) | F§5.2 |
| Builds JSON with `indent=`, f-strings, or string concatenation | B§2.3, B§2.5 |
| Emits payload keys in a different order than the schema card, or with different names | B§4 |
| Sends or accepts a `msg_type`, field, or error code not in the schema card | B§4, B§5.1 |
| Uses `isinstance(v, int)` to validate integer fields | B§3.2 |
| Rejects messages because of unknown extra fields | B§3.4 |
| Trusts the `player_id` field instead of the socket's registered identity | B§3.4 |
| Skips or reorders validation steps, or returns a different error code | B§5.2 |
| Sends `DISCONNECT` from the server, or broadcasts an `ERROR` | B§4.6, B§4.7 |
| Adds, skips, or changes a state transition | F§2.2 |
| Lets a socket exception escape a session loop (crashes the server) | F§5.3 |
| Closes a socket from a thread other than its session loop | F§5.4 |
| Accepts an alias without checking `PLAYER_ID_PATTERN` | B§2.5, B§3.2 |
| Lets a client change its own board instead of rendering `STATE_UPDATE` | B§1 |

---

## 6. Acceptance Tests

The examples in the blueprint double as test vectors. Prompt §3.3 asks the AI to write these tests, and the expected values are copied from the blueprint, never generated by the code under test.

| Test | Input | Pass condition |
|---|---|---|
| Framing | The B§8.2 stream fed to `FrameReader` in the splits of B§8.4 cases A–D | Exactly two frames, `CONNECT` then `MOVE`, in every case |
| Oversize | 4096 bytes with no `\n` | `FrameTooLarge` raised |
| Encoding | Every "Wire form" in B§4, rebuilt with the `protocol.py` builders | `encode_frame()` output is byte-identical |
| Alias safety | `build_connect("Al\nice")` | One `0x0A` byte per frame, and the server-side validator raises `INVALID_NAME` (B§8.6) |
| Parser errors | The parser-level exchanges in B§8.6 | Same `code` values |
| Full session | Client lines of B§8.5 replayed by two scripted clients, with game 1's `first_turn` fixed to Alice | Server lines match B§8.5 exactly, except `timestamp` values |
| Game errors | `NOT_YOUR_TURN`, `OUT_OF_BOUNDS`, `CELL_OCCUPIED` exchanges in B§8.6 | Same `code` and `fatal` values; room state unchanged |
| Disconnects | Scenarios A–E in B§8.7 | Same messages to the remaining player; departing socket closed; server keeps running |

---

## 7. Prompt Log

Each time a prompt from §3 is used, record it here. Note what the AI got wrong and the correction prompt (§4) that fixed it. This log is the evidence that AI output was constrained to the blueprint rather than accepted as-is.

**Status (Sprint 1):** Sprint 1 is design-only, so no application code has been generated yet. The prompts above are ready, and entries start when implementation begins in Sprint 3.

| # | Date | Tool / model | Prompt | Result | Spec violations found | Correction (§4) |
|---|---|---|---|---|---|---|
| 1 | | | §3.1 Serialization | | | |
| 2 | | | §3.2 Parsing & validation | | | |
| 3 | | | §3.3 Acceptance tests | | | |
| 4 | | | §3.4 Server engine | | | |
| 5 | | | §3.5 Client | | | |

---

## 8. Revision History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-04 | Initial AI strategy: system prompt with schema card; task prompts for serialization, parsing and validation, tests, server, and client; correction prompt; review checklist; acceptance tests; prompt log. Replaces blueprint §9 (v1.2). |
