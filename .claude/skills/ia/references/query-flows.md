# iA Query Flows — Tool Chains for Common Analysis Tasks

This reference documents optimal tool sequences for common queries. Follow these flows to minimize tool calls while maximizing insight.

---

## Tier 1: High-Impact Analysis (Most Valuable)

### QF-1: "What happens if I change this file/field?" (Full Impact Analysis)

**Standard 3-tool chain (always run all steps, then synthesize):**
```
1. ia_file_field_impact_analysis(file_name="CUSTMAST", field_name="CUSTNO", limit=500)
   → Direct PF references: *PGM, *SRVPGM, *DSPF, *FILE
   → impact_type: NEEDS_CHANGE / NEEDS_RECOMPILE / STRUCTURAL

2. ia_file_dependencies(file_name="CUSTMAST")
   → LFs / indexes / views over the PF (STRUCTURAL — must be rebuilt)
   → If none, stop here

3. ia_find_object_usages(object_name="<each_LF>")  [parallel for multiple LFs]
   → Programs/objects referencing those LFs — also impacted
```

**impact_type values:**
- `NEEDS_CHANGE` — field name found explicitly in RPG or CL source; requires source edit
- `NEEDS_RECOMPILE` — file referenced but field not in source (implicit record-format access); or `*SRVPGM` (source check by object name is unreliable)
- `STRUCTURAL` — `*FILE` or `*DSPF` that inherits field definitions via DDS REF(); must be rebuilt, not just recompiled

**Present as one response grouped by section:**
- "Directly references PF": results from step 1
- "Logical files / views over PF": results from step 2
- "Programs via LF": results from step 3

**Optimization:** Skip step 3 if ia_file_dependencies returns no LFs. Only chain `ia_call_hierarchy` for specific critical programs the user identifies.

---

### QF-2: "Map complete dependency chain for a program"

**Combined approach (2 calls max):**
```
1. ia_program_detail(program_name="ORDENTRY", section="*ALL") → Gets calls, files, subroutines, variables, overrides in ONE query
2. ia_call_hierarchy(program_name="ORDENTRY", direction="both") → Full call tree
```

**Why this works:** `ia_program_detail` with `section="*ALL"` returns 6 types of information in a single query, eliminating the need for separate calls to ia_file_overrides, ia_subroutines, ia_program_variables.

---

### QF-3: "Find dead code / unused objects for cleanup"

**Efficient approach:**
```
1. ia_unused_objects(object_type="*PGM", limit=100) → Zero-reference compiled objects
2. ia_cl_jobs(call_type="SBMJOB", limit=500)        → Subtract these called_program names from step 1 — they ARE used (scheduler-invoked)
3. ia_rpg_source_search(search_text="SBMJOB", limit=500) → RPG-side SBMJOB sites; also subtract
4. ia_uncompiled_sources(member_type="RPGLE", limit=100) → Orphaned sources never compiled
5. ia_code_complexity(member_name="*ALL", limit=50) → Also shows CALLED_BY_COUNT (0 = dead)
```

**Critical:** Steps 2-3 are mandatory before recommending any DELETE — cross-referencing with `ia_cl_jobs` typically removes ~40-50% of step 1's "unused" entries (they're SBMJOB-invoked). QCMDEXC dynamic calls remain undetectable; surface this as residual risk.

**No need to chain** `ia_object_lifecycle` for every object — the unused_objects query already confirms zero references. Only check lifecycle for specific objects the user wants to investigate.

---

### QF-4: "Assess risk before modifying a shared file"

**Two-query approach:**
```
1. ia_find_object_usages(object_name="CUSTMAST", object_type="*FILE")
   → Returns: using objects with their types
2. ia_file_fields(file_name="CUSTMAST") → Field definitions
```

**Skip** `ia_reference_count` — it's redundant when you already have `ia_find_object_usages` results (just count them).

---

### QF-5: "Find circular dependencies"

**Single query:**
```
ia_circular_deps()
```

**Only chain** `ia_call_hierarchy` if circular deps are found AND user wants to understand the specific call chain.

---

## Tier 2: Program Understanding

### QF-6: "Complete program anatomy"

