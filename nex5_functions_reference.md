# NEX5 — Functions Reference (the honest map)

As of **2026-10-05, about 02:45 SAST**. Built read-only from:

- **Code:** nex5 HEAD `fa35b6b`.
- **Live process:** NEX pid 15153, booted 02:14:05, launch env from `/proc/15153/environ`.
- **Boot log:** `/tmp/nex5_soak.log`.
- **Databases:** read-only SQLite queries on `nex5/data/*.db`, taken while NEX was running, so the counts were moving as they were read.
- **Prior verdicts:** cited from this research log by file name (mainly `dev_queue_2026-10-04_results.txt` and `mental_factors_inventory_alobha_spec.txt`).

Nothing was edited and nothing was probed.

**Not read, by instruction:** the warrant tables (`belief_survival`, `belief_use_day`, `belief_use_days`, `warrant_record_cursor`) and `fountain_retrieval_log`. The WarrantRecorder appears below only as a running loop; its internals were not inspected.

**Status tags used throughout**

| Tag | Meaning |
|---|---|
| PROVEN-READS | Flag on; beat base A **and** a length-matched neutral control C in a blind-rated A/B/C probe. |
| ARMED-UNPROVEN | Flag on, code runs or injects, but no A/B/C uptake result is on record. |
| ARMED-INERT | Flag on, but measured or structurally has no effect. |
| DISARMED | Flag off after failing its control (reason given). |
| DARK | Built but never armed. |
| DEAD | Nothing live reaches it, or it is a permanent no-op. Removal candidate. |

A "faculty" here means a **prompt block for the 3B voice** unless stated otherwise. No faculty below is claimed to be present in NEX, or experienced. A PROVEN-READS result means one thing only: when its text is in the prompt, the 3B's output measurably changes in the intended direction.

---

## 0. Top-level summary

1. **One process, one voice.**
   - NEX is a single Python process: `run.py` with a Flask GUI on `127.0.0.1:8765`, about 55 named background loops (plus per-adapter sense threads and 6 Writer threads), and 6 SQLite databases.
   - Almost every English sentence it produces comes from **qwen2.5:3b** via ollama. That includes the "separate mind" persona, which is the same model at temperature 0.9.
   - The substrate selects, stores, scores and schedules. It does not compose language. The exceptions are template output (identity_loop), quiescent fires (a bare sense line) and non-admin chat replies (a stored belief returned verbatim).
2. **Four factors are proven to read: equanimity, apramada, compass and amoha.**
   - Three of them (equanimity, apramada, compass) live **only in admin chat**, not in her autonomous thinking.
   - **Amoha** is the one proven factor on the fountain path, but its note has surfaced in **0 of 9,440** recorded fires (5,734 of them since it was armed on 09-15). Its detector never returns `clouded`, so live it never fires in the fountain.
3. **Most "always-on" fountain faculties reach only about 25% of fires.**
   - This applies to the workspace, mood, metacog, amoha, recursive-self, drive and stakes lines.
   - They all live in the DRIFT template's `focus_block`. EXPLAIN/ARGUE ("wide mode") replaces that template wholesale.
   - Fire modes over the last 14 days: DRIFT 824, EXPLAIN 964, ARGUE 893, quiescent/NULL 659.
4. **Several armed mechanisms are idle or no-ops in practice:**
   - **ThrowNet:** no session since 2026-09-18.
   - **RECONCILE:** dormant, because it needs 2 or more open problems and there are 0 open and 1 stuck.
   - **surprise_loop:** has promoted nothing since 09-22. 146 of 147 flagged surprises since then are empty-window artifacts.
   - **VALENCE_DRIVE:** a no-op since CURIOSITY was disarmed.
   - **Substrate-as-voice:** permanently disabled by `NEX5_RECONCILE=1`.
   - **Synergizer quiet trigger:** a column bug means it can never fire.
5. **The "one-pen" Writer is the main write path, but not the only one.** 31 modules open their own `sqlite3` connections and write directly. Among them are the 8 life loops, generator.py, the moltbook loops, momentum, hot_observer and persona_responder.
6. **Outward-facing and live:**
   - Moltbook posting is real (`MOLTBOOK_DRY_RUN=0`, account `nex_v4`): 1,213 posts total, 6 in the last 7 days.
   - Kokoro TTS speech is enabled.
   - 24 external feeds are polled.
   - fetch_loop GETs article URLs.
7. **The belief graph is growing and lopsided.**
   - 76,844 beliefs, 57% of them news headlines (`precipitated_from_sense`, all T7/T6).
   - Tier 4 is empty and Tier 5 has 1 belief, so the climb from T6 upward is effectively not happening.
   - Her one grounded self-fact is a T1 scorecard belief: her market calls are **48%** correct, against **50%** for a coin flip.

---

## 1. Architecture

### 1.1 Processes and supervision
- **`nex5-keepalive.service`** (systemd user unit) runs `nex_keepalive.sh`.
  - This bash supervisor parses the `launch_nex` line once and holds it in memory.
  - It also starts `power_switch.py` (:8766), the GUI "power pill".
- **`nex5-db-reaper.timer`** (nightly, about 04:00) runs `nex_db_reaper.py` as its own process.
  - It deletes rows older than 30 days from `pipeline_events`, `tree_snapshots`, `tier_snapshots` and `sense_events`, and rows older than 2 days from `residue`.
  - It works in 5,000-row batches with busy-timeout backoff.
- **`~/.nex/STANDBY`**
  - Present: the supervisor stops NEX with SIGTERM. `_graceful_standby` drains every Writer queue, then calls `os._exit(0)` to free VRAM.
  - Absent: the supervisor relaunches NEX.
