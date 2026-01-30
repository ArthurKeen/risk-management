# ArangoRDF Usage and Schema Mapping Analysis Report

**Project:** Sentries Risk Management Knowledge Graph  
**Date:** January 2025  
**Scope:** ArangoRDF usage, ontology-to-ArangoDB mapping, OFAC/synthetic data mapping, and ingest approach.

---

## 1. Executive Summary

This report analyzes how ArangoRDF is used in the project, why object properties are not appearing in the ObjectProperty collection, whether ontology-to-ArangoDB mappings are recorded on the ontology, how OFAC and synthetic data mappings are recorded, and whether ingest uses `arangoimport` as specified in the PRD.

**Findings in brief:**
- ArangoRDF is used only for ontology load via PGT; data load is Python-only.
- Object properties are stored in the **Property** vertex collection by ArangoRDF PGT, not in **ObjectProperty**; the ontology graph definition expects both.
- No ArangoDB namespace annotations (`adb:collection`) are used on the ontology.
- OFAC and synthetic mappings are hardcoded in Python; they are not recorded in the ontology or in a separate mapping artifact.
- Ingest uses Python (`import_bulk`) only; there are no `arangoimport` scripts.

---

## 2. ArangoRDF Usage

### 2.1 How ArangoRDF Is Used

**Location:** `scripts/load_data.py`

- The ontology file `sentries_ontology.owl` is parsed into an rdflib `Graph`.
- ArangoRDF is initialized: `adp = ArangoRDF(db)`.
- The only ArangoRDF call is:
  ```python
  adp.rdf_to_arangodb_by_pgt(
      name="OntologyGraph",
      rdf_graph=g
  )
  ```
- So ArangoRDF is used **only** to create and populate the ontology graph (OntologyGraph) via the Property Graph Transformation (PGT) algorithm. It is **not** used for OFAC or synthetic data.

### 2.2 ObjectProperty Collection and PGT Behavior

**Observation:** Object properties (e.g. `owned_by`, `family_member_of`) are not present in the **ObjectProperty** collection.

**Cause (ArangoRDF PGT semantics):**

