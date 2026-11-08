# World state for long-horizon video: references with roles, lifetimes and a ledger

Public summary of a design consult written 2026-11-07 for Snap, Dylan's ComfyUI front-end (part of the public
ComfyUI-MAINodes pack), on keeping what is true about a production's world over time and using it to choose what
conditions each shot and each extension segment of MiniMax H3 / Pika H3 Flex video.
**Status: design, not built.** The Extend Director part depends on the rolling-references study
([u46h_rolling_refs.md](u46h_rolling_refs.md)), which is in progress.

Related: [u46_extend.md](u46_extend.md) (extend studies U46, U46b, U46d-g), [prompt_craft_rules.md](prompt_craft_rules.md)
(the measurements M1-M7 cited here), [key_picture_freeze.md](key_picture_freeze.md),
[factcheck_subject_vs_picture.md](factcheck_subject_vs_picture.md), [ideas_and_lessons.md](ideas_and_lessons.md).

---

## 1. The problem

### Why "memory of a movie" fails

The tempting design for long video is a memory: keep everything the model has rendered and feed the relevant parts
back. The failures seen on 2026-11-07 were not storage failures. They were failures of **provenance and roles**:

- **A jump between two cuts.** The second cut of a test short jumped in set, light and pose because nothing tied its
  first frame to the first cut's last frame.
- **Set drift.** The set plates were made before the look was picked, and nothing promoted the picked look to "the set".
- **A lost effect.** A good bowl-burst effect shot could only be saved as a note: nothing could hold an effect so a later
  cut could cite it.
- **A reverting background.** In a long continuation of a woman walking down a beach, the landscape rightly drifted as
  she walked, then reverted to the initial reference background. A "where" picture was treated as "always stay here"
  instead of "this is where we are now".

The measured lever behind the last three is **which pictures are in the references at all**, not how they are cited:

| measurement | result |
|---|---|
| key picture in references on an action cut (M6) | froze 4/4 takes by 1.5-2.5 s (motion 3.3-6.5 vs 12.7-13.3 without) |
| location master cited (M2) | 13/18 takes at 640 cut to the master's framing at the end; 0/3 without it |
| master wrapped in MiniMax's `<Subject N>` definition (M7) | still a cut target: at 1280 seed 7103 cut to it at 2.88 s (2.83 s in the plain `<Picture N>` form); 3/10 takes with the master cited, 0/3 without |
| Cast pictures wrapped as `<Subject N>` (M7) | neutral: same people, motion 12.2 vs 13.45, no freeze |

So the rule is about inclusion: decide per cut which pictures go in, by their job and lifetime, and write that down.
The guide's Subject form is tidiness; it does not change the result.

### Why a fixed set photo in every shot fights a travelling camera

A picture in the references is a target. A set photo whose framing is not this cut's framing is something the take can
cut or dissolve to (M2, M7). On a long continuation, a fixed first reference pulls a travelling background back to where
it started (the beach walk), and the extend study's "Long" mode, which re-anchors to clip 1 every third clip, holds two
minutes but "the re-anchor moves the action once, then it is steady: the person ends up in one steady pose and place
that is not the prompt's" ([u46_extend.md](u46_extend.md), U46e). A fixed anchor means "always stay here".

### Dylan's words (2026-11-07 evening)

- 19:49: "the jump between the 2 scenes is a bit rough; it needs that last frame as a reference in Ref2V, talking about
  it being the first frame of the next ... also that needs to be a rule"
- 19:51: "that scene needs a proper 'this is the set' type reference for each of the shots to play from... we also now
  have a nice bowl effect shot too for the bowl fx... so maybe the path is to generate the first scene, get the
  reference, generate the next etc?"
- 20:1x: "there's the 'carry low drift forward' that works to some extent but earlier context drops out of context...
  rolling references getting added in with continuous shots help... we have a video continuation workflow but it's
  missing this important thing"
- 20:2x: "we need some kind of reference conditioning that changes as we go through and keep generating... the agent
  [should] look at what generated in their latent-resolved pieces... the clown lady video walking down the beach ... the
  landscape that rightfully drifted ends up reverting to the initial reference background conditioning... we need to
  agentically know what references are injected and change those as things evolve and continue over long horizons"