- **Boot order** (`run.py`): init 6 DBs → self-location T1 commit → Kokoro preload → modes and voices → sense scheduler → coherence gate and holding zone → dynamic (bonsai and life loops) → voice client → world model → membrane → standalone loops → resumption seeder → fountain → strikes, tools, speech → signals, diversity, edge generator, probes → AppState → decoder → Flask.
- **External dependencies:**
  - ollama at `localhost:11434`; the model is `qwen2.5:3b` because `NEX5_VOICE_MODEL` is unset.
  - Kokoro TTS in-process (voice `af_sarah`).
  - sentence-transformers `all-MiniLM-L6-v2` on CPU, for embeddings and dedup.

### 1.2 Substrate vs. 3B voice
- **Substrate:** 6 SQLite DBs, an in-memory **bonsai tree** (branch focus and texture), and the loops in §3. It decides *what* is in front of the voice and *when* a fire happens.
- **Voice:** `voice/llm.py VoiceClient.speak()` calls the ollama chat endpoint. 15 modules call the 3B:
  - Fountain path: generator (fire and RECONCILE), condenser (3–6 word droplet), `stage_gate/transformer` (RESHAPE).
  - Belief writers: synergizer, and the remember / wonder / fetch / pattern / witness / affinity loops.
  - persona_responder, world prediction_generator and strikes.
  - The chat route in `gui/server.py`.
  - `value_drift_monitor`, which is unreachable (DEAD).
- **Doctrine §0** says "substrate provides; speaking layer composes". This holds: substrate modules mostly add **prompt text**, and the 3B composes the result.

### 1.3 Writer "one-pen"
- `substrate/writer.py` runs one Writer thread per DB. Writes are queued and wrapped in `BEGIN IMMEDIATE`/`COMMIT`, and the queue drains whole on close. `Reader` opens a fresh `mode=ro` connection per read, and WAL mode means reads never block the writer.
- The beliefs Writer is wrapped by **TaggingBeliefWriter** (`tag_protocol`), which auto-tags every `INSERT INTO beliefs`.
- **Reality:** 31 modules also write directly through their own `sqlite3.connect` (`grep` census). These include:
  - the surprise, remember, wonder, fetch, identity, pattern, witness, affinity and daily_life loops;
  - generator.py, momentum, hot_observer, persona_responder and self_prediction;
  - the 3 moltbook loops, focus_loop, signal_to_problem, edge_builder and decoder_loop.
  So the one-pen rule describes the core path, not the whole system.

### 1.4 Sense → dynamic
- **SenseScheduler:** 28 adapters write `sense_events`.
  - **4 internal sensors** (always on): proprioception 10 s, meta_awareness 10 s, temporal 60 s, interoception 30 s.
  - **24 external feeds** (RSS/API), polled every 60 s to 24 h. All start at boot.
  - The boot log line says "23 adapters wired", which is a stale string.
- **A–F pipeline** (`dynamic.sense_poll`, 2.5 s):
  - Each new sense event goes through receive → match branches → magnitude → aperture gate → valence → attend the tree.
  - Each event is logged to `pipeline_events` (4.07 M rows).
  - quality_synthesis scales intake magnitude per branch: ×1.2 on high-genius branches, ×0.8 on low.
- **Distillation** (`dynamic.sense_distillation`, 60 s):
  - Turns titled external events into **T7 beliefs** with `source=precipitated_from_sense`, at most 5 per pass.
  - Near-duplicates are suppressed at remapped cosine 0.96. `internal.*` and `external.other_mind` are excluded.
  - This is the largest belief source: 44,011 total, 417 in the last 7 days.
- **Branch crystallization** (60 s): sustained hot branch focus becomes `crystallization_events` (768 total, last 10-04 00:09).

### 1.5 Belief graph
- **`beliefs`:** 76,844 rows. Distribution by tier:

| Tier | Count | Notes |
|---|---|---|
| T1 | 311 | 278 locked |
| T2 | 40 | all locked |
| T3 | 9,801 | |
| T4 | 0 | |
| T5 | 1 | |
| T6 | 6,095 | |
| T7 | 60,545 | |
| T8 | 51 | retired; 30 paused |

  Active (T1–T6, unpaused) = 16,248.

- **Edges:** `belief_edges` has 282,838 rows. Four writers build them:
  - `edge_builder`: embedding `cross_domain` edges, every 60 s.
  - `edge_generator`: heuristic edges, every 30 min.
  - `novel_association`: every 30 min.
  - harmonizer `cross_domain`: every 6 h.
- **Tier movement:**
  - `promotion.py`: corroborate / survive_challenge promote; `decay_pass` hourly demotes after 48 h idle (720 h for T6 under TIER_CLIMB).
  - `erosion`: every 6 h.
  - `harmonizer`: every 2 h; resolves contradictions and retires beliefs to T8.
  - `world_consolidation`: every 30 min; promotes across themes.
- **Belief sources, last 7 days:**

| Source | Count |
|---|---|
| precipitated_from_sense | 417 |
| remember_loop | 54 |
| counterfactual_node | 50 |
| hot_observer | 37 |
| wonder_loop | 36 |
| synergized | 27 |
| fountain_insight | 22 |
| fetch_loop | 15 |
| identity_loop | 4 |
| pattern_loop | 2 |
| witness_loop | 1 |
| scorecard | 1 |

  `surprise`, `emergent_drive` and `behavioural_observation` wrote 0.

### 1.6 Retrieval — three separate retrievers
- **Fountain** (`_retrieve_context_beliefs`):
  - About 7 "own" beliefs (most recent, T3–T7; T8 excluded) plus 2 random seeds.
  - A per-source cap; up to 2 residue beliefs from the previous cycle; a `belief_boost` time bonus.
  - On every 20th fire, 1 dormant belief is reanimated.
  - With ASSOC_RECALL, 3 of the own slots are filled by relevance from a pool of 1,500.
  - Every retrieved belief bumps `belief_activation`.
- **Chat** (`BeliefRetriever`):
  - Scores keyword overlap × confidence, with an active-branch boost.
  - When edges exist, final score = 0.4 keyword + 0.6 spreading activation.
  - The membrane router splits INSIDE (self-inquiry) from OUTSIDE (world).
