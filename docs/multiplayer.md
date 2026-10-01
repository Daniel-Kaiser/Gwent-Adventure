# Multiplayer (Challenge a Friend)

Status: **step 2 done. "Challenge a Friend" can be played.** Two browsers connect peer-to-peer over
WebRTC. There is no game server and nothing costs money to run.

* Each browser keeps its own hand and draw pile private.
* Both clients run the same rules engine on the shared public state.
* Only the acting player's decisions travel over the wire, and hidden card identities never do until
  the card is revealed.

Single player (Quick Game vs AI, tutorial, deck builder) is unchanged and still works offline. The
connection library (PeerJS) is loaded from a CDN only when a player hosts or joins with a room code.

---

## How to play with a friend

1. Both players need **exactly the same game file**. If the two copies differ in any way (even one
   balance change), the game refuses to connect with "Different game versions".
2. Main menu → **New Game** → **Challenge a Friend**. Enter your name (it is remembered).
3. One player clicks **Host a game** and gets a 6-character room code, e.g. `8A4NA5`. The code is
   shown in a text box with a **Copy** button. If copying is blocked (some iframes), select the text
   and press Ctrl+C.
4. The other player clicks **Join a game**, types the code and clicks **Join**.
5. Both players pick a deck in the usual *Select deck* popup. Randomized is allowed; incomplete decks
   cannot be picked. The match starts once both have chosen.
6. In game:
   * The friend's hand is shown as card backs only. Their draw pile is just a count.
   * A small banner at the top says when the friend is thinking or choosing.
   * Their played cards fly in from their hand and flip face up.
7. The menu (☰) has **Concede**. At the end both players see **Rematch**, which needs both to accept
   and reuses the same deck choices, and **Main Menu**.
8. **If room codes do not work**, use **Advanced: connect with manual codes**:
   * The cause may be that the matchmaking service is unreachable, or that a firewall blocks it.
   * The host sends an invite code. The joining player pastes it and sends back a reply code. The
     host pastes that and clicks **Connect**.
   * The codes are longer (about 600 characters) but need no third-party service.
9. **If you cannot connect at all**, one of you is probably behind a strict NAT (some mobile or
   corporate networks). That needs a TURN relay server. Add one in `PVP_TURN_SERVERS` near the top of
   the *PVP* section of `game.html`.

It works when opened as a local `file://`; all tests ran that way. It is built to work inside an
iframe (itch.io) as well, but that has not been tried yet.

## Testing alone with two tabs

1. Settings → **Developer mode** on.
2. New Game → Challenge a Friend → **Local test (two tabs)** → **Host**. Note the 4-letter code.
3. Open the same `game.html` in a **second tab of the same browser** and turn Developer mode on there
   too.
4. In the second tab: Local test → **Join** → enter the code.
5. Play both sides by switching tabs.

This uses a `BroadcastChannel` inside the browser, so no network is involved. It only exists for
testing and appears only in Developer mode.

---

## 1. Seats and controllers

The engine always reasons from the point of view of the **local browser**:

| seat       | single player            | PvP                               |
|------------|--------------------------|-----------------------------------|
| `'player'` | you (bottom of screen)   | you (bottom of screen)            |
| `'ai'`     | the AI opponent          | your friend (top of screen)       |

Each client sees itself as `'player'`. A seat named `'ai'` means "the opponent seat"; it does not mean
the opponent is an AI. For anything both clients must agree on, `state.pvpSeats` maps the two seats to
the canonical roles **`h`** (host) and **`g`** (guest).

Every seat has a **controller** (`controllers[side]`):

| controller             | `kind`   | `interactive` | how it decides |
|------------------------|----------|---------------|----------------|
| `LocalHumanController` | `local`  | yes | The UI renders requests from engine state (targeting session, picker modals, mulligan overlay). Mouse and keyboard input becomes actions through `submitAction`. |
| `AIController`         | `ai`     | no  | Autonomous: `aiTakeTurn()` / `aiMulligan()`. In-flow choices come from the AI policy in the engine's non-interactive branches. **Never installed in PvP** (`aiTakeTurn`/`aiMulligan` also refuse to run there). |
| `makeRemoteController(pvp)` | `remote` | yes | PvP opponent seat. Requests produce no local UI; the friend's actions arrive over the network and are applied by `pvpPump`. |
| `makeBotController(side, seed)` | `bot` | yes | Tests only. Seeded random choices through `submitAction`. |