- 20:3x, a pasted ChatGPT brainstorm (fact-checked before use; see [factcheck_subject_vs_picture.md](factcheck_subject_vs_picture.md)):
  five memories; "the renderer proposes pixels, the world controller commits reality"; planned vs observed state;
  accepting a take is not committing a memory update; pixel continuity vs world-state continuity; never retire world
  facts because their pictures stopped being useful; costume is event state; an Extend Director with a reference ledger.
- 20:4x: "let's work on the things we can work on over here that make sense given our Snap project at least, while
  [the second machine] can pick up the training and scene-building training data pieces" (so this design exposes a
  renderer-independent slot for trajectories and terrain, built later).

### Constraints

- A render never writes state. Story, style and canon writes by an agent are always proposals.
- Warn, never block.
- The front-end calls no model itself; the critic is an agent job.
- FL2VA (first/last frame) and Ref2VA (reference pictures) are separate H3 checkpoints with separate Pika adapters
  (verified). The existing extend path runs the FL2VA pair with no pictures for every take, a Ref2V take included.

---

## 2. Recommendation

Do not build a new memory engine. Four of the five memories already exist as versioned records on two clocks: identity
(the Cast record), the world (the location record: plan, master, camera marks, named plates with poses), event state
(the story clock: who carries what, costume, position and facing, folded beat by beat in story order) and recent
context (the take itself plus the extend latent carry).

What is missing is narrower and cheaper:

1. **A reference ledger** on every take and every extend segment: per conditioning input, where it came from, what job
   it does in the prompt, which model family consumed it, how long it is valid and when it was retired.
2. **A `relation` field on a cut** (continues / new angle / reverse / insert / new scene) that picks pixel continuity or
   world-state continuity, plus an `opens_on` source the compile fills and hashes, so only the downstream cut goes stale.
3. **An observed-state layer** the critic writes beside the planned state, which never becomes canon except through an
   explicit "keep as canon" proposal.
4. **One new record kind, `effect`**, for a picked effect's key frame, words, takes and known side effects.
5. **A generator-adapter boundary in extend**: the front-end decides what a continuation needs; one place decides what
   FL2VA or Ref2VA can accept.
6. **A documented, renderer-independent `motion_plan` format** for trajectories, terrain and camera paths, reserved now
   and read only when a file exists.

The split that makes this cheap: some things exist and only need a name (identity, world, event state, camera pose per
plate); some exist but are unnamed and thrown away (the per-take reference package is already built with a role, a why,
a source and an order, then the provenance is dropped when the references are written); and only a few are new
(observed state, the effect record, the relation field, the ledger's validity and retirement, the motion plan).

---

## 3. The record of world state

### What is stored

| memory | stored today | what the design adds |
|---|---|---|
| identity (who) | Cast record: name, trigger, details, LoRA, pictures with roles, voice, states with pictures | nothing structural; the ledger cites a Cast picture as a `subject` job, never as a frame |
| world (where, layout) | location record: plan, master, camera marks `{x, y, heading, eye_height, lens}`, plates `{file, from, role, pose, camera, use}` | a plate gains `side` (a camera mark or a word: "south", "reverse") and `set: true` when it is the side's "this is the set" reference made from a picked take's look; walkthrough plates stay (never retired) |
| event state (who holds what, costume, damage, positions) | story changes on beats, folded in story order; costume is story state, not identity | an **observed** layer: critic rows carry `observed [{entity, path, value, frames, confidence}]`; a read-only `planned_vs_observed` diff; a disagreement leads to a proposal or a retake, never a write to the story |
| camera state | cut camera words, the cut's plate (camera picture), plate pose, transition | `relation {to, kind}` and `opens_on {cut, at: "out" or seconds}` |
| recent context | the take (frames, sound), the extend carry (22 held frames) | the per-segment ledger on extend takes |
| effects (new) | a scene note | the `effect` record kind |
| trajectories, terrain (new) | none | the `motion_plan` format, reserved |

### Roles and lifetimes

Each reference has a job and a lifetime. In the brainstorm's terms: **who** and **look** are permanent (valid for the
production; the Cast picture is the identity root); **where** is rolling (valid for a stretch of segments, then
replaced or retired); **what** (an effect, a prop state) is beat-scoped; **just before** is the latent carry.

