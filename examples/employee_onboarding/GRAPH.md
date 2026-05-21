# LangGraph — Onboarding Agent

Each user message triggers one full graph run. State persists across turns via `MemorySaver`.

```mermaid
flowchart TD
    START(( START ))
    END_NODE(( END ))

    START -->|route_turn| RT{route_turn}

    RT -->|no case loaded| load_case
    RT -->|phase == needs_corrections| apply_corrections
    RT -->|phase == qa| qa_dispatch
    RT -->|phase == done| noop

    load_case["load_case\n─────────────\nFetch case + employee\nfrom Django API\nMark case in_progress\nEmit welcome message"]
    apply_corrections["apply_corrections\n─────────────\nParse employee reply\nExtract corrected fields\nPATCH Employee in DB\nRe-fetch case"]

    load_case --> validate_form
    apply_corrections --> validate_form

    validate_form["validate_form\n─────────────\nLLM structured-output\nchecks REQUIRED_FIELDS\nfor missing / invalid data"]

    validate_form -->|validation_missing non-empty| ask_for_corrections
    validate_form -->|validation_missing empty| brief_employee

    ask_for_corrections["ask_for_corrections\n─────────────\nEmit chat message\nlisting missing fields\nSet phase → needs_corrections\nLog message to DB"]

    brief_employee["brief_employee\n─────────────\nLLM generates personalized\nbriefing from policies\nSet phase → qa\nLog message to DB"]

    ask_for_corrections --> END_NODE
    brief_employee --> END_NODE

    qa_dispatch["qa_dispatch\n─────────────\nLog employee message\nLLM classifies intent:\nquestion or done"]

    qa_dispatch -->|last_intent == question| answer_question
    qa_dispatch -->|last_intent == done| finalize_and_notify_hr

    answer_question["answer_question\n─────────────\nRAG-style reply\ngrounded in policies\nLog message to DB"]

    finalize_and_notify_hr["finalize_and_notify_hr\n─────────────\nLLM generates closing msg\nLLM drafts HR email\nMark case completed\nSet phase → done\nLog message to DB"]

    answer_question --> END_NODE
    finalize_and_notify_hr --> END_NODE

    noop["noop\n─────────────\nPhase is done —\nno further action"]
    noop --> END_NODE
```

## Phase transitions

| Phase | Set by | Meaning |
|---|---|---|
| `new` | DB default | Case just created |
| `needs_corrections` | `ask_for_corrections` | Agent waiting for employee to provide missing fields |
| `qa` | `brief_employee` | Briefing delivered; open Q&A active |
| `done` | `finalize_and_notify_hr` | Onboarding complete; HR email simulated |

## Routing logic (per turn)

| Condition | Entry node |
|---|---|
| `state.case` is empty | `load_case` |
| `phase == needs_corrections` | `apply_corrections` |
| `phase == qa` | `qa_dispatch` |
| `phase == done` | `noop` |
