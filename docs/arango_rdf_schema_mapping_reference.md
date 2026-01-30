# ArangoRDF and Schema Mapping — Reference (Tabled)

**Status:** Tabled for later. Use this document when you return to this work.  
**Last updated:** January 2025

---

## What This Is About

When you’re ready to tackle ArangoRDF and schema mapping again, this document is your entry point. It summarizes the situation, points to the detailed docs, and lists what’s left to decide and do.

---

## Source Documents

| Document | Purpose |
|----------|---------|
| **`docs/arango_rdf_analysis_report.md`** | Full analysis: how ArangoRDF is used, why ObjectProperty is empty, ontology annotations, OFAC/synthetic mapping, ingest approach. |
| **`docs/implementation_outline_arango_rdf.md`** | Step-by-step outline of proposed changes (ObjectProperty vs Property, ontology annotations, config externalization, ingest docs). |

Read the analysis report for *why* things are the way they are; use the implementation outline for *what* to change and in what order.

---

## Quick Recap of Findings

1. **ArangoRDF** is used only to load the ontology via `rdf_to_arangodb_by_pgt()`; OFAC/synthetic data are loaded in Python.
2. **ObjectProperty collection** is empty because PGT puts all predicate resources (including object properties) in the **Property** vertex collection, not in **ObjectProperty**.
3. **Ontology ↔ ArangoDB** mapping is not recorded on the ontology; there are no `adb:collection` (or `adb:key`) annotations. ArangoRDF supports `http://arangodb/collection` for this.
4. **OFAC/synthetic → physical schema** (party_type → collection, rel_type → edge, Class _ids, propagation weights) is hardcoded in `load_data.py` (and related scripts); no external config or ontology annotations.
5. **Ingest** is Python-only (csv + `import_bulk`); the PRD says “via arangoimport” but there are no arangoimport scripts.

---

## Decisions to Make When You Tackle This

- **ObjectProperty vs Property:** Choose one: (A1) use **Property** only and drop **ObjectProperty** from graph/themes/scripts; (A2) try to populate **ObjectProperty** via custom `adb_col_statements`; (A3) implement a custom **ArangoRDFController** so `rdf:type owl:ObjectProperty` → **ObjectProperty**. Recommendation in the outline: A1.
- **Ontology annotations:** Whether to add `adb:collection` (and optionally `adb:key`) to the ontology, and whether PGT should *use* them (e.g. via `adb_col_statements`) or only *document* them.
- **OFAC/synthetic mapping:** Config format (JSON vs YAML), path (e.g. `config/schema_mapping.json`), and how to handle Class _ids (manual update after first load vs discovery script).
- **Ingest:** Docs-only (update PRD to match Python ingest) vs also adding an optional arangoimport script.

---

## Suggested Order When You Return

1. **Docs (ingest)** — Update PRD/implementation plan so they say CSV is loaded via Python (e.g. `import_bulk`), not only arangoimport.
2. **ObjectProperty / Property** — Pick and implement A1, A2, or A3 from the implementation outline.
3. **Ontology annotations** — Add ArangoDB namespace and `adb:collection` (and optionally `adb:key`) to the ontology; decide use-vs-document.
4. **Externalize mapping** — Move OFAC/synthetic mappings into a config file and load it in `load_data.py`, `generate_test_data.py`, and any other scripts that use those maps.
5. **Optional** — Add an optional arangoimport path and document its limitations.

---

## Key Files (Current State)

- **Ontology:** `sentries_ontology.owl` — no ArangoDB namespace or adb annotations.
- **Load:** `scripts/load_data.py` — ArangoRDF for ontology; Python csv + `import_bulk` for OFAC; hardcoded `collection_map`, `edge_map`, `class_map`, `weight_map`.
- **Flatten:** `scripts/flatten_ofac.py` — writes `data/parties.csv`, `data/relationships.csv`.
- **Synthetic:** `scripts/generate_test_data.py` — uses same collection/edge names; no shared config with load_data.
- **Graph definition:** `scripts/load_data.py` — `ont_vertices` includes `"ObjectProperty"`; PGT does not populate it.
- **Themes:** `docs/sentries_standard.json`, `docs/sentries_risk_heatmap.json` — node configs for **Property**, **ObjectProperty**, **DatatypeProperty**.

---

## One-Line Reminder

When you come back: open **`docs/arango_rdf_analysis_report.md`** for the full picture, then **`docs/implementation_outline_arango_rdf.md`** for the concrete change list and options (A1/A2/A3, B, C, D).
