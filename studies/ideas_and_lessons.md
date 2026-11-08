# Ideas and lessons: continuity, references and prompts

Research queue and synthesis lessons from 2026-11-07 on long-horizon MiniMax H3 / Pika H3 Flex video, measured with
our own takes (prompt-craft measurements, lab checks, a re-run of a test short, a dials bake-off and a fact-check).
**Status: the lessons are measured; the queue is open.** Rolling references in extend
([u46h_rolling_refs.md](u46h_rolling_refs.md)) is in progress.

Related: [world_state_design.md](world_state_design.md), [prompt_craft_rules.md](prompt_craft_rules.md),
[key_picture_freeze.md](key_picture_freeze.md), [factcheck_subject_vs_picture.md](factcheck_subject_vs_picture.md),
[u46_extend.md](u46_extend.md).

---

## World state, not memory of a movie

From a ChatGPT brainstorm Dylan pasted (2026-11-07 evening), kept only where the fact-check and our measurements
support it:

- **Five memories:** identity (who), world (place and layout), event state (who holds what, what broke, costume), camera
  state, recent visual context. Costume is event state, not identity.
- **Accepting a take is not committing a memory update.** "The renderer proposes pixels; the world controller commits
  reality." Keep planned state and observed state apart; generation errors never become facts.
- **Pixel continuity vs world-state continuity.** A continuing camera opens on the previous out-frame; a new angle
  carries pose, props and timing, not the image.
- **Don't feed the set photo to every shot as a frame.** Masters and key pictures are jump and freeze targets (M2, M6).
  Keep the set's identity available another way: a camera-matched view, words, and the critic.
- **Never retire world facts because their pictures stopped being useful.** The beach tower is behind the camera, not
  gone.
- **A separate track:** a map-arrows LoRA and a six-view to 4D-splat idea: world-space trajectories, terrain and a
  separate camera path, judged by world-unit path error from an unseen camera, reconciled locally (update only what
  changed; commit an event without claiming its exact geometry). It does not block the front-end work; the design
  reserves a renderer-independent `motion_plan` format for it.

---

## Things to try (research queue)

- **End on a pose the right way.** MiniMax's guide lists last frame and keyframe as legitimate standalone Picture roles.
  Test FL2VA with a real last-frame input against Ref2V with a key picture for "land on this pose" beats; the
  key-in-references route freezes (M6). Also: the key cited only inside a `<Subject N>` definition (expected to fail).
- **Rolling references in extend (U46h, running).** Family A (FL2VA latent carry: Standard, Long, a rolling recent
  keystone, a triggered re-anchor) vs Family B (Ref2VA: no references, identity only, fixed scene, isolated set, rolling
  set, agent-selected), on travelling, return-trip, state-change and costume-change scenes.
- **The burst's boom.** Re-word "a deep rushing whoosh" to "a soft rising hiss like steam rushing up out of the bowl",
  and add an audio critic pass for short sub-bass transients.
- **Identity under the Subject form vs a plain Cast line** at a framing where faces can be judged (M5 only checked
  people counts).
- **Master as a cut target at 1280 with more seeds** (3/10 is a small n), and whether a camera-matched set view (not
  the wide master) keeps the place without becoming a target.
- **Board gate.** Treat a duplicated prop or person on a storyboard frame as high severity (it spreads); count props on
  every board.
- **Chain vs side by side for the duel sequence.** The design leans side by side from set references, chaining only the
  sweat-drop insert.
- **Effect references as clips** (the bowl burst: key frame, timing, sound), cited in a later cut; test whether its
  picture is safe on held-pose cuts only.
- **Planned vs observed state.** The critic writes observed entities (prop holders, positions, landmarks) and a
  read-only diff against the plan.
- **Motion-plan interface.** World-unit trajectories, terrain and a separate camera track, once a renderer writes a file.
- **Share the measured prompt rules with the Astrid project** (pending Dylan's OK; share the measured rules, not the
  pre-measurement list).
