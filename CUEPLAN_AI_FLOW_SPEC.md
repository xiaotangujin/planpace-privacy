# CuePlan AI Flow v1

CuePlan AI Flow v1 is a public text format for AI assistants that create importable CuePlan workflows. Users can ask ChatGPT, Claude, Gemini, Kimi, or another AI to plan a trip, work block, study session, routine, event, or backup plan, then copy the generated block into CuePlan.

This specification is intentionally simpler than CuePlan's internal data model. AI assistants should generate this public format. CuePlan converts it into the app's internal workflow engine during import.

Recommended canonical URL:

```text
https://<your-domain-or-github-pages>/cueplan-ai-flow-v1
```

Temporary GitHub Pages URL pattern:

```text
https://<github-user>.github.io/<repo>/cueplan-ai-flow-v1
```

## 1. Core Principles

- Make the format easy for AI assistants to generate correctly.
- Keep all schema keys, enum values, and protocol identifiers in English.
- Generate user-facing text in the user's language.
- Do not require AI assistants to understand CuePlan's internal model.
- Do not require exact coordinates for location-based flows.
- Allow CuePlan to show a preview and ask the user to confirm uncertain fields.
- Never auto-start an imported workflow.

## 2. Language Rules

Protocol fields must always be English:

- JSON keys: `format`, `version`, `plan`, `nodes`, `end`
- Enum values: `normal`, `bPlan`, `manual`, `duration`, `location`
- Stable node IDs: `prepare`, `arrive_office`, `review`

User-facing content must follow the user's language:

- `plan.title`
- `plan.summary`
- node `title`
- node `note`
- location `placeName`
- location `addressText`

If the user writes in Chinese, generate Chinese titles and node names. If the user writes in Japanese, generate Japanese titles and node names. If the user's language is unclear, use the language of the latest user request.

Example for a Chinese user:

```json
{
  "id": "prepare_bag",
  "title": "整理书包",
  "note": "检查电脑、充电器和资料。",
  "icon": "🎒"
}
```

Example for an English user:

```json
{
  "id": "prepare_bag",
  "title": "Pack bag",
  "note": "Check laptop, charger, and documents.",
  "icon": "🎒"
}
```

## 3. Required Output Format

AI assistants should output exactly one CuePlan flow block:

````text
CUEPLAN_FLOW_V1
```json
{
  "format": "cueplan.flow",
  "version": 1,
  "plan": {
    "title": "Airport Day",
    "summary": "A step-by-step airport routine.",
    "mode": "normal",
    "icon": "✈️",
    "start": { "type": "manual" },
    "repeat": { "type": "none" },
    "nodes": [
      {
        "id": "pack",
        "type": "task",
        "title": "Pack essentials",
        "icon": "🎒",
        "end": {
          "type": "duration",
          "seconds": 900,
          "allowsOvertime": true
        }
      }
    ]
  }
}
```
````

Do not add explanations, analysis, disclaimers, or extra Markdown outside the block unless the user explicitly asks for explanation.

## 4. App Detection Rules

CuePlan can recognize an AI-generated flow when the pasted text or imported file contains at least one strong signal:

1. `CUEPLAN_FLOW_V1`
2. `"format": "cueplan.flow"`
3. `"version": 1` and a top-level `"plan"` object

When a match is found, the app may show:

```text
Detected a CuePlan workflow. Import it?
```

For localized builds, this prompt should be translated in the app UI.

## 5. Supported Modes

### 5.1 `simple`

Simple mode is for fast, friendly, linear flows. It is best for users who only need a list of actions with durations.

Use it when the user asks for a very lightweight routine and does not mention advanced controls.

Supported:

- plan title
- plan icon
- manual start
- ordered task list
- duration or manual completion per task

### 5.2 `normal`

Normal mode is the default and recommended mode for AI-generated workflows.

Supported:

- manual start
- scheduled start
- location start
- no repeat, daily, weekdays, weekly
- ordered task nodes
- app-generated start and end nodes
- duration end
- manual end
- location end
- total flow elapsed end
- clock time end
- overtime setting
- emoji icons

AI assistants should generate `normal` unless the user explicitly asks for a backup route, fallback plan, Plan B, A/B path, or route switching.

