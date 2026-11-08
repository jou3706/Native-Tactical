# Epic Systems Design: Interactive Storyline & Unique Captains

## 1. Interactive Storyline & Rebellion (Epic Story)

### Goals
- Replace static, linear quest steps with dynamic, travel-driven assignments.
- Escalating political drama leading to a military rebellion (coup) where the player overthrows the king and claims the throne.

### Architecture
- **Quest framework**: extend `qst_ash_crown` with dynamic stages tracked by globals:
  - `$g_story_stage` (0..N), `$g_story_assignment_type` (0 none, 1 travel, 2 deliver, 3 battle, 4 council), `$g_story_target_center`, `$g_story_deadline_day`, `$g_story_choice_flags`.
- **Royal assignment loop** (simple trigger, daily):
  - If no active assignment and player is a vassal of a kingdom, generate a new assignment from the king:
    - Travel to a center (scout/inspect).
    - Deliver a message/item to a lord.
    - Join a battle / reinforce a garrison.
    - Attend a council.
  - Set target center, deadline, reward. Display message + quest note.
- **Progression check** (daily):
  - When player reaches target center within deadline → complete assignment, reward (renown/gold/relation), increment `$g_story_stage`.
  - Missed deadline → relation penalty, king displeasure flag.
- **Escalation & rebellion**:
  - Track `$g_king_displeasure` (0..100) incremented by: missed assignments, refused orders, negative decisions at council, relation drops with king.
  - At threshold (e.g., 60) and after stage N, unlock "Rebellion" decision menu:
    - Player may declare rebellion → set `fac_player_supporters_faction` active, transfer loyal vassals, start civil war.
    - Or submit → king grants mercy but story continues with reduced rewards.
  - Final coup: when player army defeats king's main party / captures capital → player becomes faction leader (`slot_faction_leader = trp_player`), quest succeeds.

### Key integration points
- `script_game_start`: init globals.
- `module_simple_triggers`: daily assignment generation + progression + escalation.
- `module_game_menus`: assignment accept/decline, rebellion decision, epilogue.
- `module_quests.py`: quest notes per stage (8 slots, reuse slots for current objective).
- Existing Ashen Crown chapters remain as a parallel track; new epic stages layer on top.

### Data structures
- Assignment types enum (travel/deliver/battle/council).
- Target center stored as party id (center).
- Deadline as campaign day (`$g_cur_day` + N).
- Flags bitfield for decisions (mercy/iron, submit/rebel, etc.).

## 2. Unique Unit Captains

### Goals
- Monitor army composition by class (infantry/archer/cavalry).
- When any class count crosses a multiple of 100, generate/award a unique captain for that class.
- Captains act as field commanders linked to leadership & loyalty.

### Architecture
- **Globals**:
  - `$g_captain_count_inf/arch/cav`: current counts (updated each check).
  - `$g_captain_inf/arch/cav_awarded`: number of captains already awarded per class.
  - `$g_captain_total`: total captains.
- **Check script** (`script_tactical_captain_check`): runs every 6 hours:
  - Count party troops by class using `party_get_num_companion_stacks` + `troop_get_class`.
  - For each class: `floor(count/100)`. If greater than awarded, award new captains:
    - Increment awarded + total.
    - Grant renown (+10) and gold (+200) per captain.
    - Display message.
  - Store counts for UI.
- **Captain roster menu** (Camp → "Captain roster"): shows counts and awarded captains.
- **Future expansion (troop-level captains)**:
  - Pre-define unique captain troops in `module_troops.py` (e.g., `trp_captain_infantry_01..`).
  - On award, add captain troop to party (`party_add_members`) if not already present, and track in a captain list.
  - Link captain to class loyalty (captain morale affects party morale).
  - On captain death, remove from roster and allow re-award.

### Integration
- Simple trigger (6h) calls `script_tactical_captain_check`.
- Camp menu entry "Captain roster".
- README updated.

## 3. Build & verification
- All additions build with 0 errors and 0 warnings.
- No live-game testing; logic is build-verified only.