The engine asks `isInteractive(side)`, never "is this the human?". `isLocalSide(side)` only decides
presentation: banners, highlights, picker modals and click handlers.

### Request types

All requests are plain JSON (`notifyRequest(side, request)`):

* `{ type: 'mulligan' }`: answer with `mulligan` (optional, once) and then `mulliganDone`.
* `{ type: 'target', kind, source, candidates, selected, max, mandatory, amount }`
  * `kind` is one of: `order`, `playDamage`, `playDebuff`, `playRefresh`, `playMove`,
    `reactionBuff`, `specialDamage`, `playDamageSeq`, `playBuffSeq`, `playBuffMulti`,
    `orderMulti`, `pickMulti`, `pickOwn`.
  * Answer with `target` (toggles in multi-selects), `finish` or `cancel`.
* `{ type: 'pick', zone: 'discard' | 'deck', pile, candidates, hint }`: answer with `pick`
  (`uid: null` = skip).
* **"Each player chooses" (Rotfiend, Archgriffin)**: in PvP both players choose **at the same time**
  (`pickOwnBoth`).
  * The local seat uses the normal pick UI.
  * The remote seat gets a parallel board-pick request (`pendingBoardPick`), answered by its `target`
    action.
  * Both players see "Pick a card to be discarded", and the effect resolves once both have chosen.
  * Single player keeps the old order: the active player first, then the AI.

## 2. Actions

Every decision is a plain JSON action. There is one entry path:

```
submitAction(side, action) -> validateAction -> recordAction -> applyAction
```

| action | fields | meaning |
|---|---|---|
| `play`        | `uid`, `index` | play a hand card (`index` = board slot or `null`) |
| `order`       | `uid` | activate a ready Order |
| `endTurn`     | | end the turn (the draw happens automatically) |
| `pass`        | | pass the round |
| `mulligan`    | | swap the opening hand (once) |
| `mulliganDone`| | keep the hand / finished |
| `target`      | `uid` | answer a targeting request or a board pick |
| `finish`      | | "Done" on a multi-select |
| `cancel`      | | back out / finish early (`applyTargetingCancel`) |
| `pick`        | `uid`, `null` or `'?'` | answer a discard/deck pick; `'?'` = a card in a deck this client cannot see |

PvP routing:

* **My actions.** `submitAction('player', a)` goes to `pvpSubmitLocal`, which validates, sends
  `{ t: 'act', a }` and applies locally.
  * A `play` carries `reveal: { id }`, the identity of the card being played, which becomes public at
    that moment.
  * A pick inside my own deck (Dandelion) is sent as `'?'`.
* **The friend's actions** are queued, then applied by `pvpPump`, in order, once
  `validateAction('ai', a)` accepts them.
* **Both directions** apply an action only when `pvpReadyFor(a)` holds: no kill animation pending, no
  card or projectile in flight, and, for turn-level actions, no reactions, follow-ups, token spawns
  or end-turn draw in progress. Both clients therefore apply every action at the same logical point.
  * My own action that comes in early (for example a click during an animation) is held in
    `pvp.localQueue` and committed a moment later.
* A friend's action that stays invalid for 25 s while nothing local is gating it means the games
  diverged; the "Out of sync" popup is shown.
* Local gates are the round-end Continue overlay and the mulligan overlay. Actions that arrive while
  one is open simply wait in the queue.

## 3. Determinism

Randomness that affects game state uses seeded streams (sfc32 seeded by xmur3):

| stream | used for | who knows it in PvP |
|---|---|---|
| `RNG.shared`        | starting player | both, via commit-reveal (§5) |
| `rngPrivate(side)`  | that side's random deck, shuffle, mulligan reshuffle, Dandelion reshuffle | **only that side** |
| `RNG.brain`         | AI decisions | single player only |

Cosmetic randomness (particles, voice lines, music) stays `Math.random`.

**Card uids are opaque and deterministic.**

* **Single player:**
  * deck cards are `p0..` / `a0..` after the private shuffle;
  * tokens are `tok0..`.
* **PvP:**
  * a card in a deck has a private uid (`hd3`, `gd7`);
  * when a seat draws its n-th card, that card gets the public uid `h<n>` / `g<n>` on both clients
    (the owner's real card and the other client's placeholder). Draws therefore need no message and
    reveal nothing; only the count is public;
  * tokens are `ht<n>` / `gt<n>` from per-seat counters.
* **Iteration order:** anything whose order can change state iterates seats host-first (`seatOrder()`,
  `allBoardCards()`), so both clients burn, kill and react in the same order. In single player the
  order is the classic player-first.
