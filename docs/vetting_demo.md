# Vetting Demonstration: Transitive Risk Assessment

This document outlines the demonstration scenarios for assessing the transitive risk of individual Organizations and Persons by associating them with the OFAC sanctioned dataset.

## Overview

To demonstrate the power of graph-based risk propagation, we use a dataset of "Vetting Targets" (Organizations and Persons) that are linked to known high-risk entities from the OFAC dataset. This allows us to show how association affects risk scores, even when an entity is not explicitly listed on a sanctions list.

## Setup Instructions

To reset and run the vetting demonstration:

1. **Tag Existing Data**: Ensure all OFAC data is tagged (handled automatically by the pipeline).
2. **Generate Demonstration Data**:
   ```bash
   python3 scripts/generate_test_data.py
   ```
3. **Run Risk Propagation**:
   ```bash
   python3 scripts/calculate_inferred_risk.py
   ```

## Demonstration Scenarios

We have established deterministic scenarios to show different paths of risk inheritance:

### 1. Direct Ownership
An Organization is directly owned by a sanctioned OFAC Organization.
- **Inheritance**: 100% of owner's risk.
- **Expected Result**: 1.0

### 2. Indirect Ownership (Multi-hop)
An Organization is owned by a Holding Company, which is in turn owned by a sanctioned OFAC Organization.
- **Path**: `Sanctioned Org` -> `Holding Co` -> `Target Org`
- **Inheritance**: 100% risk propagated through the chain.
- **Expected Result**: 1.0

### 6. Deep Multi-hop (3 Hops) - The "Shell Game"
An Organization is buried under two layers of holding companies, starting from a sanctioned OFAC Organization.
- **Path**: `Sanctioned Org` -> `Layer 1` -> `Layer 2` -> `Target Org`
- **Inheritance**: 100% risk maintained throughout.
- **Expected Result**: 1.0

### 3. Sanctioned Leadership
An Organization is led by a sanctioned OFAC Person.
- **Inheritance**: 80% of leader's risk.
- **Expected Result**: 0.8

### 4. Family Association (Indirect)
A sanctioned OFAC Person is a family member of a Person, who then owns an Organization.
- **Inheritance**: 50% risk via family tie, then 100% via ownership.
- **Expected Result**: 0.5

### 5. Clean Entity
An Organization with no associations to the OFAC dataset.
- **Inheritance**: 0%.
- **Expected Result**: 0.0

## Results Summary

| Vetting Target | Association Type | Hops from OFAC | Inferred Risk |
| :--- | :--- | :--- | :--- |
| **Vetting Target - 3-Hop Link** | Deep Multi-hop Ownership | **3** | **1.0** |
| **Vetting Target - Indirect Link** | Indirect Ownership | **2** | **1.0** |
| **Vetting Target - Family Link** | Owned by Relative of OFAC | **2** | **0.5** |
| **Vetting Target - Leadership Link**| Led by OFAC Person | 1 | 0.8 |
| **Vetting Target - Direct Link** | Owned by OFAC Org | 1 | 1.0 |

## Visual Identification in ArangoDB

When using the ArangoDB Visualizer:

- **Nodes**: Vetting targets are visually distinct (Purple color or "Vial" icon).
- **Edges**: Investigative relationships are rendered with **dashed lines**, making it easy to distinguish them from official OFAC data.
- **Data Source**: Every node and edge has a `dataSource` property (either `OFAC` or `Synthetic`).
