# TicketFlow — Technical README

**Project owner:** Bohan Ma (AI Business Partner, Kubrick Group)
**Stakeholders:** Natasha Thomas (project owner)
**Last Feature Update:** July 28, 2026
**Version:** v2 (post-demo additions: Priority Tier, Start/Due Dates, Checklist, richer description, timezone fix)
**Status:** Production-ready (handover pending Natasha's cutover approval); Phase 6 (SLA alerts) ~95% built, Power Automate connector stalling, shelved for now.

---

## 1. What this is

TicketFlow is the Kubrick internal team's automated intake system. It converts Microsoft Forms submissions into Microsoft Planner tasks with auto-tagging, auto-routing to the correct Resource Managers (RMs) by region × capability, Teams notifications to assignees, and (in-progress) SLA alerts when tickets stall.

The system consists of **two separate Power Automate flows**:

1. **TicketFlow** — event-driven; fires on each form submission. Handles intake, ticket creation, RM routing, and notifications.
2. **TicketFlow — Stage 2 SLA Alerts** — time-driven; runs on a schedule. Detects tickets stuck in early stages and fires reminder DMs.

Both flows are currently built against a test Planner board ("Project TicketFlow Test Board A"). Production cutover to the real form and Planner board is pending Natasha's scheduling.

---

## 2. Key identifiers

These IDs are referenced throughout both flows. If anything is rebuilt, copy these exactly.

### Plan and group

| Item | Value |
| --- | --- |
| Group: Innovation Team | `c77d9084-e455-4a49-8b7b-292cc400edef` |
| Plan: Project TicketFlow Test Board A | `DnDvOQKb_EiZg-wUJVnzjpYAHgxP` |

### Bucket IDs (Planner stages)

| Stage | Bucket name in Planner | Bucket ID |
| --- | --- | --- |
| Stage 1 | Stage 1 - New Request | `zBpt4ehFxkyYHGe0S55hfZYAA1CL` |
| Stage 2 | Stage 2 - Assigned (Awaiting Contract) [sic — typo in bucket name; should be "Contact"] | `46b12RpLw0qd9b1EUIf7-JYAES7w` |
| Stage 3 | Stage 3 - Active | `eWziJo9EPkC89DdxhVfTHZYAJIPq` |
| Stage 4 | Stage 4 - At Risk / Stalled | `oKYWR9ig3USF0ceYb2J9jZYAKKn6` |
| Stage 5 | Stage 5 - Ready for Rotation | `Qw0SOS4fb0SkRLCg5ULCv5YAA1U4` |

### Form field IDs (Microsoft Forms internal identifiers)

| Field | ID |
| --- | --- |
| Capability Required (multi-select) | `r5ec9a97f0a07442dae2aa56ea3344b3c` |
| Region (single-select) | referenced via dynamic content as "Region Where should..." |
| Responder/Requester | `body/responder` (returns UPN) |
| Submission date | `body/submitDate` (returned in UTC) |
| Priority Tier (single-select) | referenced via dynamic content |
| Required Start Date | referenced via dynamic content |
| Required By Date | referenced via dynamic content (added June 2026) |
| Start Date Flexibility | referenced via dynamic content |
| Estimated Duration | referenced via dynamic content |
| Request Purpose | referenced via dynamic content |
| Expected Deliverables | referenced via dynamic content |
| Request Type | referenced via dynamic content |
| Ticket Title | referenced via dynamic content |

### Form Capability options (string values — exact-match sensitive)

The Capability field returns a stringified JSON array. The six possible values are:

```
Data Engineering
AI (ML & GenAI)
Platform Engineering
Data & AI Product Management
Data & AI Governance
Applied Data Intelligence
```

### Form Region options

```
UK
US
Global
```

### Form Priority Tier options (exact strings — en-dash, not hyphen)

⚠️ The character between "Tier N" and the description is `–` (en-dash, Unicode U+2013), NOT `-` (hyphen). Switch cases must match exactly.

```
Tier 1 – Revenue generating (Solutions/ BD / proposals)
Tier 2 – Strategic internal initiative
Tier 3 – Non-critical / support work
```

### Form Priority Tier → Planner Priority mapping

| Form Priority Tier | priorityInt | Planner display |
| --- | --- | --- |
| Tier 1 – Revenue generating (Solutions/ BD / proposals) | 1 | Urgent |
| Tier 2 – Strategic internal initiative | 3 | Important |
| Tier 3 – Non-critical / support work | 5 | Medium |
| (no match / default) | 5 | Medium |

### Form Start Date Flexibility options

```
None
Under 1 week
1-2 weeks
2-4 weeks
Other
```

### Form Estimated Duration options

```
< 1 week
1-2 weeks
2-4 weeks
1-2 months
2+ months
Other
```

### Planner Label colour → Capability mapping

The plan has Planner Premium enabled, so 25 label colours are available. Six are used:

| Colour | Capability label |
| --- | --- |
| Cranberry | Data Engineering |
| Orange | AI (ML & GenAI) |
| Peach | Platform Engineering |
| Light green | Data & AI Product Management |
| Teal | Data & AI Governance |
| Lavender | Applied Data Intelligence |

---

## 3. Resource Manager routing matrix

The routing is keyed on Region × Capability. UPNs follow the pattern `firstnamelastname@kubrickgroup.com` (no dot separator).

### US region

| Capability | Resource Manager | UPN |
| --- | --- | --- |
| Data Engineering | Emily Coulson | `emilycoulson@kubrickgroup.com` |
| Platform Engineering | Emily Coulson | `emilycoulson@kubrickgroup.com` |
| AI (ML & GenAI) | Selin Yolladi | `selinyolladi@kubrickgroup.com` |
| Data & AI Product Management | Selin Yolladi | `selinyolladi@kubrickgroup.com` |
| Data & AI Governance | Selin Yolladi | `selinyolladi@kubrickgroup.com` |
| Applied Data Intelligence | Selin Yolladi | `selinyolladi@kubrickgroup.com` |

### UK region

| Capability | Resource Manager | UPN |
| --- | --- | --- |
| Data Engineering | Conor McLachlan | `conormclachlan@kubrickgroup.com` |
| Platform Engineering | Jessica Cubbison | `jessicacubbison@kubrickgroup.com` |
| AI (ML & GenAI) | Chloe Miles | `chloemiles@kubrickgroup.com` |
| Data & AI Product Management | Selva Ross | `selvaross@kubrickgroup.com` |
| Data & AI Governance | Sianika Malcolm | `sianikamalcolm@kubrickgroup.com` |
| Applied Data Intelligence | Chloe Miles | `chloemiles@kubrickgroup.com` |

### Global region (triage)

Both RMs always notified — they manually reassign as needed.

| RM | UPN |
| --- | --- |
| Anna Shadbolt | `annashadbolt@kubrickgroup.com` |
| Natasha Thomas | `natashathomas@kubrickgroup.com` |

In addition to RM assignment, the **requester** is always added to the task and notified, regardless of region.

---

## 4. Flow 1: TicketFlow (intake automation)

### 4.1 Trigger

**When a new response is submitted** (Microsoft Forms connector)

Form: AIBP intake form (placeholder during development; production form pending cutover).

### 4.2 Top-level architecture

```
When a new response is submitted (trigger)
    ↓
Get response details
    ↓
Initialize variable: priorityInt (Integer)
    ↓
Initialize variable: priorityTierDescription (String)
    ↓
Switch on Priority Tier (3 cases) → set priorityInt + priorityTierDescription
    ↓
Initialize 6 Boolean variables (one per capability):
  isDataEng, isAI_MLGenAI, isPlatformEng,
  isDataAIProdMgmt, isDataAIGov, isAppliedDataIntel
    ↓
Initialize 9 String variables (one per RM):
  rmEmilyCoulson, rmSelinYolladi, rmConorMclachlan,
  rmSelvaRoss, rmChloeMiles, rmJessicaCubbison,
  rmSianikaMalcolm, rmAnnaShadbolt, rmNatashaThomas
    ↓
Apply to each (Capability — parsed from form JSON)
    ├── Switch on Current item (capability) → set the matching boolean
    └── Switch on Region:
          ├── Case UK → nested Switch on capability → set the matching UK RM variable
          ├── Case US → nested Switch on capability → set the matching US RM variable
          └── Case Global → set rmAnnaShadbolt + rmNatashaThomas directly
    ↓
Build Assignee Array (Compose)
    ↓
Format Capability String (Compose)
    ↓
Create a task (Planner) — landing in Stage 1
    ├── Title with [Region] prefix
    ├── Priority bound to priorityInt
    ├── Start Date Time bound to Required Start Date
    ├── Due Date Time bound to Required By Date
    └── 6 colour labels bound to boolean variables
    ↓
Update task details
    ├── Rich description (see 4.5)
    └── Standard checklist (3 items, all unchecked)
    ↓
Send Notification (Apply to each over assignees)
    └── Post message in chat (Teams DM)
```

### 4.3 The expressions, in detail

Each of these is the actual code pasted into the fx (Expression) tab during build. Names with underscores reflect Power Automate's internal naming convention (spaces in step names become underscores in expressions).

#### Priority Tier mapping (inside Switch on Priority Tier)

Three cases, each containing TWO Set variable actions:

**Case 1: Tier 1 – Revenue generating (Solutions/ BD / proposals)**
- Set variable: `priorityInt` = `1`
- Set variable: `priorityTierDescription` = `Tier 1 – Revenue generating (Solutions/ BD / proposals)`

**Case 2: Tier 2 – Strategic internal initiative**
- Set variable: `priorityInt` = `3`
- Set variable: `priorityTierDescription` = `Tier 2 – Strategic internal initiative`

**Case 3: Tier 3 – Non-critical / support work**
- Set variable: `priorityInt` = `5`
- Set variable: `priorityTierDescription` = `Tier 3 – Non-critical / support work`

**Default case:** empty. Initial values (priorityInt=5, priorityTierDescription="" or "Not specified") serve as the fallback.

⚠️ **The en-dash character matters.** Copy the case strings directly from the form to avoid the hyphen-vs-en-dash trap. If the Switch always falls to Default, the most likely cause is en-dash mismatch.

#### Apply to each over Capability (parses the form's stringified JSON array)

The Capability field returns text like `["Data Engineering","AI (ML & GenAI)"]` — a JSON string, not an array. We parse it:

```
json(outputs('Get_response_details')?['body/r5ec9a97f0a07442dae2aa56ea3344b3c'])
```

Inside this loop, `item()` is the current capability string. The inner Switch on Current item maps each capability to its boolean label variable. A nested Switch on Region then resolves to the correct RM variable(s).

#### Build Assignee Array (Compose, post-loop)

Deduplicates and combines all populated RM variables plus the requester into a single array. The `if(empty(...), json('[]'), createArray(...))` pattern handles unpopulated RM variables gracefully (empty strings → empty arrays, which `union()` ignores).

Multi-line readable form:

```
union(
  if(empty(variables('rmEmilyCoulson')), json('[]'), createArray(variables('rmEmilyCoulson'))),
  if(empty(variables('rmSelinYolladi')), json('[]'), createArray(variables('rmSelinYolladi'))),
  if(empty(variables('rmConorMclachlan')), json('[]'), createArray(variables('rmConorMclachlan'))),
  if(empty(variables('rmSelvaRoss')), json('[]'), createArray(variables('rmSelvaRoss'))),
  if(empty(variables('rmChloeMiles')), json('[]'), createArray(variables('rmChloeMiles'))),
  if(empty(variables('rmJessicaCubbison')), json('[]'), createArray(variables('rmJessicaCubbison'))),
  if(empty(variables('rmSianikaMalcolm')), json('[]'), createArray(variables('rmSianikaMalcolm'))),
  if(empty(variables('rmAnnaShadbolt')), json('[]'), createArray(variables('rmAnnaShadbolt'))),
  if(empty(variables('rmNatashaThomas')), json('[]'), createArray(variables('rmNatashaThomas'))),
  createArray(triggerOutputs()?['body/responder'])
)
```

Power Automate doesn't accept multi-line expressions in the fx tab. Paste as a single line:

```
union(if(empty(variables('rmEmilyCoulson')),json('[]'),createArray(variables('rmEmilyCoulson'))),if(empty(variables('rmSelinYolladi')),json('[]'),createArray(variables('rmSelinYolladi'))),if(empty(variables('rmConorMclachlan')),json('[]'),createArray(variables('rmConorMclachlan'))),if(empty(variables('rmSelvaRoss')),json('[]'),createArray(variables('rmSelvaRoss'))),if(empty(variables('rmChloeMiles')),json('[]'),createArray(variables('rmChloeMiles'))),if(empty(variables('rmJessicaCubbison')),json('[]'),createArray(variables('rmJessicaCubbison'))),if(empty(variables('rmSianikaMalcolm')),json('[]'),createArray(variables('rmSianikaMalcolm'))),if(empty(variables('rmAnnaShadbolt')),json('[]'),createArray(variables('rmAnnaShadbolt'))),if(empty(variables('rmNatashaThomas')),json('[]'),createArray(variables('rmNatashaThomas'))),createArray(triggerOutputs()?['body/responder']))
```

**Historical note:** an earlier version used `createArray()` (no arguments) instead of `json('[]')` for the empty case. This failed with "The function 'createArray' expects a comma separated list of parameters. The function was invoked with no parameters." `json('[]')` is the correct way to produce an empty array literal in Power Automate expressions.

**Output shape:** array of UPN strings, e.g. `["conormclachlan@kubrickgroup.com","bohanma@kubrickgroup.com"]`.

#### Format Capability String (Compose)

Strips JSON brackets and quotes from the raw capability field so it can be displayed cleanly in the Teams DM:

```
replace(replace(replace(outputs('Get_response_details')?['body/r5ec9a97f0a07442dae2aa56ea3344b3c'], '[', ''), ']', ''), '"', '')
```

Input: `["Data Engineering","AI (ML & GenAI)"]`
Output: `Data Engineering,AI (ML & GenAI)`

Used for human-readable display in DMs. Not used for any logic — pure presentation.

#### Submission time timezone conversion (added June 3, 2026)

Microsoft Forms returns `submitDate` in UTC. The team is Eastern Time. Display conversion:

```
convertTimeZone(outputs('Get_response_details')?['body/submitDate'], 'UTC', 'Eastern Standard Time', 'M/d/yyyy h:mm tt')
```

⚠️ **Windows timezone names only.** `'Eastern Standard Time'` is the correct Power Automate identifier and auto-handles daylight saving (EST/EDT). Do NOT use IANA names like `America/New_York` — they won't work.

Used wherever `submitDate` is displayed (task description and Teams DM body).

### 4.4 Create a task (Planner action)

| Field | Value |
| --- | --- |
| Group Id | Innovation Team (`c77d9084-e455-4a49-8b7b-292cc400edef`) |
| Plan Id | Project TicketFlow Test Board A (`DnDvOQKb_EiZg-wUJVnzjpYAHgxP`) |
| Bucket Id | Stage 1 - New Request (`zBpt4ehFxkyYHGe0S55hfZYAA1CL`) |
| Title | `[Region] - Subject - Requester Name` (composed from dynamic content) |
| Priority | `variables('priorityInt')` |
| Start Date Time | Required Start Date (dynamic content from Get response details) |
| Due Date Time | Required By Date (dynamic content from Get response details) |
| Assigned User Ids | `Build_Assignee_Array` Outputs (semicolon-separated when bound) |
| Applied Categories | 6 boolean variables bound to the 6 colour fields per the colour→capability mapping |

### 4.5 Update task details — the rich description template

The description is set in **Update task details** (not Create a task — that step doesn't accept descriptions). The June 3, 2026 production-format template:

```
Project Sponsor: [Requester Name]
Submitted: [convertTimeZone expression on submitDate]
Priority Tier: [priorityTierDescription]

     ─── Request Details ───
     Ticket Title: [Ticket Title]
     Request Type: [Request Type]
     Region: [Region]
     Capability: [Capability]

     ─── Timeframe Details ───
     Required Start Date: [Required Start Date]
     Required By Date: [Required By Date]
     Start Date Flexibility: [Start Date Flexibility]
     Estimated Duration: [Estimated Duration]

     ─── Request Objectives ───
     Request Purpose: [Request Purpose]
     Expected Deliverables: [Expected Deliverables]

     ─── Do not edit below this line ───
ALERT_UPNS: @{join(outputs('Build_Assignee_Array'), ';')}
```

**Example rendered output from a real submission:**

```
Project Sponsor: Bohan Ma
Submitted: 6/3/2026 5:52:08 AM
Priority Tier: Tier 2 – Strategic internal initiative

     ─── Request Details ───
     Ticket Title: TicketFlow Demo Task
     Request Type: Solutions
     Region: Global
     Capability: ["AI (ML & GenAI)"]

     ─── Timeframe Details ───
     Required Start Date: 2026-06-17
     Required By Date: 2026-07-05
     Start Date Flexibility: 1-2 weeks
     Estimated Duration: < 1 week

     ─── Request Objectives ───
     Request Purpose: To test the board functionality and debug Project TicketFlow for initial production operation
     Expected Deliverables: Demo build

     ─── Do not edit below this line ───
ALERT_UPNS:annashadbolt@kubrickgroup.com;natashathomas@kubrickgroup.com;bohanma@kubrickgroup.com
```

The bottom line is critical for the SLA flow. The `join()` expression converts the array of UPNs into a semicolon-separated string.

⚠️ **Cosmetic note:** the `Capability:` line currently renders the raw JSON array (`["AI (ML & GenAI)"]`). If you want a cleaner display in the description, swap the dynamic chip for the `Format Capability String` Compose output (same expression used in the DM template).

### 4.6 Update task details — standard checklist

Added per Natasha's demo request. Every new task receives a standard 3-step checklist:

| Order | Title | IsChecked |
| --- | --- | --- |
| 1 | A. Resources assigned | false |
| 2 | B. Requester notified of resources | false |
| 3 | C. Assigned resources notified | false |

Configured in the **Checklist** parameter of Update task details. Each item is a separate entry in the structured array input. Items appear in Planner in the order added.

⚠️ **Connector quirks:**
- The Title prefix (A./B./C.) is part of the text, not an auto-numbering feature. Planner does NOT auto-number checklist items.
- `IsChecked` accepts boolean `false` (no quotes) in newer connector versions. If you get a type error on save, try the string variant `"false"`.
- Planner supports up to 20 checklist items per task. Three is well within limits.
- Changing the checklist after a task is created requires re-running Update task details, NOT a separate update step.

### 4.7 Send Notification loop — Teams DMs

After task creation, an Apply to each iterates over `Build_Assignee_Array` and DMs each recipient.

**Source for Apply to each:** `outputs('Build_Assignee_Array')`

**Inside the loop: Post message in chat** (Microsoft Teams)

| Field | Value |
| --- | --- |
| Post as | Flow bot |
| Post in | Chat with Flow bot |
| Recipient | Current item (the loop's current UPN) |
| Message | HTML template (below) — must be entered via the `</>` code-view toggle in the rich-text editor |

#### The Phase 5 HTML message template (current)

```html
<h2>🎫 New Ticket Assigned</h2>
<p><strong>@{outputs('Create_a_task')?['body/title']}</strong></p>
<p><strong>Capability:</strong> @{outputs('Format_Capability_String')}<br>
<strong>Requester:</strong> @{outputs('Get_response_details')?['body/responder']}<br>
<strong>Submitted:</strong> @{convertTimeZone(outputs('Get_response_details')?['body/submitDate'], 'UTC', 'Eastern Standard Time', 'M/d/yyyy h:mm tt')}<br>
<strong>Priority:</strong> @{variables('priorityTierDescription')}</p>
<p><a href="https://tasks.office.com/kubrickgroup.com/Home/Task/@{outputs('Create_a_task')?['body/id']}">Open in Planner →</a></p>
```

**Critical gotcha:** if the HTML renders as raw text in Teams (showing literal `<h2>` tags), the message field is in plain-text mode. Toggle the `</>` icon in the toolbar to switch to code/HTML mode.

### 4.8 Known issues / accepted technical debt

- **Self-DM duplicate-member error:** if the requester is also the flow owner (e.g., during testing as Bohan), Teams returns "duplicate chat members." Doesn't break the flow — other iterations still complete — just appears as a red error in run history. In production with non-testing submitters this disappears.
- **Bucket name typo:** Stage 2 is named "Awaiting Contract" instead of "Awaiting Contact." Functionally harmless since filtering is by ID, but worth fixing for consistency.
- **Capability rendered as JSON in description:** the description shows `Capability: ["AI (ML & GenAI)"]` with brackets and quotes. Cosmetic — RMs can read it, but could be swapped to use `Format_Capability_String` for cleaner display.
- **Estimated Duration vs Required By Date overlap:** since Required By Date was added, Estimated Duration is partially redundant. Awaiting Natasha's decision on whether to delete, disclaimer, or reframe as "Effort Estimate (if known)."
- **Timezone display is hardcoded to Eastern Time:** UK requesters will see EST-converted timestamps, which may be confusing for them. Acceptable for v1 since most consumers are US-based.

---

## 5. Flow 2: TicketFlow — Stage 2 SLA Alerts

**Naming note:** the flow is named "TicketFlow — Stage 2 SLA Alerts" for historical reasons but currently implements **Stage 1** alerting only. Rename to "TicketFlow — Stage 1 SLA Alerts" or "TicketFlow — SLA Alerts" when scope is finalized.

### 5.1 Trigger

**Recurrence** — currently every 1 hour (was set to 30 minutes during testing; should be confirmed at 1 hour for production to avoid rate limiting).

### 5.2 Top-level architecture

```
Recurrence (hourly)
    ↓
List tasks (Planner) — returns all tasks across all buckets in the plan
    ↓
Filter Stage 1 Tasks (Filter array) — narrows to bucketId = Stage 1
    ↓
Apply to each (Check Each Stage 1 Task)
    ├── Get task details — fetches description containing ALERT_UPNS metadata
    ├── Extract Alert UPNs (Compose) — defensive parser; returns UPN array or []
    ├── Task Age Hours (Compose) — computes hours since createdDateTime
    └── Condition: Task Age Hours > 24
          ├── True → Apply to each (Alert Each Assignee over the UPN array)
          │            └── Post message in chat (SLA alert DM)
          └── False → no action
```

### 5.3 The expressions, in detail

#### Filter Stage 1 Tasks (Filter array)

| Field | Value |
| --- | --- |
| From | `body('List_tasks')?['value']` or via dynamic content: List tasks → value |
| Left condition value (fx) | `item()?['bucketId']` |
| Operator | is equal to |
| Right value | `zBpt4ehFxkyYHGe0S55hfZYAA1CL` (Stage 1 bucket ID — hardcoded plain text) |

#### Get task details

| Field | Value |
| --- | --- |
| Task Id (fx) | `items('Check_Each_Stage_1_Task')?['id']` |

#### Extract Alert UPNs (Compose) — the defensive parser

Handles tasks pre-dating the TicketFlow metadata change (no ALERT_UPNS line) gracefully by returning an empty array.

```
if(contains(coalesce(outputs('Get_task_details')?['body/description'], ''), 'ALERT_UPNS:'), split(trim(split(outputs('Get_task_details')?['body/description'], 'ALERT_UPNS:')[1]), ';'), createArray())
```

Breakdown:

- `coalesce(..., '')` — if description is null, treat as empty string
- `contains(..., 'ALERT_UPNS:')` — check if the marker exists
- If yes: `split(trim(split(description, 'ALERT_UPNS:')[1]), ';')`
  - `split(description, 'ALERT_UPNS:')` returns `[before_marker, after_marker]`
  - `[1]` grabs the "after" portion
  - `trim(...)` strips whitespace and newlines
  - `split(..., ';')` breaks the UPN list on semicolons
- If no: `createArray()` — empty array, downstream loop iterates zero times

**Output shape:** array of UPN strings, e.g. `["annashadbolt@kubrickgroup.com","natashathomas@kubrickgroup.com","bohanma@kubrickgroup.com"]`.

**Earlier non-defensive version** (which crashed on tasks without the marker — kept here as a warning):

```
split(trim(split(outputs('Get_task_details')?['body/description'], 'ALERT_UPNS:')[1]), ';')
```

The failure mode was `"array index '1' is outside bounds (0, 0) of array"` when `ALERT_UPNS:` wasn't found.

#### Task Age Hours (Compose)

Computes how many hours have elapsed since the task was created. Uses `createdDateTime` from the List tasks/Filter array output (the Planner connector doesn't expose `lastModifiedDateTime`).

```
div(sub(ticks(utcNow()), ticks(items('Check_Each_Stage_1_Task')?['createdDateTime'])), 36000000000)
```

Breakdown:

- `utcNow()` → current UTC time as ISO string
- `ticks(timestamp)` → converts a timestamp to "ticks" (100-nanosecond intervals since year 1) — a large integer
- `sub(A, B)` → elapsed ticks
- `36000000000` → number of 100-nanosecond ticks in one hour
- `div(elapsed, 36000000000)` → elapsed hours as a decimal number

Output example: `27.3` for a task created 27.3 hours ago.

**Design accuracy note:** `createdDateTime` is used as a proxy for "time in Stage 1." This is accurate because:
- TicketFlow creates tasks directly in Stage 1 with assignees attached, in one atomic operation
- A ticket leaving Stage 1 (moving to Stage 2+) is no longer matched by the Filter Stage 1 Tasks step, so we never read its age in a later stage
- Per Natasha's confirmation, tickets very rarely bounce backward into Stage 1 after leaving it

The known edge case: if a ticket sat in Stage 1 for 26 hours, was moved to Stage 2, then bounced back to Stage 1, its `createdDateTime` would still show original creation — so it would alert immediately on re-entry. Treated as acceptable behaviour for v1 (the ticket has indeed been waiting too long).

#### Condition

| Field | Value |
| --- | --- |
| Left value (dynamic content) | Task Age Hours → Outputs (or fx: `outputs('Task_Age_Hours')`) |
| Operator | is greater than |
| Right value | `24` (plain integer, no quotes) |

**Note:** the new Power Automate designer auto-adds an empty second condition row joined by `Or`. This is cosmetic — runtime ignores empty rows.

#### Alert Each Assignee (inner Apply to each)

Inside the True branch:

| Field | Value |
| --- | --- |
| Select an output (fx) | `outputs('Extract_Alert_UPNs')` |

Inside this inner loop: a single **Post message in chat** action.

#### Post message in chat (SLA alert)

| Field | Value |
| --- | --- |
| Post as | Flow bot |
| Post in | Chat with Flow bot |
| Recipient (fx) | `items('Alert_Each_Assignee')` — note this is the current item directly (a UPN string), NOT `items(...)?['userId']` |
| Message | HTML template (below) |

#### Stage 1 SLA alert HTML

```html
<h2>⚠️ Ticket Untriaged — Action Needed</h2>
<p>This ticket has been in <strong>Stage 1 (New Request)</strong> for over 24 hours without being moved forward.</p>
<p><strong>@{items('Check_Each_Stage_1_Task')?['title']}</strong></p>
<p>Please review and move it to Stage 2 (Awaiting Contact) — or to the appropriate next stage.</p>
<p><a href="https://tasks.office.com/kubrickgroup.com/Home/Task/@{items('Check_Each_Stage_1_Task')?['id']}">Open in Planner →</a></p>
```

Same `</>` code-view toggle required as in TicketFlow's Phase 5 message.

### 5.4 Concurrency settings (added to mitigate rate limiting)

Both Apply to each loops should have Concurrency Control turned ON with Degree of Parallelism = 1 (serial processing). Set via: click the loop → 3-dot menu → Settings → Concurrency Control.

This serializes iterations and reduces burst load on Microsoft's Teams and Planner APIs. Necessary because the unconstrained default (~20 parallel) was triggering silent stalls.

---

## 6. Architectural decisions log

Record of decisions made and their rationale.

| Decision | Rationale | Status |
| --- | --- | --- |
| Tickets land in Stage 1, not Stage 2 directly | Natasha required preserving the 5-stage flow | Implemented |
| Use `createdDateTime` instead of `lastModifiedDateTime` for SLA clock | Planner connector doesn't expose `lastModifiedDateTime`; `createdDateTime` is accurate proxy since tasks are created-and-assigned atomically | Implemented |
| Embed `ALERT_UPNS:` in task description rather than re-derive RM routing in SLA flow | Cleanest of the available options; avoids GUID→UPN lookups, premium HTTP connectors, and duplicated routing matrices | Implemented |
| Build two separate flows (event-driven TicketFlow + scheduled SLA Alerts) | Different triggers, different concerns; mixing them is fragile | Implemented |
| Use the UPN-based "Post message in chat" Teams action | GUIDs are rejected at validation; the connector requires UPN/email-format recipients | Implemented |
| Stage 1 SLA only for v1 (not Stage 2 simultaneously) | Narrow scope to ship; Natasha confirmed tickets rarely bounce backward, so Stage 2 SLA is lower priority | Implemented |
| Use `union()` to dedupe Build Assignee Array | Handles the case where requester is also an assigned RM | Implemented |
| Use `json('[]')` (not `createArray()` with no args) for empty arrays in expressions | `createArray()` requires at least one argument; throws runtime error | Implemented |
| Format Capability via `replace(replace(replace(...)))` instead of more elegant JSON parsing | Adaptive cards rejected stringified JSON arrays as invalid; the triple-replace strips brackets and quotes for clean display | Implemented |
| Map Priority Tier 3 → Medium (5), not Low (9) | Keeps non-critical work visible by default rather than visually de-emphasized; system-assigned default ≠ "deprioritize" | Implemented |
| Preserve full Priority Tier label in description and DM via `priorityTierDescription` variable | RMs see the meaningful tier name ("Tier 1 – Revenue generating") rather than just the priority flag | Implemented |
| Bind Required Start Date → Planner Start Date Time, Required By Date → Planner Due Date Time | Native Planner fields are more discoverable than description-only text | Implemented |
| Add 3-item standard checklist to every task (Resources assigned / Requester notified / Assigned resources notified) | Natasha's demo ask; standardizes workflow tracking | Implemented |
| Convert UTC `submitDate` to Eastern Time for display | Forms returns UTC, team is US-based; 4-hour timestamp discrepancy was confusing | Implemented |
| Use Windows timezone name `'Eastern Standard Time'` (not IANA) | Power Automate's `convertTimeZone()` function requires Windows timezone identifiers; auto-handles EST/EDT | Implemented |
| Add new `Required By Date` form field rather than computing Due Date from Estimated Duration | Avoids manufacturing a synthetic deadline; captures customer intent directly | Implemented (Natasha approved) |

### Decisions explicitly tabled (not built but on the roadmap)

| Tabled item | Reason |
| --- | --- |
| Stage 2 SLA alert | Out of scope for v1 per Natasha's confirmation |
| Stamper architecture for accurate per-stage timing | Workaround 3 from our analysis; createdDateTime is good enough for Stage 1 |
| Send-once dedup (prevent hourly re-alerts on the same task) | Real production concern; will need to append `[SLA_ALERTED]` marker to description after alerting, then skip on subsequent runs |
| Switch to Direct HTTP to Microsoft Graph for GUID→UPN | Premium connector risk; superseded by embedding UPNs in description |
| SharePoint list as routing source | "Cleanest" architecture but significant additional infrastructure |
| Estimated Duration field cleanup | Awaiting Natasha's decision on delete / disclaimer / rename as "Effort Estimate" |
| Multi-timezone display in description | UK consumers will see EST timestamps; acceptable for v1, revisit if complaints surface |

---

## 7. Known issues and current debugging

### Issue A: SLA flow intermittent stalling

**Symptom:** SLA flow runs that took 10-15s earlier in a session suddenly hang for 10-24+ minutes on the Apply to each (Check Each Stage 1 Task) loop, then either complete eventually or need manual cancellation. Microsoft Teams API has been observed returning `InternalServerError` (HTTP 500), which triggers Power Automate's default 4-retry exponential backoff (~36 min wall time per retry chain).

**Hypothesized causes (in priority order):**

1. **Microsoft Teams / Planner API rate limiting** — most likely. Frequent recurrence combined with multiple Stage 1 tickets × multiple assignees hits per-connection limits.
2. **Power Automate platform-level queue backup** — degraded state after many cancelled/failed runs.
3. **Action retry exhaustion** — failed actions retrying 4× with exponential backoff before surfacing.

**Steps taken:**
- Changed Recurrence from 30 min to 1 hour
- Added Concurrency Control = 1 to both loops
- Cleaned up duplicate Stage 1 test tickets
- Restricted ALERT_UPNS to just Bohan's UPN for diagnostic testing

**Resolution status:** unresolved. Flow correctness verified earlier; issue is platform/environmental, not logic.

### Issue B: Forms license expiry

**Symptom:** every ~2 weeks, the Microsoft Forms connector returns `751: "Did not find Forms licenses for current user."` Flow becomes unable to run.

**Cause:** Microsoft Forms isn't enabled tenant-wide at Kubrick. Bohan's access is granted in 2-week trial increments via IT (John).

**Mitigation:** request permanent Forms grant from Natasha and forward to IT; do not go live on a trial license that will break the flow every 2 weeks.

### Issue C: Self-DM in test environments

**Symptom:** when the flow's owner (Bohan) is also an assignee on a test ticket, that iteration's DM action errors with "duplicate chat members."

**Cause:** Microsoft Teams doesn't allow creating a 1:1 chat between a user and themselves via this action.

**Mitigation:** in production, the flow owner won't typically also be the requester or RM. For test cleanliness, can add a Condition wrapper inside Send Notification to skip iterations where Current item equals the flow owner's UPN. Not implemented in v1.

---

## 8. Operational runbook

### How to deploy to production (Phase 7)

Pending Natasha's scheduling of:
1. Real AIBP intake form availability (replaces the placeholder form used during development)
2. Real Planner board availability (replaces "Project TicketFlow Test Board A")

Once available:

1. **In TicketFlow:**
   - Change the form reference in the "When a new response is submitted" trigger
   - Change the Group Id and Plan Id in Create a task and Update task details
   - Re-verify all dynamic content bindings — when the form changes, field IDs change, and dynamic content chips need to be re-bound from the new Get response details schema
   - Specifically re-verify the Capability field ID (`r5ec9a97f0a07442dae2aa56ea3344b3c` is for the current placeholder form and will not match the real form)

2. **In the SLA flow:**
   - Update Group Id and Plan Id in List tasks
   - Update the Stage 1 bucket ID in Filter Stage 1 Tasks
   - Update the `https://tasks.office.com/...` link in the HTML template if the tenant URL differs

3. **In Planner (production board):**
   - Pre-create the 5 stage buckets with consistent names
   - Pre-create the 6 capability labels with consistent colour assignments (Cranberry / Orange / Peach / Light green / Teal / Lavender)

4. **License check before go-live:**
   - Verify Bohan's (or whoever owns the flow) Microsoft Forms license is permanently granted, not on a 2-week trial
   - Without this, the production flow will break every 2 weeks

5. **Smoke test** with one real submission end-to-end before announcing to the team.

### How to add or change a Resource Manager

Two flows affected:

1. **TicketFlow:**
   - Add the new RM as a string variable in the Initialize variables section
   - Update the relevant Switch case (Region × Capability) to set the new variable
   - Add the new variable to the union() in Build Assignee Array using the same `if(empty(...), json('[]'), createArray(...))` pattern

2. **SLA flow:**
   - No changes needed. The SLA flow reads UPNs from the task description, so it picks up the new RM automatically.

### How to add a new capability

1. Add the capability text exactly as it will appear in the form
2. In TicketFlow, add a new boolean variable + new case in the capability Switch
3. Pick a new Planner label colour and add a new case for it
4. Update the form's Capability question to include the new option
5. Update the Planner board's labels to include the new label colour

### How to add a new region

More involved:

1. In TicketFlow, add a new case in the Switch on Region
2. Inside that case, add a nested Switch on Capability for the new region's routing
3. Add any new RM variables needed
4. Update Build Assignee Array's union() to include the new variables
5. Update the form's Region question

### How to add a new Priority Tier

1. Add the tier text exactly as it will appear in the form (mind the en-dash convention)
2. In TicketFlow, add a new case to the Switch on Priority Tier
3. Inside the case, add TWO Set variable actions — one for `priorityInt`, one for `priorityTierDescription`
4. Pick the appropriate Planner priority integer (1/3/5/9) for the new tier
5. Update the form's Priority Tier question to include the new option

### How to modify the standard checklist

The 3-item checklist (Resources assigned / Requester notified / Assigned resources notified) is defined inside Update task details' Checklist parameter.

1. Edit TicketFlow → Update task details
2. Find the Checklist parameter
3. Add/remove/edit items as needed (max 20 per task)
4. Save

Existing tasks will NOT retroactively get the new checklist — only newly-created tasks will reflect the change.

### How to add a new form field to the description

1. Add the field to the Microsoft Form
2. In TicketFlow, click Get response details → re-pick the same form ID to refresh the schema (or delete and re-add the action if needed)
3. In Update task details, click the Description field
4. Position cursor at the desired location
5. Open dynamic content → pick the new field from Get response details

### How to change the displayed timezone

If the team shifts primary location (e.g., to UK):

1. Find every instance of `'Eastern Standard Time'` in the flow
2. Replace with the appropriate Windows timezone name:
   - UK: `'GMT Standard Time'`
   - Central US: `'Central Standard Time'`
   - Pacific US: `'Pacific Standard Time'`

Windows timezone names auto-handle daylight saving. Do not use IANA names.

---

## 9. Glossary

| Term | Meaning |
| --- | --- |
| AIBP | AI Business Partner — Bohan's role and the team this serves |
| RM | Resource Manager — the role responsible for assigning consultants to tickets |
| UPN | User Principal Name — Microsoft's term for an email-format user identifier (e.g., `bohanma@kubrickgroup.com`) |
| SLA | Service Level Agreement — the 24-hour commitment that tickets won't sit untriaged |
| Stage 1 / Stage 2 / etc. | The five lifecycle buckets a ticket moves through in Planner |
| Flow bot | The system identity that posts Teams DMs from Power Automate |
| Premium connector | A Power Automate connector that requires a licensed plan beyond standard Office 365 (e.g., direct HTTP to Graph) |
| GUID | Globally Unique Identifier — Azure AD's internal user ID format (e.g., `23ac0948-7b26-40a2-bb05-bff119b3165b`) |
| Priority Tier | Form-level concept (Tier 1/2/3) mapping to Planner's native Priority field |
| en-dash | The `–` character (U+2013) used in Priority Tier labels; distinct from the hyphen-minus `-` |
| Windows timezone name | Microsoft's timezone identifier format (e.g., `Eastern Standard Time`) used by Power Automate; auto-handles daylight saving |

---

## 10. Quick reference — copy-paste expressions cheat sheet

### Parse capability JSON
```
json(outputs('Get_response_details')?['body/r5ec9a97f0a07442dae2aa56ea3344b3c'])
```

### Format capability for display
```
replace(replace(replace(outputs('Get_response_details')?['body/r5ec9a97f0a07442dae2aa56ea3344b3c'], '[', ''), ']', ''), '"', '')
```

### Convert submitDate from UTC to Eastern Time
```
convertTimeZone(outputs('Get_response_details')?['body/submitDate'], 'UTC', 'Eastern Standard Time', 'M/d/yyyy h:mm tt')
```

### Build assignee array (single line)
```
union(if(empty(variables('rmEmilyCoulson')),json('[]'),createArray(variables('rmEmilyCoulson'))),if(empty(variables('rmSelinYolladi')),json('[]'),createArray(variables('rmSelinYolladi'))),if(empty(variables('rmConorMclachlan')),json('[]'),createArray(variables('rmConorMclachlan'))),if(empty(variables('rmSelvaRoss')),json('[]'),createArray(variables('rmSelvaRoss'))),if(empty(variables('rmChloeMiles')),json('[]'),createArray(variables('rmChloeMiles'))),if(empty(variables('rmJessicaCubbison')),json('[]'),createArray(variables('rmJessicaCubbison'))),if(empty(variables('rmSianikaMalcolm')),json('[]'),createArray(variables('rmSianikaMalcolm'))),if(empty(variables('rmAnnaShadbolt')),json('[]'),createArray(variables('rmAnnaShadbolt'))),if(empty(variables('rmNatashaThomas')),json('[]'),createArray(variables('rmNatashaThomas'))),createArray(triggerOutputs()?['body/responder']))
```

### Embed UPNs in description
```
ALERT_UPNS: @{join(outputs('Build_Assignee_Array'), ';')}
```

### Priority Tier Switch case strings (en-dash, copy exactly)
```
Tier 1 – Revenue generating (Solutions/ BD / proposals)
Tier 2 – Strategic internal initiative
Tier 3 – Non-critical / support work
```

### Priority Tier → priorityInt values
- Tier 1 → `1` (Urgent)
- Tier 2 → `3` (Important)
- Tier 3 → `5` (Medium)
- Default → `5` (Medium)

### Standard checklist items
```
A. Resources assigned         (IsChecked: false)
B. Requester notified of resources    (IsChecked: false)
C. Assigned resources notified         (IsChecked: false)
```

### Filter array condition (Stage 1)
- Left (fx): `item()?['bucketId']`
- Op: is equal to
- Right (text): `zBpt4ehFxkyYHGe0S55hfZYAA1CL`

### Get task details Task Id
```
items('Check_Each_Stage_1_Task')?['id']
```

### Extract Alert UPNs (defensive)
```
if(contains(coalesce(outputs('Get_task_details')?['body/description'], ''), 'ALERT_UPNS:'), split(trim(split(outputs('Get_task_details')?['body/description'], 'ALERT_UPNS:')[1]), ';'), createArray())
```

### Task Age Hours
```
div(sub(ticks(utcNow()), ticks(items('Check_Each_Stage_1_Task')?['createdDateTime'])), 36000000000)
```

### Alert Each Assignee loop source
```
outputs('Extract_Alert_UPNs')
```

### Post message in chat Recipient (SLA flow)
```
items('Alert_Each_Assignee')
```

### Planner task URL template
```
https://tasks.office.com/kubrickgroup.com/Home/Task/@{outputs('Create_a_task')?['body/id']}
```

For tasks referenced inside the SLA flow's loop, replace `outputs('Create_a_task')?['body/id']` with `items('Check_Each_Stage_1_Task')?['id']`.

### Windows timezone names reference
- `Eastern Standard Time` — US East (handles EST/EDT)
- `Central Standard Time` — US Central (handles CST/CDT)
- `Pacific Standard Time` — US West (handles PST/PDT)
- `GMT Standard Time` — UK (handles GMT/BST)
- `W. Europe Standard Time` — Western Europe (handles CET/CEST)

---

## 11. Change log

| Date | Change | Section affected |
| --- | --- | --- |
| May 27, 2026 | Initial README v1 | All |
| June 3, 2026 (v2) | Added Priority Tier field with dual-variable Switch (priorityInt + priorityTierDescription) | 2, 4.3, 6, 10 |
| June 3, 2026 (v2) | Added Start Date / Due Date binding to Planner via Required Start Date / Required By Date form fields | 2, 4.4, 6, 10 |
| June 3, 2026 (v2) | Added standard 3-item checklist to every task | 4.6, 6, 10 |
| June 3, 2026 (v2) | Restructured description with sections: Request Details / Timeframe Details / Request Objectives | 4.5 |
| June 3, 2026 (v2) | Added new "Required By Date" field to the Microsoft Form (Natasha-approved scope change) | 2, 8 |
| June 3, 2026 (v2) | Fixed timezone bug: `submitDate` now converts from UTC to Eastern Time via `convertTimeZone()` | 2, 4.3, 4.7, 6, 8, 10 |
| June 3, 2026 (v2) | Added Priority line to Phase 5 Teams DM template | 4.7, 10 |