### 5.3 `bPlan`

B-plan mode is an advanced mode for workflows with A route, B route, and shared AB nodes.

Supported:

- A-only nodes
- B-only nodes
- shared AB nodes
- route switch conditions
- immediate switch and complete current node
- switch after current node ends

B-plan imports should always show an import preview before saving because route logic is more error-prone.

## 6. Top-Level Object

```json
{
  "format": "cueplan.flow",
  "version": 1,
  "plan": {}
}
```

| Field | Type | Required | Description |
|---|---|---:|---|
| `format` | string | yes | Must be `cueplan.flow` |
| `version` | number | yes | Must be `1` |
| `plan` | object | yes | Workflow content |

## 7. Plan Object

| Field | Type | Required | Description |
|---|---|---:|---|
| `title` | string | yes | Workflow title in the user's language |
| `summary` | string | no | Short description in the user's language |
| `mode` | string | no | `simple`, `normal`, or `bPlan`; default `normal` |
| `icon` | string | no | Prefer one emoji |
| `start` | object | no | Start rule; default manual |
| `repeat` | object | no | Repeat rule; default none |
| `nodes` | array | yes | Ordered nodes |

## 8. Start Rules

### Manual Start

```json
{
  "type": "manual"
}
```

### Scheduled Start

```json
{
  "type": "scheduled",
  "time": "09:00"
}
```

`time` uses 24-hour `HH:mm`.

If the user did not provide a specific date, do not invent one. Generate the time only.

### Location Start

```json
{
  "type": "location",
  "location": {
    "placeName": "Office",
    "addressText": "100 Main Street, New York",
    "radiusMeters": 150,
    "resolvePolicy": "askUser"
  },
  "arrivalPolicy": "firstArrival"
}
```

`arrivalPolicy` values:

- `firstArrival`: trigger once after the user first enters the area
- `everyArrival`: trigger whenever the user re-enters the area

## 9. Repeat Rules

```json
{
  "type": "none"
}
```

Supported values:

- `none`
- `daily`
- `weekdays`
- `weekly`

Optional fields:

| Field | Type | Description |
|---|---|---|
| `startDate` | string | `YYYY-MM-DD` |
| `endDate` | string | `YYYY-MM-DD` |

Repeat rules are most useful for scheduled starts. Manual start flows should usually use `"type": "none"` unless the user clearly requests recurring reminders.

## 10. Node Object

| Field | Type | Required | Description |
|---|---|---:|---|
| `id` | string | yes | Stable ID using letters, numbers, underscores, or hyphens |
| `type` | string | no | `task`, `start`, or `end`; default `task` |
| `title` | string | yes | Node title in the user's language |
| `note` | string | no | Node note in the user's language; may be empty |
| `icon` | string | no | Prefer one emoji |
| `routeRole` | string | no | B-plan only: `a`, `b`, or `ab` |
| `end` | object | no | End rule; default manual |
| `switchRules` | array | no | B-plan route switch rules |

For normal mode, node order is execution order. AI assistants do not need to generate dependency lists.

## 11. End Rules

### Manual Completion

```json
{
  "type": "manual"
}
```

### Fixed Duration

```json
{
  "type": "duration",
  "seconds": 1500,
  "allowsOvertime": true
}
```

`seconds` must be a positive integer.

### Location End

```json
{
  "type": "location",
  "location": {
    "placeName": "Library",
    "addressText": "New York Public Library",
    "radiusMeters": 120,
    "resolvePolicy": "askUser"
  }
}
```

### Total Flow Elapsed

```json
{
  "type": "flowElapsed",
  "seconds": 3600
}
```

### Clock Time

```json
{
  "type": "clock",
  "time": "18:30"
}
```

## 12. Location Object

```json
{
  "placeName": "Library",
  "addressText": "New York Public Library",
  "latitude": 40.7532,
  "longitude": -73.9822,
  "radiusMeters": 150,
  "resolvePolicy": "askUser"
}
```

| Field | Type | Required | Description |
|---|---|---:|---|
| `placeName` | string | no | Place name in the user's language when appropriate |
| `addressText` | string | no | Human-readable address |
| `latitude` | number | no | Latitude |
| `longitude` | number | no | Longitude |
| `radiusMeters` | number | no | Trigger radius; default 150 |
| `resolvePolicy` | string | no | `askUser`, `useCoordinate`, or `geocodeIfAvailable` |

