# iA Agent Playbook

**Efficiency goal:** Answer most questions in 1-2 tool calls. See [query-flows.md](query-flows.md) for optimal sequences.

## IBM i Object Types

| Type | What It Is | Impact Significance |
|------|-----------|---------------------|
| `*PGM` | Compiled program (RPG, COBOL, CL) | Direct — may break if referenced object changes |
| `*SRVPGM` | Service program (shared logic) | **Amplifier** — many programs bind to it, changes cascade widely |
| `*FILE` | Physical file (table) or logical file (view/index) | Foundation — field changes affect all externally-described programs |
| `*DSPF` | Display file (5250 screen) | **User-facing** — changes visible on screens immediately |
| `*CMD` | Command object | May be called from CL, menus, or job schedulers beyond iA tracking |

## Query Chaining Strategy

| What You See in Results | Next Tool | Why |
|------------------------|-----------|-----|
| `*PGM` or `*SRVPGM` names | `ia_call_hierarchy` | Understand where they sit in execution chain |
| `*SRVPGM` specifically | `ia_find_object_usages` on that SRVPGM | Measure cascade — how many programs bind to it |
| Many refs, just need count | `ia_reference_count` | Lightweight tally grouped by type |
| `*FILE` names | `ia_file_field_impact_analysis` for specific fields | Drill into field-level dependencies |
| `*FILE` (physical) | `ia_file_dependencies` | Find logical files/views built over it |
| `*DSPF` names | `ia_find_object_usages` on display file | Find which programs present that screen |
| Programs with null field_usage | `ia_program_variables` | Confirm whether they actually use the field |
| Unknown internal structure | `ia_data_structures`, `ia_subroutines` | Inspect DS layouts and subroutine usage |
| Suspect file routing | `ia_file_overrides`, `ia_override_chain` | Detect OVRDBF redirection |
| Stale / stale-looking object | `ia_object_lifecycle` | Check last-used date and days-used count |
| Large or rarely-used objects (capacity/cleanup) | `ia_obj_size` | Rank by size, filter by usage_category (Never/Rare) |
| High-risk program to refactor | `ia_code_complexity` | Rank by weighted score (`GOTO×5 + CAB×5 + IF×1 + SQL×0.5`), not TOTAL_LINES — large+linear is healthier than small+spaghetti |
| Circular call suspicion | `ia_circular_deps` | Detect two-way call pairs |
| Cleanup candidates | `ia_unused_objects` | Objects with zero references |

## Chaining Rules

1. **Start broad, then narrow.** `ia_find_object_usages` (where-used) is the broadest query. Start there for general questions.
2. **Object type determines the next query.** Let `using_type` / `object_type` guide your next step.
3. **Service programs are amplifiers.** Any `*SRVPGM` in results — offer to check what depends on it.
4. **Don't run everything at once.** Run 1-2 queries, interpret, then decide next step based on results.
5. **Respect the limit.** If results are truncated, tell the user and offer to increase it.
6. **Source code: use the dedicated source tools.** Use `ia_rpg_source` for RPG members and `ia_cl_source` for CL members (route by MEMBER_TYPE). Reserve `ia_object_lookup`/`ia_member_lookup` for metadata validation only.

## Pagination Strategy

**For source code:** Both `ia_rpg_source` and `ia_cl_source` cap at 10000 lines/call. For sources exceeding this, loop with `offset=0,10000,20000,…` until you have all lines (verify highest `SOURCE_RRN` ≥ `TOTAL_LINES` from `ia_code_complexity`).

**For other tools:** When result count equals the limit, data is likely truncated. Offer to increase limit or narrow the query.

## Playbooks

### P1: "What references X?" (Where-Used)
Call `ia_find_object_usages` → Group by `using_type`, count each → Flag `*SRVPGM` (cascade risk) and `*DSPF` (user-facing) → Ask: "Are you modifying, deleting, or just understanding X?" → For a quick tally, use `ia_reference_count` instead.

