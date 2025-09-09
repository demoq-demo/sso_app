# AWS Identity Center OIDC Application Automation Solution

## Architecture Overview

This solution provides end-to-end automation for creating AWS Identity Center OIDC applications across multiple AWS accounts. The solution consists of two main parts:

1. **Part 1**: Application Trigger System (DynamoDB Streams)
2. **Part 2**: IDC Application Creation System (EventBridge/SSM)

## Complete Architecture Diagram

```mermaid
graph TB
    %% Styling
    classDef userAction fill:#e1f5fe,stroke:#01579b,stroke-width:3px,color:#000
    classDef dynamodb fill:#ff9800,stroke:#e65100,stroke-width:2px,color:#fff
    classDef lambda fill:#4caf50,stroke:#1b5e20,stroke-width:2px,color:#fff
    classDef ssm fill:#9c27b0,stroke:#4a148c,stroke-width:2px,color:#fff
    classDef eventbridge fill:#f44336,stroke:#b71c1c,stroke-width:2px,color:#fff
    classDef idc fill:#2196f3,stroke:#0d47a1,stroke-width:2px,color:#fff
    classDef iam fill:#795548,stroke:#3e2723,stroke-width:2px,color:#fff
    classDef account fill:#e8f5e8,stroke:#2e7d32,stroke-width:3px,color:#000
    
    %% Part 1: Application Trigger System
    subgraph "🏢 Application Account (Part 1)"
        direction TB
        
        subgraph "📋 User Input"
            USER[👤 User/DevOps<br/>Creates DynamoDB Item]:::userAction
            CF[☁️ CloudFormation<br/>Terraform]:::userAction
        end
        
        subgraph "🗄️ Storage & Processing"
            DDB1[📊 DynamoDB Table<br/>idc-application-triggers<br/>🔄 Streams Enabled]:::dynamodb
            LAMBDA1[⚡ Lambda Function<br/>idc-application-trigger<br/>🎯 Stream Processor]:::lambda
        end
        
        subgraph "🔐 Cross-Account Access"
            ROLE1[🛡️ Cross-Account Role<br/>IDCSSMCrossAccountRole]:::iam
        end
    end
    
    %% Part 2: IDC Application Creation System  
    subgraph "🏛️ Identity Center Account (Part 2)"
        direction TB
        
        subgraph "📝 Configuration Storage"
            SSM[📋 SSM Parameter Store<br/>/sso/{app-name}/config<br/>🔒 SecureString]:::ssm
        end
        
        subgraph "🔔 Event Processing"
            EB[📡 EventBridge Rule<br/>SSM Parameter Changes<br/>🎯 /sso/*/config]:::eventbridge
            LAMBDA2[⚡ Lambda Function<br/>idc-sso-operations-handler<br/>🏗️ Application Creator]:::lambda
        end
        
        subgraph "🎯 Identity Center Resources"
            IDC_APP[🔐 IDC OIDC Application<br/>Custom Provider<br/>✅ Portal Enabled]:::idc
            IDC_ASSIGN[👥 Group Assignments<br/>User Access Control]:::idc
            IDC_TIP[🎫 Trusted Token Issuer<br/>Advanced JWT Validation]:::idc
        end
        
        subgraph "🛡️ IAM Resources"
            IAM_OIDC[🔗 OIDC Identity Provider<br/>d-abc123def456.awsapps.com]:::iam
            IAM_ROLE[🛡️ IAM Role<br/>IDCOIDC-{app-type}-Role<br/>🧪 Testing Purpose]:::iam
        end
        
        subgraph "📊 Tracking"
            DDB2[📊 DynamoDB Table<br/>idc-sso-application-config<br/>📈 Deployment Tracking]:::dynamodb
        end
    end
    
    %% Flow Connections
    USER --> DDB1
    CF --> DDB1
    DDB1 -->|🔄 Stream Event| LAMBDA1
    LAMBDA1 -->|🔐 Assume Role| ROLE1
    LAMBDA1 -->|📝 Create Parameter| SSM
    
    SSM -->|📡 Parameter Change Event| EB
    EB -->|🎯 Trigger| LAMBDA2
    LAMBDA2 -->|🏗️ Create| IDC_APP
    LAMBDA2 -->|👥 Assign| IDC_ASSIGN
    LAMBDA2 -->|🎫 Create| IDC_TIP
    LAMBDA2 -->|🔗 Create| IAM_OIDC
    LAMBDA2 -->|🛡️ Create| IAM_ROLE
    LAMBDA2 -->|📊 Track| DDB2
    
    %% Account boundaries
    class "🏢 Application Account (Part 1)" account
    class "🏛️ Identity Center Account (Part 2)" account
```

