# SAPHanaSR-angi Architecture Diagrams

## 1. Component Architecture

Below is a Mermaid diagram representing the architecture and components of the SAPHanaSR-angi codebase:

```mermaid
graph TB
    %% Main Components
    subgraph "SAPHanaSR-angi"
        RA[Resource Agents]
        HOOKS[SAP HA/DR Hooks]
        TOOLS[Management Tools]
        TEST[Testing Framework]
        CONFIG[Cluster Configuration]
    end

    %% Resource Agents
    subgraph "Resource Agents"
        RA --> TOPO[SAPHanaTopology]
        RA --> CTRL[SAPHanaController]
        RA --> FS[SAPHanaFilesystem]
        
        subgraph "Libraries"
            COMMON[saphana-common-lib]
            TOPO_LIB[saphana-topology-lib]
            CTRL_LIB[saphana-controller-lib]
            CTRL_COMMON[saphana-controller-common-lib]
            FS_LIB[saphana-filesystem-lib]
        end
        
        TOPO --> COMMON
        TOPO --> TOPO_LIB
        CTRL --> COMMON
        CTRL --> CTRL_LIB
        CTRL --> CTRL_COMMON
        FS --> COMMON
        FS --> FS_LIB
        FS --> CTRL_COMMON
    end

    %% SAP HA/DR Hooks
    subgraph "SAP HA/DR Provider Hooks"
        HOOKS --> SUSCHK[susChkSrv.py]
        HOOKS --> SUSCOST[susCostOpt.py]
        HOOKS --> SUSHANSR[susHanaSR.py]
        HOOKS --> SUSTKOVER[susTkOver.py]
        HOOKS --> SUSMULTI[susHanaSrMultiTarget.py]
    end

    %% Management Tools
    subgraph "Management Tools"
        TOOLS --> SHOWATTR[SAPHanaSR-showAttr]
        TOOLS --> MONITOR[SAPHanaSR-monitor]
        TOOLS --> REPLAY[SAPHanaSR-replay-archive]
        TOOLS --> HELPER[SAPHanaSR-hookHelper]
        TOOLS --> PROVIDER[SAPHanaSR-manageProvider]
        TOOLS --> UPGRADE[SAPHanaSR-upgrade-to-angi-demo]
        
        SHOWATTR --> PYTOOLS[saphana_sr_tools.py]
    end

    %% Testing Framework
    subgraph "Testing Framework"
        TEST --> TESTBIN[Test Binaries]
        TEST --> TESTJSON[Test JSON Configs]
        TEST --> TESTWWW[Test Web Interface]
    end

    %% Cluster Configuration
    subgraph "Cluster Configuration"
        CONFIG --> SCALEUP[Scale-Up Configuration]
        CONFIG --> SCALEOUT[Scale-Out Configuration]
    end

    %% External Components
    SAP[SAP HANA Database]
    PACEMAKER[Pacemaker Cluster]
    
    %% Interactions
    SAP <--> HOOKS
    TOPO <--> SAP
    CTRL <--> SAP
    FS <--> SAP
    TOPO <--> PACEMAKER
    CTRL <--> PACEMAKER
    FS <--> PACEMAKER
    TOOLS <--> PACEMAKER
    HOOKS <--> PACEMAKER
    
    %% Legend
    classDef core fill:#f96,stroke:#333,stroke-width:2px
    classDef library fill:#9cf,stroke:#333,stroke-width:1px
    classDef external fill:#fcf,stroke:#333,stroke-width:2px
    
    class TOPO,CTRL,FS core
    class COMMON,TOPO_LIB,CTRL_LIB,CTRL_COMMON,FS_LIB library
    class SAP,PACEMAKER external
```

## 2. System Replication and Failover Workflow

This diagram illustrates the sequence of events during normal operation and failover scenarios:

```mermaid
sequenceDiagram
    participant Primary as Primary SAP HANA
    participant Secondary as Secondary SAP HANA
    participant Hooks as HA/DR Provider Hooks
    participant Topology as SAPHanaTopology
    participant Controller as SAPHanaController
    participant Cluster as Pacemaker Cluster

    Note over Primary,Secondary: Normal Operation
    Primary->>Secondary: System Replication (Sync/Async)
    
    loop Continuous Monitoring
        Topology->>Primary: Check landscape status
        Topology->>Secondary: Check landscape status
        Topology->>Cluster: Report topology information
        Controller->>Primary: Monitor primary status
        Controller->>Secondary: Monitor secondary status
        Controller->>Cluster: Report controller status
    end
    
    Note over Primary,Cluster: Failure Scenario
    Primary--xSecondary: Primary failure detected
    
    Hooks->>Cluster: srConnectionChanged event
    Hooks->>Cluster: Set SR status attribute (SFAIL/SOK)
    
    Topology->>Cluster: Detect topology change
    Controller->>Cluster: Detect primary failure
    
    alt SR Status = SOK (In Sync)
        Cluster->>Controller: Trigger promote operation
        Controller->>Secondary: Execute takeover (hdbnsutil -sr_takeover)
        Secondary->>Secondary: Become new primary
        Note over Secondary: Now acting as Primary
    else SR Status != SOK (Not In Sync)
        Note over Cluster: Avoid takeover to maintain data consistency
        Cluster->>Cluster: Wait for manual intervention
    end
    
    Note over Primary,Secondary: After Recovery
    Primary->>Cluster: Primary node recovers
    Controller->>Primary: Register as secondary (hdbnsutil -sr_register)
    Secondary->>Primary: System Replication (reversed roles)
```