From the [ArangoRDF API specification](https://arangordf.readthedocs.io/en/latest/specs.html):

- For **RDF-to-ArangoDB (PGT)**:
  - *"Contrary to RPT, regardless of whether contextualize_graph is set to True or not, **all RDF Predicates within every RDF Statement in rdf_graph will be processed as their own ArangoDB Document, and will be stored under the 'Property' Vertex Collection.**"*

So in PGT, **every** predicate (including OWL object and datatype properties) is stored as a vertex in the **Property** collection. There is no separate **ObjectProperty** (or **DatatypeProperty**) vertex collection populated by ArangoRDF.

The ontology defines `owl:ObjectProperty` and `owl:DatatypeProperty` and resources like `sentries:owned_by` with `rdf:type owl:ObjectProperty`. The PGT collection mapping could in principle use `rdf:type` to assign collections, but the documented PGT behavior is to put all predicate resources into **Property**. The project’s graph definition and scripts (e.g. `load_data.py`, `check_cross_edges.py`, theme configs) reference an **ObjectProperty** vertex collection and a **DatatypeProperty**-style distinction, but ArangoRDF PGT does not populate **ObjectProperty** (or a separate DatatypeProperty collection); it uses **Property** for all such resources.

**Conclusion:** Object properties are not “missing” from the schema; they are stored in **Property**. The **ObjectProperty** collection exists in the graph definition but is not filled by ArangoRDF PGT, so it remains empty unless populated by some other mechanism.

### 2.3 Recommendations (ArangoRDF)

1. **Align with PGT:** Treat **Property** as the single vertex collection for all ontology properties (object and datatype) and update OntologyGraph definition, themes, and scripts to use **Property** instead of **ObjectProperty** (and any separate DatatypeProperty collection), **or**
2. **Custom mapping:** Use ArangoRDF’s optional `adb_col_statements` (see Section 3) to supply a custom RDF graph that maps specific resources to **ObjectProperty** / **DatatypeProperty** collections and pass it into `rdf_to_arangodb_by_pgt`, **or**
3. **Override controller:** Implement a custom `ArangoRDFController` (e.g. `identify_best_class`) so that resources with `rdf:type owl:ObjectProperty` are assigned to an **ObjectProperty** collection and similarly for **DatatypeProperty**, and use that controller when calling ArangoRDF.

---

## 3. Ontology-to-ArangoDB Mapping and Annotations

### 3.1 ArangoDB Namespace and `adb:collection`

ArangoRDF supports explicit collection mapping via RDF statements:

- **Predicate:** `http://arangodb/collection` (referred to here as `adb:collection`).
- **Usage:** Statements of the form  
  `resource adb:collection "CollectionName"`  
  specify the ArangoDB vertex (or edge) collection for that resource.

From the API:

- `rdf_to_arangodb_by_pgt(..., adb_col_statements=Graph(...))`: *"An optional RDF Graph containing ArangoDB Collection statements of the form adb_vertex http://arangodb/collection \"adb_v_col\" .. Useful for creating a custom ArangoDB Collection mapping of RDF Resources within rdf_graph."*
- `write_adb_col_statements`: writes the mapping that PGT would use into an RDF graph so it can be inspected or edited.
- `extract_adb_col_statements`: extracts `adb:collection` statements from an RDF graph.

So the “property in the ArangoDB namespace” that allows annotating how a class or property is implemented in the physical schema is **`http://arangodb/collection`** used as a predicate with a literal collection name as object.

### 3.2 Current Ontology

**File:** `sentries_ontology.owl`

- **Namespaces declared:**  
  `owl`, `rdf`, `rdfs`, `xsd`, and the sentries default namespace.  
  **No** `xmlns` or prefix for `http://arangodb/...` (ArangoDB namespace).
- **Annotations on classes/properties:**  
  Only standard OWL/RDFS (e.g. `rdfs:domain`, `rdfs:range`, `rdf:type`, `rdfs:subClassOf`).  
  **No** `adb:collection` (or `adb:key`) statements.

So the mapping from ontology classes and properties to ArangoDB collections is **not** recorded on the ontology. It is entirely implied by ArangoRDF’s default PGT collection mapping (and, for data, by hardcoded Python maps; see Section 4).

### 3.3 Recommendations (ontology annotations)

1. **Add ArangoDB namespace** to the ontology, e.g.  
   `xmlns:adb="http://arangodb/"`.
2. **Annotate each class** with its vertex collection, e.g.  
   `sentries:Person adb:collection "Person"`  
   and similarly for Organization, Vessel, Aircraft, Entity.
3. **Annotate each object/datatype property** with the intended vertex or edge collection, e.g.  
   `sentries:owned_by adb:collection "owned_by"`  
   (if it is represented as an edge collection), or  
   `sentries:owned_by adb:collection "ObjectProperty"`  
   if the goal is to have object-property resources in an ObjectProperty vertex collection (and then use `adb_col_statements` in PGT or a custom controller to respect it).
4. **Optionally** use `write_adb_col_statements` after a test PGT run to generate a first draft of `adb:collection` statements, then add them to the ontology and maintain them there as the single source of truth for the physical schema.

---

## 4. OFAC and Synthetic Data Mapping to the Physical Schema

### 4.1 How Mapping Is Done Today

**OFAC (and CSV) → ArangoDB:**

- **Script:** `scripts/load_data.py`
- **Source:** `data/parties.csv`, `data/relationships.csv` (produced by `scripts/flatten_ofac.py` from `data/SDN_ADVANCED.XML`).
- **Mapping:** Implemented only in Python, via dicts:

  **Party type → vertex collection**
  ```python
  collection_map = {
      "4": "Person",
      "3": "Organization",
      "1": "Vessel",
      "2": "Aircraft"
  }
  ```

  **Relationship type → edge collection**
  ```python
  edge_map = {
      "15003": "owned_by",
      "15004": "family_member_of",
      "91725": "leader_of",
      "92019": "operates"
  }
  ```

  **Vertex collection → ontology Class document `_id` (for `type` edges)**
  ```python
  class_map = {
      "Person": "Class/4254344209254636453",
      "Organization": "Class/2686369784577023745",
      ...
  }
  ```

  **Edge collection → propagation weight**
  ```python
  weight_map = {
      "owned_by": 1.0,
      "leader_of": 0.8,
      "family_member_of": 0.5,
      "operates": 0.9
  }
  ```

**Synthetic data:**

- **Script:** `scripts/generate_test_data.py`
- **Mapping:** Same physical schema (Person, Organization, owned_by, leader_of, family_member_of, etc.). No separate mapping file; the script uses AQL and the same collection and edge names.

### 4.2 Where It Is Recorded

- **Not in the ontology:** The ontology does not reference OFAC subtypes, relationship type IDs, or ArangoDB collection names.
- **Not in a separate mapping document:** There is no JSON/YAML/RDF file that describes `party_type` → collection, `rel_type` → edge, or Class `_id`s.
- **Only in code:** All of the above mappings exist only as literals in `load_data.py` (and, for propagation weights, in `calculate_inferred_risk.py` or similar). CSV column names and semantics are documented in comments and in `docs/implementation_plan.md` but not in a machine-readable mapping artifact.

### 4.3 Recommendations (OFAC/synthetic mapping)

1. **Externalize mapping:** Move `collection_map`, `edge_map`, `class_map`, and `weight_map` into a config file (e.g. `config/schema_mapping.json` or `config/ofac_to_arango.yaml`) and load it in `load_data.py`.
2. **Document in ontology (optional):** If desired, use custom annotation properties (e.g. in a project namespace) to record which ArangoDB vertex/edge collection implements each class/relationship type; this would be for documentation and tooling, not for ArangoRDF unless you add support to read those annotations in a custom controller.
3. **Single source of truth:** Use the same config (or ontology annotations) for both OFAC ingest and synthetic data generation so that schema changes are made in one place.

---

## 5. Ingest: arangoimport vs Python

### 5.1 PRD and Implementation Plan

- **PRD (Week 3):** *"Use **ArangoRDF** to initialize the physical collections and load CSV data **via arangoimport**."*
- **Implementation plan:** *"Import the flattened CSVs."* (no explicit tool.)

### 5.2 Actual Implementation

- **Flattening:** `scripts/flatten_ofac.py` writes `data/parties.csv` and `data/relationships.csv` (with headers `party_id`, `primary_name`, `party_type` and `rel_id`, `from_party`, `to_party`, `rel_type`).
- **Loading:** `scripts/load_data.py`:
  - Uses Python’s `csv` module to read the CSVs.
  - Builds in-memory documents and edge lists.
  - Uses `db.collection(col_name).import_bulk(docs, overwrite=True)` for vertices and edges.
- **No** `arangoimport` or `arangosh` calls; no shell scripts that wrap `arangoimport`; no references to `arangoimport` in the codebase except in the PRD (and the PRD’s general “dot notation” note for compatibility with `arangoimport`).

So ingest is **fully Python-based** (CSV read + `import_bulk`), not `arangoimport`-based.

### 5.3 Implications

- **Pros of current approach:** Single script, no shell/exec dependency, easy to add `type` edges and propagation weights and to reuse the same code path for synthetic data.
- **Gap vs PRD:** The PRD’s “load CSV data via arangoimport” is not implemented.

### 5.4 Recommendations (ingest)

1. **Update PRD/docs:** Change the wording to “load CSV data (e.g. via Python `import_bulk`)” or similar, and add a short note that the current implementation uses Python for flexibility (type edges, weights, multi-step pipeline).
2. **Optional arangoimport path:** If you want to align with the PRD literally, add a small script that:
   - Runs after ArangoRDF (so collections exist),
   - Calls `arangoimport` for `parties.csv` and `relationships.csv` into the appropriate collections,
   - Then a separate Python step could add `type` edges and `propagationWeight` if needed, or document that these are only added when using the Python path.
3. **Keep Python as primary:** Retain the current Python ingest as the main path and document it as the reference implementation for this project.

---

## 6. Summary Table

| Topic | Current state | PRD / intended | Recommendation |
|-------|----------------|----------------|-----------------|
| **ArangoRDF usage** | Used only for ontology load via PGT | Use ArangoRDF to initialize physical collections | Keep; optionally use `adb_col_statements` or custom controller for ObjectProperty/DatatypeProperty. |
| **ObjectProperty collection** | Empty; PGT puts all predicates in **Property** | (Implicit) object properties in ObjectProperty | Use **Property** as source of truth, or add custom mapping/controller to populate **ObjectProperty**. |
| **Ontology ↔ ArangoDB mapping** | Not on ontology; inferred by PGT | (Your goal) Record on ontology | Add `adb:collection` (and optionally `adb:key`) in ontology; use ArangoDB namespace. |
| **OFAC/synthetic → schema** | Hardcoded in `load_data.py` | — | Externalize to config; optionally document in ontology. |
| **Ingest tool** | Python `csv` + `import_bulk` | `arangoimport` | Update PRD to match implementation; optionally add optional `arangoimport` path. |

---

## 7. References

- ArangoRDF docs: https://arangordf.readthedocs.io/en/latest/
- PGT collection mapping: https://arangordf.readthedocs.io/en/latest/rdf_to_arangodb_pgt.html
- ArangoRDF API (e.g. `adb_col_statements`, `write_adb_col_statements`): https://arangordf.readthedocs.io/en/latest/specs.html
- Project: `scripts/load_data.py`, `scripts/flatten_ofac.py`, `sentries_ontology.owl`, `docs/PRD.md`, `docs/implementation_plan.md`