**Single comprehensive query:**
```
ia_program_detail(program_name="ORDENTRY", section="*ALL", limit=500)
```

Returns CALLS, FILES, SUBROUTINES, VARIABLES, OVERRIDES, CALL_PARAMS in ONE query.

**Only add** `ia_code_complexity` if user specifically asks about complexity metrics.

---

### QF-7: "Understand how data flows through calls"

**Two-query approach:**
```
1. ia_call_hierarchy(program_name="ORDENTRY", direction="called") → Call targets
2. ia_call_parameters(caller_name="ORDENTRY") → Parameters at each call site
```

**Skip** `ia_data_structures` unless user specifically asks about DS layouts.

---

### QF-8: "Identify most complex/risky programs"

**Single query:**
```
ia_code_complexity(member_name="*ALL", limit=200)
```

Returns: IF_COUNT, DO_COUNT, SQL_COUNT, GOTO_COUNT, CAB_COUNT, CAS_COUNT, CALLED_BY_COUNT, TOTAL_OPERATIONS — all complexity metrics in one call.

**Default sort is TOTAL_LINES DESC — re-rank in synthesis.** Lines alone are misleading: a 5,000-line SQL-heavy program can be healthier than a 1,000-line GOTO-spaghetti program. Apply this weighted score per row:

```
score = GOTO_COUNT*5 + CAB_COUNT*5 + CAS_COUNT*5 + IF_COUNT*1 + DO_COUNT*1 + SQL_COUNT*0.5
```

Present top N by `score`, not by `TOTAL_LINES`. Tie-break with `CALL_PGM_COUNT` (more callers = higher refactor priority because impact-amplification).

**Only drill deeper** on specific programs the user identifies as concerning. Pair with `ia_object_lifecycle` to spot "orphaned + tangled" (see playbook P18).

---

### QF-9: "Find all file overrides and their chains"

**Two-query approach:**
```
1. ia_override_chain() → All chained overrides (A→B→C)
2. ia_file_overrides(member_name="<specific>") → Only if user wants details for a specific member
```

---

## Tier 3: Modernization Planning

### QF-10: "DDS to DDL migration assessment"

**Single query:**
```
ia_dds_to_ddl_status(limit=200)
```

**Only chain** `ia_find_object_usages` for specific files the user wants to prioritize.

---

### QF-11: "Repository health dashboard"

**Single query:**
```
ia_dashboard()
```

Returns: Object counts by category, line counts, library mapping — comprehensive overview in one call.

**Only add** these if user asks follow-up questions:
- `ia_code_complexity(member_name="*ALL", limit=10)` for hotspots
- `ia_unused_objects` for cleanup candidates
- `ia_exception_log` for parser issues

---

## Tier 4: Advanced Analysis

### QF-12: "Copybook change impact analysis"

**Single query:**
```
ia_copybook_impact(copybook_name="CUSTDS", limit=200)
```
Returns: All members including the copybook, with line numbers and member types.

**Chain only if** user wants to understand the programs that use those members — then use `ia_find_object_usages` on specific compiled objects.

---

### QF-13: "Service program API surface"

**Two-query approach:**
```
1. ia_srvpgm_exports(object_name="MYSRVPGM", procedure_type="EXPORT") → Exported procedures
2. ia_procedure_params(procedure_name="<specific>", library="PRDLIB") → Parameter signatures
```

**Alternative:** For procedure callers, use `ia_procedure_xref(procedure_name="X", direction="CALLERS")`.

---

### QF-14: "Procedure-level call graph"

**Single query:**
```
ia_procedure_xref(procedure_name="PROCESSORDER", direction="BOTH", limit=100)
```
Returns: Both callers and callees at procedure level — more granular than program-level ia_call_hierarchy.

---

### QF-15: "Find batch jobs and scheduler calls"

**Single query:**
```
ia_cl_jobs(call_type="SBMJOB", limit=100)
```
Returns: SBMJOB calls with job name, job queue, hold flag.

**Critical insight:** Cross-reference with `ia_unused_objects` — programs appearing "unused" may actually be scheduler-invoked.

---

### QF-16: "Program file usage with prefixes"

