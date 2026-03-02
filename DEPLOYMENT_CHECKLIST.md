# Full AI Suite - Customer Azure Deployment Checklist

> **What is this?** A step-by-step guide to deploy the entire AI suite (Divi, AI Search, Speech-to-Text, Encryption, Users, Graph, History, and Frontend) on a customer's Azure environment from scratch. Written so that someone with minimal Azure experience can follow it.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Azure Subscription & Tenant](#2-azure-subscription--tenant)
3. [Azure AD (Entra ID) - App Registrations](#3-azure-ad-entra-id---app-registrations)
4. [Resource Groups](#4-resource-groups)
5. [Azure Container Registry (ACR)](#5-azure-container-registry-acr)
6. [Azure Cosmos DB (MongoDB)](#6-azure-cosmos-db-mongodb)
7. [Azure Blob Storage](#7-azure-blob-storage)
8. [Azure Key Vault](#8-azure-key-vault)
9. [Azure OpenAI Service](#9-azure-openai-service)
10. [Azure AI Services (DeepSeek Models)](#10-azure-ai-services-deepseek-models)
11. [Azure Cognitive Search (AI Search)](#11-azure-cognitive-search-ai-search)
12. [Azure Cognitive Services (Speech)](#12-azure-cognitive-services-speech)
13. [Azure API Management (APIM)](#13-azure-api-management-apim)
14. [Azure Container Apps Environment](#14-azure-container-apps-environment)
15. [Deploy Backend Services](#15-deploy-backend-services)
16. [Azure Static Web Apps (Frontend)](#16-azure-static-web-apps-frontend)
17. [SearXNG (Web Search Engine)](#17-searxng-web-search-engine)
18. [MCP Servers (External Tooling)](#18-mcp-servers-external-tooling)
19. [Sentry (Error Monitoring)](#19-sentry-error-monitoring)
20. [Email / SMTP Setup](#20-email--smtp-setup)
21. [Linear Integration (Issue Tracking)](#21-linear-integration-issue-tracking)
22. [Deployment Configuration Repository](#22-deployment-configuration-repository)
23. [CI/CD - GitHub Actions](#23-cicd---github-actions)
24. [DNS & Custom Domains](#24-dns--custom-domains)
25. [Security Hardening & Final Review](#25-security-hardening--final-review)
26. [End-to-End Testing & Verification](#26-end-to-end-testing--verification)

---

## 1. Prerequisites

Before you touch Azure, make sure you have these ready:

### Accounts & Access
- [ ] **Azure account** with an active subscription (pay-as-you-go or Enterprise Agreement)
- [ ] **Azure AD Global Administrator** role (or someone who has it and can help you)
- [ ] **GitHub account** with access to all the source code repositories:
  - `divi` (LLM backend)
  - `diviaisearch` (RAG/document search)
  - `GenAiSpeechToText` (speech-to-text)
  - `encryptiongod` (encryption service)
  - `gen-ai-users` (user management)
  - `graph8` (Microsoft Graph wrapper)
  - `history8` (conversation history)
  - `procurementsuitefrontend` (React frontend)
- [ ] **Git** installed on your machine
- [ ] **Azure CLI** installed (`az` command) - [Install guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [ ] **Docker** installed (for building container images locally if needed)
- [ ] **Node.js 20+** installed (for local testing)
- [ ] **.NET 8 SDK** installed (for the C# services)

### Information to Gather from the Customer
- [ ] Customer's **Azure Tenant ID** (or are you creating a new tenant?)
- [ ] Desired **Azure region** (e.g., `germanywestcentral`, `westeurope`, `swedencentral`)
- [ ] Customer's **custom domain** (e.g., `ai.customerdomain.com`)
- [ ] Customer's **email domain** (for SMTP/notifications)
- [ ] List of **users** who need access to the platform
- [ ] Customer's **data residency requirements** (which region data must stay in)
- [ ] Customer's **budget/spending limits** for Azure

---

## 2. Azure Subscription & Tenant

### Set Up the Subscription
- [ ] Log into the [Azure Portal](https://portal.azure.com) with the customer's admin account
- [ ] Navigate to **Subscriptions** and verify an active subscription exists
- [ ] Note down the **Subscription ID** (you'll need it later)
- [ ] Note down the **Tenant ID** (Azure Active Directory > Overview > Tenant ID)

### Set Spending Limits (Optional but Recommended)
- [ ] Go to **Cost Management + Billing**
- [ ] Set up a **budget alert** so you don't get surprise bills
- [ ] Set appropriate spending limits for the AI services (OpenAI can get expensive)

### Register Required Resource Providers
> Azure needs you to "turn on" certain services before you can use them. This is a one-time thing.

- [ ] Go to **Subscriptions > [Your Sub] > Resource providers**
- [ ] Register (click "Register") each of these if they show as "NotRegistered":
  - [ ] `Microsoft.ContainerRegistry`
  - [ ] `Microsoft.App` (Container Apps)
  - [ ] `Microsoft.DocumentDB` (Cosmos DB)
  - [ ] `Microsoft.Storage`
  - [ ] `Microsoft.KeyVault`
  - [ ] `Microsoft.CognitiveServices` (OpenAI + Speech)
  - [ ] `Microsoft.Search`
  - [ ] `Microsoft.Web` (Static Web Apps)
  - [ ] `Microsoft.ApiManagement`
  - [ ] `Microsoft.ManagedIdentity`
  - [ ] `Microsoft.OperationalInsights` (Log Analytics)

---

## 3. Azure AD (Entra ID) - App Registrations

> This is the authentication backbone. Every service and the frontend authenticate through Azure AD. This section is critical - get it wrong and nothing will work.

### 3a. Create the Main App Registration (Frontend + Shared)

This is the primary application that users log into.

- [ ] Go to **Azure Active Directory > App registrations > New registration**
- [ ] **Name**: `Customer AI Suite` (or whatever the customer wants)
- [ ] **Supported account types**: "Accounts in this organizational directory only" (Single tenant)
- [ ] **Redirect URI**: Select "Single-page application (SPA)" and enter:
  - `https://<customer-frontend-domain>/auth-redirect`
  - (For testing, also add `http://localhost:5173/auth-redirect`)
- [ ] Click **Register**
- [ ] **Note down the Application (Client) ID** - this is your `AZURE_AD_CLIENT_ID` / `VITE_MSAL_CLIENT_ID`

### 3b. Configure API Permissions

- [ ] Go to **API permissions > Add a permission**
- [ ] Add **Microsoft Graph** permissions:
  - [ ] `User.Read` (Delegated) - So users can sign in and read their profile
  - [ ] `User.ReadBasic.All` (Delegated) - To read other users' basic info
  - [ ] `Mail.Send` (Delegated) - If using email features through Graph
  - [ ] `Calendars.Read` (Delegated) - If using calendar features
- [ ] Click **Grant admin consent for [Customer Org]** (requires admin)

### 3c. Expose an API (So Backend Services Can Validate Tokens)

- [ ] Go to **Expose an API**
- [ ] Click **Set** next to "Application ID URI" - accept the default (`api://<client-id>`) or set a custom one
- [ ] Click **Add a scope**:
  - **Scope name**: `access_as_user`
  - **Who can consent**: Admins and users
  - **Admin consent display name**: "Access AI Suite as a user"
  - **Admin consent description**: "Allow the application to access AI Suite APIs on behalf of the signed-in user"
  - **State**: Enabled
- [ ] Click **Add scope**

### 3d. Create a Client Secret (For Backend Services That Need Server-to-Server Auth)

- [ ] Go to **Certificates & secrets > New client secret**
- [ ] **Description**: "AI Suite Backend Secret"
- [ ] **Expires**: Pick an appropriate expiry (e.g., 12 months or 24 months)
- [ ] Click **Add**
- [ ] **IMMEDIATELY copy the secret value** - you will never see it again! This is your `AZURE_AD_CLIENT_SECRET`

### 3e. Configure Token Settings

- [ ] Go to **Token configuration > Add optional claim**
- [ ] Token type: **ID**
- [ ] Select claims: `email`, `family_name`, `given_name`, `preferred_username`
- [ ] Click **Add**
- [ ] Repeat for **Access** token type with the same claims

### 3f. Add Application Roles (For RBAC)

- [ ] Go to **App roles > Create app role**:
  - **Display name**: "Agents Manager"
  - **Allowed member types**: Users/Groups
  - **Value**: `agents.manager`
  - **Description**: "Can create, update, and delete AI agents"
  - **Enabled**: Yes
- [ ] Click **Apply**

### 3g. Assign Users to the Application

- [ ] Go to **Azure Active Directory > Enterprise applications**
- [ ] Find your app registration
- [ ] Go to **Users and groups > Add user/group**
- [ ] Assign the customer's users who should have access
- [ ] Assign the `agents.manager` role to admin users

### Summary - Save These Values
| Value | Environment Variable | Where It's Used |
|-------|---------------------|-----------------|
| Tenant ID | `AZURE_AD_TENANT_ID` / `AZURE_TENANT_ID` | All services |
| Client ID | `AZURE_AD_CLIENT_ID` / `AZURE_CLIENT_ID` / `VITE_MSAL_CLIENT_ID` | All services |
| Client Secret | `AZURE_AD_CLIENT_SECRET` | gen-ai-users, graph8 |
| Authority URL | `VITE_MSAL_AUTHORITY` | Frontend (format: `https://login.microsoftonline.com/<tenant-id>`) |

---

## 4. Resource Groups

> Resource Groups are like folders to organize your Azure resources. Create separate ones for better organization.

- [ ] Create **Resource Group 1** - Main services:
  - **Name**: `<customer>-ai-suite-rg` (e.g., `contoso-ai-suite-rg`)
  - **Region**: Your chosen region (e.g., `germanywestcentral`)
- [ ] Create **Resource Group 2** - AI/ML resources (optional, for separation):
  - **Name**: `<customer>-ai-models-rg`
  - **Region**: Matches your OpenAI deployment region (e.g., `swedencentral`)
- [ ] Create **Resource Group 3** - Speech services (if different region):
  - **Name**: `<customer>-speech-rg`
  - **Region**: Where Speech API is available (e.g., `francecentral`)

---

## 5. Azure Container Registry (ACR)

> This is where your Docker images live. All backend services get built into Docker images and stored here.

### Create the Registry
- [ ] Go to **Container Registries > Create**
- [ ] **Name**: `<customer>genairegistry` (must be globally unique, alphanumeric only)
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Location**: Same as your main resource group
- [ ] **SKU**: Standard (or Premium if you need geo-replication)
- [ ] Click **Create**

### Configure Access
- [ ] Go to your new ACR > **Access keys**
- [ ] **Enable** the Admin user toggle
- [ ] Note down:
  - **Login server**: `<name>.azurecr.io`
  - **Username**: (same as registry name)
  - **Password**: (generated, copy it)

### Push Initial Images
> For each service, build and push the Docker image:

- [ ] **Divi** (LLM backend):
  ```
  cd divi
  docker build -t <registry>.azurecr.io/divi:latest .
  docker push <registry>.azurecr.io/divi:latest
  ```
- [ ] **DiviAiSearch** (RAG search):
  ```
  cd diviaisearch
  docker build -t <registry>.azurecr.io/divi-ai-search:latest .
  docker push <registry>.azurecr.io/divi-ai-search:latest
  ```
- [ ] **GenAiSpeechToText** (speech):
  ```
  cd GenAiSpeechToText/GenAiSpeechToText
  docker build -t <registry>.azurecr.io/speech-to-text:latest .
  docker push <registry>.azurecr.io/speech-to-text:latest
  ```
- [ ] **EncryptionGod - TypeScript**:
  ```
  cd encryptiongod/typescript
  docker build -t <registry>.azurecr.io/encryptiongod:latest .
  docker push <registry>.azurecr.io/encryptiongod:latest
  ```
- [ ] **EncryptionGod - C#** (high-performance version):
  ```
  cd encryptiongod/c-sharp
  docker build -t <registry>.azurecr.io/encryptiongod-csharp:latest .
  docker push <registry>.azurecr.io/encryptiongod-csharp:latest
  ```
- [ ] **Gen-AI-Users**:
  ```
  cd gen-ai-users
  docker build -t <registry>.azurecr.io/gen-ai-users:latest .
  docker push <registry>.azurecr.io/gen-ai-users:latest
  ```
- [ ] **Graph8**:
  ```
  cd graph8
  docker build -t <registry>.azurecr.io/graph8:latest .
  docker push <registry>.azurecr.io/graph8:latest
  ```
- [ ] **History8**:
  ```
  cd history8
  docker build -t <registry>.azurecr.io/history8:latest .
  docker push <registry>.azurecr.io/history8:latest
  ```

---

## 6. Azure Cosmos DB (MongoDB)

> The suite uses Cosmos DB with the MongoDB API. You need **two** Cosmos DB accounts: one for application data, one for the credential manager.

### 6a. Create Primary Cosmos DB Account (Application Data)

- [ ] Go to **Azure Cosmos DB > Create > Azure Cosmos DB for MongoDB**
- [ ] **Account name**: `<customer>-cosmos-db` (globally unique)
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Location**: Your chosen region
- [ ] **Capacity mode**: Provisioned throughput (or Serverless for lower cost)
- [ ] **Version**: 4.2 or higher
- [ ] Click **Create**

Once created:
- [ ] Go to **Connection strings** and copy the **Primary Connection String**
- [ ] This becomes your `MONGODB_URI` / `MONGO_URI`

### Create Databases and Collections

- [ ] Go to **Data Explorer > New Database**: `ai-suite`
- [ ] Create collections (these get auto-created by the services, but good to know):
  - `users` - User profiles and groups (gen-ai-users)
  - `conversations` - Chat conversations (history8)
  - `messages` - Chat messages (history8)
  - `agents` - AI agent configurations (diviaisearch)
  - `documents` - Document metadata (diviaisearch)
  - `transcriptions` - Speech transcription records (GenAiSpeechToText)

### 6b. Create Credential Manager Cosmos DB Account

> This is a separate database used to securely store encrypted credentials for multi-tenant setups.

- [ ] Go to **Azure Cosmos DB > Create > Azure Cosmos DB for MongoDB**
- [ ] **Account name**: `<customer>-credential-manager-db` (globally unique)
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Location**: Same region
- [ ] **Capacity mode**: Serverless (this one is low-volume)
- [ ] Click **Create**

Once created:
- [ ] Copy the **Primary Connection String** - this is your `CREDENTIALS_URL`
- [ ] Create database: `credential-manager`
- [ ] Create collection: `credentials`

### 6c. Seed Credential Manager (If Using Centralized Config)

> Several services (graph8, history8, encryptiongod) fetch their configs from the credential manager DB instead of env vars. You need to insert encrypted credential documents.

- [ ] Generate a 32-byte hex encryption key (this is your `ENCRYPTION_KEY`):
  ```
  openssl rand -hex 32
  ```
- [ ] Store the encryption key securely (you'll need it for graph8, history8, encryptiongod env vars)
- [ ] Insert credential documents into the `credentials` collection for each service that needs them (the services will look up their configs by `CONTAINER_ID`)

---

## 7. Azure Blob Storage

> Used for file uploads, document processing, SharePoint file caching, audio files, etc.

### 7a. Create Storage Account for General Attachments

- [ ] Go to **Storage accounts > Create**
- [ ] **Name**: `<customer>genaiattachments` (globally unique, lowercase, no special chars)
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Region**: Your chosen region
- [ ] **Performance**: Standard
- [ ] **Redundancy**: LRS (Locally-redundant) or GRS (Geo-redundant) depending on requirements
- [ ] Click **Create**

Once created:
- [ ] Go to **Access keys** and copy the **Connection string** - this is your `AZURE_STORAGE_CONNECTION_STRING`
- [ ] Go to **Containers** and create:
  - [ ] `attachments` (Private access level) - for chat file uploads
  - [ ] `sharepoint-files` (Private access level) - for cached SharePoint documents

### 7b. Create Storage Account for Speech/Audio (If Different Account Needed)

- [ ] Go to **Storage accounts > Create**
- [ ] **Name**: `<customer>aiblob` (globally unique)
- [ ] **Resource group**: `<customer>-speech-rg`
- [ ] **Region**: Same as speech service region
- [ ] Click **Create**

Once created:
- [ ] Copy the **Connection string** and **Account Key**
- [ ] Create containers:
  - [ ] `audio` (Private)
  - [ ] `text` (Private)
  - [ ] `image` (Private)
  - [ ] `others` (Private)
  - [ ] `transcription` (Private)

### 7c. Configure CORS on Storage Accounts

> The frontend needs to upload files directly to blob storage in some cases.

- [ ] Go to each Storage account > **Resource sharing (CORS)**
- [ ] Add a CORS rule for Blob service:
  - **Allowed origins**: `https://<customer-frontend-domain>` (and `http://localhost:5173` for dev)
  - **Allowed methods**: GET, PUT, POST, DELETE, OPTIONS
  - **Allowed headers**: `*`
  - **Exposed headers**: `*`
  - **Max age**: 3600

---

## 8. Azure Key Vault

> Stores per-user encryption keys for the EncryptionGod service. Each user gets their own AES key stored securely in Key Vault.

### Create the Key Vault
- [ ] Go to **Key vaults > Create**
- [ ] **Name**: `<customer>-encryption-kv` (globally unique)
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Region**: Your chosen region
- [ ] **Pricing tier**: Standard
- [ ] **Permission model**: Azure role-based access control (RBAC)
- [ ] Click **Create**

Once created:
- [ ] Note the **Vault URI** (e.g., `https://<name>.vault.azure.net/`) - this is your `AZURE_KEYVAULT_URL`

### Configure Access Policies

> The EncryptionGod service needs to create, read, and manage secrets in Key Vault. It uses the user's token for RBAC access, so users need Key Vault permissions.

- [ ] Go to **Access control (IAM) > Add role assignment**
- [ ] Assign **Key Vault Secrets Officer** role to:
  - [ ] The Container App's managed identity (once created in step 14)
  - [ ] OR the application's service principal
- [ ] Assign **Key Vault Secrets User** role to users who will use encryption features

---

## 9. Azure OpenAI Service

> This is the brain of the AI suite. You need to deploy multiple AI models across potentially multiple regions (not all models are available in all regions).

### 9a. Create Azure OpenAI Resources

> You may need multiple Azure OpenAI resources in different regions because not all models are available everywhere.

**Resource 1 - Primary (Sweden Central is recommended for best model availability)**
- [ ] Go to **Azure AI services > Azure OpenAI > Create**
- [ ] **Name**: `<customer>-openai-primary`
- [ ] **Resource group**: `<customer>-ai-models-rg`
- [ ] **Region**: `swedencentral` (best model availability in EU)
- [ ] **Pricing tier**: Standard S0
- [ ] Click **Create**
- [ ] Note down the **Endpoint** and **Key** from the Keys and Endpoint page

**Resource 2 - Secondary (for models not available in primary region)**
- [ ] Create another Azure OpenAI resource if needed in `norwayeast` or `germanywestcentral`
- [ ] Note down its **Endpoint** and **Key**

**Resource 3 - Default/fallback endpoint**
- [ ] Create if needed for load distribution
- [ ] Note down its **Endpoint** and **Key**

### 9b. Deploy Models

> Go to each Azure OpenAI resource > **Model deployments > Deploy model** and create these deployments:

**On Primary Resource (swedencentral):**
- [ ] **gpt-4.1-mini** - Main chat model (cost-effective)
  - Deployment name: `gpt-4.1-mini`
  - Tokens-per-minute (TPM): At least 80K
- [ ] **gpt-4.5-preview** - Advanced reasoning model
  - Deployment name: `gpt-4.5-preview`
  - TPM: At least 40K
- [ ] **gpt-5** - Most powerful model
  - Deployment name: `gpt-5`
  - TPM: At least 40K
- [ ] **gpt-5-mini** - Efficient version of GPT-5
  - Deployment name: `gpt-5-mini`
  - TPM: At least 80K
- [ ] **gpt-5.1** - Latest model
  - Deployment name: `gpt-5.1`
  - TPM: At least 40K
- [ ] **gpt-5.1-chat** - Chat-optimized version
  - Deployment name: `gpt-5.1-chat`
  - TPM: At least 40K
- [ ] **gpt-5.2** - Newest model
  - Deployment name: `gpt-5.2`
  - TPM: At least 40K
- [ ] **gpt-5.2-chat** - Chat-optimized version
  - Deployment name: `gpt-5.2-chat`
  - TPM: At least 40K
- [ ] **gpt-image-1.5** - Image generation
  - Deployment name: `gpt-image-1.5`
  - TPM: At least 20K
- [ ] **text-embedding-3-large** - For RAG embeddings
  - Deployment name: `text-embedding-3-large`
  - TPM: At least 120K (embeddings are cheap but high-volume)
- [ ] **model-router** - Auto-routes to best model (if available)
  - Deployment name: `model-router`
  - TPM: At least 80K

**On Secondary Resource (if needed):**
- [ ] **o3-mini** - Reasoning model
  - Deployment name: `o3-mini`
  - TPM: At least 40K
- [ ] **o4-mini** - Updated reasoning model
  - Deployment name: `o4-mini`
  - TPM: At least 40K
- [ ] **gpt-4o-mini** - Default fallback model
  - Deployment name: `gpt-4o-mini`
  - TPM: At least 80K
- [ ] **o3-deep-research** - Deep research model
  - Deployment name: `o3-deep-research`
  - TPM: At least 20K

### 9c. Note Down All Model Configuration

For **each** model deployment, you need:
- [ ] Endpoint URL
- [ ] API Key
- [ ] Deployment name
- [ ] API version (use `2025-04-01-preview` or `2025-01-01-preview`)
- [ ] Resource name

> **Tip**: Create a spreadsheet mapping each model to its endpoint/key/deployment/version. You'll need this for the environment variables.

---

## 10. Azure AI Services (DeepSeek Models)

> DeepSeek models (R1 and V3) run through Azure AI Services (not regular Azure OpenAI).

### Create Azure AI Hub/Project
- [ ] Go to **Azure AI Studio** (https://ai.azure.com)
- [ ] Create a new **Hub** in your chosen region
- [ ] Create a **Project** within the hub
- [ ] Deploy models from the **Model Catalog**:
  - [ ] **DeepSeek-R1** (latest version, e.g., `DeepSeek-R1-0528`)
    - Note endpoint URL and API key
  - [ ] **DeepSeek-V3** (latest version, e.g., `DeepSeek-V3-0324`)
    - Note endpoint URL and API key
- [ ] Note the **AI Services endpoint** (e.g., `https://<hub-name>.services.ai.azure.com/`)
- [ ] Note the **API key**

---

## 11. Azure Cognitive Search (AI Search)

> Powers the RAG (Retrieval-Augmented Generation) document search. Users upload documents, they get chunked, embedded, and indexed here.

### Create the Search Service
- [ ] Go to **Azure AI Search > Create**
- [ ] **Name**: `<customer>-aisearch` (globally unique)
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Location**: Your chosen region
- [ ] **Pricing tier**: Basic (minimum) or Standard (recommended for production)
  - Basic: 15 indexes, 2GB storage
  - Standard: 50 indexes, 25GB storage
- [ ] Click **Create**

### Get the Keys
- [ ] Go to **Keys**
- [ ] Copy the **Primary admin key** - this is your `AZURE_SEARCH_ADMIN_KEY`
- [ ] Note the **Search service URL** (e.g., `https://<name>.search.windows.net`) - this is your `AZURE_SEARCH_ENDPOINT`

### Create the Index
> The diviaisearch service creates the index automatically on first use, but you should note the index name.

- [ ] Default index name: `documents-index` (this is your `AZURE_SEARCH_INDEX_NAME`)
- [ ] The index will be auto-created with the correct schema when the first document is ingested

---

## 12. Azure Cognitive Services (Speech)

> Powers the real-time speech-to-text transcription feature.

### Create the Speech Service
- [ ] Go to **Azure AI services > Speech > Create**
- [ ] **Name**: `<customer>-speech`
- [ ] **Resource group**: `<customer>-speech-rg`
- [ ] **Region**: Check [region availability](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/regions) - `francecentral` or `westeurope` are good EU options
- [ ] **Pricing tier**: S0 (Standard)
- [ ] Click **Create**

### Get the Keys
- [ ] Go to **Keys and Endpoint**
- [ ] Copy **Key 1** - this is your `AZURE_SPEECH_KEY` / `AzureSpeech.SubscriptionKey`
- [ ] Note the **Region** - this is your `AZURE_SPEECH_REGION` / `AzureSpeech.Region`

---

## 13. Azure API Management (APIM)

> Acts as a gateway/proxy in front of your backend services. Provides rate limiting, subscription keys, and a unified API surface.

### Create APIM Instance
- [ ] Go to **API Management services > Create**
- [ ] **Name**: `<customer>-apim` (globally unique, becomes `<name>.azure-api.net`)
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Region**: Your chosen region
- [ ] **Organization name**: Customer's org name
- [ ] **Administrator email**: Customer's admin email
- [ ] **Pricing tier**: Developer (for testing) or Basic/Standard (for production)
  - Developer: cheap but no SLA
  - Basic: SLA, but limited features
  - Standard: recommended for production
- [ ] Click **Create** (this takes 30-60 minutes!)

### Create Subscriptions
- [ ] Go to **Subscriptions > Add subscription**
- [ ] **Name**: "AI Suite Frontend"
- [ ] **Scope**: All APIs
- [ ] Click **Create**
- [ ] Copy the **Primary key** - this is your `VITE_API_SUBSCRIPTION_KEY`

### Configure API Proxies
> Add each backend service as an API in APIM:

- [ ] **Divi Backend API**
  - Name: `divi`
  - URL: Point to the Divi Container App URL
  - Suffix: `divi/api/v1`
- [ ] **Divi Staging** (if separate staging)
  - Suffix: `divi-staging/api/v1`
- [ ] **AI Search API**
  - Name: `divi-ai-search`
  - Suffix: `divi-ai-search`
- [ ] **AI Search Indexing API**
  - Name: `divi-ai-indexing`
  - Suffix: `indexing`
- [ ] **Graph Explorer API**
  - Name: `graph-explorer`
  - Suffix: `graph-explorer`
- [ ] **Transcription API**
  - Name: `transcription`
  - Suffix: `transcription/api`

### Configure CORS Policy on APIM
- [ ] Go to **APIs > All APIs > Inbound processing > Add policy > CORS**
- [ ] Allowed origins: `https://<customer-frontend-domain>`
- [ ] Allowed methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
- [ ] Allowed headers: `*`

---

## 14. Azure Container Apps Environment

> Container Apps is where all your backend services run. First, you create an "Environment" (shared infrastructure), then deploy individual "Apps" into it.

### 14a. Create Log Analytics Workspace
- [ ] Go to **Log Analytics workspaces > Create**
- [ ] **Name**: `<customer>-ai-logs`
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Region**: Your chosen region
- [ ] Click **Create**

### 14b. Create the Container Apps Environment
- [ ] Go to **Container Apps Environments > Create**
- [ ] **Name**: `<customer>-ai-env`
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Region**: Your chosen region
- [ ] **Log Analytics workspace**: Select the one you just created
- [ ] **Zone redundancy**: Enabled (for production) or Disabled (for cost savings)
- [ ] Click **Create**

### 14c. Link Container Registry
- [ ] In the Container Apps Environment, the Container Apps will need access to your ACR
- [ ] When creating each Container App, you'll configure the registry credentials

---

## 15. Deploy Backend Services

> Deploy each microservice as a Container App. The order matters because some services depend on others.

### Deployment Order (deploy in this order):
1. EncryptionGod (no dependencies on other services)
2. Gen-AI-Users (no dependencies on other services)
3. History8 (depends on EncryptionGod URL)
4. Graph8 (depends on EncryptionGod URL)
5. DiviAiSearch (depends on Cosmos DB and Azure Search)
6. Divi (depends on all other services)
7. SearXNG (standalone)
8. MCP Servers (depend on Divi and other services)

---

### 15a. Deploy EncryptionGod

- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `encryption-backend`
- [ ] **Container Apps Environment**: Select `<customer>-ai-env`
- [ ] **Image source**: Azure Container Registry
- [ ] **Registry**: `<customer>genairegistry.azurecr.io`
- [ ] **Image**: `encryptiongod-csharp` (recommended) or `encryptiongod`
- [ ] **Tag**: `latest`
- [ ] **CPU**: 0.5, **Memory**: 1Gi
- [ ] **Ingress**: Enabled, External, Port 3006
- [ ] **Min replicas**: 1, **Max replicas**: 3

**Environment Variables:**
- [ ] `ASPNETCORE_ENVIRONMENT` = `Production`
- [ ] `ASPNETCORE_URLS` = `http://+:3006`
- [ ] `MONGO_URI` = (Primary Cosmos DB connection string)
- [ ] `AZURE_KEYVAULT_URL` = (Key Vault URI from step 8)
- [ ] `AZURE_TENANT_ID` = (from step 3)
- [ ] `AZURE_CLIENT_ID` = (from step 3)
- [ ] `VITE_CHAT_HISTORY_API` = (will be filled after History8 is deployed)
- [ ] `SENTRY_DSN` = (from step 19, or leave empty)
- [ ] `SENTRY_ENVIRONMENT` = `production`

Once deployed:
- [ ] Note the **Container App URL** (e.g., `https://encryption-backend.<hash>.<region>.azurecontainerapps.io`)
- [ ] This is your `ENCRYPTIONGOD_URL`

### Enable Managed Identity
- [ ] Go to the Container App > **Identity > System assigned > On**
- [ ] Copy the **Object ID**
- [ ] Go back to Key Vault > **Access control (IAM)**
- [ ] Assign **Key Vault Secrets Officer** to this managed identity

---

### 15b. Deploy Gen-AI-Users

- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `gen-ai-user-api`
- [ ] **Container Apps Environment**: Select `<customer>-ai-env`
- [ ] **Image**: `gen-ai-users:latest`
- [ ] **CPU**: 0.5, **Memory**: 1Gi
- [ ] **Ingress**: Enabled, External, Port 3000
- [ ] **Min replicas**: 1, **Max replicas**: 3

**Environment Variables:**
- [ ] `NODE_ENV` = `production`
- [ ] `SERVER_HOST` = `0.0.0.0`
- [ ] `SERVER_PORT` = `3000`
- [ ] `MONGODB_URI` = (Primary Cosmos DB connection string)
- [ ] `AZURE_AD_TENANT_ID` = (from step 3)
- [ ] `AZURE_AD_CLIENT_ID` = (from step 3)
- [ ] `AZURE_AD_CLIENT_SECRET` = (from step 3d)
- [ ] `AZURE_STORAGE_CONNECTION_STRING` = (from step 7a)
- [ ] `AZURE_STORAGE_CONTAINER_NAME` = `attachments`
- [ ] `SENTRY_DSN` = (from step 19, or leave empty)
- [ ] `SENTRY_TUNNEL_URL` = (Sentry tunnel URL, or leave empty)
- [ ] `SENTRY_EVENTS_URL` = (Sentry events URL, or leave empty)
- [ ] `SENTRY_AUTH_TOKEN` = (Sentry auth token, or leave empty)
- [ ] `SMTP_HOST` = (from step 20)
- [ ] `SMTP_PORT` = `587`
- [ ] `SMTP_SECURE` = `false`
- [ ] `SMTP_USER` = (from step 20)
- [ ] `SMTP_PASS` = (from step 20)
- [ ] `SMTP_FROM` = (e.g., `Feedback <noreply@customer.com>`)
- [ ] `LINEAR_API_KEY` = (from step 21, or leave empty)
- [ ] `LINEAR_PROJECT_TENANT` = (Linear project name, or leave empty)

Once deployed:
- [ ] Note the **Container App URL** - this is your `USERS_API_BASE_URL` / `VITE_USERS_API_BASE_URL`

---

### 15c. Deploy History8

- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `gen-ai-history-api`
- [ ] **Image**: `history8:latest`
- [ ] **CPU**: 0.5, **Memory**: 1Gi
- [ ] **Ingress**: Enabled, External, Port 3000
- [ ] **Min replicas**: 1, **Max replicas**: 3

**Environment Variables (Option A - Direct Config):**
- [ ] `NODE_ENV` = `production`
- [ ] `PORT` = `3000`
- [ ] `SERVER_HOST` = `0.0.0.0`
- [ ] `MONGO_URI` = (Primary Cosmos DB connection string)
- [ ] `AAD_CLIENT_ID` = (from step 3)
- [ ] `ENCRYPTIONGOD_URL` = (from step 15a)
- [ ] `SENTRY_DSN` = (from step 19, or leave empty)
- [ ] `SENTRY_AUTH_TOKEN` = (from step 19, or leave empty)

**Environment Variables (Option B - Credential Manager):**
- [ ] `CREDENTIALS_URL` = (Credential Manager Cosmos DB connection string from step 6b)
- [ ] `CONTAINER_ID` = (the document ID in credential manager)
- [ ] `ENCRYPTION_KEY` = (32-byte hex key from step 6c)
- [ ] `KEY_VAULT_ID` = (Key Vault secret ID if using KV for config)
- [ ] `ENCRYPTIONGOD_URL` = (from step 15a)
- [ ] `PORT` = `3000`

Once deployed:
- [ ] Note the **Container App URL** - this is your `VITE_CHAT_HISTORY_API`
- [ ] **Go back to EncryptionGod** and update `VITE_CHAT_HISTORY_API` to this URL

---

### 15d. Deploy Graph8

- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `graph8`
- [ ] **Image**: `graph8:latest`
- [ ] **CPU**: 0.5, **Memory**: 1Gi
- [ ] **Ingress**: Enabled, External, Port 3000
- [ ] **Min replicas**: 1, **Max replicas**: 2

**Environment Variables:**
- [ ] `NODE_ENV` = `production`
- [ ] `HOST` = `0.0.0.0`
- [ ] `PORT` = `3000`
- [ ] `CREDENTIALS_URL` = (Credential Manager Cosmos DB connection string)
- [ ] `CONTAINER_ID` = (credential manager document ID)
- [ ] `ENCRYPTION_KEY` = (32-byte hex key)
- [ ] `ENCRYPTIONGOD_URL` = (from step 15a)

Once deployed:
- [ ] Note the **Container App URL** - this is your `VITE_GRAPH_API_BASE_URL` (or route through APIM)

---

### 15e. Deploy DiviAiSearch (Indexing + Search)

> This service runs in two modes: one for document ingestion/indexing, one for search queries. Deploy both.

**Indexing Service:**
- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `divi-ai-search-indexing`
- [ ] **Image**: `divi-ai-search:latest`
- [ ] **CPU**: 1.0, **Memory**: 2Gi (needs more resources for document processing)
- [ ] **Ingress**: Enabled, External, Port 3000
- [ ] **Min replicas**: 1, **Max replicas**: 3

**Search Service:**
- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `divi-ai-search-app`
- [ ] **Image**: `divi-ai-search:latest`
- [ ] **CPU**: 0.5, **Memory**: 1Gi
- [ ] **Ingress**: Enabled, External, Port 3000
- [ ] **Min replicas**: 1, **Max replicas**: 5

**Environment Variables (same for both, with port differences):**
- [ ] `NODE_ENV` = `production`
- [ ] `SERVER_HOST` = `0.0.0.0`
- [ ] `SERVER_PORT` = `3000`
- [ ] `API_PREFIX` = `/api/v1`
- [ ] `AZURE_TENANT_ID` = (from step 3)
- [ ] `AZURE_CLIENT_ID` = (from step 3)
- [ ] `AZURE_STORAGE_CONNECTION_STRING` = (from step 7a)
- [ ] `AZURE_STORAGE_CONTAINER_NAME` = `sharepoint-files`
- [ ] `AZURE_SEARCH_ENDPOINT` = (from step 11)
- [ ] `AZURE_SEARCH_ADMIN_KEY` = (from step 11)
- [ ] `AZURE_SEARCH_INDEX_NAME` = `documents-index`
- [ ] `AZURE_OPENAI_KEY` = (OpenAI key for embeddings)
- [ ] `AZURE_OPENAI_ENDPOINT` = (OpenAI endpoint)
- [ ] `AZURE_OPENAI_EMBED_DEPLOYMENT` = `text-embedding-3-large`
- [ ] `AZURE_OPENAI_EMBED_API_VERSION` = `2023-05-15`
- [ ] `AZURE_OPENAI_KEY_4_1_mini` = (OpenAI key for the judge model)
- [ ] `AZURE_OPENAI_ENDPOINT_4_1_mini` = (full endpoint URL for 4.1-mini with deployment path)
- [ ] `MONGODB_URI` = (Primary Cosmos DB connection string)
- [ ] `VITE_MODEL_API_BASE_URL` = (will be the Divi service URL, fill after deploying Divi)
- [ ] `FALLBACK_MODEL` = `gpt-5-mini`

Once deployed:
- [ ] Note the **Indexing Service URL** - this is your `INGESTION_BACKEND_URL` / `VITE_DIVI_AI_INGESTION_API_BASE_URL`
- [ ] Note the **Search Service URL** - this is your `VITE_DIVI_AI_SEARCH_API_BASE_URL`

---

### 15f. Deploy Divi (Main LLM Backend)

> This is the central service that handles all AI completions, streaming, and tool use.

- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `divi`
- [ ] **Image**: `divi:latest`
- [ ] **CPU**: 1.0, **Memory**: 2Gi (needs resources for streaming + MCP)
- [ ] **Ingress**: Enabled, External, Port 3000
- [ ] **Min replicas**: 1, **Max replicas**: 10

**Environment Variables (this is the big one):**

Server:
- [ ] `NODE_ENV` = `production`
- [ ] `HOST` = `0.0.0.0`
- [ ] `PORT` = `3000`
- [ ] `API_PREFIX` = `/api/v1`
- [ ] `LOG_LEVEL` = `info`

Database:
- [ ] `MONGO_URI` = (Primary Cosmos DB connection string - or a dedicated Divi Cosmos DB)

Storage:
- [ ] `AZURE_STORAGE_CONNECTION_STRING` = (from step 7a)
- [ ] `AZURE_STORAGE_CONTAINER_NAME` = `attachments`

Encryption:
- [ ] `SECRET_ENCRYPTION_KEY` = (generate with `openssl rand -base64 32`)
- [ ] `ENCRYPTIONGOD_URL` = (from step 15a)

Inter-service URLs:
- [ ] `USERS_API_BASE_URL` = (from step 15b)
- [ ] `VITE_CHAT_HISTORY_API` = (from step 15c)
- [ ] `INGESTION_BACKEND_URL` = (from step 15e)

Azure OpenAI - Default:
- [ ] `AZURE_OPENAI_API_KEY` = (default OpenAI key)
- [ ] `AZURE_OPENAI_RESOURCE_NAME` = (default resource name)
- [ ] `AZURE_OPENAI_ENDPOINT` = (default endpoint)
- [ ] `AZURE_OPENAI_API_VERSION` = `2025-01-01-preview`
- [ ] `AZURE_OPENAI_DEPLOYMENT_NAME` = `gpt-4o-mini`
- [ ] `AZURE_DEFAULT_DEPLOYMENT_NAME` = `gpt-4o-mini`
- [ ] `AZURE_DEFAULT_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_DEFAULT_API_KEY` = (API key)
- [ ] `AZURE_DEFAULT_ENDPOINT` = (full endpoint with deployment path)
- [ ] `AZURE_DEFAULT_API_VERSION` = `2025-01-01-preview`

Azure OpenAI - GPT-4.1-mini:
- [ ] `AZURE_OPENAI_GPT_4_1_mini_DEPLOYMENT` = `gpt-4.1-mini`
- [ ] `AZURE_OPENAI_RESOURCE_NAME_4_1_mini` = (resource name)
- [ ] `AZURE_OPENAI_API_VERSION_4_1_mini` = `2024-12-01-preview`
- [ ] `AZURE_OPENAI_ENDPOINT_GPT_4_1_mini` = (endpoint URL)
- [ ] `AZURE_OPENAI_API_KEY_4_1_mini` = (API key)

Azure OpenAI - O3-Mini:
- [ ] `AZURE_OPENAI_GPT_O3_MINI_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_O3_MINI_API_VERSION` = `2025-01-01-preview`
- [ ] `AZURE_OPENAI_GPT_O3_MINI_ENDPOINT` = (full endpoint URL)
- [ ] `AZURE_OPENAI_GPT_O3_MINI_API_KEY` = (API key)

Azure OpenAI - O4-Mini:
- [ ] `AZURE_OPENAI_GPT_O4_MINI_DEPLOYMENT` = `o4-mini`
- [ ] `AZURE_OPENAI_GPT_O4_MINI_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_O4_MINI_API_VERSION` = `2025-01-01-preview`
- [ ] `AZURE_OPENAI_GPT_O4_MINI_ENDPOINT` = (full endpoint URL)
- [ ] `AZURE_OPENAI_GPT_O4_MINI_API_KEY` = (API key)

Azure OpenAI - GPT-4.5-Preview:
- [ ] `AZURE_OPENAI_GPT_4_5_PREVIEW_DEPLOYMENT` = `gpt-4.5-preview`
- [ ] `AZURE_OPENAI_GPT_4_5_PREVIEW_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_4_5_PREVIEW_API_VERSION` = `2024-12-01-preview`
- [ ] `AZURE_OPENAI_GPT_4_5_PREVIEW_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_4_5_PREVIEW_API_KEY` = (API key)

Azure OpenAI - GPT-5:
- [ ] `AZURE_OPENAI_GPT_5_DEPLOYMENT` = `gpt-5`
- [ ] `AZURE_OPENAI_GPT_5_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_5_API_VERSION` = `2025-04-01-preview`
- [ ] `AZURE_OPENAI_GPT_5_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_5_API_KEY` = (API key)

Azure OpenAI - GPT-5-Mini:
- [ ] `AZURE_OPENAI_GPT_5_MINI_DEPLOYMENT` = `gpt-5-mini`
- [ ] `AZURE_OPENAI_GPT_5_MINI_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_5_MINI_API_VERSION` = `2024-12-01-preview`
- [ ] `AZURE_OPENAI_GPT_5_MINI_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_5_MINI_API_KEY` = (API key)

Azure OpenAI - GPT-5.1:
- [ ] `AZURE_OPENAI_GPT_5_1_DEPLOYMENT` = `gpt-5.1`
- [ ] `AZURE_OPENAI_GPT_5_1_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_5_1_API_VERSION` = `2025-04-01-preview`
- [ ] `AZURE_OPENAI_GPT_5_1_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_5_1_API_KEY` = (API key)

Azure OpenAI - GPT-5.1-Chat:
- [ ] `AZURE_OPENAI_GPT_5_1_CHAT_DEPLOYMENT` = `gpt-5.1-chat`
- [ ] `AZURE_OPENAI_GPT_5_1_CHAT_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_5_1_CHAT_API_VERSION` = `2025-04-01-preview`
- [ ] `AZURE_OPENAI_GPT_5_1_CHAT_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_5_1_CHAT_API_KEY` = (API key)

Azure OpenAI - GPT-5.2:
- [ ] `AZURE_OPENAI_GPT_5_2_DEPLOYMENT` = `gpt-5.2`
- [ ] `AZURE_OPENAI_GPT_5_2_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_5_2_API_VERSION` = `2025-04-01-preview`
- [ ] `AZURE_OPENAI_GPT_5_2_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_5_2_API_KEY` = (API key)

Azure OpenAI - GPT-5.2-Chat:
- [ ] `AZURE_OPENAI_GPT_5_2_CHAT_DEPLOYMENT` = `gpt-5.2-chat`
- [ ] `AZURE_OPENAI_GPT_5_2_CHAT_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_5_2_CHAT_API_VERSION` = `2025-04-01-preview`
- [ ] `AZURE_OPENAI_GPT_5_2_CHAT_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_5_2_CHAT_API_KEY` = (API key)

Azure OpenAI - Image Generation:
- [ ] `AZURE_OPENAI_GPT_IMAGE_1_5_DEPLOYMENT` = `gpt-image-1.5`
- [ ] `AZURE_OPENAI_GPT_IMAGE_1_5_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_IMAGE_1_5_API_VERSION` = `2025-04-01-preview`
- [ ] `AZURE_OPENAI_GPT_IMAGE_1_5_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_IMAGE_1_5_API_KEY` = (API key)

Azure OpenAI - Model Router:
- [ ] `AZURE_OPENAI_MODEL_ROUTER_DEPLOYMENT` = `model-router`
- [ ] `AZURE_OPENAI_MODEL_ROUTER_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_MODEL_ROUTER_API_VERSION` = `2025-01-01-preview`
- [ ] `AZURE_OPENAI_MODEL_ROUTER_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_MODEL_ROUTER_API_KEY` = (API key)

Azure OpenAI - O3 Deep Research:
- [ ] `AZURE_OPENAI_GPT_O3_DEEP_RESEARCH_DEPLOYMENT` = `o3-deep-research`
- [ ] `AZURE_OPENAI_GPT_O3_DEEP_RESEARCH_RESOURCE_NAME` = (resource name)
- [ ] `AZURE_OPENAI_GPT_O3_DEEP_RESEARCH_API_VERSION` = `2025-01-01-preview`
- [ ] `AZURE_OPENAI_GPT_O3_DEEP_RESEARCH_ENDPOINT` = (endpoint URL)
- [ ] `AZURE_OPENAI_GPT_O3_DEEP_RESEARCH_API_KEY` = (API key)

DeepSeek Models:
- [ ] `AZURE_DEEPSEEK_R1_DEPLOYMENT` = `DeepSeek-R1-0528`
- [ ] `AZURE_DEEPSEEK_R1_RESOURCE_NAME` = (AI Hub resource name)
- [ ] `AZURE_DEEPSEEK_R1_API_VERSION` = `2024-05-01-preview`
- [ ] `AZURE_DEEPSEEK_R1_ENDPOINT` = (AI Services endpoint)
- [ ] `AZURE_DEEPSEEK_R1_API_KEY` = (AI Services key)
- [ ] `AZURE_DEEPSEEK_V3_DEPLOYMENT` = `DeepSeek-V3-0324`
- [ ] `AZURE_DEEPSEEK_V3_RESOURCE_NAME` = (AI Hub resource name)
- [ ] `AZURE_DEEPSEEK_V3_API_VERSION` = `2024-05-01-preview`
- [ ] `AZURE_DEEPSEEK_V3_ENDPOINT` = (AI Services endpoint)
- [ ] `AZURE_DEEPSEEK_V3_API_KEY` = (AI Services key)

MCP Server URLs:
- [ ] `DEEP_RESEARCH_MCP_SERVER_URL` = (from step 18)
- [ ] `DOCUMENT_GENERATION_MCP_SERVER_URL` = (from step 18)
- [ ] `TRANSCRIPTION_MCP_SERVER_URL` = (from step 18)

SearXNG:
- [ ] `SEARXNG_URL` = (from step 17)

Monitoring:
- [ ] `SENTRY_DSN` = (from step 19, or leave empty)
- [ ] `SENTRY_ENVIRONMENT` = `production`

Once deployed:
- [ ] Note the **Container App URL** - this is your main Divi backend URL
- [ ] **Go back to DiviAiSearch** and update `VITE_MODEL_API_BASE_URL` to this URL + `/api/v1`

---

## 16. Azure Static Web Apps (Frontend)

> The React frontend is deployed as a static web app.

### Create the Static Web App
- [ ] Go to **Static Web Apps > Create**
- [ ] **Name**: `<customer>-ai-frontend`
- [ ] **Resource group**: `<customer>-ai-suite-rg`
- [ ] **Hosting plan**: Standard (for custom domains + auth features)
- [ ] **Deployment source**: GitHub
- [ ] **Organization**: Your GitHub org
- [ ] **Repository**: `procurementsuitefrontend`
- [ ] **Branch**: `main`
- [ ] **Build preset**: React
- [ ] **App location**: `/`
- [ ] **Output location**: `dist`
- [ ] Click **Create**

### Configure Environment Variables (Application Settings)
- [ ] Go to Static Web App > **Configuration > Application settings**
- [ ] Add all `VITE_*` variables:
  - [ ] `VITE_MSAL_CLIENT_ID` = (from step 3)
  - [ ] `VITE_MSAL_AUTHORITY` = `https://login.microsoftonline.com/<tenant-id>`
  - [ ] `VITE_MODEL_API_BASE_URL` = (Divi URL through APIM, e.g., `https://<apim>.azure-api.net/divi/api/v1`)
  - [ ] `VITE_DIVI_AI_SEARCH_API_BASE_URL` = (AI Search URL through APIM)
  - [ ] `VITE_DIVI_AI_INGESTION_API_BASE_URL` = (Indexing URL through APIM)
  - [ ] `VITE_CHAT_HISTORY_API` = (History8 URL)
  - [ ] `VITE_USERS_API_BASE_URL` = (Gen-AI-Users URL)
  - [ ] `VITE_GRAPH_API_BASE_URL` = (Graph8 URL through APIM)
  - [ ] `VITE_TRANSCRIPTION_API_BASE_URL` = (Speech-to-Text URL through APIM)
  - [ ] `VITE_API_SUBSCRIPTION_KEY` = (APIM subscription key from step 13)
  - [ ] `VITE_AZURE_SPEECH_KEY` = (from step 12)
  - [ ] `VITE_AZURE_SPEECH_REGION` = (from step 12)
  - [ ] `VITE_SENTRY_DSN` = (from step 19, or leave empty)
  - [ ] `VITE_SENTRY_ENVIRONMENT` = `production`
  - [ ] `VITE_SENTRY_TUNNEL_URL` = (Gen-AI-Users URL + `/monitoring`)
  - [ ] `VITE_DEPLOYMENT_CONFIGURATION` = (URL to deployment config JSON - see step 22)
  - [ ] `VITE_DIVI_AI_SEARCH_MAX_AGENTS` = `200`

### Configure Custom Domain (if applicable)
- [ ] Go to **Custom domains > Add**
- [ ] Enter the customer's domain (e.g., `ai.customerdomain.com`)
- [ ] Follow the DNS validation steps (add CNAME or TXT record)
- [ ] Wait for SSL certificate to be auto-provisioned

### Update Azure AD Redirect URIs
- [ ] Go back to **Azure AD > App registrations > Your app > Authentication**
- [ ] Add the production redirect URI: `https://<customer-domain>/auth-redirect`
- [ ] Add the Static Web App default URL as well: `https://<static-web-app-name>.azurestaticapps.net/auth-redirect`

---

## 17. SearXNG (Web Search Engine)

> SearXNG is a privacy-respecting meta-search engine that the AI uses for web search capabilities.

### Deploy SearXNG as Container App
- [ ] Go to **Container Apps > Create**
- [ ] **Name**: `searxng`
- [ ] **Container Apps Environment**: Select `<customer>-ai-env`
- [ ] **Image source**: Docker Hub
- [ ] **Image**: `searxng/searxng:latest`
- [ ] **CPU**: 0.5, **Memory**: 1Gi
- [ ] **Ingress**: Enabled, External (or Internal if only Divi needs it), Port 8080
- [ ] **Min replicas**: 1, **Max replicas**: 2

**Environment Variables:**
- [ ] `SEARXNG_BASE_URL` = (the container app URL, fill after creation)

Once deployed:
- [ ] Note the **Container App URL** - this is your `SEARXNG_URL`
- [ ] **Go back to Divi** and update `SEARXNG_URL`

---

## 18. MCP Servers (External Tooling)

> MCP (Model Context Protocol) servers provide extended capabilities to the AI. You need to deploy 3 MCP servers.

### 18a. Deep Research MCP Server

- [ ] Deploy as Container App (check if there's a separate repo/image for this)
- [ ] **Name**: `deep-research-mcp`
- [ ] **Ingress**: External, Port 3000 (or whatever the service uses)
- [ ] Configure with necessary environment variables (OpenAI keys, etc.)
- [ ] Note the URL - this is your `DEEP_RESEARCH_MCP_SERVER_URL` (append `/api/mcp`)

### 18b. Document Generation MCP Server

- [ ] Deploy as Container App
- [ ] **Name**: `mcp-document-generator`
- [ ] **Ingress**: External, Port 3000
- [ ] Configure with necessary environment variables
- [ ] Note the URL - this is your `DOCUMENT_GENERATION_MCP_SERVER_URL` (append `/mcp`)

### 18c. Transcription MCP Server

> This is provided by the GenAiSpeechToText service itself.

- [ ] The URL is the Speech-to-Text Container App URL + `/api/mcp`
- [ ] This is your `TRANSCRIPTION_MCP_SERVER_URL`

### Update Divi's MCP Configuration
- [ ] After all MCP servers are deployed, update Divi's environment variables with the correct URLs
- [ ] Restart the Divi container app to pick up new MCP server URLs

---

## 19. Sentry (Error Monitoring)

> Sentry captures errors and performance data from all services. You can use the hosted version or self-host.

### Option A: Use Existing Sentry Instance
- [ ] Get access to the existing Sentry instance (or create a new one at sentry.io)
- [ ] Create a new **Organization** for the customer (or use existing)

### Option B: Self-Hosted Sentry
- [ ] Deploy self-hosted Sentry (if customer requires on-prem monitoring)

### For Either Option:
- [ ] Create Sentry **Projects** for each service:
  - [ ] `divi-backend` (Node.js project)
  - [ ] `divi-history` (Node.js project)
  - [ ] `gen-ai-users` (Node.js project)
  - [ ] `procurementsuitefrontend` (React project)
- [ ] For each project, copy the **DSN** (Data Source Name)
- [ ] Create an **Auth Token** for CI/CD (source map uploads)
- [ ] Note down:
  - `SENTRY_DSN` for each service
  - `SENTRY_AUTH_TOKEN` for CI/CD
  - `SENTRY_ORG` organization slug
  - `SENTRY_PROJECT` project slug
  - `SENTRY_URL` self-hosted URL (if applicable)

---

## 20. Email / SMTP Setup

> Used for sending feedback notifications and system emails.

### Option A: Office 365 / Microsoft 365
- [ ] Create or designate a shared mailbox (e.g., `noreply@customer.com` or `automation@customer.com`)
- [ ] Ensure SMTP AUTH is enabled for this mailbox in Exchange Admin Center
- [ ] Note down:
  - `SMTP_HOST` = `smtp.office365.com`
  - `SMTP_PORT` = `587`
  - `SMTP_SECURE` = `false`
  - `SMTP_USER` = (the mailbox email)
  - `SMTP_PASS` = (the mailbox password or app password)
  - `SMTP_FROM` = `Feedback <automation@customer.com>`

### Option B: Azure Communication Services (Alternative)
- [ ] Create an Azure Communication Services resource
- [ ] Configure email sending
- [ ] Note connection details

---

## 21. Linear Integration (Issue Tracking)

> Optional - Used to automatically create support tickets from user feedback.

- [ ] Create a Linear workspace (or use existing customer's workspace)
- [ ] Go to **Settings > API > Personal API Keys**
- [ ] Create a new API key
- [ ] Note: `LINEAR_API_KEY` = (the generated key)
- [ ] Create or identify the project: `LINEAR_PROJECT_TENANT` = (project name)

> **If not needed**: Leave `LINEAR_API_KEY` empty and the feature will be disabled.

---

## 22. Deployment Configuration Repository

> The frontend loads its configuration (feature flags, branding, enabled features) from a JSON config file hosted externally.

- [ ] Create a GitHub repository for customer configuration (e.g., `customer-ai-config`)
- [ ] Create a config JSON file (e.g., `production_config.json`) based on the existing config format
- [ ] The config typically includes:
  - [ ] Enabled features / feature flags
  - [ ] Branding / theme settings
  - [ ] Available AI models for the frontend
  - [ ] Custom CSS URL (if any)
- [ ] Push to GitHub and get the raw URL
- [ ] Set `VITE_DEPLOYMENT_CONFIGURATION` to the raw GitHub URL (or host it elsewhere)

---

## 23. CI/CD - GitHub Actions

> Automated deployments on git push. Each service has its own workflow.

### 23a. Set Up GitHub-Azure OIDC Authentication

> Modern GitHub Actions use OIDC (OpenID Connect) to authenticate with Azure instead of storing long-lived secrets.

- [ ] Go to **Azure AD > App registrations > New registration**
- [ ] **Name**: `GitHub Actions OIDC`
- [ ] Register it
- [ ] Go to **Certificates & secrets > Federated credentials > Add credential**
  - **Federated credential scenario**: GitHub Actions
  - **Organization**: Your GitHub org name
  - **Repository**: Each repo name
  - **Entity type**: Branch (`main` for production, `staging` for staging)
- [ ] Repeat for each repository/branch combination
- [ ] Assign the service principal the **Contributor** role on the resource group(s)

### 23b. Configure GitHub Secrets

> In each GitHub repository, go to Settings > Secrets and variables > Actions and add:

**Common Secrets (all repos):**
- [ ] `AZURE_CLIENT_ID` = (OIDC app registration Client ID)
- [ ] `AZURE_TENANT_ID` = (Azure Tenant ID)
- [ ] `AZURE_SUBSCRIPTION_ID` = (Azure Subscription ID)

**Container App Repos (divi, diviaisearch, GenAiSpeechToText, encryptiongod, gen-ai-users, graph8, history8):**
- [ ] `REGISTRY_LOGIN_SERVER` = `<customer>genairegistry.azurecr.io`
- [ ] `REGISTRY_USERNAME` = (ACR admin username)
- [ ] `REGISTRY_PASSWORD` = (ACR admin password)
- [ ] `RESOURCE_GROUP` = `<customer>-ai-suite-rg`
- [ ] `CONTAINER_APP_NAME` = (the container app name for this service)

**Frontend Repo (procurementsuitefrontend):**
- [ ] `AZURE_STATIC_WEB_APPS_API_TOKEN` = (from Static Web App > Manage deployment token)
- [ ] `SENTRY_AUTH_TOKEN` = (from step 19)
- [ ] All `VITE_*` variables as secrets (or use a `.env.production` approach)

### 23c. Create/Update Workflow Files

- [ ] For each service, create or update the GitHub Actions workflow file in `.github/workflows/`
- [ ] Ensure the workflow:
  - [ ] Triggers on push to `main` (production) and `staging` (staging)
  - [ ] Builds the Docker image
  - [ ] Pushes to ACR
  - [ ] Deploys to the correct Container App
- [ ] For the frontend:
  - [ ] Triggers on push to `main`
  - [ ] Builds the React app with Vite
  - [ ] Deploys to Azure Static Web Apps

---

## 24. DNS & Custom Domains

### Configure Custom Domain for Frontend
- [ ] Add a CNAME record in customer's DNS:
  - `ai.customerdomain.com` -> `<static-web-app>.azurestaticapps.net`
- [ ] Or an A record if using apex domain
- [ ] Wait for DNS propagation (can take up to 48 hours, usually much faster)

### Configure Custom Domain for APIM (Optional)
- [ ] Add a CNAME record:
  - `api.customerdomain.com` -> `<apim-name>.azure-api.net`
- [ ] Upload an SSL certificate for the custom domain in APIM

### Configure Custom Domain for Container Apps (Optional)
> Usually not needed if using APIM as the gateway, but useful for direct access.

- [ ] For each Container App that needs a custom domain:
  - Go to **Custom domains > Add**
  - Follow the CNAME/TXT validation steps

---

## 25. Security Hardening & Final Review

### Network Security
- [ ] Review all Container Apps - set services that don't need direct public access to **Internal** ingress
  - EncryptionGod: Can be internal (only other services call it)
  - History8: Can be internal if accessed through APIM
  - Graph8: Can be internal if accessed through APIM
- [ ] Enable **IP restrictions** on APIM if needed (whitelist customer IPs)
- [ ] Review and tighten **CORS policies** on all services

### Secret Management
- [ ] Verify no secrets are hardcoded in any config files
- [ ] Ensure all API keys are stored as **Container App secrets** (not plain env vars)
- [ ] Rotate all default/initial passwords and keys
- [ ] Set up a key rotation schedule (especially for Azure AD client secrets - they expire!)

### Authentication & Authorization
- [ ] Verify Azure AD token validation is working on all backend services
- [ ] Test that unauthenticated requests are rejected
- [ ] Test RBAC - verify `agents.manager` role works correctly
- [ ] Verify user group-based access control in DiviAiSearch

### Data Protection
- [ ] Verify Cosmos DB has encryption at rest enabled (it's on by default)
- [ ] Verify all service-to-service communication uses HTTPS
- [ ] Verify Key Vault access policies are minimal (principle of least privilege)
- [ ] Check that Azure Storage uses HTTPS-only connections

### Monitoring & Alerting
- [ ] Set up Azure Monitor alerts for:
  - [ ] Container App failures / restarts
  - [ ] High CPU/memory usage
  - [ ] 5xx error rates
  - [ ] Cosmos DB RU consumption (to avoid throttling)
  - [ ] OpenAI API quota usage
- [ ] Verify Sentry is receiving errors from all services
- [ ] Set up Sentry alert rules for critical errors

### Backup & Recovery
- [ ] Enable **Cosmos DB continuous backup** (point-in-time restore)
- [ ] Verify Azure Storage has **soft delete** enabled
- [ ] Document the disaster recovery process

---

## 26. End-to-End Testing & Verification

### Pre-Flight Checks
- [ ] All Container Apps show "Running" status with healthy replicas
- [ ] All Container Apps have successful health checks
- [ ] Frontend loads at the custom domain (or Static Web App URL)

### Authentication Flow
- [ ] Open the frontend URL in a browser
- [ ] Click "Sign In" - verify Azure AD login page appears
- [ ] Sign in with a test user - verify redirect back to app works
- [ ] Verify user profile loads correctly (name, email, avatar)

### Core Chat Functionality
- [ ] Send a simple chat message - verify AI responds
- [ ] Verify streaming works (text appears word by word, not all at once)
- [ ] Test different AI models (switch models in the UI and send messages)
- [ ] Verify conversation history is saved and loads on refresh
- [ ] Test creating a new conversation
- [ ] Test deleting a conversation

### Document Search (RAG)
- [ ] Upload a test document (PDF, DOCX, or PPTX)
- [ ] Wait for indexing to complete
- [ ] Ask a question about the document content
- [ ] Verify the AI references the document in its answer

### Speech-to-Text
- [ ] Test the transcription feature with an audio recording
- [ ] Verify the transcription appears correctly
- [ ] Test with different languages (if applicable)

### Encryption
- [ ] Send a message in chat
- [ ] Verify the message is encrypted in the database (check Cosmos DB - messages should not be plain text)
- [ ] Verify the message decrypts correctly when viewing chat history

### Image Generation
- [ ] Ask the AI to generate an image
- [ ] Verify the image appears in the chat

### Web Search
- [ ] Ask the AI a question that requires current information
- [ ] Verify the AI performs a web search via SearXNG
- [ ] Verify search results are incorporated into the response

### Document Generation
- [ ] Ask the AI to create a document (Word, Excel, or PDF)
- [ ] Verify the document is generated and downloadable

### User Management
- [ ] Create a user group
- [ ] Add users to the group
- [ ] Verify group-based access control works (e.g., agents visible only to group members)

### Performance
- [ ] Verify chat responses start streaming within 2-3 seconds
- [ ] Test with 5+ concurrent users (if possible)
- [ ] Check Container App scaling - verify new replicas spin up under load

### Error Handling
- [ ] Verify errors appear in Sentry
- [ ] Test the feedback form - verify it creates a Linear ticket (if configured)
- [ ] Test email notifications (if configured)

---

## Quick Reference - All Services & Their URLs

Once everything is deployed, fill in this table:

| Service | Container App Name | Internal URL | External/APIM URL |
|---------|-------------------|-------------|-------------------|
| Divi (LLM) | `divi` | | |
| AI Search (Search) | `divi-ai-search-app` | | |
| AI Search (Indexing) | `divi-ai-search-indexing` | | |
| Speech-to-Text | `speech-to-text` | | |
| EncryptionGod | `encryption-backend` | | |
| Gen-AI-Users | `gen-ai-user-api` | | |
| Graph8 | `graph8` | | |
| History8 | `gen-ai-history-api` | | |
| SearXNG | `searxng` | | |
| Deep Research MCP | `deep-research-mcp` | | |
| Doc Generation MCP | `mcp-document-generator` | | |
| Frontend | (Static Web App) | N/A | |

---

## Estimated Azure Resources Cost (Monthly)

> These are rough estimates. Actual costs depend heavily on usage.

| Resource | SKU | Estimated Cost |
|----------|-----|---------------|
| Azure OpenAI (all models) | S0 | $500 - $5,000+ (usage-based) |
| Cosmos DB (2 accounts) | Serverless or 400 RU/s | $50 - $200 |
| Container Apps (8-10 apps) | Consumption | $100 - $500 |
| Azure AI Search | Basic/Standard | $75 - $250 |
| Blob Storage (2 accounts) | Standard LRS | $10 - $50 |
| Key Vault | Standard | $5 - $20 |
| Speech Services | S0 | $20 - $100 |
| API Management | Basic/Standard | $50 - $350 |
| Static Web App | Standard | $9 |
| Container Registry | Standard | $20 |
| Log Analytics | Pay-as-you-go | $20 - $100 |
| **Total Estimate** | | **$860 - $6,600+/month** |

> The biggest variable cost is Azure OpenAI. Heavy usage of GPT-5 / GPT-5.2 can get expensive fast.

---

## Troubleshooting Common Issues

### "AADSTS50011: The redirect URI does not match"
- Go to Azure AD > App registrations > Authentication
- Make sure the exact redirect URI (including `/auth-redirect`) is listed as a SPA redirect

### Container App shows "Unhealthy"
- Check Container App > Log stream for errors
- Most common cause: missing or wrong environment variables
- Check the health check endpoint is responding

### "Unauthorized" errors from backend services
- Verify `AZURE_TENANT_ID` and `AZURE_CLIENT_ID` are correct in all services
- Check that the JWT token audience matches the app registration's Client ID
- Verify the token hasn't expired

### AI responses are empty or error out
- Check Azure OpenAI deployment names match the environment variables exactly
- Verify the API keys are correct
- Check if you've exceeded the tokens-per-minute quota

### Documents fail to index
- Check Azure Search service is running and the admin key is correct
- Verify the embeddings model (`text-embedding-3-large`) is deployed and accessible
- Check blob storage permissions

### SearXNG returns no results
- Verify the SearXNG container is running
- Check if the container has outbound internet access
- Some search engines may block Azure IP ranges

---

**Total checklist items: ~300+**

**Estimated time to complete full deployment: 2-4 days** (depending on experience and Azure provisioning times)