- **VoiceEngine** (chat, `use_substrate` mode, the default at boot):
  - A 5-axis grader (semantic 0.45, confidence 0.23, tier 0.14, …) picks the best stored belief, with `min_score` 0.60.
  - Every query is logged to `throw_net_triggers` as `user_query`.

### 1.7 A fountain fire, end to end (`stage6_fountain`)
1. **Tick.** `fountain_loop` sleeps for the mode interval (120 s in mode `mind`) and skips the tick if the mode disables the fountain.
2. **Readiness** (0–1, fire at ≥ 0.7).
   - Sum: base 0.15 + hot branches 0.3 each (max 0.6) + warm branches 0.15 each (max 0.3) + consolidation 0.2 + more than 20 beliefs 0.1 + ≥ 600 s since the last fire 0.2.
   - Minus a genius penalty: up to 0.15 when the past hour's STRIKING rate is below 40%.
3. `CompetingDrives.compute_now()` writes a `drive_activations` row.
4. **Substrate-as-voice** is skipped. It runs only when `NEX5_RECONCILE != 1`, so it is DEAD live.
5. **Quiescent:** every 5th fire (`total % 5 == 4`) writes a bare sense line. No LLM, no crystallization; about 20% of fires, with mode NULL.
6. **Koan:** when `total % 8 == 7` and readiness ≥ 0.8, a koan is used (`koan_reads`).
7. **`_build_prompt`:**
   - The DRIFT system prompt carries `focus_block`: GW winner, echo, drive line, competing drives, recursive SELF, metacog, mood, amoha and the L4-stakes notice.
   - **WIDE MODES** replaces the whole system prompt with an EXPLAIN or ARGUE task on one fresh feed item about 70% of the time. The item comes from STAKES_DEEP, else CURIOSITY (off), else a uniform random pick. **This drops `focus_block`.**
   - Then the shared blocks are appended:
     - identity statement ("This is who you said you are", every fire);
     - self-narrative;
     - SOCIAL (1 persona reply);
     - recent STRIKING fires;
     - rut warning;
     - spectrum standing-points;
     - beliefs;
     - world and sense items;
     - arcs;
     - frame-dedup.
8. **Stillness:** if the own-slot retrieval duplicates recent fires and readiness is below 0.85, the fire is skipped (`stillness_log`).
9. **RECONCILE:** if there are 2 or more open or stuck problems, it pairs them and demands a concrete joint move. The prompt carries a forbidden-word list. Observations are written back, and with ABSTAIN/COMMIT_CLOSE a problem closes after 10 moves. **Currently dormant:** 0 open and 1 stuck.
10. Otherwise `voice.speak(prompt, PHILOSOPHICAL)` produces the thought.
11. **Post-fire:**
    - problem-feedback credit; branch pick (BRANCH_TEMP); `recursion_attrib`;
    - L4-stakes checks every 5th fire; HOT observer meta-belief; momentum capture;
    - `internal.fountain` sense row; condenser droplet (LLM); emphasis score;
    - `fountain_events` insert; substrate snapshot; retrieval log; prompt-block manifest;
    - a substrate parallel fire every 5th fire (`% 5 == 1`).
12. **FOUNTAIN_DEDUP** suppresses crystallization for near-duplicates of recent sibling fires.
13. **Crystallizer quality gate → coherence gate:** a survivor becomes a **T6 belief** with `source=fountain_insight` (22 in the last 7 days; 160 rejects in the same period). BESTOFCLUSTER, ATTRIB_CRYSTALLIZE and SURPRISE_WEIGHT adjust this step.
14. CARRYOVER stamps resolved/stalled on the momentum thread; a drive-probe chance; a state snapshot to `state_snapshot.json`.
15. **Downstream:**
    - genius tagger (60 s) and decoder (30 s);
    - moltbook poster (posts crystallized insights);
    - speech queue → Kokoro.

Fires per day when awake: 110 on 10-03 and 73 on 09-30. The rest of the week was mostly STANDBY.

### 1.8 Coherence gate, holding zone, ThrowNet
- **CoherenceGate:** checks every faculty-generated thought. Does it contradict what she holds, connect to what she is engaged with, add something novel? The outcome is ACCEPT, REJECT, HOLD or RESHAPE. `gate_decisions` holds 15,816 rows after daily retention.
- **HOLD** goes to the holding zone (`held_thoughts` 29,713; last 09-28). A resolver loop (300 s) settles each held thought by corroboration (N = 3), contradiction or a 24 h fade.
- **RESHAPE** goes to the ReshapeTransformer (3B).
- **ThrowNet:**
  - **Triggers:** ≥ 4 same-topic gate REJECTs in 15 min, or ≥ 3 gap deflections in 30 min.
  - **Session:** the monitor (300 s) runs pending triggers: TimeFetch (≤ 40 candidates) → RefinementEngine (top 10) → CoherenceGate → `throw_net_sessions`.
  - **Reality:** 91 sessions ever (803 accepted). The last session was **2026-09-18** and the last gate_reject trigger 09-13.
  - The 322 `user_query` rows are VoiceEngine chat logs, inserted already marked `fired=1`, so they never queue a session. **The cycle is idle in practice.**
- **CounterfactualNode** (300 s) works open problems through the same RefinementEngine → gate. After 3 ACCEPTs a problem moves to `review_queue`, which is empty. It wrote 50 beliefs in the last 7 days; the last was on 10-03.

### 1.9 Chat path (`POST /api/chat`, `gui/server.py`)
1. A session row is created. The membrane router decides INSIDE vs OUTSIDE and the retriever runs.
2. Always-on additions to `belief_text`:
   - behavioural self-model and self-model (INSIDE routes only); harmonizer tension;
   - open-problem match; metacognition; novel association; self-narrative;
   - affect; drive; SelfMindView summary; tools.
   - SocialPresence was removed and is now None.
