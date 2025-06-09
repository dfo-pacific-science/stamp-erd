# stamp-erd

**Sample Tracking Platform Overview**

NOTE: stamp_erd_dbdiagram.txt can be pasted into dbdiagram.io website app (free) to render an interactive ERD. I suggest we use [dbdiagram.io](https://dbdiagrm.io) to make updates and changes and to visualize the ERD, and then copy the code from dbdiagram.io and commit it overtop of stamp_erd_dbdiagram.txt file in this repo for version control.

STAMP.png is the erd exported from dbdiagram.io and can be replaced and updated to follow the above workflow.

## 1. Foundational Standards & Translation Layer

**To ensure long‑term interoperability and leverage broad community support, our core model is anchored in internationally recognized standards rather than bespoke DFO systems:**

- **Darwin Core Data Package (DwC‑DP):** Provides a well‑documented publishing model for Events, Occurrences, MaterialEntities, Identifications, etc. (see [DwC‑DP Quick Reference](https://gbif.github.io/dwc-dp/qrg/index.html)).
- **Darwin Core Conceptual Model:** Clarifies relationships between core classes (Event, Occurrence, Organism, MaterialEntity, Protocol, Agent, Reference, etc.) and underpins DwC‑DP semantics.
- **Dublin Core / DCMI:** Powers general-purpose metadata (e.g., citation, description fields).
- **OBO Foundry Ontologies (OBI, ENVO):** For laboratory protocols, environmental context, and material preservation semantics via URIs.

**Why a Translation Layer?**  
DFO's legacy fish‑data systems use inconsistent field names, acronyms, and table structures. By mapping them into a clean, standards‑based schema:

1. **We avoid lock‑in to any specific tool.**
2. **We make downstream data publishing, reporting, and integration far easier.**
3. **We future‑proof against evolving community vocabularies.**

---

## 2. Core Concepts & Tables

**Below are the primary tables in the ERD, how they map to DwC‑DP, and why they matter for fisheries sample tracking:**

| **Table**        | **DwC‑DP Class**     | **Purpose & Workflow Role** |
|------------------|----------------------|------------------------------|
| Project          | `dwc:Dataset`        | Top‑level logical dataset (annual program or research project). |
| Event            | `dwc:Event`          | Any action at time/place: field collection, preservation, shipment. |
| Organism         | `dwc:Organism`       | A fish or aggregation of fish involved in an Event. |
| Occurrence       | `dwc:Occurrence`     | A catch record — counts, status, lifeStage, sex, etc., per Event. |
| MaterialEntity   | `dwc:MaterialEntity` | Physical samples (scales, tissue, otoliths), tracking preservation & lineage. |
| Container        | Compliance           | Holds multiple MaterialEntities; labelled vials, envelopes, scale-books. |
| Shipment         | `dwc:Event`*         | Lab submission; a specialized Event subclass for transport to testing labs. |
| Protocol         | `dwc:Protocol`       | Standard methods (sampling, preservation, molecular) linked by URI. |
| Agent            | `dwc:Agent`          | People or organizations (collectors, lab techs). |
| Identification   | `dwc:Identification` | Taxonomic determinations, e.g. field ID vs. DNA barcoding. |
| Reference        | `dcmi:Reference`     | Publications, SOPs, scale books, lab manuals. |
| Relationship     | `dwc:Relationship`   | Generic link for edge cases (partOf, derivedFrom, subsampleOf). |

\*Technically DwC‑DP models shipments as Events with `eventType = "Shipment"`.

---

## 3. Workflow Coverage

**We cross‑walked eight major workflow phases against our model. Here's a quick audit:**

| **Phase**               | **Key Tables**               | **Supported?** |
|-------------------------|------------------------------|----------------|
| Project & Acquisition   | Project, Event               | ✔ `projectID` on Event; ties to non‑DwC DFO project codes. |
| Kit Preparation & Labelling | Container, Protocol     | ✔ `ContainerType`, label URIs; `protocolID` on Event presets kit SOP. |
| Fish Capture & Biodata  | Event, Organism, Occurrence  | ✔ `Occurrence` ties catch counts + lifeStage/sex back to `Organism`. |
| Tissue Collection       | MaterialEntity               | ✔ `type = "tissue"`, vials via `Container`, `hasDNA` flag. |
| QC & Preservation       | MaterialEntity.condition     | ✔ `condition`, `disposition`, `preservationProtocolID`. |
| Shipment to Lab         | Shipment                     | ✔ Links to `Agent`, retains `eventID` context. |
| Lab Analysis & Identification | Identification, Protocol | ✔ DNA vs. scale ID via `identificationType` + `taxonAssignmentMethod`. |
