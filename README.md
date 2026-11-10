# Round-Robin Lead Assignment for Zoho CRM (Deluge)

A round-robin assignment engine for Zoho CRM Leads that keeps a
persistent pointer, skips members who are unavailable, and writes an
audit log for every decision.

Built and verified in a real Zoho CRM org (Ultimate trial, JP data
center) against a 23-case test plan. See "Verification" and
"Limitations" below for exactly what was and was not proven.

## Why not the built-in assignment rules?

Zoho CRM's standard assignment rules are a reasonable default, but
checking them against the official documentation and community reports
left four gaps this project set out to close:

- **No documented persistent pointer.** Nothing in the official docs
  says the rotation position survives between imports. Community
  reports describe the order restarting from the top on each import,
  so the first members collect more leads than the last ones.
- **Limited trigger coverage.** Per the official FAQ, assignment rules
  fire on import, web forms, and API creation - not on records created
  manually or by workflows. A create-workflow function has no such gap.
- **One default user as the only fallback**, with no record of why a
  lead landed there.
- **No assignment log.** There is no built-in way to prove afterwards
  who was skipped, why, and what the rotation state was.

## How it works

- **Pools** (`RR_Pools`) select leads by matching one field value
  (for example a territory field), ordered by priority. One pool can
  be marked as the default.
- **Members** (`RR_Members`) belong to a pool and carry a sequence
  number plus availability settings: active flag, accepting-new flag,
  work days, a daily shift window, an absence range, and a daily cap.
- **State** (`RR_State`) holds one row per pool with the rotation
  pointer (`Last_Sequence`) and counters. The pointer update uses a
  conditional update (`If-Unmodified-Since`); a stale write is
  rejected with HTTP 412 instead of silently overwriting.
- **The engine** (`rr_engine`) claims the lead record first (status +
  a unique operation id, again conditionally), so a duplicate trigger
  exits as `SKIPPED_ALREADY_ASSIGNED` instead of double-assigning.
  It then evaluates every member, records a snapshot of the decision
  (for example `2:Shift,3:OK,1:OK`), claims the next sequence,
  updates the owner, and reads the record back to verify.
- **Logs** (`RR_Assignment_Logs`) get one row per attempt: result
  (Assigned / Fallback / Unassigned_NoEligible / Unassigned_NoPool /
  Config_Error / skips), the sequence used, the member snapshot, the
  operation id, and any error message.
- **Entry points**: a workflow rule on Lead creation
  (`rr_assign_lead_workflow`) and a detail-page button
  (`rr_reassign_btn`) that refuses to re-run within 5 minutes of the
  last assignment.

Fairness is defined narrowly and honestly: the rotation cycles
without gaps among the members who are eligible at that moment.
Members who were away do not get catch-up compensation later.

## Modules and fields

| Module | Purpose | Key fields |
|---|---|---|
| RR_Pools | Assignment pools | Pool_Key, Match_Field_API_Name, Match_Value, Priority, Timezone, Is_Default, Fallback_User_Id, Active |
| RR_Members | Pool members | Pool (lookup), User_Id, Sequence, Active, Accepting_New, Work_Days, Shift_Start/End, Absence_From/To, Daily_Cap |
| RR_State | Rotation state, one row per pool, pre-created by setup | Pool (lookup), Last_Sequence, Total_Claimed, Total_Assigned, Last_Assigned_User_Id |
| RR_Assignment_Logs | Audit log | Lead (lookup), Pool, Result, Sequence_Used, Assigned_User_Id, Members_Snapshot, Today_Count, Source, Operation_Id, Executed_At, Error_Message |

Leads get five fields: RR_Territory (the match field used in the
sample data), RR_Assignment_Status, RR_Pool_Key,
RR_Last_Assigned_At, RR_Operation_Id.

## Files

| File | What it is |
|---|---|
| `rr_engine.ds` | The assignment engine (standalone function) |
| `rr_assign_lead_workflow.ds` | Workflow wrapper: calls the engine with source `Create_Workflow` |
| `rr_reassign_btn.ds` | Button function: 5-minute debounce, then calls the engine with source `Button_Reassign` |
| `setup_stage1_create_modules.ds` | Creates the four modules via the Settings API |
| `setup_stage2_fields_pools.ds` / `setup_stage2b_fields_members_state.ds` / `setup_stage2c_fields_logs_leads.ds` | Creates all fields |
| `setup_stage3_seed_data.ds` | Sample pools, members, and state rows (edit before running) |
| `test_driver.ds` / `test_driver2.ds` / `test_extra.ds` | The scenario drivers used for verification |

