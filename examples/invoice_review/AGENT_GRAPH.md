# LangGraph Agent — Flow Diagram

```mermaid
flowchart TD
    START([START]) --> load_invoice

    load_invoice["**load_invoice**\nFetch invoice, order &\npricing list from Django API\n─────────────────\nstate: invoice, order,\npricing_list, iteration"]

    load_invoice --> extract_and_validate

    extract_and_validate["**extract_and_validate**\nValidate required fields,\nformats & header values\n─────────────────\nstate: validation_issues"]

    extract_and_validate --> check_against_order

    check_against_order["**check_against_order**\nCompare invoice lines\nagainst PO / goods receipt\n─────────────────\nstate: po_issues"]

    check_against_order --> check_against_pricing

    check_against_pricing["**check_against_pricing**\nCompare unit prices against\ncustomer pricing list\n─────────────────\nstate: pricing_issues, overall_ok"]

    check_against_pricing --> summarize_check

    summarize_check["**summarize_check**\nGenerate markdown summary\nof all findings for chat;\npersist InvoiceCheck to Django\n─────────────────\nstate: check_summary\n💬 emits chat message"]

    summarize_check --> await_user_decision

    await_user_decision{{"**await_user_decision**\n⏸ HITL interrupt\n─────────────────\nRenders approve / request-correction\n/ re-check card in CopilotKit sidebar"}}

    await_user_decision -->|approve| send_for_approval
    await_user_decision -->|correct| mark_for_correction
    await_user_decision -->|recheck| load_invoice

    send_for_approval["**send_for_approval**\nMark invoice → approved\nvia Django API\n─────────────────\nstate: sent = true"]

    mark_for_correction["**mark_for_correction**\nMark invoice → needs_correction\nvia Django API\n(employee uploads corrected invoice\nbefore triggering recheck)"]

    send_for_approval --> END_A([END])
    mark_for_correction --> END_B([END])

    style await_user_decision fill:#f0e6ff,stroke:#7c3aed,color:#000
    style load_invoice fill:#e0f2fe,stroke:#0369a1,color:#000
    style extract_and_validate fill:#e0f2fe,stroke:#0369a1,color:#000
    style check_against_order fill:#e0f2fe,stroke:#0369a1,color:#000
    style check_against_pricing fill:#e0f2fe,stroke:#0369a1,color:#000
    style summarize_check fill:#e0f2fe,stroke:#0369a1,color:#000
    style send_for_approval fill:#dcfce7,stroke:#16a34a,color:#000
    style mark_for_correction fill:#fff7ed,stroke:#c2410c,color:#000
```

## State fields populated at each node

| Node | Reads | Writes |
|---|---|---|
| `load_invoice` | `invoice_id` | `invoice`, `order`, `pricing_list`, `iteration` |
| `extract_and_validate` | `invoice` | `validation_issues` |
| `check_against_order` | `invoice`, `order` | `po_issues` |
| `check_against_pricing` | `invoice`, `pricing_list` | `pricing_issues`, `overall_ok` |
| `summarize_check` | all issues, `invoice_id` | `check_summary` (→ Django `InvoiceCheck`) |
| `await_user_decision` | `check_summary`, `overall_ok` | — (interrupt; routes via `Command`) |
| `send_for_approval` | `invoice_id` | `sent` |
| `mark_for_correction` | `invoice_id` | — |

## Loop behaviour

When the finance employee selects **Request correction**, the graph ends at
`mark_for_correction` (invoice status → `needs_correction`). The employee then
edits the invoice via the upload-correction modal and clicks **Re-run check**,
which resolves the interrupt with `recheck` — jumping back to `load_invoice`
and incrementing `iteration`.