3. If the graph match is too thin, an honest "don't know" is returned without calling the LLM.
4. **`voice_mode` is `use_substrate` at every boot:**
   - **Non-admin:** the reply is the VoiceEngine's best stored belief, **verbatim, with no LLM**.
   - **Admin with OPERATOR_MODEL:** that belief becomes grounding, and the 3B composes the reply.
5. **Admin-only prompt blocks** (flag on **and** session admin):
   - operator model and anchor; chat memory and anti-repeat;
   - compass; equanimity; apramada; affect-carry; anti-echo; compass trim;
   - provenance: a strongly mirror-dominant reply triggers **one** regeneration, which is kept only if it is less mirrored.

### 1.10 Outward channels
- **Moltbook:**
  - Loops: poster every 60 s, listener every 300 s, responder every 300 s.
  - **Real posting** (`MOLTBOOK_DRY_RUN=0`): 1,213 posts total, 6 in the last 7 days, last 10-03 23:35.
  - daily_life also schedules a moltbook DM at 12:00.
- **Speech:** Kokoro speaks queued thoughts with a minimum gap of 180 s. `speech_queue` has 4,362 rows; the last was queued 10-03 06:38.
- **Persona responder:**
  - Every 600 s, a different system prompt on the same 3B at temperature 0.9 replies to her recent thoughts. The reply is written as `external.other_mind` (59 in the last 7 days; 3,458 persona rejects).
  - `SOCIAL_N=1` feeds 1 reply back into each fire.
- **Network:** 24 feeds, fetch_loop article GETs (5 s timeout), and the world-prediction price fetch.

---

## 2. Databases (`nex5/data/`)
Row counts are `COUNT(*)`, read at about 02:30 with NEX running.

| DB | Size | Holds (key tables, rows) |
|---|---|---|
| **beliefs.db** | 300 MB | `beliefs` 76,844 · `belief_edges` 282,838 · `belief_activation` 52,220 · `belief_lineage` 13,948 · `patterns` 209,114 · `novel_association_log` 168,117 · `intake_resonance_log` 195,473 (stopped 07-13) · `groove_alerts` 84,832 · `signals` 72,866 · `world_bridge_log` 44,820 · `held_thoughts`/`held_resolutions` 29,713 each · `token_prose_stats` 22,547 · `arc_members` 17,848 / `arcs` 1,640 (dead since 05-30) · `gate_decisions` 15,816 · `fountain_crystallizations` 8,531 · `synergizer_log` 7,502 · `speech_queue` 4,362 · `residue` 3,325 · `koan_reads` 3,188 · `consolidations` 1,865 · `signal_cooldown` 1,539 · `collision_grades` 893 · `throw_net_triggers` 429 / `throw_net_sessions` 91 · `dormant_beliefs` 50 · warrant tables (not read) |
| **dynamic.db** | 1.15 GB | `pipeline_events` 4,070,683 · `word_contexts` 1,170,287 · `fountain_prompt_blocks` 179,464 · `predictions` 62,626 · `surprise_events` 62,626 · `fountain_events` 57,236 · `substrate_snapshots` 31,741 · `self_mind_snapshots` 31,620 · `recursion_attrib` 26,584 · `tree_snapshots` 24,670 · `emphasis_log` 22,747 · `crystallization_rejects` 19,280 · `tier_snapshots` 11,329 · `moltbook_post_queue` 10,849 · `substrate_fires` 8,710 · `stillness_log` 6,493 (last 05-15) · `social_presence_snapshots` 4,481 (dead since 05-30) · `persona_rejects` 3,458 · `moltbook_posts` 1,213 · `crystallization_events` 768 · `daily_activities` 719 · `identity_log` 443 · `pattern_log` 219 · `witness_log` 110 · `harmonizer_events` 109 (last 05-22) · `self_maintain_shadow` 4 · `fountain_retrieval_log` (not read) |
| **sense.db** | 505 MB | `sense_events` 808,793 (all streams; reaped at 30 d) |
| **conversations.db** | 173 MB | `narrative_log` 540,833 · `genius_tags` 59,687 · `drive_activations` 44,127 · `affect_history` 31,513 · `substrate_coherence` 27,515 · `drive_emergence_log` 17,613 · `drives_competing_log` 16,503 · `self_predictions` 15,622 · `world_predictions` 11,743 · `calibration_consults` 7,361 · `messages` 3,495 · `meta_cognition_events` 1,568 · `open_problems` 558 (557 closed, 1 stuck) · `sessions` 331 · `provenance_snapshots` 179 · `stillness_log` 159 · `genius_labels` 120 · `affect_carry` 67 · `value_drift_log` 36 (dead since 06-13) · `goals` 10 · `drives` 1 |
| **probes.db** | 0.6 MB | `probes` 42 · `probe_context` 288 (Lens-theory archaeology; last write 05-23) |
| **intel.db** | 32 KB | `analysis_snapshots`/`market_data`/`news_events`: all 0 rows. Initialised every boot, never written. **DEAD** |

**Stray, not opened by NEX:**
- `data/fountain_events.db` (0 B), repo-root `dynamic.db` (0 B) and `theory_x/substrate/beliefs.db` (0 B): empty leftovers.
- `data/strikes_catalogue.db` and `data/verification.db`: last written 07-16 and 06-06.
- `data/beliefs.db.backup_20260427_0445`.

---

## 3. Autonomous loops (cadence → what it does → evidence)

**Dynamic stage** (`stage2_dynamic.build_dynamic`):

