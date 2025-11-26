# Chapter 2 – Authentication Between AKS and Akeyless: Kubernetes Auth Method

## Why Authentication Matters

Authenticating AKS (External Secrets Operator) to Akeyless ensures that only authorized workloads can retrieve sensitive secrets securely. This prevents secret leakage and enforces security policies centrally, thus protecting your infrastructure and applications.

Without proper authentication:

Unauthorized pods could access secrets they shouldn't have permission to read.

Security incidents become harder to trace and audit.

Compliance requirements for access control and least-privilege cannot be met.

## Authentication Methods Supported by Akeyless

Akeyless supports several authentication options for Kubernetes clusters to interact securely:

API Key: Static secret-based method, simpler but less secure, often used for CI/CD automation scenarios.

Kubernetes Auth: Uses Kubernetes service account tokens tied to the cluster identity, recommended for production use for its dynamic, secure nature.

Cloud Identity Providers: Integrations with AWS IAM, Azure AD, and Google Cloud IAM for cloud-connected authentications.

For AKS + ESO integration, Kubernetes Auth is the recommended approach because it eliminates static credentials and leverages Kubernetes-native identity.

## Kubernetes Auth Method Overview

Kubernetes Auth relies on Kubernetes ServiceAccount JWT tokens to prove the identity of the caller:

AKS automatically issues a JWT token to a pod's service account when the pod starts.

Akeyless is configured with the AKS cluster's API endpoint and the cluster's CA certificate to securely communicate and verify tokens.

When a secret fetch request arrives, Akeyless validates the token directly against the AKS API server's TokenReview endpoint and verifies it belongs to an authorized service account.

This token-based authentication offers:

Dynamic, secure access without static credentials stored in configurations.

Fine-grained role binding for namespaces, service accounts, or other Kubernetes attributes.

Improved compliance and reduced security risk through centralized policy enforcement.

## How Kubernetes Auth Works: Step-by-Step Flow

1. Pod starts in AKS: Kubernetes injects a JWT token into the pod via the ServiceAccount.
2. ESO reads the token: External Secrets Operator uses this token to authenticate with Akeyless.
3. Token sent to Akeyless: ESO sends the JWT token along with the secret fetch request.
4. Token validation: Akeyless calls the AKS API server's TokenReview endpoint to verify the token is valid and identifies the correct ServiceAccount.
5. Access control check: Akeyless checks if the ServiceAccount is authorized via Access Roles to read the requested secret paths.
6. Secret returned: If authorized, Akeyless returns the secret value to ESO, which creates the Kubernetes Secret.

## Setting up Kubernetes Auth Method in Akeyless

To configure Kubernetes Auth in Akeyless, you need:

AKS API Server URL: The endpoint where Akeyless can reach your cluster's API server.

Cluster CA Certificate: The certificate authority data for TLS trust between Akeyless and AKS.

TokenReview Configuration: Akeyless uses the Kubernetes TokenReview API to validate tokens.

Steps in Akeyless:

Navigate to Authentication Methods in the Akeyless console.

Create a new Kubernetes Auth Method.

Provide the AKS cluster API server URL and CA certificate.

Configure the auth method to use TokenReview for validation.

Save and note the Auth Method Access ID for later use.

## Defining Access Roles in Akeyless

Access Roles map Kubernetes identities (ServiceAccounts, namespaces) to permitted secret paths:

Create an Access Role in Akeyless.

Bind it to the Kubernetes Auth Method.

Define rules specifying which ServiceAccount or namespace can access which secret paths.

Grant appropriate permissions (read, list) on those paths.

Example rule:

ServiceAccount: akeyless-auth-sa in namespace akeyless-auth

Allowed paths: /demo/app/*

Permissions: read, list

## How External Secrets Operator Uses Kubernetes Auth

Once configured:

ESO is deployed in AKS with a dedicated ServiceAccount (e.g., akeyless-auth-sa).

The SecretStore or ClusterSecretStore references this ServiceAccount and the Akeyless Access ID.

When ESO reconciles an ExternalSecret, it automatically uses the ServiceAccount JWT token to authenticate to Akeyless.

Akeyless validates the token and checks Access Roles before returning secrets.

This entire flow happens automatically - no manual token management required.

## Security Benefits

Kubernetes Auth provides multiple security advantages:

No static secrets: Eliminates the need to store API keys or passwords in cluster configurations.

Kubernetes-native identity: Leverages existing RBAC and ServiceAccount mechanisms.

Automatic token rotation: Kubernetes handles token lifecycle and rotation.

Centralized audit: All access requests are logged in Akeyless with identity information.

Least privilege enforcement: Fine-grained Access Roles ensure workloads only access needed secrets.

## Diagram: Kubernetes Auth Method Flow
## Diagram: Kubernetes Auth Method Flow

```
flowchart TB
    subgraph AKS["AKS Cluster"]
        Pod["Pod with ServiceAccount<br/>(akeyless-auth-sa)"]
        ESO["External Secrets Operator<br/>(ESO Controller)"]
        API["AKS API Server<br/>(TokenReview Endpoint)"]
        K8sSecret["Kubernetes Secret<br/>(app-db-secret)"]
    end
    
    subgraph Akeyless["Akeyless Platform"]
        AuthMethod["Kubernetes Auth Method<br/>(AKS API URL + CA Cert)"]
        AccessRole["Access Roles<br/>(SA: akeyless-auth-sa → /demo/app/*)"]
        Vault["Secrets Vault<br/>(/demo/app/db-password)"]
    end
    
    Pod -->|"1. JWT token mounted<br/>in pod"| ESO
    ESO -->|"2. Fetch secret request<br/>+ JWT token"| AuthMethod
    AuthMethod -->|"3. Validate token via<br/>TokenReview API"| API
    API -->|"4. Token valid +<br/>ServiceAccount identity"| AuthMethod
    AuthMethod -->|"5. Check Access Role<br/>permissions"| AccessRole
    AccessRole -->|"6. Read secret from<br/>vault"| Vault
    Vault -->|"7. Return secret value"| ESO
    ESO -->|"8. Create/Update<br/>Kubernetes Secret"| K8sSecret
    
    style AKS fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style Akeyless fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style Pod fill:#c8e6c9,stroke:#2e7d32
    style ESO fill:#c8e6c9,stroke:#2e7d32
    style API fill:#c8e6c9,stroke:#2e7d32
    style K8sSecret fill:#c8e6c9,stroke:#2e7d32
    style AuthMethod fill:#ffe0b2,stroke:#ef6c00
    style AccessRole fill:#ffe0b2,stroke:#ef6c00
    style Vault fill:#ffe0b2,stroke:#ef6c00
```
```