AI assistants must not invent coordinates.

If exact coordinates are unknown, omit `latitude` and `longitude`, then set:

```json
"resolvePolicy": "askUser"
```

CuePlan import behavior:

- Coordinates present: prefill the map.
- Address present but no coordinates: ask the user to confirm the location on the map.
- No address and no coordinates: mark the location as incomplete and ask the user to fill it.

## 13. B-Plan Extension

B-plan nodes use `routeRole`:

- `a`: only active in A plan
- `b`: only active in B plan
- `ab`: active in both A and B plans

Example:

```json
{
  "id": "client_call",
  "title": "Client Call",
  "icon": "📞",
  "routeRole": "ab",
  "end": {
    "type": "duration",
    "seconds": 1800,
    "allowsOvertime": true
  },
  "switchRules": [
    {
      "condition": "nodeOvertime",
      "targetRoute": "b",
      "timing": "afterCurrentNode"
    }
  ]
}
```

### Switch Conditions

| Value | Meaning |
|---|---|
| `nodeOvertime` | Current node is overtime |
| `nodeElapsedMoreThan` | Current node elapsed time is greater than threshold |
| `nodeElapsedLessThan` | Current node elapsed time is less than threshold |
| `flowElapsedMoreThan` | Total flow elapsed time reaches threshold |
| `locationReachedAtNodeEnd` | Location is reached when the node ends |
| `locationNotReachedAtNodeEnd` | Location is not reached when the node ends |

Conditions with thresholds must include `seconds`.

Example:

```json
{
  "condition": "nodeElapsedMoreThan",
  "seconds": 900,
  "targetRoute": "b",
  "timing": "afterCurrentNode"
}
```

### Switch Timing

| Value | Meaning |
|---|---|
| `immediately` | End current node and switch route |
| `afterCurrentNode` | Switch after current node ends |

## 14. Size Limits

Recommended import limits:

| Import Method | Recommended Limit | Notes |
|---|---:|---|
| QR code | about 3 KB | Suitable only for small flows |
| Clipboard text | 100 KB soft limit, 256 KB hard limit | Prefer JSON file for large flows |
| JSON file | 1 MB | Suitable for larger flows |
| Normal mode nodes | 100 soft limit, 200 hard limit | Large previews may be slower |
| B-plan nodes | 80 soft limit | Route validation is more complex |

## 15. Clipboard Strategy

CuePlan should not poll or silently read the clipboard in the background.

Recommended behavior:

1. On the import screen, optionally detect whether the clipboard may contain text.
2. Read full clipboard content only after user intent, such as tapping "Read Clipboard".
3. If `CUEPLAN_FLOW_V1` or `cueplan.flow` is found, show an import prompt.
4. If the content is too large, ask the user to import a JSON file instead.

## 16. Safety Rules

- Import content is data only. Never execute code.
- URLs are stored as text only and should not open automatically.
- Coordinates must be range-checked.
- Node IDs must be unique after normalization.
- Empty node titles are invalid.
- Imported plan IDs and node IDs should be regenerated internally.
- Imported workflows must not overwrite existing workflows.
- Imported workflows must not auto-start.
- B-plan flows with ambiguous route logic must open the import preview.

## 17. Normal Mode Example

````text
CUEPLAN_FLOW_V1
```json
{
  "format": "cueplan.flow",
  "version": 1,
  "plan": {
    "title": "Deep Work Morning",
    "summary": "A calm morning routine for focused work.",
    "mode": "normal",
    "icon": "🧠",
    "start": {
      "type": "scheduled",
      "time": "09:00"
    },
    "repeat": {
      "type": "weekdays"
    },
    "nodes": [
      {
        "id": "prepare",
        "title": "Prepare desk",
        "note": "Clear the desk and open only the tools needed.",
        "icon": "🧹",
        "end": {
          "type": "duration",
          "seconds": 300,
          "allowsOvertime": true
        }
      },
      {
        "id": "focus",
        "title": "Focus block",
        "note": "Work on the most important task.",
        "icon": "🎯",
        "end": {
          "type": "duration",
          "seconds": 2700,
          "allowsOvertime": false
        }
      },
      {
        "id": "review",
        "title": "Review next action",
        "icon": "✅",
        "end": {
          "type": "manual"
        }
      }
    ]
  }
}
```
````

