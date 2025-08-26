# AWS Identity Center OIDC with Cognito Cross-Account Setup

## Overview

This document provides a comprehensive end-to-end setup for implementing AWS Identity Center (IDC) OIDC integration with Amazon Cognito across multiple AWS accounts using Trusted Identity Propagation (TIP).

## Architecture Overview

```mermaid
graph TB
    %% Define styles for different components
    classDef cognitoStyle fill:#FF9900,stroke:#FF6600,stroke-width:3px,color:#fff
    classDef idcStyle fill:#232F3E,stroke:#FF9900,stroke-width:3px,color:#fff
    classDef lambdaStyle fill:#FF9900,stroke:#FF6600,stroke-width:2px,color:#fff
    classDef storageStyle fill:#3F48CC,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef appStyle fill:#146EB4,stroke:#FF9900,stroke-width:2px,color:#fff
    classDef flowStyle fill:#00C853,stroke:#00A040,stroke-width:2px,color:#fff
    classDef accountStyle fill:#f9f9f9,stroke:#333,stroke-width:2px,color:#333
    
    %% Identity Account
    subgraph IA ["🏢 Identity Account (<IDENTITY_ACCOUNT_ID>)"]
        direction TB
        CUP["👥 Cognito User Pool | 🔐 User Authentication"]
        CUC["🔑 User Pool Client | 📱 OIDC Configuration"]
        CD["🌐 Cognito Domain | 🔗 OAuth Endpoints"]
        CU["👤 Users & Groups | 📋 SSOUsers, Admins"]
        L1["⚡ Lambda Function | 📤 Parameter Creator"]
        EB1["📡 EventBridge | 🔄 CloudTrail Events"]
    end
    
    %% SSO Account  
    subgraph SA ["🏛️ SSO Account (<SSO_ACCOUNT_ID>)"]
        direction TB
        IDC["🎯 AWS Identity Center | 🔐 Central SSO Hub"]
        TIP["🛡️ Trusted Token Issuer | ✅ JWT Validation"]
        OA["📱 OIDC Application | 🔗 Cross-Account App"]
        L2["⚡ Lambda Function | 🏗️ SSO Operations"]
        EB2["📡 EventBridge Rule | 🎯 Parameter Triggers"]
        DDB["🗄️ DynamoDB Table | 📊 App Configuration"]
        SSM["📦 Parameter Store | 🔒 Secure Config"]
        CAR["🔐 Cross-Account Role | 🤝 Trust Relationship"]
    end
    
    %% User Applications
    subgraph UA ["💻 User Applications"]
        direction TB
        WA["🌐 Web Application | 🔗 Production App"]
        TA["🧪 Test OIDC App | ✅ Integration Testing"]
    end
    
    %% AWS Services
    subgraph AWS ["☁️ AWS Services"]
        direction TB
        STS["🎫 AWS STS | 🔑 Temporary Credentials"]
        S3["📦 S3 Buckets | 📁 Resource Access"]
        EC2["🖥️ EC2 Instances | 💻 Compute Resources"]
    end
    
    %% Cross-Account Connections
    L1 -.->|"🔐 Assume Role"| CAR
    CAR -.->|"📝 Create Parameter"| SSM
    SSM -->|"🚨 Parameter Event"| EB2
    EB2 -->|"⚡ Trigger Lambda"| L2
    
    %% SSO Account Internal Flow
    L2 -->|"🏗️ Create App"| OA
    L2 -->|"🛡️ Create Issuer"| TIP
    L2 -->|"📊 Store Config"| DDB
    TIP -.->|"🔗 Register with"| IDC
    OA -.->|"📱 Register with"| IDC
    
    %% Identity Account Internal Flow
    CU -->|"👤 Belongs to"| CUP
    CUP -->|"🔗 Configured with"| CUC
    CUC -->|"🌐 Hosted on"| CD
    CUC -.->|"📡 Triggers Events"| EB1
    EB1 -.->|"⚡ Invokes"| L1
    
    %% User Authentication Flow
    WA -->|"🔐 OAuth Login"| CD
    TA -->|"🧪 Test Login"| CD
    CD -->|"🎫 JWT Tokens"| WA
    CD -->|"🎫 JWT Tokens"| TA
    
    %% AWS Resource Access
    WA -.->|"🎫 JWT + TIP"| STS
    STS -.->|"🔑 Temp Credentials"| S3
    STS -.->|"🔑 Temp Credentials"| EC2
    
    %% Apply styles
    class CUP,CUC,CD,CU cognitoStyle
    class IDC,TIP,OA idcStyle
    class L1,L2 lambdaStyle
    class DDB,SSM storageStyle
    class WA,TA appStyle
    class EB1,EB2,CAR flowStyle
    class STS,S3,EC2 appStyle
    class IA,SA,UA,AWS accountStyle
```

## 🔄 Cross-Account Authentication Flow

```mermaid
sequenceDiagram
    participant U as User
    participant A as Application
    participant C as Cognito (Identity Account)
    participant I as Identity Center (SSO Account)
    participant AWS as AWS Services
    
    U->>A: 1. Access Application
    A->>C: 2. Redirect to Cognito Login
    U->>C: 3. Authenticate (username/password)
    C->>A: 4. Return Authorization Code
    A->>C: 5. Exchange Code for JWT Token
    C->>A: 6. Return JWT with Claims & Groups
    A->>I: 7. Use JWT for TIP (Optional)
    I->>AWS: 8. Assume Role with Web Identity
    AWS->>A: 9. Return Temporary Credentials
    A->>U: 10. Grant Access with Permissions
```

## OAuth 2.0 OIDC Flow Diagram

```mermaid
sequenceDiagram
    participant Browser as Browser
    participant App as Test App
    participant Cognito as Cognito
    participant Token as Token Endpoint
    
    Browser->>App: 1. Click Login
    App->>Cognito: 2. Redirect to /oauth2/authorize
    Note over Cognito: User authenticates
    Cognito->>App: 3. Redirect with authorization code
    App->>Token: 4. POST /oauth2/token (code + credentials)
    Token->>App: 5. Return JWT tokens
    App->>Browser: 6. Display user claims & groups
```
