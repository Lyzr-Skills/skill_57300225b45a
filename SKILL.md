---
name: bucket-billing-payment-tax
description: Fact bucket for WHEN and HOW the customer is invoiced and pays - payment terms (net days and basis), billing frequency, the invoice schedule (dates, periods, amounts, stubs, split invoices), payment installments within an invoice, PO requirement, PO number and PO amount, invoice-portal obligations, non-cancellable / non-refundable wording, withholding tax and gross-up, and every refund, credit, acceptance, right-of-return, dispute-hold, extended-payment or payment-contingency right outside termination. Feeds payment-term, cash-basis, uneven / split billing, PO, withholding and refund / credit checks.
metadata:
  bucket_id: billing_payment_tax
  schema_version: 2.0.0
  output_model: billing_payment_tax_models.py (BillingPaymentTaxOutput)
  brd_refs: Sec 2.3.3, 3.1, 4.2.1 (clause 6), 6.3.2, 7.2 steps 4-5, 8.1.1, 8.1.2, 8.3.1, 10.1
  tracker_rows: 18, 30, 31, 35, 52, 57
---

# Bucket: Billing, Payment & Tax

This skill is self-contained: Part A is how to extract (shared by every bucket), Part B is what
this bucket extracts, Part C is the exact output structure.

# Part A - How to extract

You extract facts for **one bucket** (Billing, Payment & Tax) from **one document**. Part B says which facts,
Part C gives the exact output structure, and the output is validated by `billing_payment_tax_models.py`
(`BillingPaymentTaxOutput`). Your output feeds three kinds of downstream checks:

| Output section | Downstream consumer | Your job |
|---|---|---|
| `values` | Fact rules in code (exact match, tolerance, date difference) | Transcribe the single value exactly + light parse |
| `records` | Calculation rules in code (sums, joins, period comparisons) | Transcribe every row of a repeating structure, completely, with stable keys |
| `evidence` | Contextual agent (policy overrides, deviations, judgements) | Capture verbatim clauses and signals; give non-binding hints only |

**Golden rule: you transcribe; code and the contextual agent decide.**

## A1. Input
- An OCR reconstruction of the PDF in **markdown**. Pages are separated by page markers
  `<<<PAGE n>>>`, where `n` is the 1-based physical page of the PDF.