**Single query:**
```
ia_program_files(member_name="ORDENTRY", library="PRDLIB", limit=50)
```
Returns: Files used with PREFIX, RENAME, record format — more detailed than ia_find_object_usages for file analysis. Use `library` to scope when the same member exists in multiple libraries.

---

### QF-17: "Scoped analysis by application area"

**Forward (area → objects):**
```
1. ia_application_area(area_name="*LIST") → List all defined areas
2. ia_application_area(area_name="MYPROJECT") → Objects in specific area
```

**Reverse (object → areas):**
```
ia_application_area(object_name="CUSTMAST") → All areas containing CUSTMAST
ia_application_area(object_name="%CUST%")   → Wildcard: areas with any CUST* object
```

---

### QF-18: "SQL name resolution"

**Single query:**
```
ia_sql_names(name_pattern="STORE%", limit=50)
```
Returns: SQL long names ↔ 10-char system names mapping.

---

## Tier 5: Source Code Analysis

### QF-19: "Read source code for a program/object" (e.g., "show me ADMINP source", "business rules of ORDENTRY")

**Two-step approach:**
```
1. ia_object_lookup(object_name="ADMINP") → Get source member, source file, library, MEMBER_TYPE
2. Route by MEMBER_TYPE:
   - RPGLE / SQLRPGLE / RPG / SQLRPG → ia_rpg_source(member_name=..., library_name=...)
   - CLLE / CLP / CL                  → ia_cl_source(member_name=..., library_name=...)
```

**Why object_lookup first:** Programs may have different source member names than object names. Object lookup returns the actual source file (SRCFILE), member name (SRCMBR), library, and MEMBER_TYPE needed to select the right source tool.

**Pagination — mandatory for sources >10,000 lines:**
Both `ia_rpg_source` and `ia_cl_source` cap `limit` at 10000 per call. Read `TOTAL_LINES` from `ia_code_complexity` first; if larger, loop with `offset=0, 10000, 20000, …` until you have all lines (stop when a call returns fewer than 10,000 rows). Verify the highest `SOURCE_RRN` returned equals `TOTAL_LINES`.

**Spec-type filtering (RPG only, when user wants specific sections):**
```
ia_rpg_source(member_name=..., library_name=..., source_spec="P")   # P=procedures, D=definitions, F=files, C=calc
```

**Present results:** Show source code with complexity metrics (IF/DO/SQL counts, executable lines from `ia_code_complexity`).

---

### QF-20: "Read source for a member name" (e.g., "show ABC member", "what does CUSTPROC do")

**Two-step approach:**
```
1. ia_member_lookup(member_name="ABC") → Get source file, library, MEMBER_TYPE, timestamps
2. Route by MEMBER_TYPE → ia_rpg_source(...) or ia_cl_source(...)
```

**Multiple members — STOP and ask:**

If `ia_member_lookup` returns multiple matches (same member name in different libraries/source files):

1. **Present version summary table:**

| Library | Source File | Type | Lines | Last Changed |
|---------|-------------|------|-------|--------------|
| PRDLIB | QRPGLESRC | SQLRPGLE | 1250 | 2025-03-15 |
| DEVLIB | QRPGLESRC | SQLRPGLE | 1312 | 2026-01-20 |
| TESTLIB | QRPGSRC | RPGLE | 980 | 2024-08-01 |

2. **Ask explicitly:** "Found N versions of 'ABC'. Which version would you like me to show? (specify library or row number)"

3. **Wait for user selection.** Do NOT show source for any version until the user chooses.

4. Once selected, call the appropriate source tool with `library_name=<chosen>`.

**Never:** Show source for the first match by default or retrieve all versions in parallel — this wastes context and may document the wrong version.

---

### QF-21: "Deep token-level analysis"

**For RPG:**
```
1. ia_rpg_source_tokens(member_name="ORDENTRY", limit=500)
```

**For CL:**
```
1. ia_cl_source_tokens(member_name="RUNJOB", limit=500)
```

**Skip** `ia_member_lookup` just to get location — go straight to the token analysis.

---

### QF-22: "Find all uses of a specific variable/field name"

**Single query:**
```
ia_file_field_impact_analysis(field_name="ORDAMT", file_name="*ALL", limit=500)
```

**Only add** `ia_program_variables` for specific programs where you need to see all variables, not just the target field.

