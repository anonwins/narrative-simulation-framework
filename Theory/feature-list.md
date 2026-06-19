# What NSF’s Narrative Stack Enables

What players can **experience** and designers can **author** using NSF’s narrative simulation modules. Genre sweet spot: investigation-heavy, dialogue-first RPGs (Disco-like, noir, sci-fi mystery, etc.).

NSF does **not** cap what your game can do — it provides the narrative brain. Game-owned mechanics (combat, crafting, etc.) compose alongside NSF; see [architecture/game-extensions.md](architecture/game-extensions.md).

Spec detail: [`systems/`](systems/) · sample games: [`architecture/samples.md`](architecture/samples.md).

---

## Core fantasy

- Play an investigator (or similar) piecing together what happened — not just following a script
- Talk, observe, travel, and deduce; the world reacts and remembers
- Fail checks and still move the story forward (fail-forward)
- Works in **3D walkable** or **2D map/UI** presentation — same story brain underneath

---

## Investigation & clues

- Structured case files: leads, tasks, evidence, theories, and a resolution
- A personal case log / chronicle that updates as you progress
- Discover hidden objects, documents, and inspectables
- Clues become **facts** the simulation knows — gates and dialogue can depend on them
- Re-question witnesses and revisit topics unless a beat is meant to be one-and-done
- Track who knows what (rumors, intel spread between characters)

---

## Dialogue & conversation

- Branching conversations with meaningful choices
- Talk to NPCs while a **companion** can comment or chime in on the side
- Voice lines via **text-to-speech** or recorded audio (game decides look & feel)
- Dialogue options can lock, unlock, or change based on what you've learned
- Some lines only open after the right clue, relationship, or time window

---

## Character mind (skills & inner life)

- Skill checks on a **2d6** model — active rolls, passive notices, retries where allowed
- Custom skill sets per game (noir instincts, sci-fi crew skills, etc.)
- **Beliefs** you pick up, wrestle with, and resolve — not a hard cap on how many you juggle
- **Conduct** that reflects how you behave (bold, humble, ruthless…) and feeds into outcomes
- **Stress, emotion, vitality, morale** — pressure that builds and can affect play
- Skills can be buffed or drained by gear, beliefs, and story events

---

## People & politics

- Relationships that shift (trust, hostility…) — shown as **labels or numbers** depending on the game
- Factions you can help, anger, or navigate (cop guild, station crew, crime families…)
- Ideology / worldview axes that move when you take sides
- One main **companion** who travels with you, trusts you (or doesn't), and reacts to your choices

---

## World, places & time

- Move between locations (crime scenes, decks, districts…) — 3D scenes or 2D maps
- Click or walk up to people and objects to interact
- Time passes in the story; deadlines, night falling, alerts counting down
- NPCs on schedules — someone might only be somewhere at the right hour
- Areas stay locked until you've earned access (keycard, reputation, proof)

---

## Money, gear & survival pressure

- Earn, spend, and run out of money
- Shops, bribes, pawn, rent, debt — financial pressure as story fuel
- Buy information or services when the plot allows
- Inventory: key items, tools, evidence objects
- Equipment that changes your effective skills
- “Broke” or desperate states that open different options

---

## Story shape

- Main plot **threads** with milestones — not just a linear quest chain
- Story flags and beats the sim tracks behind the scenes
- **Pacing**: some content only appears in the right window; side content can stay softer
- Multiple endings and outcome scoring from how you played
- Failed rolls and dead ends that **branch forward**, not hard game over

---

## Player-facing presentation

- Dialogue UI, case log, belief journal, roll feedback
- Subtitles / spoken lines (TTS built in; games skin the rest)
- Ambient and spot audio (footsteps, rain, terminal hum…)
- Adjustable text size and high-contrast UI support
- **English** out of the box; games can add more languages (sample: English + French)

---

## Saves & sessions

- Autosave and continue — pick up the investigation where you left off
- Full history of what the world believes happened (for consistency after load)

---

## What you author vs what NSF provides

| NSF provides (compose what you need) | Each game adds |
|---|---|
| Narrative modules: dialogue, facts, rolls, chronicle, thread, economy, … | Characters, setting, art, voice casting |
| Shared infrastructure: registry, events, saves, rules, content pipeline | Story content, scenes, UI skin |
| Reference samples (noir 3D + sci-fi 2D) — full stack demo | Game-owned modules + adapters (see game-extensions) |

---

## Game-owned mechanics (not in NSF v1)

NSF does not ship combat, crafting, or other genre-specific loops. Games add them as **peer modules** or **adapters** that feed outcomes into facts, flags, and events so the narrative sim still remembers. Examples:

- Combat resolved → `fact_*`, vitality loss, conduct, chronicle entries
- Chase failed → fail-forward branch via rolls + story flags
- Crafting milestone → inventory + fact registration

Details: [architecture/game-extensions.md](architecture/game-extensions.md).

---

## Not in NSF package (v1)

- Online / co-op multiplayer
- Player modding API (games ship their own content packs)
- Built-in Steam page or cloud account UI
- Shipped combat/crafting/etc. modules (game-owned unless promoted later)