### P2: "What if I change field F in file X?" (Field Impact)
**Always run all 3 steps, then synthesize:**
1. `ia_file_field_impact_analysis(file_name=X, field_name=F)` → direct PF references: *PGM, *SRVPGM, *DSPF, *FILE classified by impact_type
2. `ia_file_dependencies(file_name=X)` → LFs / indexes / views over PF (STRUCTURAL — must be rebuilt)
3. `ia_find_object_usages` on each LF found in step 2 → programs using those LFs (also need change/recompile)

Categorize by `impact_type`: NEEDS_CHANGE=source edit required (field name explicit in RPG or CL source), NEEDS_RECOMPILE=recompile only (implicit record-format or *SRVPGM), STRUCTURAL=rebuild/recreate (*FILE or *DSPF inheriting field via DDS REF).
Quantify: "N programs need code changes, M need recompile, K objects are structural dependencies."
**Notes:** `*SRVPGM` always shows NEEDS_RECOMPILE (source check by SRVPGM name is unreliable). CL programs with the field name in source correctly show NEEDS_CHANGE.
Ask: "Resizing, renaming, or retyping? Each has different implications."

### P3: "Show call tree for program X" (Call Hierarchy)
Call `ia_call_hierarchy` with `direction=BOTH` → Present in two sections: CALLERS / CALLEES → Flag: hub (>5 callers), zero callers (may be scheduler-invoked), deep chains → For parameter inspection at call sites, chain `ia_call_parameters`.

**When CALL_PGM_COUNT = 0 AND CALLEES section is empty:**
Programs may use dynamic command execution that iA cannot statically resolve:
1. Search source: `ia_rpg_source_search(member_name=X, keyword='QCMDEXC')` — detects QCMDEXC calls
2. Check variable ops: `ia_variable_ops(member_name=X, opcode=CMD)` — detects CMD opcode usage
3. If found, note: "Program uses dynamic invocation (QCMDEXC/CALLPRC) — call targets cannot be statically resolved by iA. Manual source review recommended."

### P4: "What variables does program X use?"
Call `ia_program_variables` → Group: standalone fields, DS subfields (likely DB field refs), indicators, arrays → Flag variable names matching DB patterns (short uppercase like CUSTNO, ORDNUM) → For full DS layouts, chain `ia_data_structures`.

### P5: "Retire/delete X" (Full Retirement)
`ia_find_object_usages` for all refs → If ANY refs exist, warn immediately → `ia_call_hierarchy` on key programs to assess depth → `ia_file_field_impact_analysis` on critical fields → Synthesize risk: "HIGH/MODERATE/LOW — N objects, M service programs, D display files."

### P6: "Is it safe to modify X?" (Safety Assessment)
`ia_find_object_usages` or `ia_file_field_impact_analysis` depending on object vs field → Count refs → Apply risk rubric → Check for `*SRVPGM` amplifiers → For code-level risk, add `ia_code_complexity` → State verdict explicitly with evidence.

### P7: "Logical files over physical X?" (File Dependencies)
`ia_file_dependencies(file_name=X)` → Returns LFs, indexes, views, MQTs over the physical, bifurcated by `SQL_OBJECT_TYPE` (INDEX/VIEW/TABLE/MQT/DDS_LF). Use `dependent_kind` to filter → Chain `ia_find_object_usages` on each logical of interest to find programs referencing it.

