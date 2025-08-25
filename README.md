# SSO Application Deployment Architecture

## Overview
End-to-end automated SSO application deployment system using AWS Identity Center with Lambda integration, EventBridge automation, and lifecycle management.

## Architecture Components

### Core Services
- **AWS Identity Center**: SSO application management
- **Lambda Functions**: Application operations and lifecycle management
- **EventBridge**: Automated triggers for parameter changes
- **Systems Manager**: Configuration storage and automation
- **DynamoDB**: Application metadata and TTL management
- **SNS**: Lifecycle alerts and notifications

## Architecture Diagram

```mermaid
graph TB
    subgraph "Configuration Management"
        SSM[AWS Systems Manager<br/>Parameter Store]
        SSM_CONFIG["/sso/{app_type}/config<br/>SAML Configuration"]
        SSM_INSTANCE["/sso/instance-arn<br/>Identity Center Instance"]
        
        SSM --> SSM_CONFIG
        SSM --> SSM_INSTANCE
    end
    
    subgraph "Event-Driven Automation"
        EB[EventBridge Rules]
        EB_CREATE[Parameter Change Rule<br/>Create/Update]
        EB_DELETE[Parameter Deletion Rule<br/>Delete Detection]
        
        EB --> EB_CREATE
        EB --> EB_DELETE
    end
    
    subgraph "Core Processing"
        LAMBDA_MAIN[Lambda: sso-operations-handler<br/>Main Application Logic]
        LAMBDA_LIFECYCLE[Lambda: sso-lifecycle-manager<br/>Cleanup & Alerts]
        
        subgraph "Lambda Operations"
            OP_CREATE[create_application]
            OP_MANAGE[manage_application]
            OP_ASSIGN[assign_users]
            OP_SAML[configure_saml]
        end
        
        LAMBDA_MAIN --> OP_CREATE
        LAMBDA_MAIN --> OP_MANAGE
        LAMBDA_MAIN --> OP_ASSIGN
        LAMBDA_MAIN --> OP_SAML
    end
    
    subgraph "AWS Identity Center"
        IDC[Identity Center Instance]
        IDC_APPS[SSO Applications]
        IDC_GROUPS[User Groups]
        IDC_SAML[SAML Configuration]
        
        IDC --> IDC_APPS
        IDC --> IDC_GROUPS
        IDC --> IDC_SAML
    end
    
    subgraph "Data Storage"
        DDB[DynamoDB Table<br/>sso-application-config]
        DDB_TTL[TTL Management<br/>Auto-cleanup]
        
        DDB --> DDB_TTL
    end
    
    subgraph "Notifications"
        SNS[SNS Topic<br/>sso-lifecycle-alerts]
        EMAIL[Email Notifications<br/>admin@company.com]
        
        SNS --> EMAIL
    end
    
    subgraph "Automation"
        SSM_AUTO[SSM Automation Document<br/>SSODeploymentAutomation]
        SSM_STEPS[Automation Steps<br/>Create → Configure → Assign]
        
        SSM_AUTO --> SSM_STEPS
    end
    
    subgraph "Application Types"
        APP_SAGE[SageMaker Application<br/>Production Status]
        APP_JUPYTER[Jupyter Application<br/>Development Status]
        APP_TABLEAU[Tableau Application<br/>Production Status]
        
        SSM_CONFIG -.-> APP_SAGE
        SSM_CONFIG -.-> APP_JUPYTER
        SSM_CONFIG -.-> APP_TABLEAU
    end
    
    %% Event Flow
    SSM_CONFIG -->|Parameter Change| EB_CREATE
    SSM_CONFIG -->|Parameter Delete| EB_DELETE
    EB_CREATE -->|Trigger| LAMBDA_MAIN
    EB_DELETE -->|Trigger| LAMBDA_LIFECYCLE
    
    %% Processing Flow
    LAMBDA_MAIN -->|Read Config| SSM_CONFIG
    LAMBDA_MAIN -->|Create/Update| IDC_APPS
    LAMBDA_MAIN -->|Configure SAML| IDC_SAML
    LAMBDA_MAIN -->|Assign Groups| IDC_GROUPS
    LAMBDA_MAIN -->|Store Metadata| DDB
    
    %% Lifecycle Management
    LAMBDA_LIFECYCLE -->|Query Apps| DDB
    LAMBDA_LIFECYCLE -->|Send Alerts| SNS
    DDB_TTL -->|Auto-delete| DDB
    
    %% Manual Automation
    SSM_AUTO -->|Invoke| LAMBDA_MAIN
    SSM_AUTO -->|Read Config| SSM_CONFIG
    
    %% Styling
    classDef primary fill:#e3f2fd
    classDef lambda fill:#fff3e0
    classDef storage fill:#e8f5e8
    classDef event fill:#fce4ec
    classDef app fill:#f3e5f5
    
    class IDC,IDC_APPS,IDC_GROUPS,IDC_SAML primary
    class LAMBDA_MAIN,LAMBDA_LIFECYCLE,OP_CREATE,OP_MANAGE,OP_ASSIGN,OP_SAML lambda
    class SSM,SSM_CONFIG,SSM_INSTANCE,DDB,DDB_TTL storage
    class EB,EB_CREATE,EB_DELETE,SNS,EMAIL event
    class APP_SAGE,APP_JUPYTER,APP_TABLEAU app
```