| Loop | Every | Does | Live evidence |
|---|---|---|---|
| dynamic.sense_poll | 2.5 s | A–F pipeline over new sense events → bonsai | `pipeline_events` writing |
| dynamic.aperture | 5 s | membrane aperture from aggregate texture | in-memory |
| dynamic.accumulator | 30 s | bonsai decay, membrane accumulator flush; cadence refresh about every 30 min | in-memory |
| dynamic.crystallization | 60 s | sustained hot branch → crystallization event/belief | last 10-04 00:09 |
| dynamic.consolidation | 60 s | quiet-triggered tree consolidation flag (feeds readiness +0.2) | in-memory |
| dynamic.sense_distillation | 60 s | headlines → T7 beliefs | 417 in 7 d |
| dynamic.snapshot | 60 s | `tree_snapshots`; `tier_snapshots` every 900 s | writing |
| dynamic.health | 30 s | health line to the error channel | log only |
| dynamic.emergent_drives | 12 h | drive-pressure proposals | `drive_proposals` 8, last 09-25 |
| life.witness_loop | 1,800 s; composes 04:00 | LLM "what is the blind spot" → T6 belief | last 09-30 |
| life.pattern_loop | 600 s; composes 03:00 and 15:00 | LLM names the pattern in identity statements → T6 | last 09-30 |
| life.identity_loop | 300 s; composes 00/06/12/18 | **template, no LLM** self-statement → "This is who you said you are" block | last 10-04 18:39 |
| life.affinity_loop | 1,800 s, batch 30 | 3B rates "how much is this mine" → affinity weight | runs; its own docstring (07-09) records the rating as hollow, because the 3B copies anchor labels |
| life.scorecard_loop | 3,600 s | refreshes the T1 market-scorecard belief | 10-05 02:14 |
| life.surprise_loop | 300 s | flagged surprises → T6 "surprise" beliefs | **0 since 09-22**: 146/147 flagged have NULL actual content (empty windows), filtered out |
| life.remember_loop | 600 s, cap 24/day | LLM links an old and a recent belief | 54 in 7 d |
| life.wonder_loop | 900 s, cap 16/day | LLM asks a question about an entity | 36 in 7 d |
| life.fetch_loop | 1,800 s, cap 12/day | GETs an article URL, LLM responds in one sentence | 15 in 7 d |
| life.daily_life | 60 s | scheduled day (Europe/Amsterdam tz): entries, reading, outreach DM | last 10-04 18:39 |
| sustained.focus_loop | 60 s | one open problem held; appends per-fire observations; pings chat when stuck | — |
| signals.signal_to_problem | 120 s | signals → open problems (conf ≥ 0.40, cap 5/24 h) | last problem 10-03 |
| diversity.edge_builder | 60 s | embedding `cross_domain` edges | writing |
| stage_gate.retention | 24 h | prunes `gate_decisions` / `throw_net_sessions` | — |
| moltbook poster / listener / responder | 60 / 300 / 300 s | posts crystallized insights; reads; replies | 6 posts in 7 d |

**World model and membrane:**

| Loop | Every | Does | Live evidence |
|---|---|---|---|
| world_model.decay | 1 h | demote idle beliefs | — |
| world_model.harmonizer | 2 h | contradiction scan/resolve, retire to T8 | `harmonizer_events` last row **05-22** |
| world_model.cross_domain | 6 h | harmonizer cross-domain edges | — |
| world_model.erosion | 6 h | provenance erosion pass | — |
| world_model.synergizer | 60 s check; fires on a 25 min timer | LLM fuses two cross-branch beliefs → `synergized` | 27 in 7 d; the quiet trigger is dead (§6) |
| world_model.world_consolidation | 30 min | thematic-convergence promotion (armed) | — |
| membrane.behavioural | 4 h | behavioural self-model (hedge rate, register, length) | — |

**Standalone** (`run.py`):

| Loop | Every | Does | Live evidence |
|---|---|---|---|
| fountain_loop | 120 s | §1.7 | 57,236 fires total |
| HoldingZoneResolver | 300 s | settle held thoughts | last 09-30 |
| NovelAssociation | 30 min | cross-branch similar pairs → edges/log | last 10-04 |
| ThrowNetMonitor | 300 s | run pending throw-net triggers | **idle since 09-18** |
| CounterfactualNode | 300 s | work open problems through gate | 50 beliefs in 7 d |
| SubstrateHarmonic | 300 s | coherence metric → `substrate_coherence` (log-only) | writing |
| GeniusTagger | 60 s | v2 genius score per fire → `genius_tags` (feeds readiness and the STRIKING block) | writing |
| DriveEmergence | 600 s | emergent-drive detection | `drive_formed_id` NULL in all 17,613 ticks; the single `drives` row (formed 06-22) is "disk permissions learnt bootcamp midwife", reinforced 10,471× |
| CompetingDrives | 600 s + per fire | 5-drive weights / tension block | tension_active in 15,324 of 16,503 ticks (93%) |
| AffectState | 300 s | valence/arousal/stability/mood label | writing |
| PredictiveSubstrate | 300 s | predictions + surprise scoring | writing; see surprise_loop |
| SelfMindView | 300 s | self-state snapshot + chat summary | writing |
| PersonaResponder | 600 s | other-mind replies | 59 in 7 d |
| WorldPredictionLoop | 900 s, horizon 3,600 s | BTC direction call + random control | 58 resolved in 7 d |
| SelfPredictionLoop | 900 s, horizon 420 s | predicts own firing + coin control | 78 resolved in 7 d |
| SelfMaintainShadow | 600 s after boot, then hourly | log-only "would pause N" regulator | row 4 = `insufficient` (too little history after standby) |
| WarrantRecorder | 900 s / 3,600 s | warrant use-day/survival recording | not inspected (instruction) |
| quality_synthesis | 1,800 s | genius-by-branch multiplier file read by attention | — |
| SignalLoop | 60 s | detectors + templates → `signals`, `patterns` | writing |
| DiversityLoop | 60 s | groove spotter, dormancy scan, fire-count clock (20/200/2,000 fires) | `groove_alerts` 84,832 |
| EdgeGeneratorLoop | 30 min | heuristic edges | — |
| decoder_loop | 30 s | tokenizes fires → `word_contexts` (GUI LIVE column) | writing |
| SpeechQueueConsumer | poll | Kokoro TTS | last queue 10-03 |
| SenseScheduler | per adapter | 28 adapters | writing |
| Writers ×6 | queue | DB writes | — |