## Configuration (read before use)

The published code is sanitized. Before it runs anywhere, set:

1. **Connection name.** Every `invokeurl` block uses the literal
   `"your_connection"`. Deluge requires the connection name as a
   literal in `invokeurl`, so replace it with the link name of a Zoho
   OAuth connection in your org that has CRM module, COQL, and
   settings scopes.
2. **Data center.** URLs use `www.zohoapis.com`. Change to your DC
   (`.eu`, `.in`, `.jp`, `.com.au`, `.cn`) if needed.
3. **User IDs.** Sample data and test drivers contain `YOUR_USER_ID`
   placeholders. Replace with real CRM user IDs from your org.
4. **Timezone handling - the big gotcha.** In the org this was built
   in, `zoho.currenttime` inside Deluge returned a wall clock 17
   hours behind JST (a US Pacific wall clock), and the offset changes
   with US daylight saving time. The engine therefore corrects the
   time (+17h, or +16h during US DST, computed from the wall-clock
   date) and evaluates shifts and absences against the corrected
   time. Verify what `zoho.currenttime` returns in your org before
   trusting any time-based rule; do not copy the offset blindly.
   Pools carry a Timezone label, but only `Asia/Tokyo` was exercised.

## Setup outline

1. Run the setup stages in order (modules, fields, seed data), or
   create the modules and fields manually from the table above.
2. Create exactly one `RR_State` row per pool with
   `Last_Sequence = 0`. The engine stops with `Config_Error` if a
   pool has zero or multiple state rows, and also if two members in a
   pool share a Sequence value.
3. Create a workflow rule: Leads, on Create, all records, instant
   action = the workflow wrapper function, argument `leadId` mapped
   to the Lead Id.
4. Create a detail-page button on Leads pointing at the button
   function, argument `leadId` mapped to the Lead Id.
5. Create one lead and check that an `RR_Assignment_Logs` row appears.

## Verification

Tested 2026-11-09/10 against a 23-case plan (rotation order and
wrap-around, continuation across batches and across days, territory
routing and the default pool, skips for not-accepting / inactive /
absence / shift / day-off / daily cap, fallback with and without a
fallback user, no matching pool, duplicate execution, member row
deleted mid-rotation, missing and duplicated state rows, duplicate
sequence values, shift boundaries on both sides - start inclusive,
end exclusive, and the button debounce).

Results: 15 cases passed outright, 4 passed with a reduced setup
(second pool had a single member; sequential burst of 6 instead of
20; near-simultaneous double run tested sequentially), and 1 was not
run (a true parallel API burst - the conditional update it relies on
was verified directly: a stale write returns HTTP 412).

Two findings worth knowing:

- A member row pointing at a non-existent CRM user ID cannot even be
  created - the API returns success-shaped output with a null id.
  The engine still validates members against the live user list
  (active + confirmed), but that path could not be exercised with
  real data in a single-user org.
- The daily cap counts assignments per user, not per member row, so
  in an org where several member rows point at the same user, the cap
  behaves as a per-user cap. That is by design here, but it means the
  "one member capped, others continue" case only exists in
  multi-user orgs.

## Limitations

- Verified in a **single-user org**: all member rows pointed at one
  real user, so rotation was proven at the level of sequences,
  pointers, and logs - not by watching different owners receive
  leads. The owner-update and read-back steps did run on every
  assignment.
- The daily cap is best-effort: it counts today's assignment logs for
  the create-workflow source. It is a pacing control, not a hard
  quota.
- Best-effort concurrency: the claim and pointer updates are
  conditional and conflicts surface in the log, but this is not a
  distributed lock.
- Leads only. Weights and load-based balancing are out of scope.
- `zoho.crm.deleteRecord` does not exist as a Deluge built-in in this
  context (nor does `zoho.crm.getUsers`); the test drivers delete via
  `invokeurl` DELETE calls. Regex escapes in `replaceAll` need a
  single backslash in the editor - a doubled backslash compiles but
  throws at runtime.

## Status

Working prototype built as a portfolio evidence pack. Not client
work, and not a packaged product. License: TBD by the author.
