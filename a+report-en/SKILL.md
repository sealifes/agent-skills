---
name: a+report
description: Search GitHub commits within a date range and generate A+ project work reports (.docx format)
---

# A+ Project Work Report Generator

## Prerequisites

- GitHub MCP tool access required
- document-processing-docx skill or equivalent Word output capability required

---

## Execution Pipeline

```
[Date Range Input] → [GitHub Search] → [Classification] → [Report Generation] → [.docx Output]
```

---

## STEP 1: GitHub Search Parameters

```yaml
# Required parameters
organization: atayalan
target_authors:
  - sealifes
  - hangli

# Search scope: enumerated repositories
repositories:
  - fgc
  - chorus-agent
  - chorus-edge-k3s-installer
  - chorusLiteInstaller
  - serviceagent
  - mgmt-plane
  - edge-agent
  - nf-wrapper
  - vpp-controller
  - codes
  - vpp-arm-buildenv

# Branch strategy
branch_selection: Select 4 most recently updated branches per repo
deduplication: Same commit SHA across multiple branches counts as one
```

---

## STEP 2: Classification Rules

```
Classification Decision Tree:

1. Check repository name
   ├─ Contains "edge" → classify as edge/
   ├─ Is vpp-controller → classify as edge/
   ├─ Is vpp-arm-buildenv → classify as edge/
   └─ Otherwise → classify as chorus/

2. If PR/commit spans both categories
   └─ Classify as edge/
```

Output directory structure:
```
output/
├── chorus/
│   └── chorus-YYYY-M-D-weekday.docx
└── edge/
    └── edge-YYYY-M-D-weekday.docx
```

---

## STEP 3: Report Generation Rules

### 3.1 Quantity and Content

| Item | Rule |
|------|------|
| Reports per PR | 6 |
| Words per report | ~300 words |
| Content strategy | **Expand**: Include technical background, terminology explanations |
| Prohibited | Large blocks of repeated/duplicated text |

### 3.2 Date Scheduling Algorithm

```python
def schedule_report_dates(pr_date: date, count: int = 6) -> list[date]:
    """
    Start from the first Wednesday after PR date, alternate Wed/Fri
    """
    # Find first Wednesday after PR date
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
Priority rules:
1. Only one report per day (across both chorus/edge categories)
2. On conflict, defer the later PR's report dates to next available Wed/Fri
3. Maintain a global occupied_dates set to prevent collisions
```

---

## STEP 4: Word Document Format Specification

```yaml
format: .docx
filename_pattern: "{category}-{YYYY}-{M}-{D}-{weekday}.docx"
# Examples: chorus-2025-11-5-wed.docx, edge-2025-11-7-fri.docx

typography:
  font_size: 11pt
  line_spacing: 1

constraints:
  max_pages: 1  # Trim content if exceeded
```

---

## STEP 5: Report Content Template

```markdown
# {Category} Work Report - {YYYY}-{MM}-{DD}

## {Issue/PR Number}: {Title}

### Project Information
- **Repository**: {repo_name}
- **Date**: {YYYY}-{M}-{D}-{Weekday}

### Change Summary
{2-3 sentences describing the purpose and background of this change}
{May include extended technical concepts such as protocols, architectural design rationale}

### Technical Details
Files modified:
1. `{file_path_1}` - {Brief description of file's role}
2. `{file_path_2}` - {Brief description of file's role}

Total: {N} lines changed ({added} additions, {deleted} deletions).
{Describe the code-level change logic}

### Impact Scope
{Explain the impact of this change on system behavior}
{If applicable, describe relationships with other modules}
```

---

## Edge Case Handling

| Scenario | Handling |
|----------|----------|
| Fewer than 6 commits in date range | Generate only as many reports as commits exist; do not fabricate |
| PR has no meaningful commit message | Infer change purpose from diff content |
| Multiple PRs on same day | Sort by commit timestamp, schedule sequentially |
| Empty search results | Report "No commits matching criteria found in specified range" |

---

## Example Output

```
# Chorus Work Report - 2025-11-05

## G1-3261: Remove ranNode Backup Skip Mechanism in Chorus Mode

### Project Information
- **Repository**: fgc
- **Date**: 2025-11-5-Wed

### Change Summary
This commit introduces an important behavioral adjustment to the AMF (Access and Mobility Management Function) controller. AMF is a critical network function in the 5G core network responsible for managing UE (User Equipment) access and mobility. In the previous implementation, when the system operated in Chorus mode, ranNode (Radio Access Network node) backup operations were skipped. This change removes that skip logic, ensuring the ranNode backup mechanism in Chorus mode operates consistently with standard mode.

### Technical Details
Files modified:
1. `apps/amfctrl/amf/wps_amfRanDb.cpp` - RAN database management module, maintains base station connection state
2. `apps/amfctrl/amfToDb/src/wps_amfToDb_redis.cpp` - AMF to Redis sync module, handles state persistence

Total: 27 lines changed (10 additions, 17 deletions). By removing conditional branches, this change unifies the data backup strategy across different operational modes, reducing code complexity and improving maintainability.

### Impact Scope
This change ensures RAN node states are properly backed up to Redis in Chorus deployment environments, enhancing system fault tolerance. When AMF restarts or fails over, complete ranNode connection information can be recovered from Redis, preventing base stations from needing to re-register.
```
