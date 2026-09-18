# Battle for Biternia — Game Rules Spec

Source: official rulebook (`battle-for-biternia_rules_web.pdf`).
Scope: **2-player standard game only.** The 3-4 player team variant and the
optional Catapult variant are explicitly **out of scope for v1** and are not
covered below.

> **Open item:** this rulebook describes the rules *engine* (phases, keywords,
> combat math) but does not contain the literal text of all 76 Hero Power
> cards or the 24 Basic cards. Those need to be transcribed from the physical
> card set before `state-schema.md` / card data can be finalized. Hero
> summaries in this doc are flavor/balance notes only, not implementable
> card text.

## Objective

Each player controls a team of 2 Heroes. Each side defends one **Elemental
Bit** (guarded by up to 3 Towers) and tries to destroy the opponent's
Elemental Bit first.

**Win condition:** first player to reduce the opponent's Elemental Bit to 0
HP wins immediately.

## Components (digital-relevant subset)

- 19 Heroes, each with: a Hero Card (name, class, base HP, trait, class icon)
  and its own set of Power Cards (A/B/C/Z skill buttons; Z = "Ultimate",
  unlockable only at Hero level 4).
- 24 Basic Action Cards, usable by any Hero: Strike x4, Parry x4, Dash x2,
  Defend x2 per player's starting deck (12-card basic deck in 2-player mode).
- 2 Elemental Bits (1 per player) with HP trackers.
- Up to 3 Towers per player (6 total) with HP trackers, each guarding a path
  to that player's Bit.
- Tokens: Gold, Shield, Stone, +1 Attack, Chill, Death Mark, Confuse, Favored
  Enemy, Bear Trap, Ice Wall (Bear Trap/Ice Wall/Runika artifacts/Catapult
  are variant-only — out of scope for v1 unless later added).
- Initiative token (passes each round).

## Setup (2-player standard)

1. Place board; place 3 Gold tokens on each gold-icon space.
2. Place HP trackers + standees on both players' Towers and Bits.
3. Give each player 2 Hero Mats.
4. **Draft Heroes** (snake draft, initiative player picks first):
   - Round 1: initiative player picks 1 Hero, other player picks 2.
   - Round 2: initiative player picks 2, other player picks 2.
   - Round 3: initiative player picks 1.
   - Result: each player ends with exactly 2 Heroes. When a Hero is drafted,
     give the opposing player that Hero's reference card.
5. **Starting powers:** each player secretly picks 1 Power Card per drafted
   Hero (recommend the "A" card) → hand of 2 cards. Cannot pick the
   Ultimate ("Z") card until that Hero reaches level 4.
6. **Starting deck:** each player gets a 12-card basic deck (Strike x4, Parry
   x4, Dash x2, Defend x2), shuffled once (decks are **never reshuffled**
   again for the rest of the game — see Shuffle-Free Zone below). Draw 4 →
   total starting hand = 8 cards (2 power + ... wait, see note).

   > Rulebook text: preliminary hand is 4 (1 power card per Hero, 2 Heroes),
   > then draw 4 from the basic deck → **8 cards in hand** at game start.

7. **Deploy Heroes:** initiative player places 1 Hero standee on any of their
   own Tower spaces; other player does the same; alternate until both
   Heroes per player are deployed. Max 2 Heroes per space during deployment
   (no limit after deployment completes).
8. Set each Hero's level cube to 1, and HP cube to (base HP + level).

### Shuffle-Free Zone (important invariant)

Decks are **never shuffled** after the game begins. When a deck is
exhausted, flip its discard pile over as the new deck **without shuffling**.
This is load-bearing for ability cooldown/respawn pacing — the digital
implementation must preserve **strict FIFO card order**, not randomize on
reshuffle.

## The Game Round — 4 Phases

Each round: **Orders → Movement → Action → Cleanup**, then pass initiative
and start the next round.

### Phase 1: Orders (simultaneous, hidden)

- Each player assigns exactly one card from hand, face-down, to each of
  their living Heroes. Defeated Heroes are not assigned a card.
- If the Hero already has a face-up card from last round, the new face-down
  card is placed overlapping it so the old card's **defense value stays
  visible** to both players until revealed in the Action phase.
- Digital implication: this phase is simultaneous/blind — both players
  submit before either sees the other's assignment. Server must hold both
  submissions and only reveal after both are locked in.

### Phase 2: Movement (alternating)

- Initiative player moves or passes with 1 Hero; other player does the
  same; alternate until every Hero has moved-or-passed exactly once.
- Move = standee moves to one adjacent space (adjacent = connected by a path
  **and not obstructed by an enemy Tower**, unless that Tower is destroyed).
- Passing forfeits that Hero's move for this round only.
- Defeated Heroes cannot move — they must pass (rulebook explicitly notes
  players should pass defeated Heroes early since it costs nothing).
