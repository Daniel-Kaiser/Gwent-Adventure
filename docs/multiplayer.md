# Multiplayer preparation (step 1 of PvP)

Status: **step 1 done. No networking yet, and gameplay is unchanged.** `game.html` is now structured so
that a later step can add peer-to-peer PvP (WebRTC, two browsers, no server).
In that design each browser keeps its own hand and draw pile private. Both clients run the same
rules engine on the shared public state, and only the acting player's decisions go over the wire.

This document describes:

* the architecture introduced in step 1
* the contracts that step 2 must keep
* the open risks

---

## 1. Seats and controllers

The engine always reasons from the point of view of the **local browser**:

| seat       | meaning in single player | meaning in PvP (step 2)          |
|------------|--------------------------|-----------------------------------|
| `'player'` | you (bottom of screen)   | you (bottom of screen)            |
| `'ai'`     | the AI opponent          | the remote human (top of screen)  |

Each client sees itself as `'player'`. A seat named `'ai'` in the code means
"the opponent seat"; it does not mean the opponent is an AI.

Every seat has a **controller** (`controllers[side]`, defined in the
*CONTROLLERS & ACTIONS* section of `game.html`):

| controller             | `kind`   | `interactive` | how it decides |
|------------------------|----------|---------------|----------------|
| `LocalHumanController` | `local`  | yes | The UI renders requests from engine state (targeting session, picker modals, mulligan overlay). Mouse and keyboard input becomes actions through `submitAction`. |
| `AIController`         | `ai`     | no  | Autonomous. `onTurnStart` runs `aiTakeTurn()` and `mulligan` runs `aiMulligan()`. In-flow choices are made by the AI policy in the engine's non-interactive branches. |
| `makeRemoteController(transport)` | `remote` | yes | **Stub for step 2.** Receives `onRequest(side, request)` and forwards it to the transport. Actions from the peer are fed to `submitAction(side, action)`. |
| `makeBotController(side, seed)`   | `bot`    | yes | Test only (dev hook). Answers every request with seeded random choices through `submitAction`. |

The engine never asks "is this the human?". It asks `isInteractive(side)`:

* **Interactive sides** get *requests* (a targeting session, a discard or deck pick, the mulligan). The
  engine then waits for an answering action.
* **Non-interactive sides** (the AI) compute the choice inline in the engine branch guarded by
  `!isInteractive(side)`.

In PvP both seats are interactive, so the AI branches are never reached.

`isLocalSide(side)` only decides **presentation**:

* whether the targeting noodle, banners and highlights are shown
* whether picker modals open
* whether click handlers are attached

Requests for a non-local interactive seat produce no local UI. That seat's controller gets
`onRequest(...)` instead.

### Controller interface

```js
{
  kind: 'local' | 'ai' | 'remote' | 'bot',
  interactive: boolean,
  autoEndTurn?(): boolean,          // interactive only: may the engine auto-end this side's turn?
  onTurnStart?(side): void,         // called ~900 ms after the side's turn starts (scheduleTurnStart)
  onRequest?(side, request): void,  // a decision is needed (see request types)
  mulligan?(side): void,            // non-interactive only: take the mulligan decision now
}
```

### Request types (`notifyRequest(side, request)`)

All requests are plain JSON:

* `{ type: 'mulligan' }`: decide on the opening hand. Answer with `mulligan` (optional, once) and then
  `mulliganDone`.
* `{ type: 'target', kind, source, candidates: [uid], selected: [uid], max, mandatory, amount }`
  * `kind` is the targeting session kind:
    * single picks: `order`, `playDamage`, `playDebuff`, `playRefresh`, `playMove`, `reactionBuff`, `specialDamage`
    * sequential picks: `playDamageSeq`, `playBuffSeq`
    * multi-selects: `playBuffMulti`, `orderMulti`, `pickMulti`
    * own-card pick: `pickOwn`
  * Answer with `target` (repeated for multi-selects, where it toggles), `finish` or `cancel`.