## 18. Chinese Normal Mode Example

````text
CUEPLAN_FLOW_V1
```json
{
  "format": "cueplan.flow",
  "version": 1,
  "plan": {
    "title": "上班前准备",
    "summary": "帮用户按部就班完成出门前准备。",
    "mode": "normal",
    "icon": "🏠",
    "start": {
      "type": "manual"
    },
    "repeat": {
      "type": "none"
    },
    "nodes": [
      {
        "id": "check_bag",
        "title": "检查包",
        "note": "确认钥匙、钱包、证件和电脑都带好。",
        "icon": "🎒",
        "end": {
          "type": "duration",
          "seconds": 180,
          "allowsOvertime": true
        }
      },
      {
        "id": "leave_home",
        "title": "准备出门",
        "note": "关灯、关窗、锁门。",
        "icon": "🚪",
        "end": {
          "type": "manual"
        }
      }
    ]
  }
}
```
````

## 19. B-Plan Example

````text
CUEPLAN_FLOW_V1
```json
{
  "format": "cueplan.flow",
  "version": 1,
  "plan": {
    "title": "Client Meeting Backup",
    "summary": "A meeting flow with an online backup route.",
    "mode": "bPlan",
    "icon": "👥",
    "start": {
      "type": "manual"
    },
    "repeat": {
      "type": "none"
    },
    "nodes": [
      {
        "id": "prepare",
        "title": "Prepare agenda",
        "icon": "📝",
        "routeRole": "ab",
        "end": {
          "type": "duration",
          "seconds": 600,
          "allowsOvertime": true
        }
      },
      {
        "id": "in_person_room",
        "title": "Set up meeting room",
        "icon": "🏢",
        "routeRole": "a",
        "end": {
          "type": "location",
          "location": {
            "placeName": "Office meeting room",
            "addressText": "Company office",
            "radiusMeters": 100,
            "resolvePolicy": "askUser"
          }
        },
        "switchRules": [
          {
            "condition": "locationNotReachedAtNodeEnd",
            "targetRoute": "b",
            "timing": "afterCurrentNode"
          }
        ]
      },
      {
        "id": "remote_setup",
        "title": "Prepare video call",
        "icon": "💻",
        "routeRole": "b",
        "end": {
          "type": "duration",
          "seconds": 480,
          "allowsOvertime": true
        }
      },
      {
        "id": "send_summary",
        "title": "Send summary",
        "icon": "📨",
        "routeRole": "ab",
        "end": {
          "type": "manual"
        }
      }
    ]
  }
}
```
````

## 20. Prompt Template for Other AI Assistants

Users can paste this prompt into another AI assistant:

```text
Create a CuePlan AI Flow v1 workflow.
Output exactly one CUEPLAN_FLOW_V1 block.
Do not add explanations outside the block.
Use English JSON keys and enum values exactly as specified.
Use my language for the plan title, summary, node titles, node notes, place names, and addresses.
Default to normal mode unless I explicitly ask for a backup route, fallback plan, Plan B, or A/B route switching.
If a location is needed but exact coordinates are unknown, provide placeName and addressText only, omit latitude and longitude, and set resolvePolicy to askUser.
Specification URL: <Canonical URL>
```

## 21. 中文说明

这个标准给其他 AI 阅读时以英文为主，是为了减少不同模型对协议字段的误解。实际生成给用户看的流程标题、节点名称、说明、地点名称，仍然应该使用用户自己的语言。

简单理解：

- 协议字段固定英文，例如 `format`、`nodes`、`duration`。
- 用户看到的内容按用户语言生成，例如“检查包”“准备出门”。
- 普通流程默认用 `normal`。
- 只有用户明确要备用路线、A/B 方案、B 计划时，才用 `bPlan`。
- AI 不知道准确坐标时，不要乱写经纬度，只写地址，并设置 `resolvePolicy` 为 `askUser`。