* **Dev hooks for tests:**
  * `window.__gwaNextSeeds` / `window.__gwaNextDecks`;
  * `__gwaDev.setSeeds`, `setDecks`, `useBot`, `snapshot`;
  * PvP: `__gwaDev.pvpBot`, `pvpNoShuffle`, `fakeBuild`, `pvpTamperDeck`, `netLog`,
    `pvpSimulateDrop(ms)`, `pvpInfo()`, `pvpCanonical()`.

### Per-turn state hash

At every turn boundary (end of `endTurn`, including round end) each client hashes a canonical public
state (`pvpCanonicalState`). It covers:

* round, turn, active seat;
* per seat, host first: board (uid, card, power, Order used, owner), discard pile, hand uids, deck
  count, passed flag and score.

The hashes are exchanged. A mismatch shows **"Out of sync"** (Keep playing / End match) and writes
both states to the console.

### Single-player determinism test

Two pages with the same seeds play bot vs AI and bot vs bot and must produce identical logs, action
logs and final state.

## 4. Public vs private state

| zone / field | visibility |
|---|---|
| hand (cards and order) | **private**: the friend sees card backs (count + uids only) |
| deck / draw pile | **private**: the friend sees the count |
| private seed, deck salt | **private** (the deck list and salt are revealed at game end, see §5) |
| boards, powers, buffs, flags | public |
| discard piles | public |
| counts, passed, actedThisTurn, orderUsedThisTurn, mulliganUsed | public |
| round, turn, scores, starting player, shared seed | public |

The friend's hand and deck exist on this client as placeholders: `{ uid, hidden: true, owner: 'ai' }`.

* `pvpRevealHandCard` turns a placeholder into a real card when it is played.
* `pvpDeckTopReveal` does the same for Maxii's top card.
* A card that returns to hand from the board (Ciri) stays known, because its identity was already
  public.

### Hidden-information audit

| card / mechanic | what it touches | PvP handling |
|---|---|---|
| **Maxii van Dekkar** | reveals its owner's own top deck card | the owner sends `{ t: 'reveal', key: 'deckTop', id }` when the effect resolves; the other client waits for it (`pvpDeckTopReveal`) |
| **Dandelion** | owner picks from their own deck, card on top, rest reshuffled | the pick is sent as `'?'`; the other client only keeps a count; the log says "puts a card on top" |
| draws, redraw to 4, mulligan | owner's own deck | deterministic public uids, no identities, no message |
| `deckBottomChoice` (old Angoulême effect) | the opponent's deck | **unused by any card (dormant)**; it would need the deck owner to reveal the whole deck |
| round-end return to hand (Ciri) | public card back into a hand | stays known |
| AI | its own zones | never runs in PvP |
| hover inspector / opponent hand rendering | | the opponent hand renders card backs only; placeholders are never rendered |
| log | | names come only from public cards; "Opponent" becomes the friend's escaped name |

Result: **no card reads the opponent's hand or deck.** The automated tests check, during whole games,
that the other client's DOM never contains the friend's hand cards. They also check that every card
id received over the wire was already public by then.

## 5. Network

### Transports

All three share one interface: `send(str)`, `close()`, `onmessage`, `onstatus('open'|'down'|'up')`.

* **Room codes (PeerJS)**
  * Library: `peerjs@1.5.4`, loaded lazily from unpkg with jsdelivr as fallback.
  * Broker: the free public PeerJS cloud.
  * ICE: Google STUN, plus an optional TURN entry in `PVP_TURN_SERVERS`.
  * Room id: `gwentadv-v1-<6 chars>`; the prefix avoids collisions with other apps.
  * The guest reconnects automatically every 3 s if the data connection drops. The host accepts a
    reconnect only from the same guest token.
* **Manual codes**
  * Plain `RTCPeerConnection` with an ordered data channel and non-trickle ICE (gathering waits up to
    5 s).
  * The offer/answer SDP is deflate-compressed and base64-encoded: `GWAO1z:…` for the invite,
    `GWAA1z:…` for the reply.
  * There is no automatic reconnect; ICE may still recover a brief outage by itself.
* **Local**
  * `BroadcastChannel('gwa-local-<code>')`, with `localStorage` events as a fallback.
  * Developer mode only.

### Session (`makePvpSession`)

* Every message is `{ seq, m }` and is kept in an outbox.
* The receiver drops duplicates. On a gap it asks `{ c: 'resume', last }`, and the sender replays
  everything after `last`.