* `{ type: 'pick', zone: 'discard' | 'deck', pile: side, candidates: [uid], hint }`: a discard or deck
  picker. The discard picker is used by Yennefer, Caretaker, Sigrdrifa, King Bran, Grave Hag,
  Monster Egg and Whispering Hillock; the deck picker by Dandelion. Answer with
  `pick` (`uid: null` = skip).
* Turn-level decisions (play, order, end turn, pass) are not separate requests. When it is a side's
  turn and the engine is idle, that side may submit one of them. `onTurnStart` is the hint.

## 2. Actions

Every decision of an interactive side is a plain JSON action. It enters the engine through **one** path:

```
submitAction(side, action)  ->  validateAction(side, action)  ->  recordAction(...)  ->  applyAction(side, action)
```

| action | fields | meaning |
|---|---|---|
| `play`        | `uid`, `index` (board slot or `null` = rightmost) | play a hand card |
| `order`       | `uid` | activate a ready Order |
| `endTurn`     | | end the turn (the end-of-turn draw happens automatically) |
| `pass`        | | pass the round |
| `mulligan`    | | swap the whole opening hand (once) |
| `mulliganDone`| | keep the hand / finished |
| `target`      | `uid` | answer the open targeting request; toggles in multi-selects |
| `finish`      | | "Done" on a multi-select (`pickMulti`) |
| `cancel`      | | back out or finish early. The engine decides per kind (`applyTargetingCancel`). |
| `pick`        | `uid` or `null` | answer the open discard or deck pick |

Details:

* `validateAction` rejects anything illegal right now (wrong turn, wrong side, card not a candidate, not
  plain JSON, and so on) before it touches state. **Remote input must always go through it.**
* Confirmation dialogs ("cancel this ability?") and gating such as the tutorial or hold-to-pass are
  **local UI**. Only the final decision becomes an action.
* `state.actionLog` records every applied action, plus the AI's turn-level decisions (`by: 'ai'`), as
  `{ n, side, by, turn, ...action }`.

## 3. Determinism

All randomness that affects game state goes through seeded streams (sfc32 seeded by xmur3):

| stream | used for | who knows the seed in PvP |
|---|---|---|
| `RNG.shared`        | starting player | both (commit-reveal, see §5) |
| `rngPrivate(side)`  | that side's random deck building, initial shuffle, mulligan reshuffle, Dandelion's reshuffle | **only that side** |
| `RNG.brain`         | AI decisions (`aiShouldPassWhenAhead`) | single player only |

Rules:

* `shuffle(arr, rng)` must be given a stream whenever the result matters. Without one it falls back to
  `Math.random`, which is for cosmetic use only.
* Cosmetic randomness may stay `Math.random`: particles, crumbs, voice-line choice, music track, deck
  ids in the deck builder. It must never feed back into state.
* **Card uids are opaque and deterministic.**
  * A side's cards are numbered `p0, p1, ...` / `a0, a1, ...` **after** the private shuffle, so a uid
    never reveals the card's identity.
  * Tokens are `tok0, tok1, ...` from a counter reset each game. Both clients create tokens in the same
    order, so their uids match.
  * Developer-added cards use the next `p<n>`.
  * The tutorial keeps its fixed hand-written uids.
* Seeds can be forced for tests: `window.__gwaNextSeeds = { shared, player, ai }` (or
  `__gwaDev.setSeeds`) before a game starts. `state.seeds` stores them. In PvP a client would only have
  `shared` and its own seed.
* Decks can be forced the same way: `window.__gwaNextDecks = { player: [ids], ai: [ids] }` (or
  `__gwaDev.setDecks`). In PvP each client builds only its own deck.
* Engine flows are async (animations, `setTimeout`). The logical order of state changes still has to be
  identical on every client. This is guaranteed as long as decisions are only taken once the engine is
  idle, which is what the controllers do.
  * "Idle" includes `pendingKills === 0`: a killed card stays on the board, at 0 power, until its
    death, burn or eat animation ends. The AI (`buffReactionsIdle`), multi-hit damage
    (`applyMultiDamage`, the AI's sequential hits, Birna's follow-up) and the bots wait for those
    removals before the next decision or hit. Otherwise "X is destroyed" (and the own-discard
    reactions it triggers) could land before or after the next hit depending on frame timing.

