# Lending governance examples

Business definitions and pipeline definitions for a small servicing book: member, product, loan, and payment. A load publishes only after the written table matches the approved dictionary for that layer.

## How the files relate

| Artifact | Scope | What it answers |
|---|---|---|
| `glossary/` | One set for the whole book | What does this business word mean? |
| `contacts/lending.yml` | One list for the lending domain | Who is accountable? |
| `dictionaries/<layer>/` | One file per entity per layer | How is that meaning stored here? |
| `pipelines/<layer>/` | One file per entity per layer | How is the table loaded? |

A term has one definition. Bronze, silver, and gold dictionaries cite that term and describe their own representation. A transformation that only changes encoding (file code `A`, conformed value `current`, rate stored as percent or as a decimal) keeps the same term. A transformation that creates a new meaning (a 30+ delinquency flag, a current product list taken from the latest snapshot) gets its own term or a different grain on the dictionary.

`certified_for` on a term lists the layers allowed to cite it. `term.ssn` is bronze only. `term.delinquent_30_plus` is gold only.

## Walkthrough

**Same meaning, different encoding.** `term.loan_status` is the servicing status of a booked loan on the as-of date. Bronze `STATUS_CD` keeps `A/D/C/P`. Silver and gold `loan_status` use `current/delinquent/charged_off/paid_off`. `term.interest_rate` is the note's nominal annual rate. Bronze stores percent (`8.99`). Silver and gold store a decimal (`0.089900`).

**New meaning.** Gold `is_delinquent_30_plus` cites `term.delinquent_30_plus`. It must not cite `term.loan_status`. The rule is contractual days past due of 30 or more, and `loan_status = delinquent`.

**Same terms, different grain.** Product attributes keep their terms at every layer. Bronze and silver are one row per product per as-of date. Gold is one row per product, chosen as the latest silver snapshot. The grain change is on the dictionary, not a second glossary entry.

**Restricted identifier.** Bronze member `TAX_ID` cites `term.ssn` and is `restricted`. Silver and gold have no attribute for it.

**Events that undo other events.** `term.payment_effect` allows `posted`, `reversal`, and `nsf`. A reversal is a new payment id with a negative amount. `term.payment_reversal` is that concept. Silver and gold payment dictionaries list it under `concepts`.

**Not publishable yet.** `dictionaries/gold/borrower_role.yml` is `draft`. It describes joint obligors. `dict.gold.loan` still uses `term.primary_obligor` for the single member on the loan. A draft dictionary cannot certify a table.

## Publish check

Before a table is published, the runner checks the dictionary named by the pipeline:

- Dictionary `status` is `approved`, and `contact: contact.lending` resolves.
- Gold has a business owner, a data steward, and a technical owner. Silver has a steward and a technical owner. Bronze has a technical owner.
- Column names and physical types match the attributes.
- Primary-key columns are unique. `nullable: false` and `allowed_values` hold.
- Every `glossary_term` and every entry in `concepts` exists, is `approved`, and lists this layer in `certified_for`.

## Pipelines

Bronze, silver, and gold use the same keys. A file source has a path. A table source has a relation. Names of sources and steps are temp views. A transform step is one `SELECT`, or a function that returns a DataFrame registered under the step name. The next step queries that name. Optional `dq` on a step returns the bad rows. `expect: empty` and `on_fail: quarantine` drop them from the view. `on_fail: fail` stops the pipeline.

`sink.sql` is one `INSERT`, `INSERT OVERWRITE`, or `MERGE` into `target.name`. `target.location` makes that table external. Pre steps run first. Post steps run after the dictionary check succeeds. `depends_on` orders pipelines when a source is another pipeline's table.

`pipelines/silver/loan.yml` is the fullest example: two sources, a function step, and step DQ. `pipelines/gold/product.yml` is the grain change, via `INSERT OVERWRITE`. `pipelines/gold/borrower_role.yml` is disabled.

## Gaps

These showed up while writing the pipelines against the dictionaries.

1. **Lineage columns sit outside the dictionary.** Bronze attaches `_source_path`, `_file_checksum`, and `_ingest_timestamp`, then lists them in `dictionary_check.ignore_columns`. Silver uses `_ingest_timestamp` to pick a row and then drops it. The dictionary never defines those columns.
2. **Bronze append is not idempotent.** The sink is `INSERT`. Loading the same landing file again appends a second copy. The duplicate DQ only sees rows inside the current run. Silver merge hides the replay for snapshot entities. Bronze itself keeps both copies.
3. **Prose rules are copied into SQL.** Days-past-due by status, payment components summing to the total, and the reversal sign are written in dictionary `rule` or `representation` text and again in step DQ. Nothing checks that the two still match.
4. **Payment-to-loan DQ has no shared date.** `pipeline.silver.payment` and `pipeline.gold.payment` treat a loan as present if any snapshot contains that `loan_id`. A payment can pass because an older snapshot exists, or fail because this run has not published that loan yet.
5. **Borrower role cannot run.** `dict.gold.borrower_role` is `draft`, and no landing file contains co-obligors. The pipeline is `enabled: false` and has no sources or sink.
6. **Quarantine and audit tables have no dictionaries.** Step DQ writes to `lending.quarantine.*`. Every post step inserts into `lending.audit.pipeline_run`. Those tables are not certified.
7. **`depends_on` is outside the step list.** Multi-source pipelines need a run order. The step SQL cannot say that `pipeline.silver.loan` waits for `pipeline.silver.member`.
8. **Two source shapes.** File sources use `path`, `format`, and `attach_metadata`. Table sources use `relation`. The rest of the file is the same.
9. **`lending.functions.dedupe_latest` is not in this directory.** Only `pipeline.silver.loan` calls it. Member, product, and payment dedupe with SQL so both step kinds are visible. The function has to return the input columns, one row per key.
10. **The external table is not created by the sink SQL.** `MERGE` and `INSERT` assume `target.name` already exists at `target.location`. The runner still has to create that Delta table from the dictionary, plus any `ignore_columns`, before the sink runs.
11. **Gold product replaces the whole table.** `INSERT OVERWRITE` is a different outcome from the `MERGE` used for loans, members, and payments. A failed check after that write depends on the runner restoring the previous Delta version.

## Layout

```
example/
  contacts/lending.yml
  glossary/{member,product,loan,payment}.yml
  dictionaries/{bronze,silver,gold}/{member,product,loan,payment}.yml
  dictionaries/gold/borrower_role.yml
  pipelines/{bronze,silver,gold}/{member,product,loan,payment}.yml
  pipelines/gold/borrower_role.yml
  index.yml
```
