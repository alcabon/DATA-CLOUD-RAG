
```mermaid
flowchart TB

  subgraph SOURCES["Connectors / Streams"]
    DSO["DSO<br/>(transient schema)"]
  end

  subgraph LAKE["Data Lake — materialized"]
    DLO_CRM["DLO: Individual — raw<br/>(Salesforce CRM)"]
    DLO_S3["DLO: Customer — raw<br/>(AWS S3)"]
    DLO_SVC["DLO: Contact — raw<br/>(Service Cloud)"]
    DLO_KAV["DLO: Knowledge__kav — raw<br/>(all fields, all records)"]
    DLO_KAV_F["DLO: Knowledge__kav — filtered<br/>(PublishStatus = Online)"]
    UDLO["UDLO<br/>(unstructured file references)"]
  end

  subgraph MODEL["Data Model — virtual views"]
    DMO_IND["DMO: Individual"]
    DMO_CP["DMO: Contact Point Email"]
    DMO_KAV["DMO: Knowledge Article"]
    UDMO_X["UDMO: unstructured content"]
  end

  subgraph VECTOR["Search and Vector — materialized"]
    CDMO["CDMO<br/>(chunked text)"]
    IDMO["IDMO<br/>(vector embeddings)"]
  end

  subgraph IR["Identity Resolution — materialized / hybrid"]
    LINK_IND["Unified Link Individual<br/>(crosswalk)"]
    UNIFIED_IND["Unified Individual<br/>(hybrid)"]
    LINK_CP["Unified Link Contact Point<br/>(crosswalk)"]
    UNIFIED_CP["Unified Contact Point<br/>(hybrid)"]
  end

  subgraph DERIVED["Derived Outputs — materialized"]
    CIO["Calculated Insight Object"]
    SEG["Segment Membership Table"]
  end

  subgraph LEGEND[" Legend "]
    direction LR
    L1["Materialized<br/>(real storage)"] ~~~ L2["Hybrid<br/>(view over materialized data)"] ~~~ L3["Virtual view<br/>(no storage)"] ~~~ L4["Transient<br/>(momentary)"]
  end

  DSO -. ingest .-> DLO_CRM
  DSO -. ingest .-> DLO_S3
  DSO -. ingest .-> DLO_SVC
  DSO -. ingest .-> DLO_KAV

  DLO_CRM -->|mapping| DMO_IND
  DLO_S3 -->|mapping| DMO_IND
  DLO_SVC -->|mapping| DMO_IND
  DLO_SVC -->|mapping| DMO_CP

  DLO_KAV -->|Data Transform| DLO_KAV_F
  DLO_KAV_F -->|mapping| DMO_KAV

  UDLO -->|mapping| UDMO_X

  DMO_KAV -->|Search Index Config| CDMO
  UDMO_X -->|Search Index Config| CDMO
  CDMO -->|embedding model| IDMO

  DMO_IND -->|Identity Resolution| LINK_IND --> UNIFIED_IND
  DMO_CP -->|Identity Resolution| LINK_CP --> UNIFIED_CP

  DMO_IND -->|scheduled SQL| CIO
  UNIFIED_IND -->|segment publish| SEG
  CIO -->|segment criteria| SEG

  SEG ~~~ LEGEND
  IDMO ~~~ LEGEND

  classDef materialized fill:#1B3A5C,color:#ffffff,stroke:#2E75B6,stroke-width:1px
  classDef hybrid fill:#5B8FB9,color:#ffffff,stroke:#2E75B6,stroke-width:1px,stroke-dasharray: 4 3
  classDef virtual fill:#EAF1F8,color:#1B3A5C,stroke:#2E75B6,stroke-width:1px,stroke-dasharray: 4 3
  classDef transient fill:#ffffff,color:#888888,stroke:#999999,stroke-width:1px,stroke-dasharray: 2 2

  class DLO_CRM,DLO_S3,DLO_SVC,DLO_KAV,DLO_KAV_F,UDLO,CDMO,IDMO,LINK_IND,LINK_CP,CIO,SEG materialized
  class UNIFIED_IND,UNIFIED_CP hybrid
  class DMO_IND,DMO_CP,DMO_KAV,UDMO_X virtual
  class DSO transient
  class L1 materialized
  class L2 hybrid
  class L3 virtual
  class L4 transient

```

Done — added right where it belongs:

**Core objects (updated)**