### Determinism test

`__gwaDev.useBot(side, seed)` replaces a seat's controller with a seeded bot. The test runs two
headless pages with identical seeds:

* bot vs AI: the AI on its normal autonomous path
* bot vs bot: both seats interactive, which exercises the exact path a remote player will use

It requires identical `state.log`, `state.actionLog` and final public state.

How to run it (Playwright):

1. For each of two pages, call `__gwaDev.setSeeds(...)`, optionally `__gwaDev.setDecks(...)`, then
   `__gwaDev.useBot('player', seed)` (and `useBot('ai', seed)` for bot vs bot), then `startNewGame()`.
2. Click `#roundContinueBtn` whenever it appears.
3. When `state.phase === 'gameEnd'`, compare `__gwaDev.snapshot()` from both pages.

Step 1 results:

* 3 seeds with random decks, each run as bot vs AI and as bot vs bot: all 6 runs identical.
* 3 seeds with monster-only decks (eat/discard heavy), each run as bot vs AI and as bot vs bot: all 6
  runs identical.

The AI code is written for the `'ai'` seat only, so "AI vs AI" is done as bot (player seat) vs AI
(opponent seat) and as bot vs bot. Letting the AI play the bottom seat would need a perspective
refactor of the AI. That work is not needed for PvP.

## 4. Public vs private state

| zone / field | visibility |
|---|---|
| hand (cards and their order) | **private** to the owner; the opponent knows the count and opaque uids |
| deck / draw pile (cards and order) | **private** to the owner; the opponent knows the count |
| private seed | **private** |
| boards, powers, buffs, flags (`orderUsed`, `playedTurn`, bloodthirst, ...) | public |
| discard piles | public |
| hand/deck counts, passed, actedThisTurn, orderUsedThisTurn, mulliganUsed | public |
| round, turn, scores, starting player, shared seed | public |

Data model:

* `publicView(viewer)` returns what a client may show or send. The opponent's hand and deck appear as
  `{ uid, hidden: true }` placeholders (`hiddenCard(uid)`) plus counts.
* `revealCard(side, uid, id)` turns a placeholder into a real card instance when the owner reveals it.
* In single player nothing is hidden yet. Both seats hold real cards, and the UI simply never shows the
  opponent's hand or deck.

### Hidden-information audit (cards that read or reveal hidden zones)

