# stamp-erd

**Sample Tracking Platform Overview**

NOTE: stamp_erd_dbdiagram.txt is written in Database Markup Language (DBML) and can be pasted into dbdiagram.io website app (free) to render an interactive ERD. I suggest we use [dbdiagram.io](https://dbdiagram.io) to make updates and changes and to visualize the ERD, and then copy the code from dbdiagram.io and commit it overtop of stamp_erd_dbdiagram.txt file in this repo for version control.

## 1. Foundational Standards & Translation Layer

**To ensure long‑term interoperability and leverage broad community support, our core model is anchored in internationally recognized standards rather than bespoke DFO systems:**

- **Darwin Core Data Package (DwC‑DP):** Provides a well‑documented publishing model for Events, Occurrences, MaterialEntities, Identifications, etc. (see [DwC‑DP Quick Reference](https://gbif.github.io/dwc-dp/qrg/index.html)).
- **Darwin Core Conceptual Model:** Clarifies relationships between core classes (Event, Occurrence, Organism, MaterialEntity, Protocol, Agent, Reference, etc.) and underpins DwC‑DP semantics.
- **Dublin Core / DCMI:** Powers general-purpose metadata (e.g., citation, description fields).
- **OBO Foundry Ontologies (OBI, ENVO):** For laboratory protocols, environmental context, and material preservation semantics via URIs.

**Why a Translation Layer?**  
DFO's legacy fish‑data systems use inconsistent modelling approaches to link various concepts. Furthermore field names are often highly bespoke, and acceptable values (enums) are rarely defined using strictly controlled controlled vocabulary. 

The goal of aligning with the DwC-DP model is to use a highly flexible and extremely well thought out set of concepts and how they relate to make it easier to integrate different approaches found in DFO for modelling data collection and lab analyses. The goal is not to strictly adhere to the concept and term names used in DwC-DP when presenting materialized views to users, but to use the concepts and terms to map disparate schemas to. **By doing so we:**

- Establish a single, well-defined schema that captures full provenance—from field collection through lab analysis—so scientists can trace each sample’s lineage and validate results.  
- Provide consistent table and field names with explicit foreign-key relationships, making ETL pipelines and SQL queries straightforward for developers while delivering the contextual joins researchers need.  
- Embed controlled vocabularies and standard identifiers (DwC-DP terms, OBO URIs) to ensure data is interoperable, discoverable, and reusable across projects and teams.  
- Normalize disparate legacy and lab systems into a unified model, enabling seamless cross-study and multi-disciplinary analyses (e.g., linking environmental covariates with biological measurements).  
- Preserve event, occurrence, and material metadata in structured form to support reproducible research, audit trails, and adherence to FAIR data principles.  

---

## 2. Core Concepts & Tables

**Below are the primary tables in the ERD, how they map to DwC‑DP, and why they matter for fisheries sample tracking:**
2. Core Concepts & Tables  
Below are the primary tables/concepts in the ERD, how they map to DwC-DP terms, and why they matter for fisheries sample tracking:

| Concept (Table)                 | DwC-DP Concept (UpperCamelCase) or Terms (lowerCamelCase)                                       | Purpose & Workflow Role                                                                                                                                                                                                                                                               |
|-------------------------|-----------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Project**             | `projectID`, `projectTitle`, `fundingAttributionID` | Captures the high-level “Project” at the parent event (e.g., trip) level, linking all related child events, occurrences, and samples.                                                                                                                                                |
| **Trips & Actions**     | `dwc:Event`                                         | Models everything from “trip” and “haul” to preservation and shipment as Events with `eventType` (e.g., MaterialGathering, Preservation, Shipment). Use `parent_event_id` to nest sites under trips. Capture metadata, methods and external IDs via `event_assertion`, `event_protocol`, and `event_identifier`. |
| **Catch (SampleGroup)** | `dwc:Event` (`eventType` = catch),`dwc:Occurrence`, <br>`dwc:Organism`                | Catches are events. `occurrenceID` rows are counts of each species in a catch, ie. represent each species (e.g., “20 Chinook”, “5 Coho” as seperate rows with `organismQuantity` = 20 or 5 and `organismQuantityType` = Count). Each 'occurrenceID` links to the catch `eventID`. Link each `occurrenceID` to an `organism` for individual-level metadata (sex, length, weight etc), which effectively separates total catch from the subset you actually sample.                               |
| **Specimens/Samples**           | `dwc:MaterialEntity`                                 | Represents each tissue sample taken from the fish for further analysis as a `MaterialEntity`, link sub-samples with whole samples (eg. splitting a liver in half) with `parentMaterialEntityID`. Link samples to the whole fish via `organismID` and catch via `occurrence_id` and `eventID`.                                                                                  |
| **Lab results**         | `dwc:MaterialAssertion`                             | Attaches lab measurements/analysis (genetic assignments, age etc.) to material records via MaterialAssertion linked to `material_entity_id`.                                                                                                                                    |
| **Container**           | *Not a native DwC-DP class*                         | Physical sample containers (vials, envelopes, scale-books) are managed via **material_identifier** records on MaterialEntity (use `identifierType = "containerID"`) or with a `Container` concept table with `containerType` (vial, 96-well box, binder), `containerPurpose` attributes (archive, shipping etc.).                                                                                                                  |
| **Shipment**            | `dwc:Event` (`eventType = "Shipment"`)              | Captures sample dispatch to testing labs as specialized Events.                                                                                                                                                                                                                       |
| **Protocol**            | `dwc:Protocol`                                      | References standard methods (sampling, preservation, molecular) by URI where available; linked to Events or Analyses via `Protocol` table.                                                                                                                             |
| **Agent**               | `dwc:Agent`                                         | Represents people or organizations (collectors, lab techs); linked via `event_agent_role`, `occurrence_agent_role`, etc.                                                                                                                                                            |
| **Identification**      | `dwc:Identification`                                | Captures taxonomic determinations (field ID vs. DNA barcoding), linked to Occurrence or Material via `identificationID`, `identifiedByID`.                                                                                                                                            |                                                                                                                                                            |
| **Relationship**        | `dwc:Relationship`                                  | Provides a generic linking mechanism for edge cases.                                                                                                                                                                                 |

---

## 3. Workflow Coverage

**We cross-walked six major workflow phases against our model. Here's a quick audit:**

| **Phase**               | **Key Tables**                                        | **Supported?**                                                                                                                                                                                                   |
|-------------------------|-------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Project & Acquisition   | Event                                                 | ✔ `projectID`, `projectTitle`, `fundingAttributionID` on Event; ties to external project codes.                                                                                                                 |
| Fish Capture & Biodata  | Event, Occurrence, Organism                           | ✔ Occurrences per species (counts, `organismQuantity`, `lifeStage`, `sex`) link back to Organism for individual metadata (length, weight, etc.).                                                                  |
| Tissue Collection       | MaterialEntity, material_identifier                   | ✔ `MaterialEntity` rows for each specimen and subsample (`parentMaterialEntityID`); container tracking via `material_identifier` (`identifierType = "containerID"`).                                               |
| QC & Preservation       | MaterialEntity, Protocol                              | ✔ Preservation metadata (`condition`, `disposition`) on MaterialEntity; preservation methods linked via `Protocol` (`protocolType = "preservation"`).                                                             |
| Shipment to Lab         | Event (`eventType = "Shipment"`), event_identifier, Agent | ✔ Sample dispatch as Events (`eventType="Shipment"`); external shipment IDs via `event_identifier`; sending `Agent` roles; materials link back via `derivation_event_id`.                                        |
| Lab Analysis & Results  | Identification, Protocol, MaterialAssertion           | ✔ Taxonomic determinations via `Identification`; analysis methods via `Protocol`; lab measurements captured in `MaterialAssertion` (`assertionType="resultsAvailable"`, analyte values, priority flags, etc.). |