---

## Tier 6: Discovery & Inventory

### QF-23: "What objects exist matching a pattern?"

**Single query:**
```
ia_object_lookup(object_name="%ORD%", limit=100)
```

Supports `%` wildcards. Returns type, library, attribute for all matches.

---

### QF-24: "List all service programs and their callers"

**Two-query approach:**
```
1. ia_object_lookup(object_name="%SRV") → Find SRVPGMs
2. ia_find_object_usages(object_name="<srvpgm>", object_type="*SRVPGM") → For each SRVPGM of interest
```

---

## Optimization Decision Tree

```
User Question
    │
    ├─► "What uses X?" ──────────► ia_find_object_usages (single call)
    │                              └─► Chain ia_call_hierarchy ONLY if *SRVPGM in results
    │
    ├─► "Impact of changing X?" ─► ia_file_field_impact_analysis (single call)
    │                              └─► Chain ia_find_object_usages ONLY for *SRVPGM amplifiers
    │
    ├─► "What does X call?" ─────► ia_call_hierarchy direction=called (single call)
    │
    ├─► "Tell me about program X" ► ia_program_detail section=*ALL (single call)
    │                               └─► Covers: calls, files, subroutines, vars, overrides
    │
    ├─► "Dead code (compiled)?" ─► ia_unused_objects (single call)
    │
    ├─► "Orphaned sources?" ─────► ia_uncompiled_sources (single call)
    │
    ├─► "Complexity hotspots?" ──► ia_code_complexity member_name=*ALL (single call)
    │
    ├─► "Find object named X" ───► ia_object_lookup with % wildcards (single call)
    │
    ├─► "Repository overview" ───► ia_dashboard (single call)
    │
    ├─► "Copybook change impact?" ─► ia_copybook_impact (single call)
    │
    ├─► "SRVPGM API surface?" ────► ia_srvpgm_exports (single call)
    │
    ├─► "Procedure callers?" ─────► ia_procedure_xref (single call)
    │
    └─► "Batch jobs?" ────────────► ia_cl_jobs (single call)
```

---

## Anti-Patterns (Avoid These)

| Bad Pattern | Why It's Bad | Better Approach |
|-------------|--------------|-----------------|
| `ia_find_object_usages` then `ia_reference_count` on same object | Redundant — just count the where_used results | Use only `ia_find_object_usages` |
| `ia_member_lookup` just for location before `ia_rpg_source_tokens` | Only returns location metadata, not content | Go straight to `ia_rpg_source_tokens` |
| `ia_program_summary` + `ia_program_variables` + `ia_subroutines` | Three queries for one program | Use `ia_program_detail section=*ALL` |
| `ia_object_lifecycle` for every unused object | Unnecessary — unused_objects already confirms zero refs | Only check lifecycle for specific objects |
| Chaining to `ia_call_hierarchy` for every *PGM result | Overkill — most programs don't need deep analysis | Only chain for *SRVPGM or critical programs |
| Ranking complexity by TOTAL_LINES only | Large+linear is healthier than small+GOTO-spaghetti | Weighted score: `GOTO×5 + CAB×5 + IF×1 + SQL×0.5` (see QF-8) |
| Recommending DELETE from `ia_unused_objects` alone | Misses SBMJOB / QCMDEXC scheduler-invoked programs (~40-50% false positives) | Always cross-reference `ia_cl_jobs` + `ia_rpg_source_search('SBMJOB')` first |
| Recommending DELETE from `ia_object_lifecycle` alone | LASTUSED_DATE can lag for cold-stored objects | Always verify with `ia_find_object_usages` before any deletion recommendation |

---

## Response Efficiency Rules

1. **Answer in one query when possible.** Most questions can be answered with a single well-chosen tool.

2. **Chain only when results demand it.** Seeing `*SRVPGM` in results = chain. Seeing only `*PGM` = usually stop.

3. **Use section filters.** `ia_program_detail section=FILES` is faster than `section=*ALL` if user only asks about files.

4. **Respect result counts.** If < 10 results, summarize and stop. If > 50, offer to narrow before deep-diving.

5. **Don't pre-chain.** Run the first query, interpret results, THEN decide if chaining adds value.