## Sequence Diagram - End-to-End Flow

```mermaid
sequenceDiagram
    participant 👤 User as User/DevOps
    participant 📊 DDB1 as DynamoDB<br/>(App Account)
    participant ⚡ L1 as Lambda 1<br/>(Stream Processor)
    participant 🛡️ Role as Cross-Account<br/>Role
    participant 📋 SSM as SSM Parameter<br/>(IDC Account)
    participant 📡 EB as EventBridge<br/>(IDC Account)
    participant ⚡ L2 as Lambda 2<br/>(IDC Creator)
    participant 🔐 IDC as Identity Center
    participant 🛡️ IAM as IAM Resources
    participant 📊 DDB2 as DynamoDB<br/>(IDC Account)
    
    %% Styling
    Note over 👤 User, 📊 DDB1: 🏢 Application Account (Part 1)
    Note over 📋 SSM, 📊 DDB2: 🏛️ Identity Center Account (Part 2)
    
    %% Part 1: Application Trigger
    rect rgb(255, 248, 225)
        Note over 👤 User, ⚡ L1: 🚀 Part 1: Application Request Processing
        👤 User->>📊 DDB1: 📝 Create DynamoDB Item<br/>{app_name, callback_url, groups}
        📊 DDB1->>⚡ L1: 🔄 DynamoDB Stream Event<br/>(INSERT/MODIFY)
        ⚡ L1->>⚡ L1: 🔍 Extract Application Config<br/>Validate Required Fields
        ⚡ L1->>🛡️ Role: 🔐 Assume Cross-Account Role<br/>ExternalId: idc-sso-integration
        🛡️ Role-->>⚡ L1: ✅ Temporary Credentials
    end
    
    %% Cross-Account Parameter Creation
    rect rgb(240, 248, 255)
        Note over ⚡ L1, 📋 SSM: 🌉 Cross-Account Bridge
        ⚡ L1->>📋 SSM: 📝 Create SSM Parameter<br/>/sso/{app_name}/config<br/>SecureString with OIDC config
        ⚡ L1->>📊 DDB1: ✅ Update Status: deployed<br/>Add deployed_time
    end
    
    %% Part 2: IDC Application Creation
    rect rgb(248, 255, 248)
        Note over 📋 SSM, 📊 DDB2: 🏗️ Part 2: IDC Application Creation
        📋 SSM->>📡 EB: 📡 Parameter Store Change Event<br/>Source: aws.ssm
        📡 EB->>⚡ L2: 🎯 Trigger Lambda<br/>operation: manage_application
        ⚡ L2->>📋 SSM: 📖 Read Parameter Config<br/>Parse JSON configuration
        ⚡ L2->>🔐 IDC: 🏗️ Create OIDC Application<br/>Custom Provider + Portal Options
        🔐 IDC-->>⚡ L2: ✅ Application ARN<br/>apl-xxxxxxxxx
    end
    
    %% Group Assignments
    rect rgb(255, 240, 245)
        Note over ⚡ L2, 🔐 IDC: 👥 User Access Configuration
        ⚡ L2->>🔐 IDC: 🔍 Resolve Group Names<br/>DisplayName → GroupId
        ⚡ L2->>🔐 IDC: 👥 Create Application Assignment<br/>Assign Groups to Application
        🔐 IDC-->>⚡ L2: ✅ Assignment Complete<br/>(or warning if conflict)
    end
    
    %% Advanced Features
    rect rgb(248, 240, 255)
        Note over ⚡ L2, 🛡️ IAM: 🎫 Advanced OIDC Features
        alt TIP Enabled
            ⚡ L2->>🔐 IDC: 🎫 Create Trusted Token Issuer<br/>OIDC_JWT Configuration
            🔐 IDC-->>⚡ L2: ✅ Issuer ARN
            ⚡ L2->>🔐 IDC: 🔗 Configure Application Grant<br/>JWT Bearer Token
        end
        
        ⚡ L2->>🛡️ IAM: 🔗 Create OIDC Identity Provider<br/>https://d-abc123def456.awsapps.com
        ⚡ L2->>🛡️ IAM: 🛡️ Create IAM Role<br/>IDCOIDC-{app-type}-Role
        🛡️ IAM-->>⚡ L2: ✅ Resources Created
    end
    
    %% Tracking and Completion
    rect rgb(255, 255, 240)
        Note over ⚡ L2, 📊 DDB2: 📊 Deployment Tracking
        ⚡ L2->>📊 DDB2: 📊 Store Deployment Metadata<br/>Application ARN, Groups, Status
        ⚡ L2-->>👤 User: ✅ Application Ready!<br/>🔗 Portal URL Available<br/>🎫 Manual Credentials Setup Required
    end
```

