# Implementation Outline: ArangoRDF and Schema Mapping Changes

**Purpose:** Outline of proposed changes from the ArangoRDF analysis report, for review before implementation.  
**Reference:** `docs/arango_rdf_analysis_report.md`

---

## Overview

Changes are grouped into four areas. Each section lists scope, options, affected files, and steps. Dependencies between areas are noted.

| Area | Scope | Depends on |
|------|--------|------------|
| A. ObjectProperty / Property alignment | Ontology graph and UI use of Property vs ObjectProperty | None |
| B. Ontology annotations (adb:collection) | Record physical schema on the ontology | None (can inform A) |
| C. OFAC/synthetic mapping externalization | Move mappings out of code into config | None |
| D. Ingest documentation | PRD and docs vs Python ingest | None |

---

## A. ObjectProperty vs Property Alignment

**Problem:** PGT puts all predicate resources in **Property**; **ObjectProperty** is empty. Graph definition and themes reference both.

**Options (choose one):**

### Option A1: Use Property only (align with PGT)

- **Goal:** Single source of truth: **Property** holds all ontology properties (object + datatype).
- **Changes:**
  1. **Ontology graph definition** (`scripts/load_data.py`): Remove **ObjectProperty** (and **DatatypeProperty** if present) from `ont_vertices` and from any edge definitions that reference them. Use **Property** only for ontology property vertices.
  2. **Theme configs** (`docs/sentries_standard.json`, `docs/sentries_risk_heatmap.json`, etc.): Remove or repurpose **ObjectProperty** / **DatatypeProperty** node configs; ensure **Property** is styled (and optionally use rules to distinguish by type if stored as an attribute).
  3. **Scripts** that reference ontology collections (`check_cross_edges.py`, `install_theme.py` pruning, etc.): Replace **ObjectProperty** (and **DatatypeProperty**) with **Property** where they denote ontology property vertices.
  4. **Docs:** Update any references to “ObjectProperty collection” to “Property collection” (e.g. analysis report, walkthrough).

- **Files:** `scripts/load_data.py`, `docs/sentries_standard.json`, `docs/sentries_risk_heatmap.json`, `scripts/check_cross_edges.py`, `scripts/install_theme.py` (if it prunes by collection name), any other scripts that list ontology vertex collections; docs.

### Option A2: Populate ObjectProperty via custom mapping

- **Goal:** Keep **ObjectProperty** (and optionally **DatatypeProperty**) and have PGT populate them.
- **Changes:**
  1. **Custom adb_col_statements:** Build a small RDF graph (or OWL fragment) that states, for each object property resource (e.g. `sentries:owned_by`), `adb:collection "ObjectProperty"` (and for datatype properties, `adb:collection "DatatypeProperty"` if desired). Pass this into `rdf_to_arangodb_by_pgt(..., adb_col_statements=...)`.  
     **Caveat:** ArangoRDF docs say the mapping process “relies heavily on mapping certain RDF Resources to the ‘Class’ and ‘Property’ ArangoDB Collections” and “it is currently not possible to overwrite any RDF Resources that belong to these collections.” So overwriting predicate resources from Property to ObjectProperty may not be supported; needs verification with ArangoRDF behavior.
  2. **Fallback:** If overwriting is not allowed, implement **Option A3** (custom controller).
  3. **Graph/theme/scripts:** Keep **ObjectProperty** (and **DatatypeProperty**) in graph definition, themes, and scripts (no removal).

- **Files:** `scripts/load_data.py` (build and pass `adb_col_statements`), possibly a small helper or inline RDF; rest unchanged.

### Option A3: Custom ArangoRDFController

- **Goal:** Use PGT but assign collections by `rdf:type` (e.g. `owl:ObjectProperty` → **ObjectProperty**, `owl:DatatypeProperty` → **DatatypeProperty**).
- **Changes:**
  1. **New module** (e.g. `scripts/arango_rdf_controller.py`): Implement a subclass of `ArangoRDFController` that overrides `identify_best_class` (or the relevant method that decides vertex collection). For resources that are predicates with `rdf:type owl:ObjectProperty`, return the URI whose local name is `"ObjectProperty"`; for `owl:DatatypeProperty`, return the URI for `"DatatypeProperty"`; otherwise delegate to super.
  2. **load_data.py:** Instantiate ArangoRDF with this controller (e.g. `ArangoRDF(db, controller=MyController(...))`) when calling `rdf_to_arangodb_by_pgt`.
  3. **Verification:** Run ontology load and confirm **ObjectProperty** and **DatatypeProperty** are populated as expected.

