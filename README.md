# YouTube Unified Feed Processing System

## Overview
This document describes the GCP-based pipeline for processing YouTube partner feeds. The system ingests a unified feed, processes it through configurable transformations, and delivers customized outputs to multiple partners.

![Feed Processing Pipeline](diagram-feed-processing.png)

## Architecture Diagram

```mermaid
graph TD
    subgraph GCP_Project["GCP Project"]
        direction TB
        GCS_Input[("GCS Bucket (yt-unified-feed-input)")] -- "Step 1: Feed Upload" --> CF_Trigger(Cloud Function: Feed Trigger)
        CF_Trigger -- "Step 2: Publish Message" --> PubSub(Pub/Sub: new-yt-feed-topic)
        
        PubSub -- "Step 3: Trigger Job" --> Dataflow

        subgraph Dataflow["Dataflow Pipeline (Python/Beam)"]
            direction TB
            DF_ReadGCS[Read Generic Feed] -- "Step 4: Load Config" --> Firestore
            DF_ReadGCS --> DF_ReadConfig[Read Partner Config]
            DF_ReadConfig --> DF_FanOut{Fan-Out per Partner}
            DF_FanOut --> DF_Map[Map Data]
            DF_Map --> DF_Validate[Validate Data]
            DF_Validate --> DF_Format[Format Output to JSON]
            DF_Format --> DF_WriteGCS[Write Customized Feed]
        end

        DF_WriteGCS --> GCS_Output_A[("GCS Bucket (Partner A)")]
        DF_WriteGCS --> GCS_Output_B[("GCS Bucket (Partner B)")]
        DF_WriteGCS --> GCS_Output_C[("GCS Bucket (Partner C)")]

        Firestore[("Firestore (Partner Configs)")]

        Dataflow -- "Logs/Metrics" --> Monitor(Cloud Monitoring & Logging)
        CF_Trigger -- "Logs/Metrics" --> Monitor
        Monitor -- "Alerts" --> OpsTeam[Operations/Alerting]

        AdminTool[Admin Tool / CI/CD] -- "Manages" --> Firestore
        AdminTool -- "Deploys" --> Dataflow
        AdminTool -- "Deploys" --> CF_Trigger
    end

    YT[External: YT System] --> GCS_Input
    GCS_Output_A --> PartnerA[Partner A Platform]
    GCS_Output_B --> PartnerB[Partner B Platform]
    GCS_Output_C --> PartnerC[Partner C Platform]

    style GCS_Input fill:#f9f,stroke:#333,stroke-width:2px
    style GCS_Output_A fill:#f9f,stroke:#333,stroke-width:2px
    style GCS_Output_B fill:#f9f,stroke:#333,stroke-width:2px
    style GCS_Output_C fill:#f9f,stroke:#333,stroke-width:2px
    style PubSub fill:#bbf,stroke:#333,stroke-width:2px
    style CF_Trigger fill:#ffcc66,stroke:#333,stroke-width:2px
    style Dataflow fill:#ccf,stroke:#333,stroke-width:4px
    style Firestore fill:#e7cfa8,stroke:#333,stroke-width:2px
    style Monitor fill:#9cf,stroke:#333,stroke-width:2px
