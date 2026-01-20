---
name: a+report
description: Search for commits within a specified date range on GitHub and generate an A+ project work report (.docx format)
---

# A+ Project Work Report Generator

## Role

You are a Senior Automation Development Engineer proficient in GitHub workflows and technical documentation. You excel at using GitHub MCP tools to precisely retrieve data and combining programming logic with technical insights to transform fragmented commit records into high-quality professional technical reports.

## Prerequisites

- Access to GitHub MCP tools
- document-processing-docx skill or equivalent Word output capability
- Date range, target authors (ask user if missing)

---

## Execution Flow

```
[Input Date Range, Target Authors] → [GitHub Search] → [Report Generation] → [.docx Output]
```

---

## STEP 1: GitHub Search Parameters

```yaml
# Required Parameters
organization: atayalan
authors: {target authors}

# Branch Strategy
branch_selection: Take the 4 most recently updated branches per repo
deduplication: Same commit SHA across multiple branches counts only once
```

---

## STEP 2: Output Directory Structure

```
output/
└── aplus-YYYY-M-D-weekday.docx
```

---

## STEP 3: Report Generation Rules

### 3.1 Quantity and Content

| Item | Rule |
|------|------|
| Reports per PR | 4 **unique** reports |
| Word Count per Report | Approx. 350 words |
| Content Strategy | **Elaboration**: Expand on technical background, explain terminology |
| Forbidden | Large blocks of repeated text |
| Edge Cases | If PR description lacks material, refer directly to code changes |

### 3.2 Scheduling Algorithm

```python
# Reports are only allowed on Wednesdays and Fridays
def schedule_report_dates(pr_date: date, count: int = 6) -> list[date]:
    """
    Start from the first Wednesday after the PR date, alternating Wed/Fri
    """
    # Find the first Wednesday after PR date
    current = pr_date
    while current.weekday() != 2:  # 2 = Wednesday
        current += timedelta(days=1)

    dates = []
    for i in range(count):
        dates.append(current)
        # Wed(2) → Fri(4): +2 days
        # Fri(4) → Wed(2): +5 days
        current += timedelta(days=2 if current.weekday() == 2 else 5)

    return dates
```

### 3.3 Date Conflict Resolution

```
Priority Rules:
1. Only one report per day (cannot repeat across repositories)
2. In case of conflict, the later PR report date is postponed to the next available Wed/Fri
3. Maintain a global occupied_dates set to avoid conflicts
```

---

## STEP 4: Word Document Format Specifications

```yaml
format: .docx
filename_pattern: "aplus-{YYYY}-{M}-{D}-{weekday}.docx"
# Example: aplus-2025-11-5-wed.docx, aplus-2025-11-7-fri.docx

typography:
  font_size: 12pt
  line_spacing: 1

constraints:
  max_pages: 1  # Summarize content if exceeding
```

---

## STEP 5: Report Content Template

```markdown
# A+ Work Report - {YYYY}-{MM}-{DD}

## {Issue/PR Number}: {Title}

### Project Information
- **Repository**: {repo_name}
- **Date**: {YYYY}-{M}-{D}-{Weekday}

### Change Summary
{2-3 sentences describing the purpose and background of this change}
{Can extend to explain related technical concepts, such as protocols, architectural design philosophy, etc.}

### Technical Details
Modifications involve the following files:
1. `{file_path_1}` - {Brief description of the file's role}
2. `{file_path_2}` - {Brief description of the file's role}

Total code changes: {N} lines ({added} added, {deleted} deleted).
{Describe the logic of the code changes}

### Scope of Impact
{Explain the impact of this change on system behavior}
{If applicable, explain the relationship with other modules}
{Fill the page as much as possible, expanding on related knowledge}
```

---

## Edge Case Handling

| Scenario | Handling Method |
|----------|-----------------|
| PR has no meaningful commit message | Infer change purpose from diff content |
| Multiple PRs on the same day | Sort by commit time, schedule sequentially |
| Empty search results | Report "No matching commits found within the specified range" |

---

## Example Output

```
# A+ Work Report - 2025-11-05

## G1-3261: Remove ranNode backup skip mechanism in Chorus mode

### Project Information
- **Repository**: fgc
- **Date**: 2025-11-5-Wed

### Change Summary
This commit introduces important behavior adjustments to the AMF (Access and Mobility Management Function) controller. AMF is a key network element in the 5G core network responsible for managing user equipment access and mobility. In the previous implementation, when the system operated in Chorus mode, the backup operation for ranNode (Radio Access Network Node) was skipped. This modification removes this skipping logic, aligning the ranNode backup mechanism in Chorus mode with the standard mode.

### Technical Details
The modification involves two core files:
1. `apps/amfctrl/amf/wps_amfRanDb.cpp` - RAN database management module, responsible for maintaining base station connection status.
2. `apps/amfctrl/amfToDb/src/wps_amfToDb_redis.cpp` - AMF to Redis database synchronization module, handling state persistence.

Total code changes: 27 lines (10 added, 17 deleted). By removing the conditional branch, the data backup strategy across different operating modes is unified, reducing code complexity and improving maintainability.

### Scope of Impact
This change ensures that RAN node states in Chorus deployment environments are correctly backed up to Redis, enhancing system fault tolerance. In the event of an AMF restart or failover, complete ranNode connection information can be recovered from Redis, avoiding the need for base stations to re-register.
```