- **Files:** New `scripts/arango_rdf_controller.py` (or under a small `arango_rdf_controller` package); `scripts/load_data.py`.

**Recommendation:** A1 is the smallest change and matches PGT semantics; A2/A3 are only needed if you must keep separate **ObjectProperty** / **DatatypeProperty** collections.

---

## B. Ontology Annotations (adb:collection)

**Problem:** Ontology does not record how classes and properties map to ArangoDB collections.

**Goal:** Add `adb:collection` (and optionally `adb:key`) so the ontology is the single source of truth for the physical schema.

**Changes:**

1. **Ontology file** (`sentries_ontology.owl`):
   - Add ArangoDB namespace, e.g. `xmlns:adb="http://arangodb/"` (and ensure it is declared in the root element).
   - For each **class** used as vertex collection: add one triple per class, e.g.  
     `sentries:Person adb:collection "Person"`  
     Same for Entity, Organization, Vessel, Aircraft (values: `"Entity"`, `"Organization"`, `"Vessel"`, `"Aircraft"`).  
     Use the exact collection names that exist in ArangoDB.
   - For each **object property** (owned_by, family_member_of, leader_of, operates): add, e.g.  
     `sentries:owned_by adb:collection "owned_by"`  
     (if the implementation uses an edge collection named `owned_by`).  
     If you adopt Option A1, you may still annotate that the “vertex” representation of the property resource lives in **Property**, e.g. `adb:collection "Property"`, for documentation.
   - For each **datatype property** (riskScore, primaryName, inferredRisk, dataSource): add, e.g.  
     `sentries:riskScore adb:collection "Property"`  
     (or a dedicated collection name if you introduce one).
   - Optionally add `adb:key` for resources where you want to fix document keys (see ArangoRDF docs).

2. **Load path:** If we want PGT to **use** these annotations:
   - Either ensure the ontology is loaded as-is (with these triples) into the rdflib graph before `rdf_to_arangodb_by_pgt`, and that ArangoRDF reads `adb:collection` from that graph (see API: `adb_col_statements` can be built from the same graph), **or**
   - After adding annotations to the OWL file, run `write_adb_col_statements` once, export the result, and then use that as `adb_col_statements` in future runs (or merge those statements into the ontology file for a single source of truth).
   - **Implementation step:** Confirm in ArangoRDF whether `adb_col_statements` must be a separate graph or can be the same graph that contains the ontology (with adb triples). Then implement load_data.py accordingly.

3. **Documentation:** Short note in README or implementation_plan: “Physical schema mapping is recorded on the ontology via the ArangoDB namespace (adb:collection).”

**Files:** `sentries_ontology.owl`; optionally `scripts/load_data.py` if we pass `adb_col_statements` derived from the ontology; README or `docs/implementation_plan.md`.

**Dependency:** Can be done independently. If Option A2 is chosen, the same `adb:collection` values can be used in the custom `adb_col_statements` graph.

---

## C. OFAC / Synthetic Mapping Externalization

**Problem:** `collection_map`, `edge_map`, `class_map`, `weight_map` (and any similar maps) are hardcoded in `load_data.py`; no single external artifact for OFAC/synthetic → physical schema.

**Goal:** Externalize these mappings to a config file and load them in code.

**Changes:**

1. **New config file** (e.g. `config/schema_mapping.json` or `config/ofac_to_arango.yaml`):
   - **Party type → vertex collection:** e.g. `{"4": "Person", "3": "Organization", "1": "Vessel", "2": "Aircraft"}`.
   - **Relationship type → edge collection:** e.g. `{"15003": "owned_by", "15004": "family_member_of", "91725": "leader_of", "92019": "operates"}`.
   - **Vertex collection → ontology Class document _id** (for type edges): e.g. `{"Person": "Class/4254344209254636453", ...}`.  
     **Note:** These Class _ids are created by ArangoRDF at ontology load time; they may be hash-based and could change if the ontology or ArangoRDF version changes. Options: (a) document that this mapping must be updated after the first ontology load, or (b) have a small script that discovers Class _ids from the DB and writes them into the config.
   - **Edge collection → propagation weight:** e.g. `{"owned_by": 1.0, "leader_of": 0.8, "family_member_of": 0.5, "operates": 0.9}`.

