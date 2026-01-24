# Goal: Phase 1 - Bootstrap Risk Knowledge Graph

Initialize the Risk Management Knowledge Graph by processing the OFAC Advanced XML dataset, defining an ontology, and loading the data into ArangoDB.

## Proposed Changes

### 1. OWL Ontology Design
Create `sentries_ontology.owl` to define the domain-specific classes and relationships based on OFAC schema:
- **Classes:** `Person`, `Organization`, `Vessel`, `Aircraft`.
- **Properties:** `owned_by`, `family_member_of`, `leader_of`, `operates`.

### 2. Pre-processing Script [NEW]
Build `scripts/flatten_ofac.py` to:
- Parse `SDN_ADVANCED.XML` using `lxml` for memory efficiency.
- Extract `DistinctParty` (Vertices) and `ProfileRelationship` (Edges).
- Use dot-notation for nested attributes (e.g., `identity.primary_name`).
- Output files: `parties.csv`, `relationships.csv`.

### 3. Data Loading [NEW]
Create `scripts/load_data.py` using `ArangoRDF` to:
- Map OWL classes to ArangoDB vertex collections.
- Map OWL properties to ArangoDB edge collections.
- Import the flattened CSVs.

### 5. Multi-Graph Definition [NEW]
Refine the ArangoDB graph definitions:
- **`OntologyGraph`**: Includes `Class`, `Property`, `Ontology` vertices and `domain`, `range`, `subClassOf`, `type` edges.
- **`DataGraph`**: Includes `Person`, `Organization`, `Vessel`, `Aircraft` vertices and domain-specific relationships (`owned_by`, etc.).
- **`KnowledgeGraph`**: Combines both Ontology and Data graphs.
- Update `scripts/load_data.py` to create these graphs and `scripts/install_theme.py` to register themes for all relevant graphs.

# Goal: Phase 2 - Risk Analytics & Logic

Build the computational engine for threat detection and risk propagation.

## Proposed Changes

### 1. Direct Risk Scoring [NEW]
Build `scripts/calculate_direct_risk.py` to:
- Assign base risk scores to entities based on specific metadata (e.g., sanction tags).
- Normalize scores between 0.0 and 1.0.

### 2. Inferred Risk Engine (AQL Path-Based) [NEW]
Implement native ArangoDB graph traversals to:
- Compute all paths from an entity to sanctioned nodes.
- Take the product of weights on edges (Ownership: 1.0, Leadership: 0.8, Family: 0.5).
- Multiply by base risk scores on vertices.
- Sum the results to produce a unified `inferredRisk` score.
- Save these as views or stored procedures for real-time analysis.

### 3. Visual Analysis [NEW]
- Save analytic queries to `_editor_saved_queries` for browser-based exploration.
- Enhance themes to visually distinguish inferred vs direct risk.

# Goal: Phase 3 - Analytics & Advanced Discovery

Transform the Knowledge Graph into an interactive risk intelligence platform by surfacing expert queries in the ArangoDB UI.

## Proposed Changes

### 1. Expert Query Library [NEW]
Save the following queries to `_editor_saved_queries`:
- **Sentries - Dynamic Vetting**: Vets a specific `@entityID` using the AQL path-based model.
- **Sentries - Top Threats**: Lists the top 20 High-Risk Organizations based on total exposure.
- **Sentries - Risk Concentration**: Identifies owners with the highest aggregate risk across their portfolio.

### 2. Implementation Script [NEW]
Create `scripts/install_dashboard.py` to automate the registration of these queries.

## Verification Plan

### Manual Verification
- Log into ArangoDB UI.
- Navigate to "Queries" -> "Saved Queries" and execute "Sentries - Dynamic Vetting".