- A Bit is not reachable past its own Towers until at least one of that
  player's Towers is destroyed.

### Phase 3: Action (alternating activation)

- Initiative player activates 1 Hero, then the other player activates 1
  Hero; alternate until no Hero has a face-down card. If one player runs
  out first, the other activates their remaining Heroes in any order.
- **Activating a Hero:**
  1. Discard any face-up card currently on the Hero (into that player's
     discard pile).
  2. Flip the assigned face-down card face-up.
  3. Choose one: (a) resolve the card's printed effects in order, or
     (b) discard it and perform a Universal Action instead (Farm or Rest;
     Hire is variant-only).
- **Attacking:** `Attack X` = deal X damage to a chosen target, unless the
  card specifies otherwise. Default target = any enemy Hero/Structure in
  the *same space* as the attacker.
  - Physical (⚔) vs Magic (✨) icons: cosmetic/flavor only, no baseline
    game-rule difference unless a specific card references the type.
- **Defense:** `damage = max(0, attack value - target's current face-up
  defense value)`. No face-up card ⇒ defense = 0. Structures always have
  defense = 1 (flat, no card).
- **Multi-effect cards:** resolve each listed effect in printed order;
  each Attack on a card applies defense/modifiers independently.
- **Hit effects** (text after "Hit:") only trigger if that attack dealt
  **≥1 actual damage** (i.e., not fully absorbed by defense/shield). For
  Area attacks, only targets who took damage get the Hit effect.
- **Targeting rule:** if a card requires a target and none is legal (e.g.
  no enemy present in the Hero's space), skip only that effect; the rest of
  the card still resolves. The card's defense value still applies whether
  or not any of its effects had a legal target.
- **Mismatched cards:** if a Hero is somehow assigned another Hero's power
  card (validation should prevent this, but rules cover it defensively),
  it must be treated as a forced Universal Action — no benefit, including
  no defense value.

#### Attack Keywords

| Keyword | Rule |
|---|---|
| **Ranged** | May target an enemy Hero/Structure in the *same or adjacent* space (adjacency still blocked by an intact enemy Tower). A Bit is not "adjacent" to its own Towers for ranged-attack purposes until that Tower is destroyed. |
| **Area** | Hits **all** enemies (Heroes + Structures) in the attacker's space. Not a "targeted" attack — never affects friendlies, and effects that require "a target" don't apply since Area has none. |
| **Channeled** | Old card is discarded and its defense applies immediately as normal, but the card's *other* effects are deferred. After all Heroes/Structures have activated (before resolving Base Defenses), the initiative player resolves one of their Heroes' channeled effects, then the other player does the same; alternate until all channeled effects resolve. If the channeling Hero is stunned/defeated before resolution, the deferred effect is lost (this is intentional counterplay). |
| **Interrupt** | While assigned face-down to a Hero, its owner may reveal and resolve it *immediately after a target is chosen* for an enemy Hero's ability (or immediately after the ability is announced, if it's targetless) — before that ability resolves. If a target becomes illegal because of the interrupt, that action doesn't happen. Nested interrupts resolve last-declared-first (LIFO stack). |

### Base Defenses (Towers & Bits attack back)

- After all Heroes have activated but **before** resolving Channeled
  effects: each player's Towers and Bit(s) attack. All Structures use
  **Area Attack 4** against enemy Heroes in their space.
- Initiative player activates one of their Structures, then the other
  player does the same; alternate until all eligible Structures have
  attacked.
- **Destroying a Tower:** remove its standee; owner (the destroyer) gains 3
  gold; the *defending* player's Bit takes 3 damage (ignores Bit's
  defense). Destroyed Towers don't respawn and no longer block movement or
  ranged attacks through that space.
- **Destroying the Bit = the destroyer wins immediately.**
- The Bit cannot be targeted/damaged by attacks until **at least one** of
  its Towers has been destroyed — but it still takes 3 "collateral" damage
  (ignoring its own defense) whenever a friendly Tower is destroyed *or* a
  friendly Hero is defeated, even while otherwise "protected."
- Once ≥1 Tower is down, the Bit becomes attackable **through that lane
  only** — other intact Towers still block movement/attacks through their
  own lane.

### Universal Actions

Available to any Hero on activation instead of using its assigned card
(discard the card without resolving it):

- **Farm:** discard all cards on this Hero; gain 1 gold (+1 more if
  standing on a bonus-gold space).
- **Rest:** discard all cards on this Hero; heal self 3.
- ~~Hire~~ — variant rule (Catapult), out of scope for v1.

Using a Universal Action leaves the Hero with defense value 0 until its
next activation (no face-up card).

### Phase 4: Cleanup