---

## 4. Faculties and mental factors (real status)

Verdict sources: `dev_queue_2026-10-04_results.txt` (uptake A/B/C, n = 40) and `mental_factors_inventory_alobha_spec.txt`. Fountain surfacing counts come from `fountain_prompt_blocks`, which covers 9,440 fires since 09-08.

### 4a. PROVEN-READS (beat A and C)
- **Equanimity (upekkhā), `NEX5_EQUANIMITY`.** Admin chat only.
  - Prompt block when one of her own pulls (rāga/dveṣa) fires hot and karuṇā is not live.
  - B>A +0.291, B>C +0.267 (10-02).
  - Inert on neutral turns; reads on emotionally loaded ones.
- **Apramāda, `NEX5_APRAMADA`.** Admin chat only.
  - Vigilance cue naming a held thread at risk of being dropped.
  - B>A +0.175, B>C +0.213.
- **Compass, `NEX5_COMPASS` (+TRIM).** Admin chat only.
  - Weighs moral weight, amoha clarity and the affliction reads, then returns a held-open stance.
  - B>A +0.225, B>C +0.275.
  - **Caveats:** it speaks whenever any affliction reads HIGH, and mana reads on every turn, so it speaks on essentially every turn and its discrimination is void (inventory). Its moral-weight clause catches only 4 of 10 moral questions (results).
- **Amoha, `NEX5_AMOHA`.** Fountain, DRIFT only.
  - B>A +0.287, B>C +0.287 (forced-injection probe).
  - **Live:** the note requires `detect()=="clouded"`. It has surfaced in **0 of 9,440** recorded fires, including 0 of 5,734 since it was armed on 09-15. Proven when present; absent in practice.
  - Also feeds the compass clarity read.

### 4b. ARMED-UNPROVEN (on, no A/B/C uptake on record)
- **Global Workspace.** Picks one "[WORKSPACE]" lead from surprise, momentum, bonsai, stakes and drive. DRIFT only; carried in 2,871 fires. The empty-window fix (`GW_SURPRISE_NONEMPTY`) was armed today after a replay PASS.
- **Metacog voice.** Fixed "Self-observation: …" strings; 1,188 fires.
- **Recursive self (L3 "SELF:").** Not flag-gated; 88 fires, last 09-30. Its own teeth-test docstring calls the verdict ambiguous.
- **Momentum / Carryover / Self-narrative / identity statement.**
  - These are continuity blocks. The inventory files them as smṛti analogs that were never probed.
  - The carried thread sometimes arrives in Chinese: 3B fires in Chinese show up as carried blocks in the manifests.
- **HOT observer.** Classifies fires and writes meta-beliefs about them (37 in 7 days, 6,146 total). It is commentary, not a calibrated faculty: the metacognitive calibration test failed its pre-registered bar (`metacog_calibration_results.txt`).
- **L4 stakes.** A world-contact ratio check and a maxDF* drift tripwire every 5th fire, plus a GW stakes candidate.
- **Social (persona + SOCIAL_N=1 + SOCIAL_DEPTH).** The "other mind" is the same 3B under a different prompt.
- **STAKES_DEEP.** The wide-mode pick follows a held drive. The only drive is keyword salad, and the code yields on "junk" drives. Whether it ever wins was not measured.
- **Chat mechanics:**
  - operator model and anchor (Jon intake doc as held context);
  - chat memory and anti-repeat regeneration;
  - anti-echo;
  - provenance (mirror side only; code comment: "grounding did not discriminate").
- **Affect-carry.** Partly reads: the felt-tone part does **not** read against a length-matched sham (n = 64, 09-28), but pacing does (replies 7–15 words shorter).
- **Substrate (non-prompt) faculties:**
  - AffectState valence/arousal;
  - PredictiveSubstrate surprise;
  - DriveEmergence and CompetingDrives (agency probe: drives add ΔAUC +0.013 over habit, **below** the +0.02 bar);
  - SelfMindView;
  - self- and world-prediction (an honest scorecard, not a faculty).

### 4c. ARMED-INERT
- **Mood (`NEX5_MOOD`, compositional_emotion).** Carried in 2,479 fires. Valence sits near +0.73 constant and mood variation does not steer (inventory, 10-03). The emotion label is naming only.
- **`NEX5_VALENCE_DRIVE`.** A **no-op**: it lives inside `_curiosity_pick`, which runs only when `NEX5_CURIOSITY=1`, and that flag is now off. Removal candidate.
- **Affinity.** Runs, but its own docstring records the 3B self-rating as hollow (anchor-copying; it rates headlines as "mine").

### 4d. DISARMED (flag off; why)
- **Compassion (karuṇā).** Beats A, B>C +0.000 (6c203c3, 10-04).
- **Afflictions cluster note** (rāga/dveṣa/māna/avidyā/moha).
  - avidya, dvesa and moha are inert; mana beats A but not C (b941c79). Carried in 420 fires between 09-15 and 10-03.
  - The **detectors still run** inside the compass.
- **Curiosity pick.** Inert against the uniform pick (7fd3dd5).
- **Vīrya.** Counterproductive, B>C −0.150 (e40367e).
- **Praśrabdhi.** Inert (e40367e).
- **Śraddhā.** Raised hedging, so it works backwards (99ab815).
- **NEX5_WARRANT** (warrant re-ranking). Off; only the recorder runs.