| card / mechanic | what it touches | PvP handling |
|---|---|---|
| **Maxii van Dekkar** (`playDamageFromDeckTop`) | reveals the top card of its **owner's own** deck, which deals damage equal to its power and is shown face up beside the pile | the owner's client sends `reveal {uid, id}` for the top card before the effect resolves |
| **Dandelion** (`deckTopChoice`) | the owner looks at **their own** deck, puts one card on top and reshuffles the rest | resolved on the owner's client only; the opponent learns nothing (the reshuffle uses the owner's private stream). The opponent's client just keeps "deck = N unknown". |
| `deckBottomChoice` handler (formerly Angoulême's "look at the opponent's deck, bury a card") | the **opponent's** deck | **No current card uses it.** It is dormant. If revived it would require the deck owner to reveal the whole deck to the chooser, and the owner then reshuffles with their own seed (the call already passes `chooser`). |
| draws (end-of-turn, redraw to 4, mulligan) | the owner's own deck | the owner announces the drawn **uids** (not identities) |
| round-end "return to hand" (e.g. Ciri) | a public card goes back into a hand | its identity is already public; it stays known |
| AI | reads only its own hand and deck (`aiPlayDamageFor` peeks its own top card for Maxii) | not present in PvP |
| local draw animation (`flyPlayerDraw`) | peeks the local seat's own top card | local only |

Result: **no current card reads the opponent's hand or deck.** Only Maxii reveals a hidden card (the
owner's own top card).

## 5. Planned network protocol (step 2 sketch)

Transport: a WebRTC data channel (ordered, reliable). Signalling is out of band (copy-paste offer and
answer, or a tiny relay). All messages are JSON with `{ t, seq, ... }`, where `seq` increases per sender.

1. **`hello`** `{ version, protocol: 1, gameHash }`: both sides check they run the same `game.html` build
   (hash of the rules and card DB) and abort on mismatch.
2. **Deck commit** `{ t: 'deckCommit', hash }`: each side commits `H(deckIds sorted + salt)` so a deck
   cannot be swapped later. The salt and list are revealed at game end (or on dispute) for verification.
3. **Seed exchange (commit-reveal)**: neither side picks the shared seed alone.
   * `{ t: 'seedCommit', hash: H(rA) }` both ways, then `{ t: 'seedReveal', r: rA }` both ways.
   * Each side verifies the other's commit.
   * `shared = H(rA || rB)` determines the starting player and any future public randomness.
   * Each side's **private** seed never leaves its client.
   * Each client shuffles its own deck with its private stream and announces only the resulting uid
     list (`p0..p24` order is implied, so in practice just the count).
4. **`action`** `{ t: 'action', side, action }`: an action of the sender's seat. The receiver maps the
   sender's `'player'` to its own `'ai'` seat, runs `submitAction('ai', action)`, and rejects it (with
   `resync`) if `validateAction` fails.
5. **`reveal`** `{ t: 'reveal', cards: [{ uid, id }] }`: sent **before** an action that makes a hidden
   card public (`play`; Maxii's top card; a card leaving hand or deck in any other way). The receiver calls
   `revealCard`.
6. **`draw`** `{ t: 'draw', uids: [...] }`: which opaque uids moved from deck to hand. Identities are not
   sent.
7. **`resync`** `{ t: 'resync', turn, stateHash }`: periodic or on demand. Both sides hash
   `publicView(...)` with seat names normalised (local seat first) and compare. On mismatch, fall back
   to the last agreed snapshot or abort.
8. **`concede`** `{ t: 'concede' }`: ends the game. A disconnect timeout counts as a concede after a grace
   period.

Ordering rule: an action is applied only when the local engine is idle and in the same logical step as
the sender's. Requests are generated identically on both clients because the engine is deterministic.
The non-acting client therefore already *expects* the answer and queues it until its engine reaches
that point.

## 6. Known risks / TODO for step 2

* **AI policy is still inline** in the engine's non-interactive branches. It is not wrapped in
  `AIController` methods. This is harmless for PvP (no AI), but a full extraction is still open.
* **Remote seat UI**:
  * Banners such as "Opponent is choosing..." are not shown yet. A non-local targeting session just
    shows nothing locally.
  * The cancel-confirm dialog and the discard/deck picker are local-only by design.
* **Round-end "Continue" overlay** is a local gate (not an action). In PvP both clients must reach the
  next round independently. Actions that arrive while the overlay is open must be queued.
* **Hidden placeholders**: engine code that inspects card fields of a hand card on the opponent seat
  would break on a `{ hidden: true }` placeholder. A review found only count-based uses (render,
  `turnEndable`, `drawsLeftThisRound`). This must be re-verified when step 2 actually installs
  placeholders.
* **Timing**: engine flows use animation callbacks and timers. Determinism holds because state changes
  happen in a fixed logical order and decisions wait for an idle engine. Two parallel flows racing each
  other would break it; the determinism test is the guard.
* **Validation surface**:
  * `validateAction` covers turn/side/candidate legality.
  * A malicious peer could still send a legal but impossible card *identity* in `reveal`. The deck
    commit (§5.2) is what lets the cheated party detect that at game end.
* **Tutorial** keeps its scripted AI and fixed uids. It is single-player only.
* `__gwaDev` (seeds, bots, snapshot) is a hidden developer hook with no UI.