1. Each player gains 2 gold.
2. Each player may pay gold to **level up** any number of Heroes, any
   number of times, each paid individually.
   - **Level-up cost = current level + 1** (e.g. level 2→3 costs 3 gold).
   - On level-up: increment level (also raises max HP by 1), then secretly
     add 1 of that Hero's Power Cards to hand (Ultimate/Z card unlocks only
     at level 4).
3. Each player may discard **any number** of cards from hand (including
   just-gained level-up cards — this is intentional, discarding faster
   feeds cards back through the deck sooner).
4. Each player draws back up to **hand size 8**. If the deck runs out mid-draw,
   flip the discard pile as the new deck (no shuffle — see Shuffle-Free Zone).
5. Pass the initiative token to the other player.

#### Respawning (part of Cleanup via deck cycling)

- A defeated Hero's Hero Card was placed in the player's discard pile when
  defeated (see Defeated Heroes below) and cycles through the deck exactly
  like an action card.
- When a Hero Card would be **drawn**, instead: place it back on its Hero
  Mat at full HP, and return its standee to the board on any space
  containing one of that player's remaining (non-destroyed) Towers or their
  Bit. It's active again next round.

### Defeated Heroes

When a Hero's HP reaches 0:

1. Remove its standee from the board.
2. Put its Hero Card into the owner's discard pile (re-enters the deck
   cycle — see Respawning).
3. Discard any action cards assigned to it.
4. **Opponent** gains gold equal to the defeated Hero's current level.
5. Owner's Elemental Bit takes 3 damage (ignores Bit's defense).
6. Discard all tokens on/associated with the defeated Hero (Ice Wall, Bear
   Trap, Death Mark, etc. — except tokens whose card text says otherwise,
   e.g. Favored Enemy).
7. Defeated Heroes still take part in the Movement phase but can only pass
   (their pass should be resolved as early as possible per the rulebook's
   tip — no real gameplay effect, just table-talk timing, likely a no-op
   digitally beyond "auto-pass").

Note: Shield tokens and +1 Attack tokens are explicitly **not** removed
when the Hero who placed them (not the Hero carrying them) is defeated —
only the token's own removal condition applies.

## Tokens & Keyword Definitions

| Token/Keyword | Effect |
|---|---|
| **Confuse** | Place token. Next round's Orders phase: that Hero is auto-assigned the top card of its owner's deck instead of a hand card (owner may look at it after assigning). Remove all confusion tokens after that assignment. Does not affect the Hero during the round it was applied; not removed if the Hero who applied it is defeated. |
| **Heal X** | Target friendly Hero in caster's space; +X HP, capped at max HP (overheal lost). |
| **Immobilize** | Standee laid on its side. Next time it would move for any reason, it stands up instead (move is consumed/skipped). |
| **Pierce X** | Attack ignores the first X points of the target's defense. |
| **Stun** | Discard all of that Hero's cards immediately, face-up and face-down both. Structures cannot be stunned. |
| **Structures** | Towers + Bits. Effects that "cannot target Structures" exclude both. |
| **Shield X** | Target friendly Hero in caster's space gains X shield tokens. Each point of damage (after defense is applied) removes 1 shield token instead of dealing damage; excess damage beyond shield count applies normally. Not removed when the placing Hero is defeated. |
| **+1 Attack token** | +1 to all of that Hero's attacks. Removed at end of that Hero's *next* activation, or immediately if stunned (regardless of whether an attack occurred). Not removed if the placing Hero is defeated. |
| **Self / Other** | "Self" restricts targeting to the caster only; "Other" excludes the caster as a legal target. |
| **Stealth** | Hero cannot be targeted by attacks/abilities that require choosing a target. Targetless effects (e.g. Area attacks) still affect it. |
| **Gold / Stone / Chill / Death Mark / Favored Enemy / Bear Trap / Ice Wall** | Effects are defined per-card (not in base rules) — **needs card-text transcription** before these can be implemented generically. |

## Out of Scope for v1 (explicitly deferred)

- 3-4 player team rules (shared team gold/initiative, 6-card decks, etc.)
- Catapult variant (Hire action, jungle spaces)
- Runika's artifact tokens (Battle Fist / Auto Deflector / Shield Amulet) —
  tied to a specific Hero's kit, deferred until Hero card data is entered
- "To the Pain" arena component (not explained in this rulebook excerpt —
  needs clarification if it's needed for v1)

## Still Needed Before This Spec Is Implementation-Ready

1. Full transcribed text for all 24 Basic cards + 76 Hero Power cards
   (this rulebook only gives mechanics + hero flavor/balance summaries, not
   card text).
2. Board layout: exact space graph (which spaces connect to which, gold
   spaces, jungle spaces, Tower/Bit positions) — the rulebook shows a board
   image but a digital implementation needs this as structured graph data.
3. Clarify "To the Pain Arena" component — unclear from this excerpt what
   it does or whether it's used in the standard 2-player game.