### 4e. DARK (built, never armed)
- **Alobha** (b5dd629). Lowers focal re-grip from 0.754 to 0.562, but the effect on what she writes is +0.021, below the bar. Lives inside `_curiosity_pick`.
- **Conviction.** `theory_x/stage_affect/conviction.py` is **untracked (uncommitted)**; Held/Unsettled grouping was inert.
- **Off by default, never on the line:**
  - `COMPASSION_V2`; `SUBSTRATE_EMIT`; `EDGE_WEIGHTED_FOUNTAIN`;
  - `RETRIEVAL_DEBOILER`; `INTAKE_TIGHT_CORE`; `BRIDGE` (parked);
  - the `CONTINUITY_N` / `SELF_LAYER_N` / `SENSE_OVERWHELM_N` prompt layers;
  - `PARK_CAP`; `NEX_TAG_FEEDBACK_ON`.

### 4f. DEAD / removal candidates (faculty code no live path reaches)
- **Substrate-as-voice (Intervention C).** Disabled while `NEX5_RECONCILE=1`.
- **Generator §9 reanimation governor and its rut-mirror.** Off via `NEX5_GOVERNOR_OFF=1`; reanimation stays at every 20th fire.
  - A separate rut-notice in `SelfNarrative.tick` is still on (`NEX5_RUT_MIRROR_OFF` unset), but its last `self_rut_notice` belief was written 2026-05-31.
- **Modules unreachable from `run.py`** (static import trace plus name check):
  - `value_drift_monitor`; `source_identity`;
  - `recursion_attribution` and `recursion_teeth_test` (offline tests);
  - `social_presence` (removed 07-27); the `arcs/*` reader (removed 07-27);
  - `drive_history` (removed 09-09); `auto_probe/groove_breaker` (`ENABLED=False`);
  - `subject_fidelity`; `spark_cycle/*`; `diversity/signal_cleanup`;
  - `genius_provenance(_tag)`; `probes/probe_db`; `stage_warrant/warrant`.

---

## 5. Launch flags on the live line (57)
The live environment equals HEAD `fa35b6b` exactly. All flags are `=1` unless a value is shown. **Path** column: F = fountain fire, C = admin chat, L = background loop, X = crystallizer, S = substrate/graph.

| Flag | Path | What it drives | Status |
|---|---|---|---|
| MOLTBOOK_DRY_RUN=0 | L | real posting to moltbook (not dry run) | live, outward |
| NEX5_ABSTAIN_CLOSE | F | RECONCILE closes a problem on abstain after ≥ 10 moves | dormant (RECONCILE idle) |
| NEX5_AFFECT_CARRY | C | conversational affect carry block | partly reads (pacing only) |
| NEX5_AMOHA | F | clear-seeing note when clouded (DRIFT) | PROVEN-READS; 0 live surfacings |
| NEX5_ANTILOOP | F | RECONCILE told all prior move signatures | dormant |
| NEX5_APRAMADA | C | held-thread vigilance cue | PROVEN-READS |
| NEX5_ASSOC_RECALL | F | 3 of 7 own retrieval slots by relevance | ARMED-UNPROVEN |
| NEX5_ATTRIB_CRYSTALLIZE | X | wide-mode crystals attributed to the focal item | on |
| NEX5_BESTOFCLUSTER | X | crystallize the best of a paraphrase cluster | on |
| NEX5_BRANCH_TEMP=0.35 | F | focus-weighted branch pick temperature | on |
| NEX5_CARRYOVER | F | stamp resolved/stalled on the momentum thread | ARMED-UNPROVEN |
| NEX5_CHAT_ANTIECHO | C | strip verbatim spans latched from the user | on |
| NEX5_CHAT_MEMORY | C | dialogue memory + anti-repeat regeneration | on |
| NEX5_COMMIT_CLOSE | F | RECONCILE closes a problem on a concrete artifact after ≥ 10 moves | dormant |
| NEX5_COMPASS | C | moral compass stance | PROVEN-READS; fires every turn |
| NEX5_COMPASS_TRIM | C | strip compass boilerplate echoed in replies | cosmetic |
| NEX5_DELIVER_N=10 | F | RECONCILE demands a deliverable after 10 moves | dormant |
| NEX5_DRIVE_CONTENT | L | content-topic drive synthesis | on; produced the keyword-salad drive |
| NEX5_EQUANIMITY | C | hold-steady stance on hot pulls | PROVEN-READS |
| NEX5_FOCUS_ROTATE | F | drop a stale focal cluster from the wide-mode pool | on |
| NEX5_FOUNTAIN_DEDUP | X | skip crystallizing paraphrase bursts | on |
| NEX5_FRAME_DEDUP | F | anti-frame note + header dedup | on |
| NEX5_GLOBAL_WORKSPACE | F | GW lead line (default on anyway) | ARMED-UNPROVEN; DRIFT only |
| NEX5_GOVERNOR_OFF | F | disables the reanimation governor and rut-mirror | on (= those are off) |
| NEX5_GW_SURPRISE_NONEMPTY | F | GW ignores empty prediction windows | armed 10-05, replay PASS |
| NEX5_HOT_OBSERVER | F | per-fire meta-belief | commentary; calibration FAILED |
| NEX5_INTAKE_RESONANCE_OFF | S | turns intake-resonance logging/tagging **off** | on (= feature off) |
| NEX5_L4_STAKES | F | world-contact check, drift tripwire, stakes notice | ARMED-UNPROVEN |
| NEX5_METACOG_VOICE | F | self-observation line (DRIFT) | ARMED-UNPROVEN |
| NEX5_MOMENTUM | F | carried thread across fires (default on) | ARMED-UNPROVEN |
| NEX5_MOOD | F | composed-emotion line (DRIFT) | ARMED-INERT |
| NEX5_OPERATOR_ANCHOR | C | second-person addressee anchor for Jon | on |
| NEX5_OPERATOR_MODEL | C | Jon's intake doc as held context + HYBRID compose | on |
| NEX5_PERSONA_RESPONDER | L | other-mind loop (600 s) | on |
| NEX5_PORT=8765 | — | GUI port | — |
| NEX5_PROVENANCE | C | mirror read + one less-mirrored regeneration | on (mirror side only) |
| NEX5_QUALITY_SYNTH | L | genius-by-branch intake multiplier | on |
| NEX5_RECONCILE | F | pair 2 problems, demand a joint move; also kills substrate-voice | dormant (1 problem) |
| NEX5_RECONCILE_WB | F | RECONCILE writes moves back to problems | dormant |
| NEX5_RUT_EDGE | F | rut-warning block | on |
| NEX5_SCORE_F2_V2 | L | genius score V2 anti-template metric + weights | on |
| NEX5_SELF_MAINTAIN_SHADOW | L | log-only active-set regulator | on; `insufficient` until history refills |
| NEX5_SELF_NARRATIVE | F | self-narrative block + init | ARMED-UNPROVEN |
| NEX5_SELF_PRED | L | self-prediction loop | on |
| NEX5_SIG_QUALITY | L | stricter entity filter in signal_to_problem | on |
| NEX5_SOCIAL_DEPTH | L | persona keeps an interlocutor model | on |
| NEX5_SOCIAL_N=1 | F | 1 persona reply per fire prompt | on |
| NEX5_STAKES_DEEP | F | held drive steers the wide-mode pick | ARMED-UNPROVEN |
| NEX5_SURPRISE_WEIGHT | X | surprise weighting at crystallization (default on) | on |
| NEX5_SYNTH_FRESH | S | synergizer recency preference (fresh side) | on |
| NEX5_SYNTH_FRESH_SIBLING | S | synergizer descendant penalty | on |
| NEX5_TIER_CLIMB | S | T6 idle window 720 h (vs 48 h) | on; T4/T5 still near-empty |
| NEX5_VALENCE_DRIVE | F | mood tilts the curiosity pick | **no-op** (CURIOSITY off) |
| NEX5_WARRANT_RECORD | L | warrant recorder loop | on (not inspected) |
| NEX5_WIDE_MODES | F | EXPLAIN/ARGUE override (drops focus_block) | on |
| NEX5_WORLD_CONSOLIDATE | S | thematic promotion every 30 min | on |
| NEX5_WORLD_PRED | L | world (BTC) prediction loop | on |