| Term | Full name | Status | What it holds |
|---|---|---|---|
| **DSO** | Data Source Object | Transient | The schema definition for a connector. Data passes through it but never persists here — it lands directly in the raw DLO |
| **DLO** | Data Lake Object | Materialized | Physical storage (Parquet files, effectively) holding either raw ingested data or the output of a Data Transform |
| **UDLO** | Unstructured Data Lake Object | Materialized | References to unstructured files (PDFs, docs, transcripts) on blob storage — one row per *file*, not per record |
| **DMO** | Data Model Object | Virtual view | The canonical/custom structure unifying one or more DLOs. No storage of its own — queries push down to whichever DLO(s) feed it |
| **UDMO** | Unstructured Data Model Object | Virtual view | Same role as DMO, but for unstructured content — the structural layer over a UDLO, used as the basis for chunking and embedding |
| **CDMO** | Chunk DMO | Materialized | The actual chunked text pieces, produced by a Search Index Configuration |
| **IDMO** | Index DMO | Materialized | The vector embeddings, one per chunk — this is the literal vector store |


| Term | Status | What it holds |
|---|---|---|
| **CIO** (Calculated Insight Object) | Materialized | Computed metric values, written on the insight's own schedule |
| **Segment Membership Table** | Materialized | Which unified profiles qualify for a segment, generated at publish time |
| **Unified Individual / Unified Contact Point [Type]** | Hybrid | A DMO on the surface, but sitting on top of materialized matching/reconciliation output underneath |
| **Unified Link Individual / Unified Link Contact Point [Type]** | Materialized | The crosswalk connecting each source record's ID to its resulting unified ID |

**The operations (what labels the arrows)**

| Operation | What it actually does |
|---|---|
| **Ingest** | A Data Stream pulling source records into a raw DLO |
| **Mapping** | Field-level translation from a DLO into a DMO — no row filtering, purely structural |
| **Data Transform** | Reshapes and/or filters a DLO into a new DLO (batch or streaming) |
| **Identity Resolution** | Matches and reconciles source records, producing the Unified Link + Unified [Object] pair |
| **Search Index Config** | Chunks a DMO/UDMO's content field, producing a CDMO |
| **Embedding model** | Converts each chunk in a CDMO into a vector, producing the IDMO |
| **Scheduled SQL** | The query a Calculated Insight runs on its own cadence |
| **Segment publish / segment criteria** | The trigger that materializes a Segment Membership Table |


```mermaid
flowchart TB

  subgraph STREAM["Streaming Data Transform — continuous, picks up new/changed data"]
    direction TB
    S_SRC["Source object<br/>(single DLO)"]
    S_OP{{"Reshape<br/>(filter, normalize, merge, split)"}}
    S_TGT1["Target object 1"]
    S_TGT2["Target object 2<br/>(optional, multiple targets)"]
    S_SRC --> S_OP
    S_OP --> S_TGT1
    S_OP --> S_TGT2
  end

  subgraph STREAM_UC["Streaming use cases"]
    direction LR
    UC1["Normalize with UNION<br/>denormalized to normalized"] ~~~ UC2["Merge multiple streams<br/>same keys, avoid dup in DMO"] ~~~ UC3["Split one stream<br/>profile + engagement to 2 DLOs"]
  end

  subgraph BATCH["Batch Data Transform — node canvas, repeatable / scheduled"]
    direction TB
    B_IN1["Input node: Case records<br/>(DLO)"]
    B_IN2["Input node: Account records<br/>(DLO)"]
    B_FILTER{{"Filter node<br/>escalated cases only"}}
    B_JOIN{{"Join node<br/>fully qualified key (FQK)"}}
    B_AGG{{"Aggregate node<br/>roll up to account level"}}
    B_TRANSFORM{{"Transform node<br/>concat, standardize, sentiment"}}
    B_UPDATE{{"Update node<br/>swap values on key match"}}
    B_OUT["Output node<br/>(DLO — must match DLO inputs)"]

    B_IN1 --> B_FILTER --> B_JOIN
    B_IN2 --> B_JOIN
    B_JOIN --> B_AGG --> B_TRANSFORM --> B_UPDATE --> B_OUT
  end

  subgraph LEGEND[" Legend "]
    direction LR
    LG1["DLO-type object"] ~~~ LG2["DMO-type object"] ~~~ LG3["Transform node (operation)"] ~~~ LG4["Use case example"]
  end

  STREAM -. illustrated by .-> STREAM_UC
  STREAM ~~~ LEGEND
  BATCH ~~~ LEGEND

  classDef dlo fill:#1B3A5C,color:#ffffff,stroke:#2E75B6,stroke-width:1px
  classDef dmo fill:#EAF1F8,color:#1B3A5C,stroke:#2E75B6,stroke-width:1px,stroke-dasharray: 4 3
  classDef op fill:#C9821E,color:#ffffff,stroke:#8a5a14,stroke-width:1px
  classDef usecase fill:#5B8FB9,color:#ffffff,stroke:#2E75B6,stroke-width:1px

  class S_SRC,S_TGT1,S_TGT2,B_IN1,B_IN2,B_OUT dlo
  class S_OP,B_FILTER,B_JOIN,B_AGG,B_TRANSFORM,B_UPDATE op
  class UC1,UC2,UC3 usecase
  class LG1 dlo
  class LG2 dmo
  class LG3 op
  class LG4 usecase
```