- **Public ComfyUI-MAINodes main:** fix the "Pika + reference pictures" preset's checkpoint/adapter pair (pending
  Dylan's OK).

---

## Synthesis: continuity, references and prompts

Each line: the lesson, then the evidence.

1. **Pictures beat words; fix the picture first.** A test short's duplicate bowl was born on the approved storyboard
   frame. On a clean board every prompt drew one bowl (6/6), even naming the bowl four times; on the old board no
   wording removed it (3/3). Boards passed on "no high" notes, so a duplicated prop stayed "low" and spread to every
   take: count props on boards before any video.
2. **A picture in the references is a target, whatever its syntax.** A key or peak picture on an action cut: the take
   turns into it by 1.5-2.5 s and freezes (4/4; motion 3.3-6.5 vs 12.7-13.3). The location master is a cut target (3/10
   at 1280, 13/18 at 640, 0/3 without). Wrapping it in `<Subject N>` (MiniMax's guide) is neutral and does not stop it
   (M7). Inclusion is the lever.
3. **Continuity across a cut comes from pictures.** C1→C2 jumped in set, light and pose until C2 opened on C1's
   out-frame (then seamless). A new angle must carry pose, props and timing, not the old image. Keep one set reference
   per location side, made from the picked look.
4. **Count props and score late frames against every cited picture after each take.** Prop counts and copy scores caught
   every cut, freeze and second bowl at 0.92-0.99. One of our own picks missed a duplicate bowl because framing was
   argued before counting.
5. **Model families are real.** FL2VA and Ref2VA are separate H3 checkpoints with separate Pika adapters (verified).
   The public ComfyUI-MAINodes main "Pika + reference pictures" preset loads the FL2VA pair (a bug; the development
   branch is right). The extend path continues Ref2V takes on the FL2VA pair with no pictures.
6. **Long continuation needs references with roles and lifetimes:** who and look permanent, where rolling, what
   beat-scoped, just-before as the latent carry; plus a ledger, guards against promoting a bad frame, and rolling prompt
   text. A fixed first reference reverts a travelling background (the beach walk); Long's clip-1 anchor loops the action
   ([u46_extend.md](u46_extend.md)).
7. **Verify pasted research before building.** In the ChatGPT brainstorm, the Subject/Picture and checkpoint-family
   claims held, 12/16 papers held, one study did not exist, and our own first reading ("Subject syntax may well be why")
   was wrong once measured ([factcheck_subject_vs_picture.md](factcheck_subject_vs_picture.md)).
8. **Sound.** Burst beats bring booms: the critic caught a -6.9 dBFS sub-bass hit that an audio-tagging pass missed, and
   "a deep rushing whoosh" is the likely source. An earlier "16 dB" train-vs-hiss claim did not reproduce.
9. **Dials.** Look dials need k=6 on Ref2V and audio set to protect; weather dials change the sound anyway; close-ups
   need about half the wide's strength; a frame of the master shot gives an insert continuity in one call.

### From the rolling-references lab (U46h, in progress)

- **A face detector finds a face on an empty plate.** SAM 3.1 asked for "face" returned a 1.0 x 0.47 box over the sky and
  pier of a cleaned beach plate, so the guard said "a person is still in the cleaned picture" and the fixed-set arm ran
  with no set picture. Fix: count only face boxes under 10 % of the picture and lean on the person-mask fraction. A
  guard that refuses must show its numbers in the ledger (it did, which is how this was caught).
- **Frame caches keyed by run and clip name serve old frames after a re-render.** A real tool should key extracted frames
  by the video file, not the run name.
- **A 1280x704 H3 clip costs 96-102 s on an RTX 5090** (59 s sampling plus text encoder and decode, with another process
  holding 4 GB of the card), not the ~30 s measured earlier at 832x480. Budget lab grids from a measured first clip, and
  drop secondary scenes to 832 rather than dropping arms.
