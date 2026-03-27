# Metadata Agent Query Test Results

## API Reference

| Endpoint | Method | URL |
|----------|--------|-----|
| Health check | `GET` | `http://metadata-agent-alb-153222846.us-east-1.elb.amazonaws.com/health` |
| Query | `POST` | `http://metadata-agent-alb-153222846.us-east-1.elb.amazonaws.com/query` |

**Query request body:**
```json
{ "question": "your question here" }
```

**Example:**
```bash
curl -s http://metadata-agent-alb-153222846.us-east-1.elb.amazonaws.com/query \
  -H "Content-Type: application/json" \
  -d '{"question":"What modalities are available for HPAP-010?"}'
```

---

## Q1: "show me recent orders"

| Component | Value | Status |
|-----------|-------|--------|
| `database` | `donors` | ✅ Correct |
| `sql` | `SELECT donor_ID, clinical_diagnosis, age_years, sex, BMI, C-Peptide FROM donors.Metadata ORDER BY donor_ID DESC LIMIT 10` | ✅ Valid SQL, executed successfully |
| `explanation` | "Shows the 10 most recent donor records ordered by donor ID in descending order" | ✅ Accurate description of the SQL |
| `rows` | 10 donor records (HPAP-187 down to HPAP-119) with clinical fields | ✅ Returned expected data |
| `row_count` | `10` | ✅ Correct |

**Overall**: ✅ Success — the agent mapped a vague "orders" question onto donor records reasonably.

---

## Q2: "Which immune markers are common to CODEX and IMC?"

| Component | Value | Status |
|-----------|-------|--------|
| `database` | `modalities` | ✅ Correct |
| `sql` | `SELECT DISTINCT c.Donor, 'Both CODEX and IMC' FROM modalities.CODEX c INNER JOIN modalities.IMC i ON c.Donor = i.Donor` | ❌ Incorrect — joined on `Donor` instead of comparing marker/antibody columns |
| `explanation` | "Finds donors who have data available in both CODEX and IMC imaging modalities" | ⚠️ Accurate description of what the SQL does, but does not answer the original question |
| `rows` | 50 donors present in both CODEX and IMC tables | ❌ Wrong answer — returned shared donors, not shared immune markers |
| `row_count` | `50` | ✅ Correct count for what was returned |

**Overall**: ❌ Incorrect answer — agent misinterpreted "immune markers" as donor records. Schema likely lacks column-level detail for CODEX/IMC marker panels.

---

## Q3: "How many donors are suitable for cross-modality training?"

| Component | Value | Status |
|-----------|-------|--------|
| `database` | N/A | ❌ Not reached |
| `sql` | Not returned — query execution failed | ❌ Agent generated SQL referencing `Donor` column that does not exist in the queried table |
| `explanation` | Not returned | ❌ Not reached |
| `rows` | Not returned | ❌ Not reached |
| `row_count` | Not returned | ❌ Not reached |
| `error` | `DB error 500: Unknown column 'Donor' in 'field list'` | ❌ Schema mismatch — wrong column name used |

**Overall**: ❌ Hard failure — server returned a 500 error. The agent guessed the column name incorrectly (likely `donor_ID` is the correct name, not `Donor`).

---

## Q4: "What modalities are available for HPAP-010?"

| Component | Value | Status |
|-----------|-------|--------|
| `database` | `modalities` | ✅ Correct |
| `sql` | UNION across 17 modality tables filtering by `Donor = 'HPAP-010'`, joined against `Overview` | ✅ Valid SQL, correctly structured |
| `explanation` | "Finds all available data modalities for donor HPAP-010 by checking across all modality tables" | ✅ Accurate |
| `rows` | 9 modalities: Bulk ATAC-seq, Bulk RNA-seq, Flow Cytometry, BCR-seq, TCR-seq, Perifusion, Histology, CyTOF, Oxygen Consumption | ✅ Specific and correct |
| `row_count` | `9` | ✅ Correct |

**Overall**: ✅ Success — agent correctly fanned out across all modality tables and returned accurate results.

---

## Summary

| # | Question | Result | Failure Reason |
|---|----------|--------|----------------|
| 1 | show me recent orders | ✅ Success | — |
| 2 | Which immune markers are common to CODEX and IMC? | ❌ Wrong answer | Misinterpreted "markers" as donor records; schema lacks marker column detail |
| 3 | How many donors are suitable for cross-modality training? | ❌ DB Error 500 | Generated incorrect column name (`Donor` vs `donor_ID`) |
| 4 | What modalities are available for HPAP-010? | ✅ Success | — |
