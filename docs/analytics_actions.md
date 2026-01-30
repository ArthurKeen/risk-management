# Analytics Canvas Actions

## Overview

Analytics canvas actions have been added to **DataGraph** and **KnowledgeGraph** to enable tracing paths from selected entities to sanctioned entities.

## Installed Actions

### DataGraph
- **[Person] Trace to Sanctioned Entities**
- **[Organization] Trace to Sanctioned Entities**

### KnowledgeGraph
- **[Person] Trace to Sanctioned Entities**
- **[Organization] Trace to Sanctioned Entities**

## How They Work

### [Person] Trace to Sanctioned Entities
Traces paths from selected Person(s) to sanctioned entities (entities with `riskScore > 0`) via:
- **Ownership** (`owned_by`)
- **Leadership** (`leader_of`)
- **Family ties** (`family_member_of`)

**Query Logic:**
- Traverses 1-3 hops from the selected person
- Follows relationships in any direction
- Filters to only entities with `riskScore > 0`
- Returns up to 50 paths

### [Organization] Trace to Sanctioned Entities
Traces paths from selected Organization(s) to sanctioned entities via:
- **Ownership** (`owned_by`)
- **Leadership** (`leader_of`)
- **Operations** (`operates`)

**Query Logic:**
- Traverses 1-3 hops from the selected organization
- Follows relationships in any direction
- Filters to only entities with `riskScore > 0`
- Returns up to 50 paths

## Usage

1. **Select one or more Person or Organization nodes** in the graph
2. **Right-click** on a selected node
3. **Choose "Canvas Action"** from the context menu
4. **Select the appropriate analytics action:**
   - "[Person] Trace to Sanctioned Entities" for Person nodes
   - "[Organization] Trace to Sanctioned Entities" for Organization nodes
5. **View the results** - paths to sanctioned entities will be displayed on the canvas

## Use Cases

- **Risk Assessment**: Quickly identify if a person or organization has connections to sanctioned entities
- **Due Diligence**: Trace ownership chains to find hidden risk
- **Compliance**: Verify if business partners are linked to sanctioned parties
- **Investigation**: Discover multi-hop relationships that might not be obvious

## Technical Details

- **Traversal Depth**: 1-3 hops (configurable in query)
- **Result Limit**: 50 paths per action
- **Filter**: Only returns paths ending at entities with `riskScore > 0`
- **Performance**: Optimized for real-time analysis

## Future Enhancements

Potential improvements:
- Add risk score aggregation along paths
- Show path risk multipliers
- Filter by specific sanction lists
- Add visualization of risk propagation weights
- Support for tracing FROM sanctioned entities TO targets