**In code but off** (61 `NEX5_` names). They fall into:
- the disarmed and dark flags in §4d/§4e;
- defaults or plumbing: `VOICE_URL`, `VOICE_MODEL`, `DATA_DIR`, `HOST`, `GUI_*`, `SPEECH_*`, `PERSONA_INTERVAL`, `SELF_PRED_INTERVAL`, `WORLD_PRED_*`, `QUIET_HOURS`;
- the kill-switches `GENIUS_PROMPT_OFF`, `GENIUS_GRADE_OFF`, `FOUNTAIN_READINESS_MOD_OFF` and `RUT_MIRROR_OFF`, all unset, so those features are **on**. The self_narrative rut-notice that `RUT_MIRROR_OFF` controls has not written since 05-31.

`NEX5_SYNTH_EMIT` and `NEX5_RECONCILE_PXB` are still named in comments only; their code was removed on 07-25.

---

## 6. Known dead or inert code (verified 2026-10-05)
1. **Synergizer quiet trigger** (`stage3_world_model/__init__.py` ~l.217) queries `sense_events WHERE ts > ?`. The column is `timestamp`, so the query always raises and the error is swallowed. Only the 25 min timer ever fires.
2. **Substrate-as-voice** (`generator._maybe_substrate_voice`) is unreachable while `NEX5_RECONCILE=1`.
3. **VALENCE_DRIVE and ALOBHA** live inside `_curiosity_pick`, which is unreachable with CURIOSITY off.
4. **The amoha fountain note** has 0 surfacings ever: the detector never returns `clouded`.
5. **surprise_loop** has written 0 beliefs since 09-22, because flagged surprises are empty-window artifacts with NULL `actual_content`. The predictive side still flags them: 146/147 at score ≈ 1.0. The GW fix covers the workspace only, not this loop.
6. **ThrowNet** has had no sessions since 09-18. `user_query` rows are logs pre-marked fired, and gate_reject triggers have stopped.
7. **RECONCILE** and its 6 satellite flags (ANTILOOP, DELIVER_N, ABSTAIN/COMMIT_CLOSE, RECONCILE_WB, PARK_CAP) are dormant whenever there are fewer than 2 open or stuck problems (now: 1).
8. **intel.db** has 3 tables with 0 rows. It is initialised at boot and nothing writes it.
9. **Tables frozen since removal or cut:**
   - `arcs`/`arc_*` (05-30); `social_presence_snapshots` (05-30); `stillness_log` in dynamic.db (05-15);
   - `harmonizer_events` (05-22); `value_drift_log` (06-13); `intake_resonance_log` (07-13, flag OFF);
   - `word_tags`/`coincidence_context` (08-16); `auto_probe_log` (05-01).
10. **Feed files not wired into the scheduler:** `ap_news`, `papers_with_code`, `philpapers`, `reuters`. `ml_conferences` is wired but produced 0 events in 7 days.
11. **Unreachable modules** are listed in §4f. Also unreferenced at the repo root: `flag_genius.py`, `nex_next_dev.py`, `proof_of_concept.py`, `train_curator.py`, `verifier_*`, `scripts_moltbook_seed.py`. `nex_db_reaper.py` and `power_switch.py` run as their own processes.
12. **Empty stray DB files:** `data/fountain_events.db`, repo-root `dynamic.db`, `theory_x/substrate/beliefs.db`.
13. **DriveEmergence** has never set `drive_formed_id` (0 of 17,613 ticks). The one live drive topic is keyword salad, and it feeds the drive line and STAKES_DEEP.
14. **Readiness** runs `SELECT COUNT(*) FROM beliefs` on every check. The cost is small, but it is pure overhead for a constant +0.1.
15. **The boot log string "23 adapters wired"** is stale; the real count is 28.