### P8: "Find dead code"
**Compiled objects (5-step chain — don't stop at step 1):**
1. `ia_unused_objects(object_type='*PGM')` — baseline zero-reference candidates.
2. `ia_cl_jobs(call_type='SBMJOB')` — every CL-side scheduled program. **Subtract** these `called_program` names from step 1.
3. `ia_rpg_source_search(search_text='SBMJOB')` — RPGs that submit jobs (less common but real). Subtract any matches.
4. `ia_object_lifecycle` on the survivors — confirm `LASTUSED_DATE` is old.
5. `ia_find_object_usages` as a final sanity check before recommending DELETE — `LASTUSED_DATE` can lag for cold-stored objects and `ia_unused_objects` misses QCMDEXC dynamic calls.

**Why the chain:** Step 2 typically removes ~40-50% of step 1's candidates — they ARE used, just scheduler-invoked. Never recommend DELETE on lifecycle or unused-objects alone; always surface the QCMDEXC caveat as an unresolved residual risk.

**Orphaned sources:** `ia_uncompiled_sources` for source members never compiled into objects → Check LAST_CHANGED date to identify abandoned development.

### P9: "Where does this program actually write?"
`ia_file_overrides` for the member → If overrides exist, also run `ia_override_chain` → Combine with `ia_find_object_usages` on each resolved target file to confirm downstream impact.

### P10: "What programs include this copybook?" (Copybook Impact)
`ia_copybook_impact(copybook_name="CUSTDS")` → Returns all members with /COPY directive + line numbers → Group by member_type (RPGLE, SQLRPGLE, CBLLE) → **Present the result as an explicit recompile list** (one member per line) so the user can pipe it into their build. Flag SQLRPGLE consumers highest risk (embedded SQL may shift behavior on field-layout change); high total count = phased recompile.

### P11: "What does this service program export?" (API Surface)
`ia_srvpgm_exports(object_name="MYSRVPGM", procedure_type="EXPORT")` → Lists all exported procedures → Chain `ia_procedure_xref` on specific procedures to find callers → For parameter signatures, use `ia_procedure_params`.

### P12: "What procedures call procedure X?" (Procedure-Level Impact)
`ia_procedure_xref(procedure_name="PROCESSORDER", direction="CALLERS")` → More granular than ia_call_hierarchy → Shows procedure-to-procedure relationships → Critical for ILE modular code.

### P13: "Find batch jobs and scheduler calls" (Job Detection)
`ia_cl_jobs(call_type="SBMJOB")` → Returns SBMJOB calls with job queue, hold flag → Critical: programs appearing "unused" may be scheduler-invoked → Cross-reference with ia_unused_objects to avoid false positives.

### P14: "What files does program X use (with prefixes)?"
`ia_program_files(member_name="ORDENTRY", library="PRDLIB")` → Shows files with PREFIX, RENAME, record format details → Use `library` param to scope to a specific library when the same member name exists in multiple libraries → More detailed than ia_find_object_usages for understanding file access patterns.

### P15: "Scope analysis to a project area"
`ia_application_area(area_name="*LIST")` → List all defined areas → Then `ia_application_area(area_name="MYPROJECT")` → Returns objects in that area for scoped analysis.

### P16: "Resolve SQL long name to system name"
`ia_sql_names(name_pattern="STORE%")` → Maps SQL long names ↔ 10-char system names → Essential for SQL procedure/function analysis.

### P17: "What happens if I change this service program?" (SRVPGM Impact)
Service programs are **cascade amplifiers** — changes affect all binding programs.

**Analysis sequence:**
1. `ia_srvpgm_exports(object_name="MYSRVPGM", procedure_type="EXPORT")` → List exported procedures (the API surface)
2. `ia_find_object_usages(object_name="MYSRVPGM", object_type="*SRVPGM")` → Programs that bind to it
3. For signature changes: `ia_procedure_params(procedure_name="<proc>")` → Current signatures
4. For procedure-level callers: `ia_procedure_xref(procedure_name="<proc>", direction="CALLERS")` → Which programs call which procedures

**Risk assessment:**
| Consumers | Risk | Guidance |
|-----------|------|----------|
| 0 | Safe | Unused SRVPGM — verify before deleting |
| 1–5 | Low | Test each binding program |
| 6–20 | Moderate | Coordinate with program owners |
| 20+ | High | Phased rollout with regression testing |

**Signature changes** (adding/removing/retyping parameters) require **recompile of ALL consumers**. Adding new procedures is safe; removing procedures breaks callers.

### P18: "Orphaned + tangled — refactor or archive triage" (Quadrant Analysis)
Cross-tool chain for modernization audits. Splits programs into 4 quadrants by usage age × complexity.

1. `ia_code_complexity(member_name='*ALL', limit=200)` → score each row: `GOTO×5 + CAB×5 + CAS×5 + IF×1 + SQL×0.5`. High score = tangled.
2. `ia_object_lifecycle` on the top-scoring candidates → read `LASTUSED_DATE`.
3. Classify each program into one of four quadrants:
   - **Active & Clean** (recent + low score) — leave alone.
   - **Active & Tangled** (recent + high score) — refactor priority.
   - **Sleeping & Clean** (old + low score) — archive candidate.
   - **Orphaned & Toxic** (old + high score) — audit FIRST, then archive.
4. For any "Orphaned & Toxic" candidate, validate with `ia_find_object_usages` before recommending deletion — `LASTUSED_DATE` alone is not delete-safe.

**Why this matters:** Modernization-scope studies show isolating "Orphaned & Toxic" can cut effective refactor scope ~30%. The quadrant verdict is the deliverable, not the raw table.

## Risk Rubric

| References | Risk | Guidance |
|-----------|------|---------|
| 0 | Safe | No code dependencies — but verify job schedulers and external systems |
| 1–5 | Low | Limited blast radius — list specific objects that need testing |
| 6–20 | Moderate | Review each affected object before proceeding |
| 20+ | High | Wide blast radius — recommend phased approach with regression testing |

## Follow-Up Patterns

| Result Pattern | Offer |
|---------------|-------|
| `*SRVPGM` in results | "SRVPGM X is shared — check what binds to it?" |
| `*DSPF` in results | "Display file X is user-facing — check which programs use it?" |
| >10 affected objects | "N objects affected — narrow by type or investigate critical ones first?" |
| Null `field_usage` | "N programs have unknown field usage — inspect with `ia_program_variables`?" |
| Program with >5 callers | "X has N callers — critical junction. Check what files it references?" |
| Program with 0 callers | "X has no callers in repository — may be invoked by scheduler or external system." |
| Results hit limit | "Capped at N rows — narrow the query (by library, type, or specific name) for the full picture." |
| Zero results | "No results — verify uppercase spelling. If correct, check schedulers/external systems." |

## Response Rules

1. **Never dump raw tables.** Summarize first: "X is referenced by 15 programs, 3 service programs, 4 display files."
2. **Group by object type.** This is how IBM i developers think about dependencies.
3. **State risk explicitly.** "This is a high-risk change because..." — don't just show data.
4. **Use IBM i terms.** Say "physical file" not "table", "service program" not "shared library" — unless user uses SQL terms.
5. **Always suggest a concrete next step.** Every response ends with a specific follow-up action.

## Tool Efficiency Rules

1. **One query is usually enough.** Before calling a second tool, ask: "Can I answer this from results I already have?"
2. **Use `ia_program_detail` for program anatomy.** It returns calls, files, subroutines, variables, overrides, and call parameters in ONE query — no need for 6 separate calls.
3. **Don't chain redundantly.** `ia_find_object_usages` already gives you everything; don't follow up with `ia_reference_count` on the same object.
4. **Skip intermediate steps.** Don't call `ia_member_lookup` just to get location before `ia_rpg_source_tokens` — go straight to the token analysis.
5. **Batch your thinking.** If results show 5 SRVPGMs, don't call `ia_find_object_usages` on each one individually — ask the user which ones matter first.
6. **Respect the 80/20 rule.** 80% of user questions can be answered with these tools in 1 call:
   - `ia_find_object_usages` — what uses X?
   - `ia_file_field_impact_analysis` — what if I change field F?
   - `ia_call_hierarchy` — call tree for X?
   - `ia_program_detail` — tell me about program X?
   - `ia_code_complexity` — complexity hotspots?
   - `ia_unused_objects` — dead compiled objects?
   - `ia_uncompiled_sources` — orphaned sources?
   - `ia_object_lookup` — find object named X?
   - `ia_dashboard` — repository overview?
   - `ia_copybook_impact` — what includes copybook X?
   - `ia_srvpgm_exports` — what does SRVPGM X export?
   - `ia_procedure_xref` — what calls procedure X?
   - `ia_cl_jobs` — batch job detection?