### The reference ledger

One entry per conditioning input, written on the take beside its record pins:

```
ledger: [{
  id: "L1",
  file: "<previous cut's take>_f89.png",
  source: {take: "<take>", frame: 89, at_s: 3.708, crop: null},   // or {record: "location:…@9", plate: "P1b"}, {record: "cast:…@5", picture: "front"}, {record: "effect:…@1", key: 0}
  role: "opening",      // opening | continuity | identity | state | set | camera | place | key | effect | map | voice | music
  job: "picture",       // picture (a standalone <Picture N> frame anchor) | subject (cited inside a <Subject N> definition) | first_frame | latent (the carry) | keystone | audio
  cite: "<Picture 1>",  // what the prompt calls it; "<Subject 2> via <Picture 3>" for a subject
  purpose: "the shot opens exactly on C1's out-frame (Dylan, 2026-11-07 19:49)",
  family: "ref2v",      // ref2v | fl2v | t2v, read from the pipeline at apply time
  valid: {scope: "cut"},           // cut | beat:<id> | segment:<k..m> | production
  mechanism: {slot: "ref_images", index: 0},   // or {node: "first_frame"} | {carry: 22} | {keystone: "front", source_clip: 1} | {slot: "ref_audios", index: 0}
  retired: null         // or {at: "segment 4", why: "the camera left the stall: replaced by L7"} (extend only)
}]
```

Every field except `job`, `valid`, `mechanism` and `retired` is already computed during compile; the ledger stops
throwing it away. Each entry whose source is a take frame adds one hash to the cut's `from` pins, so the existing stale
check can say "its opening frame changed (C1's active take or trim moved)" with no new dependency engine.

What it buys: the four questions an agent asks before re-injecting ("what is in there, as what, from where, still
valid?") are answered from the take; the critic can score frames against each cited picture by its job and say "the take
dissolved to the master, drop it" instead of "prompt ignored". What it costs: about 1-2 KB of JSON per take (6-9
entries).

### Three clocks

Records version on the production clock (`cast:kon@5`). A beat is a moment on the story clock (story order, not telling
order). Every time inside a take is **take seconds at 24 fps** (words, dial keys, the out-frame). The edit maps take
seconds to the timeline through the cut's trim. So `opens_on {at: "out"}` resolves to the previous cut's trim out-point,
then to a frame by one formula. In one hand-made test the out-point 3.75 s was opened on frame 89 (3.708 s), a one-frame
slip that is invisible at a cut but should be one shared formula.

---

## 4. Two acceptance paths

1. **Accepting a take** changes which take the cut uses and which out-frame downstream cuts open on. It writes nothing
   to Cast, location or story. It may be the person's pick or, under a collaborate/automatic setting, an agent's pick
   recorded as the agent's.
2. **Committing a state update** is a record version or a story change: "keep as canon" from a take, a new plate or set
   reference (a location version), a saved Cast state, "keep as effect". Story writes are proposals at every setting.
   Record writes (a set reference) stay direct and versioned, because a picture is something the person can see and undo.

The **evidence hierarchy** follows: canon (story and record versions the person made or accepted) > accepted
observations (observed rows on a take the person made active, or that passed the critic with no high note) > unverified
candidates (observed rows on takes nobody picked, every agent proposal). The compile reads only the first tier; the
second is what `planned_vs_observed` surfaces as "the picture says X, the story says Y"; the third is visible and
nothing more. Only Dylan commits canon; agents propose with frames as evidence.

Rejected: letting the critic's observations write story state under automatic (the brainstorm's "world controller
commits reality" read literally). It contradicts proposals-at-every-setting, and the critic's floor is 24/29 with 0
false alarms on stills but unmeasured on counts in video. A wrong observation committed as canon would fold into every
later cut. It becomes right only after a measured fixture floor on video observations and an explicit decision by Dylan.

---

## 5. Continuity policy: pixel versus world-state

`relation.kind` decides. It is set by the agent or person and linted when missing (for example when two consecutive cuts
share a plate); it is never guessed into a render, because a guess picks the wrong cut on a cross-cut.