## 3. Scale-Up vs Scale-Out Deployment Scenarios

```mermaid
graph TB
    %% Scale-Up Scenario
    subgraph "Scale-Up Deployment"
        subgraph "Site 1"
            SU_P[Primary HANA Instance]
        end
        
        subgraph "Site 2"
            SU_S[Secondary HANA Instance]
        end
        
        SU_P -- "System Replication" --> SU_S
    end
    
    %% Scale-Out Scenario
    subgraph "Scale-Out Deployment"
        subgraph "Site 1 (Primary)"
            SO_P_M[Primary Master Node]
            SO_P_W1[Primary Worker Node 1]
            SO_P_W2[Primary Worker Node 2]
            
            SO_P_M --- SO_P_W1
            SO_P_M --- SO_P_W2
        end
        
        subgraph "Site 2 (Secondary)"
            SO_S_M[Secondary Master Node]
            SO_S_W1[Secondary Worker Node 1]
            SO_S_W2[Secondary Worker Node 2]
            
            SO_S_M --- SO_S_W1
            SO_S_M --- SO_S_W2
        end
        
        SO_P_M -- "System Replication" --> SO_S_M
        SO_P_W1 -- "System Replication" --> SO_S_W1
        SO_P_W2 -- "System Replication" --> SO_S_W2
    end
    
    %% Annotations
    classDef primary fill:#f96,stroke:#333,stroke-width:2px
    classDef secondary fill:#9cf,stroke:#333,stroke-width:1px
    
    class SU_P,SO_P_M,SO_P_W1,SO_P_W2 primary
    class SU_S,SO_S_M,SO_S_W1,SO_S_W2 secondary
```

## Component Descriptions

### Resource Agents
- **SAPHanaTopology**: Analyzes SAP HANA database topology and reports to the cluster
- **SAPHanaController**: Manages SAP HANA databases in System Replication, controls start/stop and monitors status
- **SAPHanaFilesystem**: Manages filesystem resources for SAP HANA

### SAP HA/DR Provider Hooks
- **susHanaSR.py**: Informs cluster about System Replication state changes
- **susChkSrv.py**: Checks server availability
- **susCostOpt.py**: Optimizes costs in the SAP HANA environment
- **susTkOver.py**: Handles takeover operations
- **susHanaSrMultiTarget.py**: Handles multi-target replication scenarios

### Management Tools
- **SAPHanaSR-showAttr**: Displays cluster attributes and status
- **SAPHanaSR-monitor**: Monitors the SAP HANA SR cluster
- **SAPHanaSR-replay-archive**: Replays archived cluster data
- **SAPHanaSR-hookHelper**: Assists with hook operations
- **SAPHanaSR-manageProvider**: Manages HA/DR providers
- **saphana_sr_tools.py**: Python library for SAP HANA SR tools

### Testing Framework
- Contains various test scripts and configurations for validating cluster behavior

### Cluster Configuration
- Contains example configurations for Scale-Up and Scale-Out scenarios

## Deployment Scenarios

### Scale-Up Deployment
- Single HANA instance per site
- Simpler configuration and management
- Suitable for smaller deployments
- Resource agents manage failover between the two instances

### Scale-Out Deployment
- Multiple HANA instances per site (master + worker nodes)
- Higher performance and capacity
- More complex configuration
- Resource agents manage both the node-level and site-level failover
- Requires coordination between multiple nodes during failover

## Workflow Summary

1. **SAPHanaTopology** monitors the SAP HANA landscape and reports to the cluster
2. **SAPHanaController** manages the SAP HANA databases based on topology information
3. **SAP HA/DR Hooks** communicate system replication state changes to the cluster
4. **Management Tools** provide visibility and control over the cluster state
5. During failover, the cluster orchestrates takeover operations between primary and secondary sites

This architecture enables high availability for SAP HANA databases using SUSE's cluster technology, supporting both Scale-Up and Scale-Out deployment scenarios.
