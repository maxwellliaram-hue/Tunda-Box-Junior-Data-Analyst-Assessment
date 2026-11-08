# Tunda Box – Junior Data Analyst Assessment

| File | What it is |
|---|---|
| `RESPONSES.md` | Complete written answers to every question, in the order of the assessment |
| `analysis.ipynb` | Executed notebook: data audit, all SQL (run on SQLite), spreadsheet cross-checks, A/B test, mix-shift decomposition |
| `data/` | The two sample tables as CSV |
| Assessment copy | [Link to your Google Drive copy of the assessment document] |

Run it: `pip install pandas scipy jupyter` then `jupyter notebook analysis.ipynb`.

## How I approached it

**1. Audit before analysing.** The sample data has deliberate traps: a duplicated order (1006), mixed city spellings, a NULL amount (1009), a KES 185,000 outlier (1013), an orphan customer (999 on order 1020), an order that predates signup (1003), cancelled and refunded orders carrying amounts, and an incomplete final month. Each of these changes at least one answer, so I audited first and wrote down cleaning rules rather than patching queries ad hoc.

**2. State definitions, then show sensitivity.** Where a judgement call moves the answer (is 185,000 real? is the denominator all customers or only buyers?), I picked a defensible default, said so, and showed the alternative. Example: Nairobi is the top city under every plausible reading except "the 185,000 order is genuine".

**3. Write for the reader.** The manager answer (S2b) leads with the number and the single caveat that could flip it. The Head of Growth answer (Part 5b) leads with the decision.

**4. Separate "no evidence" from "no effect".** The A/B test is inconclusive (p = 0.33; CI -1pp to +3pp), not negative, and the right action is to collect more data rather than ship or kill.

**5. Check the story behind an aggregate.** The conversion drop disappears within each channel; it is a mix shift (Simpson-style). I decomposed it (-0.96pp mix, +0.10pp rate) rather than just asserting it.

## Key results at a glance
- Orders by status: completed 16 (17 rows incl. duplicate), refunded 2, cancelled 2
- Top city: Nairobi, KES 20,650 (completed, cleaned)
- Never ordered: customers 109, 110
- Repeat purchase rate: 40% of all customers (50% of buyers)
- Monthly revenue: Jun 9,000 → Jul 6,800 (-24.4%) → Aug 7,000 (+2.9%) → Sep 10,300 (+47.1%, partial month)
- A/B test: +1.0pp (+20% relative), p = 0.33, needs about 8,150 per arm
- Conversion fall: mix effect from paid traffic tripling, not worse channel performance

## Honest limitations
Small sample extract; SQLite used for execution (Postgres variants noted in comments and `RESPONSES.md`); the treatment of order 1013 is an assumption pending confirmation from the source system.
