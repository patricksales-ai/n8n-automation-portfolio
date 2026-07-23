# docs

- **`sample-chases.csv`** — the shape of the `Chases` ledger tab. Create a Google Sheet with this
  header row (tab named `Chases`) and point the Sheets nodes at it. The rows here are fictional;
  `level` is the numeric form of `tier` (friendly=1, firm=2, final=3) and is what the dedupe gate
  compares to decide whether an invoice has escalated.

- **`workflow-chaser.png`** — the daily chaser canvas, annotated: trigger → tier → ledger/dedupe →
  AI writer → tier routing → digest approval → send + log, with the paid-detection branch below.

- **`workflow-weekly-brief.png`** — the Monday brief canvas: trigger → receivables summary →
  AI insight → branded HTML → send.

- **Demo recording** — added in a later pass.

> Everything in this folder is fake sample data. No real customer, invoice, or contact information
> appears anywhere in this repository.
