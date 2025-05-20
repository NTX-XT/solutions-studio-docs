# Solution Studio Architecture Diagrams

This document provides visual representations of key Solution Studio architecture concepts using mermaid.js diagrams.

## Table of Contents
- [Core Component Architecture](#core-component-architecture)
- [Blueprint Lifecycle](#blueprint-lifecycle)
- [Solution Execution Flow](#solution-execution-flow)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Data-Driven Orchestration Process](#data-driven-orchestration-process)
- [Blueprint and Variation Model](#blueprint-and-variation-model)

## Core Component Architecture

The following diagram illustrates the core components of Solution Studio and how they interact:

```mermaid
flowchart TD
    subgraph "Runtime Environment"
        OE["Orchestration Engine"]
        RunFlow["Run Flow Workflow"]
    end

    subgraph "Core Components"
        NA["Nintex Apps"]
        NT["Nintex Tables"]
        NW["Nintex Workflows"]
        FP["Feature Packs"]
    end

    subgraph "Builder Interface"
        SS["Solution Studio App"]
    end

    subgraph "User Interfaces"
        PA["Portal App"]
        AA["Admin App"] 
    end

    NA -- "Form definitions\nand UI models" --> NT
    SS -- "Creates" --> NA
    SS -- "Configures" --> NW
    SS -- "Defines/Stores" --> NT
    OE -- "Coordinates" --> RunFlow
    RunFlow -- "Invokes" --> NW
    RunFlow -- "Reads from" --> NT
    NW -- "Reads/Writes" --> NT
    FP -- "Provides reusable" --> NA
    FP -- "Provides reusable" --> NW
    PA -- "Interacts with" --> NA
    PA -- "Submits data to" --> NT
    PA -- "Triggers" --> OE
    AA -- "Manages" --> NT
    AA -- "Controls" --> OE

    classDef component fill:#f5f5f5,stroke:#333,stroke-width:2px;
    classDef runtime fill:#d8e8f4,stroke:#2980b9,stroke-width:2px;
    classDef interface fill:#e8f8e8,stroke:#27ae60,stroke-width:2px;
    classDef userinterface fill:#f5e8f4,stroke:#8e44ad,stroke-width:2px;

    class NA,NT,NW,FP component;
    class OE,RunFlow runtime;
    class SS interface;
    class PA,AA userinterface;
```

## Blueprint Lifecycle

The following diagram illustrates the lifecycle states of a blueprint or variation in Solution Studio:

```mermaid
stateDiagram-v2
    [*] --> Draft: Create Blueprint
    Draft --> Testing: Test Configuration
    Testing --> Draft: Modify/Fix Issues
    Testing --> Published: Verify & Publish
    Published --> Draft: Deactivate & Edit
    Published --> [*]: Decommission
    
    state Draft {
        [*] --> Building
        Building --> Configuring
        Configuring --> Reviewing
        Reviewing --> [*]
    }
    
    state Testing {
        [*] --> VerifyingForms
        VerifyingForms --> TestingWorkflows
        TestingWorkflows --> ValidatingOutputs
        ValidatingOutputs --> [*]
    }
    
    state Published {
        [*] --> Active
        Active --> Inactive: Deactivate
        Inactive --> Active: Activate
        Active --> [*]
    }
```

## Solution Execution Flow

This diagram shows how a solution instance executes through the orchestration engine:

```mermaid
flowchart TD
    %% Main components
    StartSubmission([Start Form Submission]) --> MainOrch
    MainOrch["Orchestration Engine<br/>(Run Flow Workflow)"] --> LoadConfig
    LoadConfig["Load Blueprint/Variation<br/>Configuration"] --> IdentifyStages
    IdentifyStages["Identify Stages<br/>& Sequence"] --> StageLoop

    %% Stage execution loop
    subgraph StageLoop["Stage Execution Cycle"]
        direction LR
        GetCurrentStage["Get Current<br/>Stage"] --> LoadStageConfig
        LoadStageConfig["Load Stage<br/>Configuration"] --> InvokeStage
        InvokeStage["Invoke Stage<br/>Workflow"] --> WaitCompletion
        WaitCompletion["Wait for Stage<br/>Completion"] --> EvalOutcome
        EvalOutcome{"Evaluate<br/>Outcome"} --> NextStage
        NextStage["Determine<br/>Next Stage"]
        NextStage --> |"Next Stage<br/>Available"| GetCurrentStage
    end

    StageLoop -->|"Final Stage<br/>Complete"| EndInstance([Solution Instance Complete])

    %% Stage workflow execution
    InvokeStage --> |"Invoke"| StageWorkflow

    subgraph StageWorkflow["Stage Workflow Execution"]
        direction TB
        GetContext["Get Instance & Stage<br/>Context"] --> LoadJson
        LoadJson["Load JSON<br/>Configuration"] --> Deserialize
        Deserialize["Deserialize<br/>JSON"] --> MapVars
        MapVars["Map to Workflow<br/>Variables"] --> ExecLogic
        ExecLogic["Execute Workflow<br/>Logic"] --> ReturnOutcome
        ReturnOutcome["Return Outcome<br/>(Complete/Reject/Failed)"]
    end

    ReturnOutcome --> WaitCompletion

    %% Data stores
    Tables[("Nintex Tables<br/>Steps Table")] -.-> LoadStageConfig
    Tables -.-> LoadJson

    %% Classes for styling
    classDef startNode fill:#7CFC00,stroke:#006400,stroke-width:2px,color:#000
    classDef endNode fill:#FF6347,stroke:#8B0000,stroke-width:2px,color:#000
    classDef process fill:#87CEFA,stroke:#4682B4,stroke-width:2px
    classDef decision fill:#FFD700,stroke:#DAA520,stroke-width:2px
    classDef database fill:#9370DB,stroke:#4B0082,stroke-width:2px
    classDef subGraph fill:#F5F5F5,stroke:#333,stroke-width:1px

    %% Apply classes
    class StartSubmission,EndInstance startNode
    class LoadConfig,IdentifyStages,GetCurrentStage,InvokeStage,WaitCompletion,NextStage,GetContext,LoadJson,Deserialize,MapVars,ExecLogic,ReturnOutcome,LoadStageConfig process
    class EvalOutcome decision
    class Tables database
    class MainOrch,StageWorkflow process
```

## Entity Relationship Diagram

This diagram shows the relationship between the key persistent entities in Solution Studio:

```mermaid
erDiagram
    FLOW {
        string FlowId PK
        string Name
        string Description
        string Type "Blueprint or Variation"
        string Status "Draft, Testing, or Published"
        string ParentFlowId FK "For variations, points to blueprint"
        json Configuration
    }
    
    STAGE {
        string StageId PK
        string FlowId FK
        string Key
        string Title
        string WorkflowId FK
        int Sequence
        boolean IsEnabled
    }
    
    STEP {
        string StepId PK
        string StageId FK
        string Key "e.g., action-task, review-task"
        json Configuration "Serialized form configuration"
        string FormId FK "Points to configuration form"
    }
    
    INSTANCE {
        string InstanceId PK
        string FlowId FK
        string Status "In Progress, Completed, etc."
        timestamp CreatedDate
        string CreatedBy
        timestamp ModifiedDate
        json Data "Form submission data"
    }
    
    TASK {
        string TaskId PK
        string InstanceId FK
        string StageId FK
        string StepId FK
        string Title
        string Status
        string AssignedTo
        timestamp DueDate
        json FormData
    }
    
    FORM {
        string FormId PK
        string Name
        string Type "Start, Task, Configuration"
        json UIModel
    }
    
    WORKFLOW {
        string WorkflowId PK
        string Name
        string Type "Stage, Notification, etc."
        string Status "Active, Draft"
    }
    
    FLOW ||--o{ STAGE : "contains"
    STAGE ||--o{ STEP : "configured by"
    FLOW ||--o{ INSTANCE : "has instances"
    INSTANCE ||--o{ TASK : "has tasks"
    STAGE ||--o{ TASK : "generates"
    STEP ||--o{ TASK : "configures"
    FORM ||--o{ STEP : "defines structure for"
    WORKFLOW ||--o{ STAGE : "associated with"
```

## Data-Driven Orchestration Process

This diagram illustrates the JSON serialization and deserialization process that drives the data-centric orchestration:

```mermaid
flowchart TB
    subgraph "Configuration Phase (Builder)"
        Builder([Builder]) --> ConfigForm["Configuration Form"]
        ConfigForm --> UIModel["UI Model (Squid UI)"]
        UIModel --> Serialize["JSON Serialization"]
        Serialize --> SaveTables["Save to Tables"]
    end
    
    subgraph "Execution Phase (Runtime)"
        StartForm([Start Form Submission]) --> RunFlow["Run Flow Workflow"]
        RunFlow --> LoadBP["Load Blueprint/Variation"]
        LoadBP --> GetStage["Get Current Stage"]
        GetStage --> LoadConfig["Load Stage Configuration"]
        LoadConfig --> InvokeWF["Invoke Stage Workflow"]
        InvokeWF --> StageWF["Stage Workflow"]
        StageWF --> GetStepConfig["Get Step Configuration"]
        GetStepConfig --> Deserialize["JSON Deserialization"]
        Deserialize --> MapVariables["Map to Workflow Variables"]
        MapVariables --> ExecuteLogic["Execute Dynamic Logic"]
        ExecuteLogic --> CompleteStage["Complete Stage"]
        CompleteStage --> ReturnToFlow["Return to Run Flow"]
        ReturnToFlow --> NextStage["Next Stage or Complete"]
    end
    
    SaveTables -.-> LoadConfig
    Tables[("Nintex Tables\n(Steps, Instances, Flows)")] <-.-> SaveTables
    Tables <-.-> GetStepConfig
    
    classDef builderPhase fill:#e6f7ff,stroke:#1890ff,stroke-width:2px;
    classDef executionPhase fill:#fff7e6,stroke:#fa8c16,stroke-width:2px;
    classDef storage fill:#f6ffed,stroke:#52c41a,stroke-width:2px;
    classDef actor fill:#f9f0ff,stroke:#722ed1,stroke-width:2px;
    
    class Builder,StartForm actor;
    class ConfigForm,UIModel,Serialize,SaveTables builderPhase;
    class RunFlow,LoadBP,GetStage,LoadConfig,InvokeWF,StageWF,GetStepConfig,Deserialize,MapVariables,ExecuteLogic,CompleteStage,ReturnToFlow,NextStage executionPhase;
    class Tables storage;
```

## Blueprint and Variation Model

This diagram shows the relationship between blueprints and variations, and how they utilize components:

```mermaid
flowchart TB
    Blueprint["Blueprint\n(Master Definition)"] --> |"Clone"| Var1["Variation 1\n(IT Ticketing)"]
    Blueprint --> |"Clone"| Var2["Variation 2\n(HR Onboarding)"]
    Blueprint --> |"Clone"| Var3["Variation 3\n(Custom Scenario)"]
    
    subgraph "Shared Components"
        FP["Feature Packs"]
        FWF["Foundation Workflows"]
        TF["Task Forms"]
        SF["Start Forms"]
    end
    
    subgraph "Blueprint Definition"
        S1["Stage 1\n(Submission)"]
        S2["Stage 2\n(Review)"]
        S3["Stage 3\n(Action)"]
        S4["Stage 4\n(Resolution)"]
    end
    
    Blueprint --- S1
    Blueprint --- S2
    Blueprint --- S3
    Blueprint --- S4
    
    FP --> |"Provides"| S2
    FP --> |"Provides"| S3
    FP --> |"Provides"| S4
    
    FWF --> |"Execute"| S2
    FWF --> |"Execute"| S3
    FWF --> |"Execute"| S4
    
    TF --> |"UI for"| S2
    TF --> |"UI for"| S3
    TF --> |"UI for"| S4
    
    SF --> |"UI for"| S1
    
    Var1 --> |"Customizes"| Var1Config["Custom Configuration\n(JSON)"]
    Var2 --> |"Customizes"| Var2Config["Custom Configuration\n(JSON)"]
    Var3 --> |"Customizes"| Var3Config["Custom Configuration\n(JSON)"]
    
    Var3 --> |"Uses"| CustomWF["Custom Workflow"]
    Var2 --> |"Disables"| S4
    
    classDef blueprint fill:#d0e8ff,stroke:#4a86e8,stroke-width:2px;
    classDef variation fill:#d9ead3,stroke:#6aa84f,stroke-width:2px;
    classDef component fill:#fff2cc,stroke:#f1c232,stroke-width:2px;
    classDef stage fill:#f4cccc,stroke:#cc0000,stroke-width:2px;
    classDef config fill:#d9d2e9,stroke:#8e7cc3,stroke-width:2px;
    
    class Blueprint blueprint;
    class Var1,Var2,Var3 variation;
    class FP,FWF,TF,SF,CustomWF component;
    class S1,S2,S3,S4 stage;
    class Var1Config,Var2Config,Var3Config config;
```