* A ping is sent every 1.5 s. After 6 s of silence the session is "down": a top banner shows
  "Connection lost – reconnecting… (Ns)" with **End match**, and the transport tries to reconnect.
* When traffic comes back, both sides send `resume` and the missed messages are replayed.
* After 30 s down the session is lost, and a "Connection lost" popup leads back to the menu.
* A deliberate exit sends `{ c: 'bye' }` (also on page close). The other side immediately shows
  "Opponent left".

### Match protocol (`m.t`)

1. **`hello`** `{ proto, build, name }`
   * `build` is the SHA-256 of the whole inline game script, so any difference in rules or cards is
     refused with a clear message.
2. **`commit`** `{ deckHash, deckCount, seedHash }`
   * `deckHash = SHA-256('deck:' + sorted ids + '|' + salt)`
   * `seedHash = SHA-256('seed:' + r)`
   * It is sent right after the deck is picked.
3. **`seed`** `{ r }`
   * Each side reveals its `r` only after receiving the other's commitment, then verifies the other's
     `r` against its commitment.
   * `shared = SHA-256('shared:' + r_host + '|' + r_guest)`, so neither side picks it alone. The host
     starts if the first shared draw is below 0.5.
4. **`act`** `{ a }`: one action of the sender's seat (§2).
5. **`reveal`** `{ key: 'deckTop', id }`: Maxii's top card.
6. **`hash`** `{ k, h }`: the per-turn state hash.
7. **`deck`** `{ ids, salt }`: sent at game end. The receiver checks:
   * the hash matches the commitment;
   * every card the friend played is contained in the list.

   Otherwise the end screen shows a cheating warning.
8. **`concede`**, **`rematch`** (both must send it; same deck choices, fresh salt and seed), **`leave`**.

## 6. Testing

Automated tests, in the container: Playwright with real mouse clicks for connecting and deck selection,
and seeded bots driving both seats through the PvP action path. Results:

* **Local transport, full games:**
  * random decks, and monster/reveal-heavy decks (Rotfiend, Archgriffin, Whispess, Monster Egg, Hillock,
    Maxii, Dandelion, ...): every turn hash equal;
  * both deck checks verified;
  * no hidden card identity in the other page's DOM or received messages.
* **Manual WebRTC between two separate browser contexts:** connect by pasting codes, then a full game.
* **Room-code flow** (Host / Join popups, wrong code, guest reconnect after the data connection
  closes): tested against a **stand-in for PeerJS**, because the sandbox cannot reach the real broker.
* **Real-mouse scenarios:**
  * Archgriffin both-pick across the two pages;
  * Ghoul → Monster Egg discard pick;
  * Whispering Hillock → discard pick.
* **Desync injection** (a power changed on one page): "Out of sync" on both pages.
* **Disconnect:**
  * 10 s drop: banner on both pages, resume, game finishes in sync;
  * 60 s drop: "Connection lost".
* **Opponent closes the tab:** "Opponent left".
* **Concede, cheat detection and rematch:** the conceding side and the winner see the right end
  screens; a tampered deck list is flagged; a rematch starts a new synced game.
* **Build mismatch:** refused on both sides.

**Not tested** (needs a real internet setup):

* the real PeerJS cloud broker and the CDN download;
* STUN/TURN across real NATs;
* the game inside an itch.io iframe;
* Safari/Firefox (the tests use Chromium);
* real long-distance latency.

## 7. Known risks / TODO

* **PeerJS public broker:** free and best-effort; it may be down or rate-limited. The manual codes are
  the fallback. Self-hosting a PeerJS server, or adding TURN, would be the robust upgrade.
* **Strict NATs** need TURN (`PVP_TURN_SERVERS`); none is configured.
* **Reconnect** works while both pages stay open:
  * the session replays missed messages after a drop;
  * a **page reload** cannot resume (the game state lives in the page), so the match ends.
  * Manual-code connections cannot re-signal after a full ICE failure.
* **Validation:** `validateAction` checks legality, and the deck commit catches fake identities at the
  end. A modified client could still, for example, **look at its own deck order** (it is private) or
  stall. The end-of-game check reports a mismatch but cannot undo the game.
* **Determinism** rests on "apply at a settled point"; the per-turn hash is the guard. If a desync ever
  shows up, the console holds both canonical states.
* The AI policy is still inline in the engine's non-interactive branches. That is harmless for PvP.
* The tutorial keeps its scripted AI and fixed uids; it is single-player only.
