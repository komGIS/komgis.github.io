# LangGraph Agent — Flow Diagram

```mermaid
flowchart TD
    START([START]) --> load_context

    load_context["**load_context**\nFetch claim + documents\nMark claim 'in review'"]
    summarize_docs["**summarize_docs**\nLLM: summarise all attached docs"]
    extract_data["**extract_data**\nLLM: structured extraction\n(parties, amounts, flags…)\nPersist to Django"]
    present_summary["**present_summary**\nPost summary + red-flags\nto chat"]

    await_qa{{"**await_qa** ⏸\ninterrupt — Q&A gateway"}}

    answer_question["**answer_question**\nLLM: answer the question\nAppend to qa_history"]

    check_policy["**check_policy**\nFetch policy docs from Django\n(product-scoped + global)"]
    suggest_decision["**suggest_decision**\nLLM: recommend approve / reject\nPersist recommendation to Django"]

    await_final_decision{{"**await_final_decision** ⏸\ninterrupt — final decision gateway"}}

    approve_and_email["**approve_and_email**\nLLM: draft finance email\nCreate DecisionEmail record\nPATCH claim → approved"]
    reject_and_record["**reject_and_record**\nCreate internal record\nPATCH claim → rejected"]

    END_NODE([END])

    load_context --> summarize_docs
    summarize_docs --> extract_data
    extract_data --> present_summary
    present_summary --> await_qa

    await_qa -- "action = ask" --> answer_question
    answer_question -- "loop back" --> await_qa
    await_qa -- "action = proceed" --> check_policy

    check_policy --> suggest_decision
    suggest_decision --> await_final_decision

    await_final_decision -- "decision = approved" --> approve_and_email
    await_final_decision -- "decision = rejected" --> reject_and_record

    approve_and_email --> END_NODE
    reject_and_record --> END_NODE

    style await_qa        fill:#f5a623,color:#000
    style await_final_decision fill:#f5a623,color:#000
    style START           fill:#4caf50,color:#fff
    style END_NODE        fill:#e53935,color:#fff
```

## Node reference

| Node | BPMN task | What it does |
|---|---|---|
| `load_context` | `Task_Load_Context` | GET `/api/claims/<id>/`, marks claim **in_review** |
| `summarize_docs` | `Task_Summarize_Docs` | LLM prose summary of every attached document |
| `extract_data` | `Task_Extract_Data` | Structured extraction via `_ExtractedData` Pydantic model; persisted to Django |
| `present_summary` | `Task_Present_Summary` | Posts summary + red-flags as an `AIMessage` |
| `await_qa` ⏸ | `Gateway_Questions` | `interrupt()` — waits for `{ action: "ask" \| "proceed" }` |
| `answer_question` | `Task_Answer_Questions` | LLM answers the question; appends to `qa_history`; loops back to `await_qa` |
| `check_policy` | `Task_Check_Policy` | Fetches policy docs from Django (product-scoped + global) |
| `suggest_decision` | `Task_Suggest_Decision` | LLM recommendation via `_DecisionOutput`; persisted to Django |
| `await_final_decision` ⏸ | `Gateway_Final_Decision` | `interrupt()` — waits for `{ decision: "approved" \| "rejected", notes }` |
| `approve_and_email` | `Task_Simulate_Email` | Drafts finance email, creates `DecisionEmail`, PATCHes claim **approved** |
| `reject_and_record` | `Task_Reject` | Creates internal `DecisionEmail` record, PATCHes claim **rejected** |

⏸ = LangGraph `interrupt()` — human-in-the-loop pause point