2. **load_data.py:**
   - Load the config file (JSON or YAML) at startup.
   - Replace literal `collection_map`, `edge_map`, `class_map`, `weight_map` with the loaded dicts.
   - Keep the same variable names and logic so the rest of the script is unchanged.

3. **generate_test_data.py** (and any other scripts that create synthetic data):
   - Use the same config (or the same vertex/edge collection names and weights) so that schema and weights stay in one place. Either load the same file or import a shared constant list of collection names/weights.

4. **calculate_inferred_risk.py** (if it contains a hardcoded weight map):
   - Load propagation weights from the same config (or a shared module) so weights are defined once.

5. **Documentation:** README or implementation_plan: “OFAC and synthetic data mapping to the physical schema is defined in `config/schema_mapping.json` (or .yaml).”

**Files:** New `config/schema_mapping.json` (or `.yaml`); `scripts/load_data.py`; `scripts/generate_test_data.py`; `scripts/calculate_inferred_risk.py` (if applicable); README or docs.

**Optional:** Document in the ontology (e.g. custom annotation property) that a given class or relationship “is loaded from OFAC with party_type X” / “rel_type Y”; this would be for humans/tooling only unless we add code to read it.

---

## D. Ingest Documentation (and Optional arangoimport)

**Problem:** PRD says “load CSV data via arangoimport”; implementation uses Python only (csv + import_bulk).

**Goal:** Make documentation match reality and optionally support an arangoimport path.

**Changes (minimal – docs only):**

1. **PRD** (`docs/PRD.md`):
   - In “Week 3” (or equivalent): Change “load CSV data via arangoimport” to “load CSV data (e.g. via Python import_bulk)” or similar, and add one sentence that the current implementation uses Python for flexibility (type edges, propagation weights, single script).

2. **Implementation plan** (`docs/implementation_plan.md`):
   - Where it says “Import the flattened CSVs”, add “(current implementation: Python csv + import_bulk; optional arangoimport path can be added).”

3. **README** (`README.md`):
   - No change required if it already says “Load ontology and data” and points to `load_data.py`; optionally add “Data is loaded via Python (import_bulk) for type edges and propagation weights.”

**Changes (optional – add arangoimport path):**

4. **New script** (e.g. `scripts/import_csv_arangoimport.sh` or `.py` that shells out):
   - Assumes DB and collections already exist (after ArangoRDF + load_data ontology step, or after creating DataGraph collections manually).
   - Calls `arangoimport` for `data/parties.csv` into a temporary or staging collection (or one collection per party type if you define them), then optionally a second step for `data/relationships.csv`.
   - Document that `type` edges and `propagationWeight` are **not** set by this script; they would require a follow-up Python step or manual AQL. So “full” ingest remains the Python path unless we add that follow-up.

5. **README:** Add a short “Alternative: arangoimport” subsection that describes the optional script and its limitations.

**Files:** `docs/PRD.md`, `docs/implementation_plan.md`, `README.md`; optionally new `scripts/import_csv_arangoimport.sh` (or small Python wrapper) and README subsection.

---

## Implementation Order (suggested)

1. **D (docs only)** – Quick and non-invasive; aligns PRD with current behavior.
2. **A (choose A1, A2, or A3)** – Resolves ObjectProperty vs Property and avoids confusion in graph/themes/scripts.
3. **B (ontology annotations)** – Adds adb:collection (and optionally adb:key) so the ontology documents the physical schema; can be done in parallel with A if desired.
4. **C (externalize mapping)** – Reduces magic numbers and centralizes OFAC/synthetic mapping.
5. **D (optional arangoimport)** – Only if you want to offer an alternative ingest path.

---

## What to Decide Before Implementation

- **A:** Choose Option A1 (Property only), A2 (custom adb_col_statements), or A3 (custom controller). Recommendation: A1 unless you have a strong need for separate ObjectProperty/DatatypeProperty collections.
- **B:** Whether PGT should **use** the ontology’s adb:collection triples (e.g. by passing them as `adb_col_statements`) or only **document** them in the OWL file. If “use”, we need to confirm how ArangoRDF accepts adb_col_statements (same graph vs separate graph).
- **C:** Config format (JSON vs YAML) and exact path (e.g. `config/schema_mapping.json`). How to handle Class _ids (manual update after first load vs discovery script).
- **D:** Docs-only vs also adding an optional arangoimport script.

Once these are decided, implementation can proceed step by step as in this outline.