## Component Details

### 🏢 Part 1: Application Trigger System

| Component | Purpose | Technology |
|-----------|---------|------------|
| **DynamoDB Table** | Store application requests | `idc-application-triggers` with Streams |
| **Lambda Function** | Process stream events | `idc-application-trigger` |
| **Cross-Account Role** | Secure parameter creation | `IDCSSMCrossAccountRole` |

### 🏛️ Part 2: IDC Application Creation System

| Component | Purpose | Technology |
|-----------|---------|------------|
| **SSM Parameter Store** | Store OIDC configurations | `/sso/{app-name}/config` |
| **EventBridge Rule** | Detect parameter changes | SSM Parameter Store events |
| **Lambda Function** | Create IDC applications | `idc-sso-operations-handler` |
| **Identity Center** | OIDC application hosting | Custom provider with portal |
| **IAM Resources** | Testing infrastructure | OIDC providers and roles |

## Key Features

### 🔄 **Automated Workflow**
- **Input**: DynamoDB item creation
- **Processing**: Cross-account parameter creation
- **Output**: Fully configured IDC OIDC application

### 🛡️ **Security**
- Cross-account roles with external ID
- Least privilege permissions
- Encrypted parameter storage

### 📊 **Monitoring**
- DynamoDB deployment tracking
- CloudWatch logs for debugging
- Status updates throughout process

### 🎯 **Flexibility**
- Support for multiple applications
- Configurable group assignments
- Optional advanced features (TIP)

## Usage Example

```bash
# 1. Create application request
aws dynamodb put-item --table-name idc-application-triggers --item '{
  "application_name": {"S": "my-web-app"},
  "status": {"S": "pending"},
  "callback_url": {"S": "https://myapp.example.com/callback"},
  "issuer": {"S": "https://d-abc123def456.awsapps.com"},
  "groups": {"SS": ["AdminGroup", "DeveloperGroup"]}
}'

# 2. System automatically processes and creates IDC application
# 3. Check deployment status
aws dynamodb get-item --table-name idc-application-triggers \
  --key '{"application_name":{"S":"my-web-app"}}'
```

## Benefits

- ✅ **Fully Automated**: No manual IDC console work
- ✅ **Multi-Account**: Works across AWS account boundaries  
- ✅ **Scalable**: Handle multiple applications simultaneously
- ✅ **Auditable**: Complete deployment tracking
- ✅ **Secure**: Proper IAM roles and permissions
- ✅ **Flexible**: Configurable for different use cases