| relation | first frame | set / place pictures | Cast | key picture | words the compile adds |
|---|---|---|---|---|---|
| `continues` (same camera, action carries on) | the previous cut's active take at its trim out-point, role opening: "<Picture 1> is the first frame: the shot opens exactly on it" | none by default: the frame holds the set | Subjects | none on an action cut (M6) | the beat as one ordered action from the previous cut's end state |
| `new_angle` (same side, new framing) | none (the storyboard frame stays the board) | the side's set reference as camera: "this cut's camera on the empty set: keep its framing, set and light"; master left out and said (M2) | Subjects | none on an action cut | pose, props and timing carried in words ("At the start of the cut, …") |
| `reverse` (the other side) | none | the reverse side's set reference; no frame of the master shot (M7: a picture of another framing is a cut target however it is cited); light and palette in words and in the dial rows the wide used | Subjects | none on an action cut | as new angle, plus screen direction ("Kuroe stays frame left": one take put him frame right and was rejected for it) |
| `insert` | none | a frame of the master cut as continuity | Subjects | a held pose may keep its key ("keep the key only when a 2 s build then a hold is what the cut wants") | as today |
| `new_scene` | none | this location's side reference as camera; the master only when no plate exists | Subjects | none | none |

**What replaces "the set photo in every shot"**, in order of evidence:

1. **For a continuing camera, the out-frame is the set photo.** It is the one picture that is the right framing, light
   and pose at once, so it is a legal frame anchor (the guide's "first frame" case). A hand-made test proved it on one
   pair: the second cut's frame 0 matched the first cut's out-frame in camera, set, light and pose, and the jump was gone.
2. **For a new angle, a camera-matched set reference**: the side's plate with people removed, made from the picked
   look. It is still a frame anchor, but of this cut's framing, so there is nothing else to cut to. Proved on one cut
   (the stall matched; the one failure was screen direction, a words problem).
3. **The set's identity by words and by the critic, never by a picture of another framing.** On a single-camera cut a
   place picture whose framing is not this cut's stays out of the references whatever the syntax (M7). The set's
   topology rides in words from the location record (notes, the side's landmarks, what stays empty) and in the critic's
   cross-view check against the master.

A companion result binds the storyboards: on a bad board no wording removed a duplicated bowl (3/3 kept it), so the
inclusion policy's first line is "fix the pictures before video".

Rejected policies: "always the set photo as a picture" (13/18 takes cut to it at 640); "words only, no pictures of the
place" (loses the set within two cuts; it is what the extend path does today); "always open on the previous out-frame"
(a reverse angle opening on the forward frame is a jump cut or a morph). The relation field is the smallest thing that
picks the right policy per cut.

---

## 6. Chain mode

Chain is a **derived order, not a mode switch**. The dependency edges come from `opens_on` (C2 waits on C1's approved
out-frame), the insert's master frame (the insert waits on the wide), a set reference's source (C4 waits on the side's
set reference, which waits on C1's pick) and cited effects. A `render_order` walk returns groups: cuts with no unmet edge
render in parallel; a cut whose edge points at a cut with no active take reads "waits on C1 (no active take yet)". An
upstream re-take makes downstream cuts stale through the existing pins, and the existing staged re-run (plates, then
boards, then cuts) re-runs them under the person's chain setting, stopping before video by default. "Make this the set"
is offered when a take is accepted on a cut whose side has no set reference yet.

**Cost:** serial time only along edges. On the duel sequence the planned edges are the sweat-drop insert off the clash
wide and at most one more, so one or two waits of about 3 minutes each (about 2.3 minutes a take at 1280x704 with a 2x
upscale). On the test short there are three edges. The person's pick is the real cost: under an "assist" setting every
edge waits for a human.

Rejected: a strict serial lock (warn, never block; Dylan renders cuts out of order on purpose, and a lock would make
parallel new-angle cuts wait for nothing). Rejected: no chain support (hand-chaining took four tool calls per cut, the
picture had no role, and nothing flagged C2 when C1 changed).

---

## 7. Effect references

A record kind `effect`, cited by a cut's `effects` list:

```
effect:<id>@n  body: {
  name: "gold burst from the bowl",
  keys: [{file, from: {take, frame: 0}, note: "the peak board, jacket recoloured"}],
  words: "a small flame of light kindles in the bowl while his eyes widen, then a full burst of gold light erupts upward out of the broth, lighting his face from below",
  sound: "a soft rising hiss like steam rushing up out of the bowl",
  takes: [...],
  side_effects: ["a deep rushing whoosh wording gave an instant sub-bass boom at 0.85 s", "the key's palette pulled the jacket khaki until recoloured at the source"],
  dials: [...optional dial rows that made it...],
  rating, by
}
```

"Keep as effect" writes version 1 from a take (its key board if it has one, else the frame), the executed prompt's
relevant sentence and the take id. A cut citing it gets the words in the action sentence and the sound line. **Its
picture goes into the references only on a held-pose cut** (M6: the key is reached by about 2 s and held); on an action
cut only if a planned check surprises (the key cited only inside a Subject, no frame sentence; pass means motion after
2.5 s ≥ 10 and key copy score < 0.85 throughout; expected to fail given M7). The ledger entry is beat-scoped.

What it buys: the bowl burst (and the duel's clash) become reusable with their known traps attached. Rejected: a library
card (prompt phrases with no picture provenance, so a re-made key could not be followed) and a scene note (readable, but
nothing can cite it).

---

## 8. The Extend Director and the generator-adapter boundary

Build the boundary and the per-segment ledger now (they are a correctness fix and plumbing); build the decide-and-
re-inject loop after the rolling-references study, with its defaults read from that study's table.

### The adapter boundary

The front-end decides what a continuation needs as a ledger (identity Subjects, a "where" entry, the carried frames, the
audio decision, the rolling words). One adapter per model family decides what that family accepts:

| ledger job | FL2VA pair | Ref2VA pair |
|---|---|---|
| `latent` carry (22 held frames) | yes (measured, U46b-f) | **unknown until U46h arm B0**: "whether the Ref2VA checkpoint honours a masked carry is measured, not assumed" |
| `first_frame` | yes | no (Ref2V reads no first frame; the opening rides as `<Picture 1>`) |
| `subject` pictures (identity, set) | no: identity rides as words | yes, up to 9 pictures, each about 880 tokens at 1280x704 |
| `keystone` (Long's re-anchor) | yes | unknown (U46h's rolling keystone arm is FL2VA) |
| `audio` references | not wired | up to 3 |

(Ref2VA limits as documented: 9 pictures, 3 videos, 3 audios, 12 files.)

Today extending a Ref2V take silently switches it to the FL2VA pair and drops its pictures. The first fix changes no
behaviour: the family is picked from the take's own preset, and the answer says per ledger entry what was carried or
dropped ("Mio's pictures: dropped (fl2v carries no reference pictures); her Cast line is in the words"). Continuing a
Ref2V take on the Ref2VA pair with Subjects waits on U46h B0.

### The loop, per segment k

1. **Plan.** The segment's ledger is the previous one with retirements applied, plus this stretch's words (the rolling
   prompt: the beat text for the time the segment covers, or the continuation sentence; never the whole scene again).
   Identity entries are valid for the production and never retired. The "where" entry is valid for segments k..m. The
   carry is latent (or a keystone on Long's anchor clips).
2. **Render** through the family adapter.
3. **Look** (the critic as an agent job, on the segment's frames): prop and people counts; the copy score of the last
   second's frames against each cited picture (a high score against the "where" picture on a travelling shot is the
   beach-walk revert in a number); landmark presence versus what the "where" entry claims; identity against the Cast.
4. **Decide by intent.** Keep the ledger; or **promote** a frame (a critic-passed frame of this segment becomes the new
   "where" entry and the old one is retired with a why); or **retire** without replacement (the shot has left the set;
   words carry the place until a new one is worth a picture); or **re-inject** (identity drift above a threshold: re-cite
   the Cast front picture, or push it through a keystone on FL2VA).
5. **Write** the segment's ledger onto the extend take and go on.

### Guards

- Promote only a frame with no high critic note and an identity score above the Cast floor (the character score's 0.45
  line).
- The Cast picture is the identity root and is never replaced by a rendered frame; only the person saves a Cast state.
- Every promotion is undoable: re-run from segment k with the ledger as it stood.
- A promoted "where" frame is never written into the location record (that is a plate, a version the person sees).
- **Never retire a world fact.** Retirement applies to the ledger's picture, not the place: the lifeguard tower is still
  in the plan; only its picture stopped being the right conditioning.
- Keep the audio decision separate from the picture decision (cutting the audio carry at anchors fixed a drone, U46g).

### What waits on U46h, what does not

Waits: the default arm per family (fixed set picture, isolated set picture, rolling set picture, agent-selected, or
Family A's rolling keystone); whether Ref2VA honours the carry at all (if not, Family B is dead as a continuation and the
Director rides on FL2VA with keystones as the only picture mechanism, so "rolling where-reference" means "rolling
keystone"); the promotion threshold (what copy score against the set picture means "stuck"). Does not wait: the adapter
table and the honest answer, the per-segment ledger shape, a `segments` plan input on extend, and the critic's segment
observation.

Rejected: waiting on U46h for all of extend (hides a live bug, the silent family switch); building the whole loop now
(every default would be a guess; U46e already showed the fixed anchor resets the action, and there is no number yet for
the rolling arm).

---

## 9. The critic as observer

The critic's nature does not change; its packet and write-back do. The packet gains the take's ledger with each cited
picture, the executed prompt with its prop list, frames every 0.5 s plus first and last, and for a cut pair the planned
relation ("continues: B's frame 0 should equal A's frame at 3.75 s") and the landmarks the location side should show.
The critic writes `observed [{entity, path, value, frames, confidence}]` (for example Kuroe's left hand holds the monocle,
frames 58-90, 0.8), a `destination` for a cut or dissolve ("f97 → the master, 0.92") and per-ledger-entry copy scores.
`planned_vs_observed` diffs observed against the planned end state for that beat and lists disagreements with frames;
the agent proposes "keep as canon" or a retake.

A real disagreement already turned up: one cut ended with the monocle on the character's chest while the next opened with
it raised. Fixtures are real SFW footage of original characters (a duplicated bowl with a dissolve to the master, a cut
to the master at about 3 s, that monocle pair, a clean continuing pair, a car crossing a wide). A reading rule ships only
after the fixtures hold.

No local VLM pre-screen in the front-end (it calls no model, and a VLM is a download that is Dylan's call). The copy
score is the one measure the front-end computes itself; counts and landmarks stay with the agent.

---

## 10. A renderer-independent, versioned motion-plan interface

Reserve a record kind `motion_plan` and define its body now; build the reader when a renderer (for example a separate
map-and-trajectory training project) writes a file.

```
motion_plan:<id>@n  body: {
  name: "Liora's run to the circle",
  location: "location:<id>@n",                 // the plan whose frame this uses
  frame: { origin: "plan top-left", x_right: true, y_down: true, units: "m",
           m_per_plan_unit: 42.0,              // from the location's scale anchors or the renderer's own calibration
           up: "+z" },
  story: { book: "book:<id>", beats: [{beat: "b4", t0_s: 0.0, t1_s: 5.17}] },   // story time ↔ plan time
  terrain: { kind: "heightmap" | "mesh" | "flat", file: "<id>_terrain.png|.glb", origin: [x, y], cell_m: 0.25 } | null,
  tracks: [{ entity: "cast:<id>", samples: [{ t: 0.00, x, y, z, heading_deg, speed_mps,
             action: "run" | "jump" | "stand" | "cast", phase: "windup" | "release" | "land", contact: ["L", "R"] }] }],
  camera: [{ name: "C4", samples: [{ t, x, y, z, heading_deg, pitch_deg, roll_deg, lens_mm }] }],   // its own track, never derived from an entity
  source: { tool: "…", version: "…", by: "…" },
  notes
}
```

The existing camera pose (`{x, y, heading, eye_height, lens}` in plan fractions) maps onto it by
`x_m = x * m_per_plan_unit`, `z = eye_height`. A future reader would draw the paths on the set plan, compile blocking
words for a cut from the track at its beat ("Liora runs from the stair door to the circle and plants her feet facing
north"), pass the camera path to any adapter that takes one (none today; an H3 ControlNet route is unmeasured), and give
the critic the expected landmarks per camera sample. The advice behind the shape: world-space trajectories, not arrow
pixels; the camera gets its own path; evaluate in world units from an unseen camera.

Why a record kind and not a file beside the location: versions (a re-drawn path is v2), pins (a cut that compiled its
blocking from v1 reads stale on v2) and discoverability. Why not inside the location record: a location is permanent
world; a motion plan is event state on the story clock, and several plans share one location.

---

## 11. Options considered for the record

| option | why it lost | when it would win |
|---|---|---|
| **A. Extend the existing records and story clock, add an observed layer, a ledger and two kinds (recommended)** | (wins) it changes nothing about what runs; it names it | every production; loses only if positions change every second rather than per beat, which the motion plan covers |
| B. One monolithic `world_state` record (identity, world, events, camera, context in one versioned body) | every pin, swap and staged re-run is keyed on Cast and location records; a second home for the same facts is a two-sources-of-truth bug; it versions event state on the wrong clock (a costume change is a story fact, not a production version) | a production with no story structure |
| C. Retrieval memory over frames (a vector store of every rendered frame, as in REEL, A²RD, "More Memory, Better Videos?") | generation errors would become facts; the paper's own result is that retrieval "can also reinforce obsolete appearances after the story changes them" (the beach-walk failure by another route); needs a model service inside the front-end | later, as a critic aid ("find the frame most like this one") |
| D. External world model / 4D scene (splat, or an Oneira-style state table rendered to coarse conditioning) | no H3 route consumes coarse conditioning video today; it is a separate research track | it is exactly what the `motion_plan` slot is reserved for |

Literature used as corroboration of the shape, not as evidence about H3 (12 of 16 papers in the pasted brainstorm were
verified): StateAgent (planned vs observed state in continuation, SCS-All 45.2 → 69.3), "More Memory, Better Videos?",
Keepsake (fixed-capacity spatial memory with relational eviction), Oneira (an explicit world state managed by a coding
agent) and ATAS (scene memory in a shared frame plus event memory indexed by scene, event and time).

Decision criteria: (1) does it respect "a render never writes state"; (2) does it reuse the existing pin/stale/re-run
machinery or duplicate it; (3) is its behaviour measured or guessed; (4) does it unblock the next production soon.

---

## 12. Risks

1. **The out-frame is mid-motion** and the next cut opens blurred or holds it (Ref2V treats Picture 1 as the exact
   opening, so the first 2-6 frames inherit it). Warning: the next cut's frames 0-12 keep a copy score ≥ 0.95 against
   Picture 1 for more than 0.5 s, or the first frame's Laplacian variance is under half the take's median. Action: a lint
   offers the nearest sharp frame within ±3 frames; the exact out-frame stays the default.
2. **The Subject form read as permission to put the master back in.** M7: 3/10 takes cut to it with it cited, 0/3
   without; the syntax does not help. Action: the inclusion policy is an error-level lint whatever the citation form; the
   ledger's `job` is documentation, not permission.
3. **A chain stale storm.** Re-taking C1 after C2, the set reference and C4 were made from it re-flags all three. Action:
   re-runs stop before video by default, and the answer names the cost first ("~9 min").
4. **The effect picture freezes an action cut** (M6: 4/4). Warning: copy score against the effect key ≥ 0.85 from about
   1.5 s and motion after 2.5 s under 6. Action: the picture is cited on action cuts only if the check passes.
5. **Ref2VA ignores the masked carry** (U46h B0). Warning: a frame-difference spike at the join well over 3x the median,
   and DINOv3 layout disagreeing with the previous clip's tail. Action: the Director is designed only after U46h; the
   adapter table makes that a data change.
6. **Observed-state false positives** (counts on 30 px figures at 640). Action: observations carry confidence and
   frames; under 0.7 they are listed, not offered as proposals; the fixture floor (24/29 on stills) gates the rule.
7. **The reference budget overflows** (two Casts × two pictures, the set reference, an opening, an effect and a third
   character's picture is 8-9 entries against a cap of 9). Action: a fixed drop order: place/master first, then map, then
   an effect's picture, then Cast back views; never the opening, the camera or a Cast front.
8. **A new record kind missed in one of its enumerations** (client and server keep separate lists). Action: list every
   enumeration first and change them in one commit with tests.
9. **Clock confusion** between take seconds and the edit timeline (a 3-frame slip). Action: `at` is take seconds; "out"
   is the default; the compile prints both clocks.
10. **Chain mode slows a production more than it helps** if many new angles are marked `continues`. Action: the lint
    only suggests `continues` when the plate is shared and the transition says so; new angles are the default.

---

## 13. Assumptions

Verified: the compile already builds a reference package with role, why, source and order, capped at 9; the stale check
already compares source hashes; trims are the edit clock and words/dial keys are take seconds; "keep as canon" is the
only take-to-story path and is a proposal; extend runs the FL2VA pair with no reference node and drops Cast pictures to
names; MiniMax's Ref2VA guide distinguishes `<Subject N>` (content, cited inside the definition) from a standalone
`<Picture N>` (a frame anchor); M6, M2, M1 and M7 as measured; one hand-made out-frame continuation and one set reference
made from the picked look each fixed their cut.

Unverified:

- A Subject-cited picture with no frame role does not freeze an action cut. M7 makes this unlikely, and the design
  already assumes it false.
- Ref2VA honours a masked latent carry (U46h B0, running).
- The out-frame index formula is the same in the edit and in frame extraction. If not, a one-frame duplicate or skip at
  every chained cut; the fix is one formula.
- A continuity opening picture and the first-frame brick do not fight over `<Picture 1>`.
- The duel sequence needs at most two chain edges.

Planned checks before building the record (about 2 hours, mostly CPU): the effect-picture-as-Subject freeze test (one
640 render pair, about 3 minutes); the out-frame formula; which picture becomes `<Picture 1>`; every place a new record
kind touches; counting chain edges on real books; the budget drop order on a crowded cut; whether the critic packet can
read the ledger with no new storage; whether old takes' role strings can be migrated into ledgers; and U46h's verdict.

---

## 14. Rollout order

Three stages, each a day or two, ordered by what the next production touches first:

1. **Continuity source and set reference:** `opens_on` with its stale key and the critic's cut-pair check; the set
   reference per side; the measured corrections (key picture never on an action cut; master out of single-camera cuts
   by default and said, as an error-level lint whatever the citation form).
2. **The world-state record:** the ledger, `relation` and the continuity policy, cite by job (Subject form as tidiness),
   derived render order, the `effect` kind, the observed layer, the extend adapter table with the honest answer, the
   `motion_plan` kind reserved, and "never retire a world fact".
3. **The Extend Director** (after U46h): the segment loop, promotion guards, rolling prompt text, and family defaults
   from U46h's table.

A reader for `motion_plan` comes when a file exists. The duel sequence's twelve cuts are almost all new angles (each
at most 5.17 s), so it needs world-state continuity far more than chained out-frames, and the Director is not on its
path.

---

## 15. Open questions (Dylan's calls)

1. **For the duel sequence, render cuts in order, each waiting for the pick of the one before, or side by side from the
   set references with only the sweat-drop insert waiting on the clash wide?** The cuts are almost all new angles, so
   in-order buys little continuity; each wait is about three minutes of render plus a look. The design leans side by side
   with the insert chained, and chaining only where a cut truly continues.
2. **When a long continuation finds the set has rightly changed, may the agent swap the "where" picture by itself inside
   that job, logged and undoable, or should every swap wait for Dylan?** This is the line between accepting a take and
   changing what is true. The swap never touches the location record; it only changes which picture the next segment
   reads. The design leans: let it swap inside the job, logged and undoable, with the Cast picture never swappable.
3. **Trajectory and terrain format: the set plan as the ground frame (metres via a `m_per_plan_unit` number from the
   plan's scale anchors), or the renderer's own world units with a one-time calibration to the plan?** The design leans
   metres in the plan's frame, so one drawing on the set plan is one path. Nothing blocks on it now.

Not asked, because measurements or earlier decisions settle them: Subject form for Cast pictures (M7 neutral: tidiness);
a continuing cut opens on the previous cut's out-frame (19:49); every cut plays from a set reference per side and good
effects become references (19:51); extend gets rolling references after the U46h lab (19:53, 19:56); story, style and
dial writes are proposals; the master stays out of single-camera cuts and a key never goes into an action cut; re-run
chains stop before video by default; Pika at 6 steps; judge with sound on.