- Markdown formatting in the input (`#` headings, `**bold**`, `_italic_`, `|` table cells, `-` bullets,
  `>` quotes, backticks, `<br>`, `\` escapes) is layout produced by the OCR step, not document text.

Use nothing else: not Salesforce, not Anaplan, not other documents, not general knowledge about the
customer. If a fact is not in this document, it is `not_found`.

## A2. Applicability
Read Part B section 1 and the document.
- Bucket does not apply -> `applicable: false`, a one-sentence `applicability_reason`, and
  `values`, `records`, `evidence` set to `null`. `observations` and `notes` are `[]`.
- Bucket applies -> `applicable: true` and fill **every** field Part B defines.

"Applies" means the document is the kind that could contain these facts, not that it does.
Silence in an applicable document is evidence (no opt-out clause, no budget cap), so it is reported
as `not_found`, never skipped.

## A3. Citations
Every `found`, `inferred` or `unreadable` fact carries `sources`, a list of
`{"page_number": int, "quote": [string, ...]}`. The quotes are used to **highlight the fact in the
original PDF**, so they must match the OCR text exactly.

**Rules for each string in `quote`:**
1. It is a verbatim span from **one single line** of the page. Never join two lines in one string.
2. It contains **plain text only**: no markdown syntax (`#`, `**`, `__`, backticks, `>`, leading
   `-` / `*` / `+` bullets, `<br>`, HTML tags, `\` escapes) and **never a `|` character**. Drop the
   formatting characters and keep the words between them: `**Order Schedule Number:** ACM-1`
   is quoted as `Order Schedule Number: ACM-1`.
3. Copy the characters exactly as the OCR text has them, including OCR typos, casing, punctuation,
   currency symbols and number formatting. No ellipses, no paraphrase, no translation.
4. Make it long enough to identify the fact unambiguously on the page (label + value, or the
   sentence that states it), but no longer than the line.

**When a fact spans several lines or cells, use one string per line or cell, in reading order:**
- Address block on 5 lines -> 5 strings.
- Label on one line, value on the next -> 2 strings.
- Table row -> one string per relevant cell (`| Solution Architect | 200 |` ->
  `["Solution Architect", "200"]`).
- A printed value that itself contains `|` (e.g. DocuSign `10-Apr-26 | 5:35:23 PM EDT`) -> split at
  the pipe: `["10-Apr-26", "5:35:23 PM EDT"]`. The `raw` field still keeps the full printed value.
- A fact on two pages -> two source objects, one per page.

**By status:**
- `inferred`: quote the text you inferred from. Never invent text for something that is absent.
- `unreadable`: quote the partially legible text as the OCR gives it; if nothing is legible, use
  `["[illegible]"]` with the page number.
- `not_found` / `not_applicable`: `sources` is `[]`.

## A4. Status and confidence
| Status | Use when | raw / parsed |
|---|---|---|
| `found` | Stated in the document | raw = as printed (formatting removed); parsed = light parse |
| `inferred` | Not stated directly but follows from specific text | raw = the text relied on; parsed = your reading; confidence at most `medium` |
| `not_found` | Bucket applies, document is silent | both null, no sources, no candidates |
| `not_applicable` | This field cannot exist in this document type | both null, no sources, no candidates |
| `unreadable` | Text is there but illegible | raw = what is legible; parsed = null |

`confidence`: `high` = explicit and unambiguous; `medium` = explicit but wording or formatting
leaves some doubt, or inferred; `low` = conflicting or barely legible.

## A5. Light parsing
`raw` is the truth (as printed, markdown removed); `parsed` is a convenience that code re-checks.
Put `parsed: null` rather than guessing.

| Type in Part C | parsed format | Rules |
|---|---|---|
| Text | string | Trim whitespace and line breaks; do not rewrite. |
| Number | number | Remove thousands separators; respect decimal commas (`1.234,56` -> 1234.56). Percentages in percent units (`99.5%` -> 99.5). |
| Boolean | true / false | Only from explicit wording; otherwise null. |
| Date | `YYYY-MM-DD` | Only if unambiguous. `03/04/2026` with no locale evidence -> null + `normalization` note. |
| Time | `HH:MM:SS` (24 h) | Seconds `00` if not printed. |
| Timestamp | `YYYY-MM-DDTHH:MM:SS` | Local time as printed; **no conversion, no offset**. |
| UtcOffset | `+HH:MM` / `-HH:MM` | Only when printed as an offset or an unambiguous zone listed in Part B. |
| Money | `{"amount": number, "currency_iso": "USD"}` | ISO 4217. `$` alone is ambiguous -> `currency_iso: null` unless the document names the currency. |
| CountryCode | ISO 3166-1 alpha-2 (`US`, `GB`) | |
| SubdivisionCode | ISO 3166-2 (`US-IL`, `CA-ON`) | |
| LanguageCode | ISO 639-1 (`en`, `fr`) | |
| CurrencyCode | ISO 4217 (`USD`, `EUR`) | Only when the document names the currency or uses an unambiguous symbol (`€`, `£`). |
| enum (`"a" \| "b"`) | one listed value | If nothing fits, use `other` where listed and keep the wording in raw. Never force a fit. |

**Never compute.** No totals, sums, differences, term lengths, execution dates, discounts, FX or
time-zone conversions. If the document prints a total, transcribe it. Code does the arithmetic.

## A6. Conflicts -> candidates
If the document states the same fact differently in two places, put the reading you judge primary
in the value, every other reading in `candidates` (each with its own sources), set
`confidence: low`, and add a `conflict` note. Do not reconcile.

## A7. Records (repeating structures)
- One item per row / entry **as printed**. Do not merge, split, de-duplicate or re-order.
- `key`: follow the key rule in Part B. Keys must be unique within the collection.
- Leaves are `{"raw", "parsed"}`; `status` and `sources` sit on the item (quote the row's line or cells).
- A column that is blank for a row -> `{"raw": null, "parsed": null}`.
- `completeness`: `items_extracted` = number of items you output; `items_declared` = a count the
  document itself prints, else null; `continues_beyond_extraction` = true if the table visibly
  continues somewhere not in this document; `status` = `complete` / `partial` / `unknown`.
- `declared_totals`: every total / subtotal printed for the collection, transcribed.
- Empty collection: `items: []`, `items_extracted: 0`, status `complete`.

## A8. Evidence (for the contextual agent)
**Clauses**: capture each clause matching one of Part B's clause categories.
- `clause_id`: `C1`, `C2`, … in document order.
- `text_verbatim`: the clause text with markdown formatting removed and line breaks kept.
- Length: full text for categories marked ★ (full-verbatim). Otherwise up to 1,500 characters; if
  cut, set `text_truncated: true` and stop at a sentence end.
- `sources`: one string per line of the clause (and one source object per page).
- `overrides_hint`: fill it when the clause says it modifies, replaces, supplements or conflicts with
  another document or clause; otherwise null.

**Signals**: short factual pointers explaining why something may matter.
- `signal_id`: `S1`, `S2`, … Link `clause_id` when the signal comes from a captured clause.
- Record weak signals too. `description` states what is printed, not a conclusion.

**Hints**: one entry for every hint Part B defines, every time the bucket applies.
- `likely` must cite at least one `signal_id`; `unlikely` / `unclear` may have an empty list.
- Hints are leanings, not verdicts.

## A9. Observations and notes
- `observations`: useful facts this bucket does not define but that matter to contract review
  (cited). Do not repeat facts already in a field.
- `notes`: `conflict`, `normalization`, `missing` (expected but absent), `multi_document`,
  `truncation`, `other`, with `field_path` when about a specific field.

## A10. Fact paths
Downstream checks, agent findings and human overrides address facts by path:
`billing_payment_tax.values.<field>`, `billing_payment_tax.records.<collection>[<key>].<leaf>`,
`billing_payment_tax.evidence.clauses[<clause_id>]`, `billing_payment_tax.evidence.signals[<signal_id>]`.

## A11. What the validator checks
`billing_payment_tax_models.py` rejects the output if any of these fail; it is retried once with the errors,
then routed to a human.
1. Structure, field names, types and enum values match Part C exactly (no extra fields).
2. Every quote string is plain text (no `|`, no markdown syntax, no line break) and is found on a
   single line or table cell of its page.
3. Status rules: `found` / `inferred` / `unreadable` have sources; `not_found` / `not_applicable` have
   none and null raw / parsed; `inferred` is never `high` confidence.
4. `values` / `records` / `evidence` are null exactly when `applicable` is false.
5. `items_extracted` equals the number of items; record keys are unique; every item has a source.
6. Clause and signal IDs are unique and every reference to them exists.
7. Every hint appears exactly once; `likely` hints cite a signal; ★ clauses are not truncated.

# Part B - Billing, Payment & Tax

**Core rule: transcribe the money-timing terms as printed; code applies the thresholds.** The
60-day analysis threshold (#30), the 120-day cash-basis alert (Rule 6.3.2), the stub-period count
(Rule 8.1.1) and the SSA default of Net 30 when an order is silent are all applied by code. Report
silence as `not_found`; never fill in a default.

## 1. Applicability
**Applies** to any document with payment, invoicing, PO, tax or refund / credit terms: OS, SSA /
customer-paper master agreement (clause 6 and equivalents), SOW, change order, amendment, PO,
partner agreement, marketplace private offer.

**Does not apply** to documents with none of these (e.g. an SLA exhibit, a DPA, a Deal Desk email).

## 2. Ownership boundaries
| Fact | Owner bucket | Note |
|---|---|---|
| Line prices, totals, currency | Pricing | invoice amounts are here |
| Refunds or credits that arise on **termination** / opt-out | Term & Termination | all other refunds and credits are here |
| SLA service credits (availability, support) | SLA & Support | non-SLA credits are here |
| PO document's own number and its link to the OS / SOW | Document Identity & References | PO number printed **for invoicing** and the PO amount are here |
| Bill-to address and tax IDs | Parties & Geography | |
| Tax code and rate for the country | code (BRD 3.4 table) | only contractual tax terms (withholding, gross-up, exemption) are extracted |
| Hours upload portal for the PM | Services Delivery | AR invoice-upload portal is here |
| Milestone amounts tied to services deliverables | Services Delivery | payment installments that split an invoice are here |

## 3. values (fact rules)
| Field | Type | Extraction notes |
|---|---|---|
| `payment_terms_days` | Number | "Net 45", "within 60 days of invoice" -> 45 / 60. Different days in different places (body vs table, OS vs attached terms) -> `candidates` + `conflict` note. "Due on receipt" -> 0. |
| `payment_terms_basis` | enum `from_invoice_date`, `from_receipt_of_invoice`, `end_of_month_following`, `from_delivery`, `from_acceptance`, `other` | What the period counts from. |
| `billing_frequency` | enum `annual_in_advance`, `semi_annual_in_advance`, `quarterly_in_advance`, `monthly_in_advance`, `upfront_full_term`, `in_arrears`, `milestone`, `usage_based`, `other` | As **stated**. If the stated frequency and the printed schedule disagree, keep the stated value and add a signal; do not infer frequency from the schedule. |
| `po_required` | Boolean | From a "PO Required" field or wording such as "a purchase order is required". |
| `po_number` | Text | As printed, including placeholders such as "TBD" or "Exempt" (code interprets them). |
| `po_amount` | Money | Total amount on a PO document; on other documents only if a PO amount is printed. |
| `invoice_portal_required` | Boolean | Invoices must go through a portal (Coupa, Ariba, Tungsten, government e-invoicing). |
| `invoice_portal_name` | Text | |
| `fees_non_cancellable` | Boolean | SSA clause 6 standard wording. |
| `fees_non_refundable` | Boolean | SSA clause 6 standard wording. Exceptions go in `refund_credit_rights`. |
| `withholding_tax_pct` | Number | Rate the customer may deduct, if printed. |
| `tax_gross_up` | Boolean | true = customer must gross up for withholding (Anaplan standard "free and clear of deductions"); false = text says it will not gross up / may deduct without gross-up. |

## 4. records (calculation rules)

### invoice_schedule: one item per scheduled invoice
| Leaf | Type | Notes |
|---|---|---|
| `invoice_date` | Date | Only if a date is printed. Event-based triggers ("upon execution") -> raw kept in `invoice_trigger_raw`, date null. |
| `invoice_trigger_raw` | Text | "Upon execution", "on each anniversary", "upon acceptance of Milestone 2". |
| `period_start` / `period_end` | Date | Service period the invoice covers, if printed. |
| `amount` | Money | |
| `percent_of_total` | Number | Only if printed as % of the total contract value. Percentages of something else ("50% of Year 1") stay in `description_raw`. |
| `is_stub_period` | Boolean | true only when the schedule labels it a stub / pro-rated / partial period. Code detects unlabelled stubs from the dates. |
| `description_raw` | Text | Row wording that does not fit elsewhere. |

Key: printed invoice / installment number (`1`, `1a`, `Y2`), else `I1…`.

### payment_installments: payments that split ONE invoice
Only when an invoice is payable in several dates or milestones ("Invoice 1 payable 50% within 30
days and 50% on 1 December 2026"). Key `P1…`. Leaves: `invoice_ref` (the invoice key it splits),
`due_date`, `due_trigger_raw`, `amount`, `percent`.

### refund_credit_rights: every right that gives money back or delays / conditions payment
| Leaf | Type | Notes |
|---|---|---|
| `right_type` | enum `refund_right`, `fee_credit`, `non_sla_service_credit`, `acceptance_condition`, `right_of_return`, `billing_dispute_hold`, `extended_payment`, `payment_contingency`, `set_off`, `other` | `payment_contingency` = pay-when-funded, payment subject to budget / grant / customer approval. `extended_payment` = deferred or instalment payment beyond normal terms. |
| `holder` | enum `customer`, `anaplan`, `either` | Who benefits. |
| `amount` / `percent` | Money / Number | If stated. |
| `condition_raw` | Text | When the right applies. |
| `description_raw` | Text | Short wording. |

Key `<right_type>:<n>`. Record weak or conditional rights too.

## 5. evidence (contextual agent)
**Clause categories** (★ = full-verbatim): `payment_terms`, `invoicing`, `billing_schedule` ★,
`purchase_order` ★, `invoice_portal`, `taxes`, `withholding_tax` ★, `refund_credit` ★, `acceptance` ★,
`payment_contingency` ★, `billing_dispute` ★, `late_payment`, `other_billing`.

**Signal types**:
| Type | Record when |
|---|---|
| `non_standard_payment_days` | Payment days are printed and are not 30 (state the days; code judges) |
| `payment_contingency_language` | Pay-when-funded, subject to appropriation / budget / approval |
| `withholding_language` | Customer may withhold or deduct taxes |
| `gross_up_language` | Any wording on grossing up (for or against) |
| `tax_exempt_language` | Customer claims tax exemption / exemption certificate |
| `stub_period` | A labelled stub, pro-rated or partial period |
| `split_first_invoice` | The first year's / first invoice's fees are split across several invoices or payment dates |
| `non_annual_billing` | Billing other than annual in advance (quarterly, monthly, arrears, full upfront) |
| `invoice_portal_language` | Invoices must go through a portal |
| `po_required_language` | PO required / PO number must appear on invoices |
| `refund_language` | Refund rights outside termination |
| `acceptance_language` | Payment or fees depend on customer acceptance |
| `dispute_hold_language` | Customer may withhold disputed amounts |
| `other` | Anything else about billing or payment a reviewer should see |

**Hints** (each exactly once):
| Hint | Lean on… | Feeds |
|---|---|---|
| `split_billing` | Initial invoice split into several invoices or payment dates, or billing not proportional to the period | #35 Uneven Billing, Rules 8.1.1 / 8.1.2 |
| `cash_collection_risk` | Payment is contingent, deferred or exposed to holds beyond normal terms | #52 Cash Basis (with payment days) |
| `custom_withholding_tax` | Withholding-tax terms differ from "free and clear of deductions" | #57 Bookings Adjustment, Rule 8.3.1 |
| `refund_or_credit_rights` | Refund, credit, acceptance, right of return, dispute hold or extended payment rights exist | #31 No extended terms / refund / credit |

## 6. Code-side derivations (never done by the model)
- `effective_payment_days`: OS value, else governing SSA value, else 30 (SSA clause 6.1), recording the source used.
- Stub count: compare each invoice period with the term (Term & Termination) and anniversaries; >= 2 stubs -> Rule 8.1.1.
- First-invoice split: more than one invoice covering the first period, or any `payment_installments` on the first invoice -> Rule 8.1.2.
- Proportionality: invoice amount / period length vs the order's annual value (Pricing).
- PO checks: `po_number` on OS = PO document number = SF `PO_Number__c`; PO amount >= amount due.

## 7. Validations fed (rule registry)
| Rule | Type | Check | Fact paths |
|---|---|---|---|
| #30 Payment Term <= 60 days | fact | `effective_payment_days` <= 60 -> 'Y', else 'N<days>' (e.g. 'N90'); conflicts between OS, SSA and PO flagged | `values.payment_terms_days` (OS, SSA, PO) |
| #52 Cash Basis (Rule 6.3.2) | fact + context | > 120 days -> Cash-Basis Revenue Alert + Commissions Deferral Review; payment contingency -> flag for the accountant | `values.payment_terms_days`, `records.refund_credit_rights[payment_contingency]`, hint `cash_collection_risk` |
| #35 Uneven Billing (Rules 8.1.1 / 8.1.2) | calc | >= 2 stubs, first invoice split, amounts not proportional, or non-annual billing without matching SF instructions -> TRUE + schedule | `records.invoice_schedule`, `records.payment_installments`, `values.billing_frequency`, hint `split_billing` |
| BRD 8.1 Quota deferral | context | Split billing -> HITL to Bookings / Commissions for quota credit deferral | same as #35 |
| #18 PO Needed (Rule 2.3.3) | fact | 'Yes' / 'No' / 'Exempt' + PO number; PO received and mapped; PO number on PO = OS = SF; currency equal | `values.po_required`, `values.po_number`, `values.po_amount` |
| BRD 7.2 s4 PO coverage | calc | SOW spend <= PO amount, else "PO Under-Funded for SOW Spend" | `values.po_amount` + Services Delivery / Pricing |
| BRD 7.2 s5 AR invoice portal | fact | Portal required -> operations task for Accounts Receivable | `values.invoice_portal_required`, `values.invoice_portal_name` |
| #31 No extended terms / refund / credit | context | Any refund, credit, acceptance, return, dispute hold, extended payment right -> label(s); none (and none in Term & Termination) -> 'Y' | `records.refund_credit_rights`, hint `refund_or_credit_rights` |
| #28 SSA clause 6 (Rule 4.2.1) | fact | Net 30, non-cancellable, non-refundable, taxes free and clear (gross-up) -> deviations listed | `values.payment_terms_days`, `fees_non_cancellable`, `fees_non_refundable`, `tax_gross_up` on the SSA |
| #57 / Rule 8.3.1 Withholding | context | Custom withholding -> HITL to Bookings; proportional quota reduction = withholding % (BRD 10.2) | `values.withholding_tax_pct`, `values.tax_gross_up`, hint `custom_withholding_tax` |
| BRD 10.1 Payment Override Alert | context | Non-standard payment terms passed to the accountant for billing schedule setup | `evidence.clauses[payment_terms, billing_schedule]` |

# Part C - Output structure

Return **one JSON object** exactly in this shape. Every key shown is required (use `null` /
`[]` as the rules allow); no other keys are allowed.

## C1. Shared shapes

```jsonc
Source        = { "page_number": int >= 1, "quote": [string, ...] }        // >= 1 plain single-line string
Value<T>      = { "raw": string | null, "parsed": T | null,
                  "status": "found" | "not_found" | "not_applicable" | "inferred" | "unreadable",
                  "confidence": "high" | "medium" | "low",
                  "sources": [Source], "candidates": [Candidate] }
Candidate     = { "raw": string, "parsed": string | number | boolean | Money | null, "sources": [Source] }
Leaf<T>       = { "raw": string | null, "parsed": T | null }
Money         = { "amount": number | null, "currency_iso": string | null }   // ISO 4217, 3 letters
Item          = { "key": string, "status": "found" | "inferred" | "unreadable", "sources": [Source] (>= 1), ...leaves }
Collection<I> = { "completeness": Completeness, "declared_totals": [DeclaredTotal], "items": [I] }
Completeness  = { "items_extracted": int, "items_declared": int | null,
                  "continues_beyond_extraction": boolean, "status": "complete" | "partial" | "unknown" }
DeclaredTotal = { "label_raw": string, "value": Leaf<Money>, "sources": [Source] }
Clause        = { "clause_id": string, "clause_number_raw": string | null, "heading_raw": string | null,
                  "text_verbatim": string, "text_truncated": boolean, "category_hints": [category],
                  "overrides_hint": OverridesHint | null, "sources": [Source] }
OverridesHint = { "target_document_raw": string | null, "target_clause_raw": string | null,
                  "relationship": "modifies" | "replaces" | "supplements" | "conflicts" | "unclear" }
Signal        = { "signal_id": string, "signal_type": signal_type, "description": string, "entity_raw": string | null,
                  "strength": "strong" | "moderate" | "weak", "clause_id": string | null, "sources": [Source] }
Hint          = { "hint": hint_name, "value": "likely" | "unlikely" | "unclear", "signal_ids": [string] }
Observation   = { "label": string, "value_raw": string, "sources": [Source] }
Note          = { "type": "conflict" | "normalization" | "missing" | "multi_document" | "truncation" | "other",
                  "field_path": string | null, "message": string }

Scalar types: Text = string, Number = number, Boolean = boolean, Date = "YYYY-MM-DD",
Time = "HH:MM:SS", Timestamp = "YYYY-MM-DDTHH:MM:SS", UtcOffset = "+HH:MM" | "-HH:MM",
CountryCode = "US", SubdivisionCode = "CA-ON", LanguageCode = "en", CurrencyCode = "USD", Money as above.
```

## C2. `billing_payment_tax` output

```jsonc
{
  "bucket": "billing_payment_tax",
  "schema_version": "2.0.0",
  "applicable": true,                // false -> values, records, evidence are null
  "applicability_reason": string,

  "values": {
    "payment_terms_days": Value<Number>,
    "payment_terms_basis": Value<"from_invoice_date" | "from_receipt_of_invoice" | "end_of_month_following" | "from_delivery" | "from_acceptance" | "other">,
    "billing_frequency": Value<"annual_in_advance" | "semi_annual_in_advance" | "quarterly_in_advance" | "monthly_in_advance" | "upfront_full_term" | "in_arrears" | "milestone" | "usage_based" | "other">,
    "po_required": Value<Boolean>,
    "po_number": Value<Text>,
    "po_amount": Value<Money>,
    "invoice_portal_required": Value<Boolean>,
    "invoice_portal_name": Value<Text>,
    "fees_non_cancellable": Value<Boolean>,
    "fees_non_refundable": Value<Boolean>,
    "withholding_tax_pct": Value<Number>,
    "tax_gross_up": Value<Boolean>
  },

  "records": {
    "invoice_schedule": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "invoice_date": Leaf<Date>,
      "invoice_trigger_raw": Leaf<Text>,
      "period_start": Leaf<Date>,
      "period_end": Leaf<Date>,
      "amount": Leaf<Money>,
      "percent_of_total": Leaf<Number>,
      "is_stub_period": Leaf<Boolean>,
      "description_raw": Leaf<Text>
    }>,
    "payment_installments": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "invoice_ref": Leaf<Text>,
      "due_date": Leaf<Date>,
      "due_trigger_raw": Leaf<Text>,
      "amount": Leaf<Money>,
      "percent": Leaf<Number>
    }>,
    "refund_credit_rights": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "right_type": Leaf<"refund_right" | "fee_credit" | "non_sla_service_credit" | "acceptance_condition" | "right_of_return" | "billing_dispute_hold" | "extended_payment" | "payment_contingency" | "set_off" | "other">,
      "holder": Leaf<"customer" | "anaplan" | "either">,
      "amount": Leaf<Money>,
      "percent": Leaf<Number>,
      "condition_raw": Leaf<Text>,
      "description_raw": Leaf<Text>
    }>
  },

  "evidence": {
    "clauses": [Clause],
    "signals": [Signal],
    "hints": [Hint]
  },

  "observations": [Observation],
  "notes": [Note]
}
```

Allowed lists for this bucket:

- `category` (clause `category_hints`): `payment_terms`, `invoicing`, `billing_schedule` ★, `purchase_order` ★, `invoice_portal`, `taxes`, `withholding_tax` ★, `refund_credit` ★, `acceptance` ★, `payment_contingency` ★, `billing_dispute` ★, `late_payment`, `other_billing`
  (★ = full-verbatim, never truncated)
- `signal_type`: `non_standard_payment_days`, `payment_contingency_language`, `withholding_language`, `gross_up_language`, `tax_exempt_language`, `stub_period`, `split_first_invoice`, `non_annual_billing`, `invoice_portal_language`, `po_required_language`, `refund_language`, `acceptance_language`, `dispute_hold_language`, `other`
- `hint_name` (each exactly once): `split_billing`, `cash_collection_risk`, `custom_withholding_tax`, `refund_or_credit_rights`
