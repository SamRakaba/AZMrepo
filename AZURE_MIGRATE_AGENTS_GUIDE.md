# Azure Migrate CSV Processing Agents - Copilot Studio Guide

A comprehensive guide for building Copilot Studio agents that use LLM-based analysis to process Azure Migrate export CSV files, consolidate application, SQL Server, and web application inventories, and generate downloadable reports.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Agent 1: File Upload Handler](#agent-1-file-upload-handler)
5. [Agent 2: Application Inventory Processor](#agent-2-application-inventory-processor)
6. [Agent 3: SQL Server Inventory Processor](#agent-3-sql-server-inventory-processor)
7. [Agent 4: Web App Inventory Processor](#agent-4-web-app-inventory-processor)
8. [Agent 5: Report Generator](#agent-5-report-generator)
9. [Orchestrating the Agents](#orchestrating-the-agents)
10. [Power Automate Flows](#power-automate-flows)
    - [Understanding HTTP Action URI Values](#understanding-http-action-uri-values)
11. [Testing and Validation](#testing-and-validation)
12. [Troubleshooting](#troubleshooting)
13. [Resources](#resources)

---

## Overview

### Purpose

This guide provides step-by-step instructions for building a suite of Microsoft Copilot Studio agents that:

1. **Accept file uploads** - Allow users to upload one or more Azure Migrate export CSV files
2. **Process Application Inventory** - Use the Copilot agent's LLM reasoning to analyze the `ApplicationInventory` sheet, identify noise (patches, updates, OS components, drivers), consolidate applications by exact name match, and preserve version variants as separate entries
3. **Process SQL Server Inventory** - Use the Copilot agent's LLM reasoning to analyze SQL Server sheets, consolidate instances, group by version, remove updates and dependent clients, and generate a unique list of SQL Server versions and entries
4. **Process Web App Inventory** - Use the Copilot agent's LLM reasoning to analyze web application/web server sheets and produce a unique list of web apps
5. **Generate Reports** - Create a consolidated spreadsheet with sheets for unique applications, SQL Server instances, and web apps
6. **Enable Download** - Provide users with a downloadable link to the generated spreadsheet

> **Key Design Principle:** Agents 2, 3, and 4 use the Copilot agent's LLM model to perform all analysis, consolidation, and noise detection. Power Automate is used **only** when needed — specifically for reading raw data from files and writing results to spreadsheets. This LLM-first approach replaces the previous pattern-matching and loop-based Power Automate logic with intelligent reasoning.

### Azure Migrate CSV File Structure

The Azure Migrate export files contain the following sheets:

#### ApplicationInventory Sheet
| Column | Description |
|--------|-------------|
| MachineName | Name of the machine |
| Application | Application name |
| Version | Application version |
| Provider | Application provider/vendor |
| MachineManagerFqdn | Fully qualified domain name of the machine manager |

#### SQL Server Sheet
| Column | Description |
|--------|-------------|
| MachineName | Name of the machine |
| Instance Name | SQL Server instance name |
| Edition | SQL Server edition |
| Service Pack | Service pack level |
| Version | SQL Server version |
| Port | SQL Server port |
| MachineManagerFqdn | Fully qualified domain name of the machine manager |

#### Database Sheet (Other Databases)
| Column | Description |
|--------|-------------|
| MachineName | Name of the machine |
| Database Type | Type of database (Oracle, MySQL, PostgreSQL, etc.) |
| Version | Database version |
| MachineManagerFqdn | Fully qualified domain name of the machine manager |

#### Web Applications Sheet
| Column | Description |
|--------|-------------|
| MachineName | Name of the machine |
| WebServerType | Web server type (IIS, Apache, Tomcat, Nginx, etc.) |
| WebAppName | Name of the web application |
| VirtualDirectory | Virtual directory path |
| ApplicationPool | Application pool name (IIS) |
| FrameworkVersion | Framework or runtime version (e.g., .NET 4.8, PHP 7.4) |
| MachineManagerFqdn | Fully qualified domain name of the machine manager |

---

## Architecture

### Agent Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              User Interface                                  │
│                        (Copilot Studio Chat Interface)                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AGENT 1: File Upload Handler                            │
│  • Accepts CSV file uploads                                                 │
│  • Validates file format                                                    │
│  • Stores files in temporary storage (Azure Blob Storage recommended)       │
│  • Coordinates LLM-based processing via topic redirects                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────────┐
│ AGENT 2: App Inventory│ │ AGENT 3: SQL Server   │ │ AGENT 4: Web App      │
│ (LLM-based analysis)  │ │ (LLM-based analysis)  │ │ (LLM-based analysis)  │
│ • Read raw app data   │ │ • Read raw SQL data   │ │ • Read raw web data   │
│   (Power Automate)    │ │   (Power Automate)    │ │   (Power Automate)    │
│ • LLM identifies noise│ │ • LLM consolidates    │ │ • LLM consolidates    │
│ • LLM consolidates by │ │   by version          │ │   web apps            │
│   exact name match    │ │ • LLM removes updates │ │ • LLM removes noise   │
│ • LLM preserves       │ │   & dependent clients │ │ • Generate unique list│
│   version variants    │ │ • Generate unique list│ │                       │
│ • Generate unique list│ │                       │ │                       │
└───────────────────────┘ └───────────────────────┘ └───────────────────────┘
                    │                 │                 │
                    └─────────────────┼─────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AGENT 5: Report Generator                               │
│  • Combine processed data from all agents                                   │
│  • Create multi-sheet Excel file (Power Automate)                           │
│  • Store in temporary storage (Azure Blob Storage or SharePoint/OneDrive)   │
│  • Generate download link                                                   │
│  • Notify user with download URL                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Copilot Agent Orchestration                              │
│  • Agent topics coordinate processing sequence                              │
│  • LLM performs all analysis and consolidation                              │
│  • Power Automate used only for file I/O                                    │
│  • Error handling and user notifications                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| File Upload Handler | Accept CSV files, store in temp storage, coordinate processing topics | Copilot Studio + Power Automate (file storage only) |
| Temporary Storage | Store uploaded files temporarily | Azure Blob Storage (recommended) or SharePoint |
| App Inventory Processor | LLM-based application consolidation and noise detection | Copilot Studio LLM + Power Automate (data read only) |
| SQL Server Processor | LLM-based SQL Server consolidation and version grouping | Copilot Studio LLM + Power Automate (data read only) |
| Web App Processor | LLM-based web application consolidation | Copilot Studio LLM + Power Automate (data read only) |
| Report Generator | Create final spreadsheet with consolidated data | Power Automate + Excel Connector |
| Orchestration | Agent topics coordinate workflow via LLM | Copilot Studio Topics |

---

## Prerequisites

### Required Access

**For IT Admins / Developers** (one-time setup):

1. **Microsoft 365 License** with:
   - Microsoft Copilot Studio access
   - Power Automate access
   - Microsoft Excel Online

2. **Temporary Storage Setup** (choose one option):
   - **Option A: Azure Blob Storage (Recommended)** - Create ONE shared storage account for all users
   - **Option B: SharePoint/OneDrive** - Requires users have SharePoint or OneDrive access

3. **Permissions for Setup**:
   - Copilot Studio environment creator/maker
   - Power Automate flow creator
   - Azure Storage Blob Data Contributor (for Option A - admin only)
   - SharePoint site contributor (for Option B)

4. **Environment Setup**:
   - Azure Storage Account with containers (for Option A)
   - A dedicated SharePoint site or OneDrive folder (for Option B)
   - Azure AD application (optional, for advanced authentication)

**For End Users** (using the agent):

| Storage Option | What Users Need |
|----------------|-----------------|
| **Option A: Azure Blob Storage** | ✅ NO Azure access needed - users only interact through Copilot chat |
| **Option B: SharePoint** | ⚠️ Users need SharePoint/OneDrive access to download reports |

### Option A: Prepare Azure Blob Storage (Recommended - No SharePoint/OneDrive Required)

Azure Blob Storage provides temporary file storage without requiring users to have SharePoint or OneDrive access. This is the recommended approach for scenarios where:
- Users don't have SharePoint/OneDrive licenses
- You need isolated temporary storage for processing
- You want to avoid SharePoint storage quota constraints
- You need programmatic file retention policies

#### Access Model - Who Needs What Access?

> **Important**: End users do NOT need direct access to Azure Blob Storage. The organization creates ONE shared storage account, and the Power Automate flows handle all storage operations using service credentials.

| Role | Access Required | What They Do |
|------|-----------------|--------------|
| **IT Admin / Developer** | Azure Storage Blob Data Contributor | Creates the storage account ONCE, configures containers, sets up Power Automate connections |
| **End Users** | NO Azure storage access needed | Simply interact with the Copilot agent - upload files through chat, receive download links |
| **Power Automate (Service)** | Storage connection with SAS token or Managed Identity | Automatically reads/writes files on behalf of users |

**How It Works:**
1. **One-time setup**: An IT admin or developer creates a single Azure Storage Account for the organization
2. **Shared storage**: All users share this storage account - files are isolated by session IDs
3. **No user credentials needed**: Users never see or access the storage directly
4. **Secure access**: Power Automate flows use pre-configured connections (SAS tokens or Managed Identity) to access storage
5. **Automatic cleanup**: Lifecycle management policies automatically delete temporary files after 7 days

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│   End User      │────▶│  Copilot Agent   │────▶│  Power Automate     │
│ (No Azure access│     │  (Chat interface)│     │  (Service account)  │
│    needed)      │     │                  │     │                     │
└─────────────────┘     └──────────────────┘     └──────────┬──────────┘
                                                            │
                                                            ▼
                                               ┌─────────────────────┐
                                               │  Azure Blob Storage │
                                               │  (Shared by all     │
                                               │   users, isolated   │
                                               │   by session ID)    │
                                               └─────────────────────┘
```

#### Setup Instructions (For IT Admin / Developer)

1. **Create an Azure Storage Account**:
   ```
   Resource Group: rg-azure-migrate-processing
   Storage Account Name: stazuremigrateproc<unique-suffix> (must be globally unique, e.g., stazuremigrateproc123)
   Region: (Select your preferred region)
   Performance: Standard
   Redundancy: LRS (Locally-redundant storage) for temporary files
   ```

2. **Create Blob Containers**:
   ```
   Container: uploads
   Public access level: Private (no anonymous access)
   
   Container: processing
   Public access level: Private (no anonymous access)
   
   Container: reports
   Public access level: Private (no anonymous access)
   ```

3. **Set Up Folder Structure (Virtual Directories)**:
   ```
   uploads/
       └── {SessionID}/
           └── raw files here
   processing/
       └── {SessionID}/
           ├── applications_consolidated.json
           ├── sql_consolidated.json
           └── webapps_consolidated.json
   reports/
       └── {SessionID}/
           └── ConsolidatedReport_{timestamp}.xlsx
   ```

4. **Configure Lifecycle Management (Auto-cleanup)**:
   Create a lifecycle management rule to automatically delete temporary files:
   ```json
   {
     "rules": [
       {
         "name": "DeleteTempFilesAfter7Days",
         "enabled": true,
         "type": "Lifecycle",
         "definition": {
           "filters": {
             "blobTypes": ["blockBlob"],
             "prefixMatch": ["uploads/", "processing/"]
           },
           "actions": {
             "baseBlob": {
               "delete": {
                 "daysAfterModificationGreaterThan": 7
               }
             }
           }
         }
       }
     ]
   }
   ```

5. **Generate SAS Token or Configure Connection** for Power Automate:
   - Navigate to your Storage Account → Shared access signature
   - Configure permissions: Read, Write, Delete, List, Add, Create
   - Set expiry date appropriately
   - Generate SAS token for use in Power Automate

### Option B: Prepare SharePoint Storage (Alternative)

If you prefer to use SharePoint for storage (requires users have SharePoint access):

1. **Create a SharePoint Site** (or use existing):
   ```
   Site Name: Azure-Migrate-Processing
   Template: Team Site
   ```

2. **Create Document Libraries**:
   - `Uploads` - For incoming CSV files
   - `Processing` - For intermediate files
   - `Reports` - For generated output files

3. **Set Up Folder Structure**:
   ```
   /Uploads/
       └── {SessionID}/
           └── raw files here
   /Processing/
       └── {SessionID}/
           ├── applications_consolidated.json
           ├── sql_consolidated.json
           └── webapps_consolidated.json
   /Reports/
       └── {SessionID}/
           └── ConsolidatedReport_{timestamp}.xlsx
   ```

---

## Agent 1: File Upload Handler

### Purpose
This agent is the **starting point of the Azure Migrate processing flow**. It provides users with instructions to upload Azure Migrate extracted CSV files, accepts the file uploads, validates them, stores files in temporary storage (Azure Blob Storage recommended), and coordinates the LLM-based processing sequence by redirecting to the analysis topics (Agents 2, 3, 4) in order.

> **Important**: This agent is:
> - **NOT conversational** - It follows a structured flow without general chat capabilities
> - **Triggered on file(s) upload** - The main flow activates when users upload files
> - **The coordinator of the processing flow** - After upload, it triggers each LLM analysis topic in sequence (Application → SQL Server → Web App → Report)
> - Designed to work with Azure Blob Storage as temporary storage, which does NOT require users to have SharePoint or OneDrive access

### Storage Options

| Option | User Requirements | Best For |
|--------|-------------------|----------|
| **Azure Blob Storage (Recommended)** | No SharePoint/OneDrive access required | Organizations where users don't have SharePoint access, or need isolated temporary storage |
| **SharePoint/OneDrive** | Users must have SharePoint/OneDrive access | Organizations already using SharePoint with appropriate user licenses |

---

### Step 1: Create the Agent in Copilot Studio

#### Step 1.1: Access Copilot Studio

1. Open your web browser (Microsoft Edge, Chrome, or Firefox recommended)
2. Navigate to the URL: **https://copilotstudio.microsoft.com**
3. If prompted, sign in with your Microsoft 365 credentials:
   - Enter your email address in the **Sign in** field
   - Click the **Next** button
   - Enter your password in the **Password** field
   - Click the **Sign in** button
4. If prompted for multi-factor authentication, complete the verification process
5. Wait for the Copilot Studio home page to load completely

#### Step 1.2: Select Your Environment

1. Look at the top-right corner of the Copilot Studio interface
2. Click on the **Environment selector** dropdown (shows your current environment name)
3. From the dropdown list, select your target environment:
   - Select **Production** for production agents
   - Select **Development** or **Sandbox** for testing
4. Wait for the environment to switch (the page will refresh)

#### Step 1.3: Create a New Agent

1. On the Copilot Studio home page, locate the left navigation panel
2. Click on **Copilots** in the left navigation menu
3. On the Copilots page, click the **+ Create** button in the top-left area
4. A dropdown menu will appear with options:
   - Click on **New copilot**
5. The "Create a copilot" wizard will open

#### Step 1.4: Configure Basic Agent Settings

1. In the "Create a copilot" wizard, you will see several fields to fill out:

2. **Name field**:
   - Click in the **Name** text field
   - Type exactly: `Azure Migrate File Handler`
   - This name will be displayed to users and in the agent list

3. **Description field**:
   - Click in the **Description** text field
   - Type exactly: `Handles Azure Migrate CSV file uploads and initiates processing workflow for application, SQL Server, and web app inventory consolidation`

4. **Language dropdown**:
   - Click on the **Language** dropdown
   - Scroll through the list and select **English (en-US)**
   - Note: You can select additional languages later if needed

5. **Icon (optional)**:
   - Click on the **Icon** area if you want to upload a custom agent icon
   - Click **Upload** to select an image file (PNG, JPG, or SVG format)
   - Recommended size: 48x48 pixels or larger
   - Skip this step if you want to use the default icon

6. **Review all settings** before proceeding:
   ```
   Name: Azure Migrate File Handler
   Description: Handles Azure Migrate CSV file uploads and initiates processing workflow for application, SQL Server, and web app inventory consolidation
   Language: English (en-US)
   ```

7. Click the **Create** button at the bottom of the wizard

8. Wait for the agent to be created (this may take 10-30 seconds)

9. You will be automatically redirected to the agent's configuration page

---

### Step 2: Configure Agent Settings

#### Step 2.1: Navigate to Agent Settings

1. With your new agent open, look at the left navigation panel
2. Click on **Settings** in the left navigation menu
3. The Settings page will open with multiple tabs/sections

#### Step 2.2: Configure Generative AI Settings

1. In the Settings page, click on the **Generative AI** tab/section
2. You will see options for configuring AI behavior

3. **Orchestration setting**:
   - Locate the **Orchestration** option
   - You will see two options: **Classic** and **Generative**
   - Click on **Classic** to select it
   - **Important**: Classic mode ensures the agent follows structured conversation flows rather than generating free-form responses

4. **Generative answers setting**:
   - Locate the **Generative answers** toggle
   - Click the toggle to turn it **OFF** (disabled)
   - This prevents the agent from generating answers outside of your defined topics

5. **Boost conversations setting**:
   - Locate the **Boost conversations** toggle
   - Click the toggle to turn it **OFF** (disabled)
   - This ensures the agent strictly follows your conversation design

6. Click the **Save** button at the top of the page

#### Step 2.3: Configure Agent Details and Instructions

1. In the Settings page, click on the **Agent details** tab/section (may also be labeled "Details" or "Copilot details")

2. Scroll down to find the **Instructions** section (may also be labeled "Agent instructions" or "System prompt")

3. Click in the **Instructions** text area

4. Clear any existing text and paste the following instructions exactly:

```
You are the Azure Migrate File Handler agent. Your role is to:

1. GUIDE users to upload Azure Migrate extracted CSV files
2. ACCEPT file uploads (CSV or Excel format) containing Azure Migrate export data
3. VALIDATE uploaded files have the expected format
4. COORDINATE the LLM-based processing workflow after successful upload

IMPORTANT BEHAVIOR:
- You are NOT a general-purpose conversational agent
- You should ONLY handle Azure Migrate CSV file upload requests
- Always start by providing clear instructions for file upload
- Do NOT engage in off-topic conversations
- If users ask unrelated questions, redirect them to upload their files

EXPECTED FILE FORMATS:
- CSV files exported from Azure Migrate
- Excel files (.xlsx) with sheets: ApplicationInventory, SQL Server, WebApplications
  (The file may also contain a Database sheet — this is accepted but not processed by
  the current agents)

PROCESSING ARCHITECTURE (LLM-first approach):
- After the file is uploaded and stored, you coordinate the processing sequence
  by triggering each analysis topic in order
- Agents 2, 3, and 4 use your LLM reasoning to analyze raw data from the file.
  Power Automate is used ONLY for reading raw data from files and writing results
  to spreadsheets — all consolidation and noise detection is performed by LLM
  analysis within Copilot Studio topics
- The processing sequence is:
  1. "Process Application Inventory" topic — LLM analyzes and consolidates apps
  2. "Process SQL Server Inventory" topic — LLM consolidates SQL instances
  3. "Process Web App Inventory" topic — LLM consolidates web apps
  4. Report generation — consolidated data written to a spreadsheet

WORKFLOW:
1. Greet the user and explain the purpose
2. Provide instructions for uploading Azure Migrate CSV files
3. Wait for file upload
4. Store the uploaded file and record the file path
5. Trigger the processing topics in sequence (Application → SQL → Web App → Report)
6. Provide the user with a download link to the consolidated report

RESPONSE STYLE:
- Be concise and professional
- Use bullet points for instructions
- Include emojis sparingly for visual clarity (📋, ✅, ⬆️, 📁)
- Always confirm actions and next steps
```

5. Click the **Save** button at the top of the page

6. Wait for the confirmation message "Settings saved" to appear

---

### Step 3: Create Global Variables

Variables store information during the conversation. In Copilot Studio, variables are created **within topics** using nodes on the authoring canvas. To make a variable available across all topics (global), you change its scope in the variable properties panel.

> **Note**: Copilot Studio does not have a separate "Variables" settings page. Variables are created within topic flows using "Set a variable value" nodes, and then their scope is changed to "Global" to make them accessible across all topics.

#### Step 3.1: Create a Setup Topic for Global Variables

We'll create a dedicated topic to initialize all global variables needed for the agent.

##### Step 3.1.1: Create the Initialization Topic

1. In the left navigation panel, click on **Topics**
2. Click the **+ Add a topic** button at the top
3. From the dropdown menu, select **From blank**
4. A new blank topic canvas will open

##### Step 3.1.2: Configure Topic Name

1. At the top of the topic canvas, click on **Untitled** to edit it
2. Type the topic name: `Initialize Global Variables`
3. Press **Enter** to confirm the name

##### Step 3.1.3: Add Trigger Phrase

1. In the **Trigger phrases** section, add a single trigger:
   - Type: `Initialize session` and press **Enter**

> **Note**: This topic will be called programmatically at the start of conversations rather than triggered by user input directly.

##### Step 3.1.4: Create the sessionId Variable

1. Below the trigger phrases section, click the **+** button to add a node
2. From the dropdown menu, select **Variable management** → **Set a variable value**
3. In the "Set variable" node that appears:
   - Click on **Set variable** dropdown
   - Click **Create new**
   - In the variable name field, type: `sessionId`
   - Click **Create** or press **Enter**
4. For the **To value** field:
   - Click the field and type a placeholder value: `""` (empty string)
   - This will be set by Power Automate later
5. **Change the variable scope to Global**:
   - Click on the variable name `sessionId` in the node
   - In the **Variable properties** panel that appears on the right:
     - Look for the **Scope** dropdown (may show "Topic" by default)
     - Click the dropdown and select **Global**
   - The variable will now be prefixed with `Global.` (shown as `Global.sessionId`)
6. Click outside the panel to close it

##### Step 3.1.5: Create the uploadStatus Variable

1. Below the previous node, click the **+** button
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `uploadStatus`
   - Click **Create**
4. For **To value**, type: `"Pending"`
5. **Change scope to Global**:
   - Click on the variable name `uploadStatus`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.6: Create the downloadUrl Variable

1. Click the **+** button below the previous node
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `downloadUrl`
   - Click **Create**
4. For **To value**, type: `""`
5. **Change scope to Global**:
   - Click on the variable name `downloadUrl`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.7: Create the processingStatus Variable

1. Click the **+** button below the previous node
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `processingStatus`
   - Click **Create**
4. For **To value**, type: `"Not Started"`
5. **Change scope to Global**:
   - Click on the variable name `processingStatus`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.8: Create the errorMessage Variable

1. Click the **+** button below the previous node
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `errorMessage`
   - Click **Create**
4. For **To value**, type: `""`
5. **Change scope to Global**:
   - Click on the variable name `errorMessage`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.9: Create the uploadedFilePath Variable

> **Note**: This variable stores the path to the uploaded file in temporary storage. It is used by Agents 2, 3, and 4 when calling their data extraction tools (e.g., "Read Application Inventory Data", "Read SQL Server Inventory Data", "Read Web App Inventory Data").

1. Click the **+** button below the previous node
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `uploadedFilePath`
   - Click **Create**
4. For **To value**, type: `""`
5. **Change scope to Global**:
   - Click on the variable name `uploadedFilePath`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.10: Create the consolidatedApplications Variable

> **Note**: This variable is set by the "Process Application Inventory" topic (Agent 2) to store the LLM-analyzed unique application list as a JSON string.

1. Click the **+** button below the previous node
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `consolidatedApplications`
   - Click **Create**
4. For **To value**, type: `""`
5. **Change scope to Global**:
   - Click on the variable name `consolidatedApplications`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.11: Create the consolidatedSQLInstances Variable

> **Note**: This variable is set by the "Process SQL Server Inventory" topic (Agent 3) to store the LLM-analyzed unique SQL Server list as a JSON string.

1. Click the **+** button below the previous node
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `consolidatedSQLInstances`
   - Click **Create**
4. For **To value**, type: `""`
5. **Change scope to Global**:
   - Click on the variable name `consolidatedSQLInstances`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.12: Create the consolidatedWebApps Variable

> **Note**: This variable is set by the "Process Web App Inventory" topic (Agent 4) to store the LLM-analyzed unique web application list as a JSON string.

1. Click the **+** button below the previous node
2. Select **Variable management** → **Set a variable value**
3. In the "Set variable" node:
   - Click **Set variable** → **Create new**
   - Type variable name: `consolidatedWebApps`
   - Click **Create**
4. For **To value**, type: `""`
5. **Change scope to Global**:
   - Click on the variable name `consolidatedWebApps`
   - In the Variable properties panel, change **Scope** to **Global**

##### Step 3.1.13: Add End Conversation Node

1. Click the **+** button below the last variable node
2. Select **Topic management** → **End current topic**
3. This ensures the topic ends cleanly after initialization

##### Step 3.1.14: Save the Topic

1. Click the **Save** button at the top-right of the canvas
2. Wait for the "Topic saved" confirmation

#### Step 3.2: Verify Global Variables

After saving the topic, verify your global variables are created:

1. Open any topic in your agent (or create a test topic)
2. Add a "Send a message" node
3. Click the **{x}** icon (Insert variable) in the message text
4. In the variable picker, you should see your global variables listed with the `Global.` prefix:

```
┌────────────────────────────────────┬─────────┬────────────────────────────────────────────────────┐
│ Variable Name                      │ Type    │ Purpose                                            │
├────────────────────────────────────┼─────────┼────────────────────────────────────────────────────┤
│ Global.sessionId                   │ String  │ Unique identifier for the current session          │
│ Global.uploadStatus                │ String  │ Current status of file upload                      │
│ Global.downloadUrl                 │ String  │ URL for downloading the report                     │
│ Global.processingStatus            │ String  │ Current status of processing workflow               │
│ Global.errorMessage                │ String  │ Error message if processing fails                  │
│ Global.uploadedFilePath            │ String  │ Path to the uploaded file in temporary storage      │
│ Global.consolidatedApplications    │ String  │ LLM-analyzed unique application list (JSON)         │
│ Global.consolidatedSQLInstances    │ String  │ LLM-analyzed unique SQL Server list (JSON)          │
│ Global.consolidatedWebApps         │ String  │ LLM-analyzed unique web app list (JSON)             │
└────────────────────────────────────┴─────────┴────────────────────────────────────────────────────┘
```

> **Tip**: Global variable names must be unique across all topics in the agent. Once created, these variables can be accessed and modified from any topic by using the **{x}** variable picker.

---

### Step 4: Create Topics

Topics define the conversation flows that the agent uses to interact with users.

#### Step 4.1: Create Topic 1 - Welcome and Upload Instructions

This is the main entry point topic where users start their interaction.

##### Step 4.1.1: Create New Topic

1. In the left navigation panel, click on **Topics**
2. The Topics page will display existing system topics
3. Click the **+ Add a topic** button at the top
4. From the dropdown menu, select **From blank**
5. A new blank topic canvas will open

##### Step 4.1.2: Configure Topic Name and Description

1. At the top of the topic canvas, you will see "Untitled" as the topic name
2. Click on **Untitled** to edit it
3. Type the topic name: `Welcome and Upload Instructions`
4. Press **Enter** to confirm the name
5. Click on **Add description** (if available)
6. Type: `Main entry topic that greets users and prompts for Azure Migrate file upload`

##### Step 4.1.3: Add Trigger Phrases

Trigger phrases are the words or sentences that activate this topic.

1. In the topic canvas, locate the **Trigger phrases** section at the top
2. You will see a text field labeled "Add phrases"
3. Click in the text field

4. Type the first trigger phrase and press **Enter**:
   - Type: `Start` and press **Enter**

5. Type the second trigger phrase and press **Enter**:
   - Type: `Hello` and press **Enter**

6. Type the third trigger phrase and press **Enter**:
   - Type: `Upload files` and press **Enter**

7. Type the fourth trigger phrase and press **Enter**:
   - Type: `Process Azure Migrate files` and press **Enter**

8. Type the fifth trigger phrase and press **Enter**:
   - Type: `I have Azure Migrate export files` and press **Enter**

9. Type the sixth trigger phrase and press **Enter**:
   - Type: `Help me consolidate applications` and press **Enter**

10. Type the seventh trigger phrase and press **Enter**:
    - Type: `I need to upload CSV files` and press **Enter**

11. Type the eighth trigger phrase and press **Enter**:
    - Type: `Azure Migrate CSV processing` and press **Enter**

12. Type the ninth trigger phrase and press **Enter**:
    - Type: `Consolidate application inventory` and press **Enter**

13. Type the tenth trigger phrase and press **Enter**:
    - Type: `Process migration assessment` and press **Enter**

14. Type the eleventh trigger phrase and press **Enter**:
    - Type: `Begin file upload` and press **Enter**

15. Type the twelfth trigger phrase and press **Enter**:
    - Type: `Process CSV` and press **Enter**

16. Verify all trigger phrases are added:
    ```
    Trigger phrases:
    1. Start
    2. Hello
    3. Upload files
    4. Process Azure Migrate files
    5. I have Azure Migrate export files
    6. Help me consolidate applications
    7. I need to upload CSV files
    8. Azure Migrate CSV processing
    9. Consolidate application inventory
    10. Process migration assessment
    11. Begin file upload
    12. Process CSV
    ```

##### Step 4.1.4: Build the Conversation Flow - Add Message Node

1. Below the trigger phrases, you will see the conversation canvas
2. Click the **+** button to add a new node
3. From the dropdown menu, select **Send a message**

4. In the message node that appears, click in the text area
5. Type or paste the following welcome message exactly:

```
Welcome! I'm your Azure Migrate CSV Processor. 🤖

📋 **Instructions for Uploading Azure Migrate CSV Files:**

1. Export your data from Azure Migrate portal
2. Ensure your files contain the required sheets:
   • ApplicationInventory (Machine names, Applications, Versions)
   • SQL Server (Instance names, Editions, Versions)
   • WebApplications (Web server types, Web app names, Frameworks)
   ℹ️ Files may also include a Database sheet — this is accepted but not currently processed
3. Files can be in CSV or Excel (.xlsx) format
4. You can upload one or multiple files at once

📁 **What I'll do with your files:**
• Use AI-powered analysis to identify noise and duplicates
• Consolidate application, SQL Server, and web app inventories
• Generate a downloadable report with unique entries

⬆️ **Please upload your Azure Migrate CSV file(s) to begin processing.**
```

6. Click outside the text area to save the message

##### Step 4.1.5: Build the Conversation Flow - Add Question Node for File Upload

1. Below the message node, click the **+** button
2. From the dropdown menu, select **Ask a question**

3. In the question node:
   - Click in the **Ask a question** text field
   - Type: `Please upload your Azure Migrate export file(s). You can upload one or more CSV or Excel files.`

4. **Configure the response type**:
   - Click on **Identify** dropdown
   - Scroll through the list and select **File**
   - This enables file upload capability for this question

5. **Set the variable to save the response**:
   - Below the question, find **Save response as**
   - Click on **Select a variable** or **Create new variable**
   - If creating new: Type variable name: `uploadedFiles`
   - Set the variable type to: **File** or **Array of Files**
   - Click **Save** or **Done**

6. **Configure file upload options** (if available):
   - Look for **File upload settings** or **Options**
   - Enable **Allow multiple files**: Toggle to **ON**
   - Set **Accepted file types** (if available): `.csv, .xlsx, .xls`
   - Set **Maximum file size** (if available): `25 MB`

##### Step 4.1.6: Build the Conversation Flow - Add Condition Node

1. Below the question node, click the **+** button
2. From the dropdown menu, select **Add a condition**

3. In the condition node:
   - Click on **Select a variable**
   - Select: `uploadedFiles`
   - Click on the **operator dropdown**
   - Select: **is not empty** or **has value**

4. The condition will create two branches:
   - **If true (left branch)**: Files were uploaded
   - **If false (right branch)**: No files uploaded

##### Step 4.1.7: Configure the TRUE Branch (Files Uploaded)

> **Note:** In the current version of Copilot Studio (2025), the **"Call an action"** menu option has been replaced by **"Add a tool"**. The former **"Actions"** page has been replaced by the **"Tools"** page. Power Automate flows are now added as **tools** — select the **+** (Add node) icon in a topic and choose **Add a tool**, then select a flow or create a **New Agent flow**. For agent-level tools, use the **Tools** page in the left navigation panel.

**Part A: Add a Confirmation Message**

1. Click the **+** button below the **TRUE** branch label on the authoring canvas
2. From the dropdown menu that appears, select **Send a message**
3. In the message node that appears, click in the text area
4. Type or paste the following confirmation message exactly:

```
✅ **Files received successfully!**

I'll now store your file and begin the AI-powered analysis. The processing sequence is:

1. 📁 Storing file in temporary storage
2. 📊 Analyzing Application Inventory (LLM-based noise detection & consolidation)
3. 🗄️ Analyzing SQL Server Inventory (LLM-based version grouping & consolidation)
4. 🌐 Analyzing Web App Inventory (LLM-based consolidation)
5. 📝 Generating your consolidated report

⏳ Each step will provide a summary as it completes.
```

5. Click outside the text area to save the message

**Part B: Add a Power Automate Flow Call Node (File Upload)**

> **Note:** The Power Automate agent flow that stores the uploaded files will be fully configured in **Step 5**. The steps below add the flow tool node to the canvas now so the topic structure is complete. You will return here to configure the inputs and outputs once the flow is ready.

6. Click the **+** (Add node) button below the message node you just added (still inside the TRUE branch)
7. From the dropdown menu, select **Add a tool**
8. You have two options in the tool selection panel:
   - **New Agent flow** – Select this to create a new agent flow template with the required trigger and response action already configured. You will build the processing logic in Step 5. For now, click **Publish** to save the empty template, then click **Go back to agent**.
   - **Select an existing flow** – If you have already created and published the **Handle File Upload – Azure Migrate** flow (from Step 5), select it here.
9. If you created a new agent flow template and returned to the topic, an **Action** node will appear in the canvas — this is expected and will be fully configured in Step 5.

> **Tip:** If you prefer to skip adding the flow tool for now, you can come back after completing Step 5. Locate the TRUE branch in this topic, click **+**, select **Add a tool**, and choose the flow to add and configure it at that point.

**Part C: Add Topic Redirects for LLM-Based Processing Sequence**

After the file upload flow returns, the agent coordinates the processing by redirecting to each analysis topic in sequence. Each topic (Agent 2, 3, 4) calls its own data extraction tool, then the LLM analyzes the raw data and stores the results in a global variable.

> **Note:** The processing topics referenced below are created in Agents 2, 3, and 4 respectively. You will add these redirect nodes after completing those agent sections. For now, you can add placeholder "Send a message" nodes or skip this part and return later.

10. Click the **+** (Add node) button below the Action node (still inside the TRUE branch)
11. From the dropdown menu, select **Redirect to another topic**
12. Select: **Process Application Inventory** (created in Agent 2, Step 4)
    - This topic calls the "Read Application Inventory Data" tool, then the LLM analyzes the data and stores results in `Global.consolidatedApplications`

13. Click the **+** (Add node) button below the redirect
14. Select **Redirect to another topic**
15. Select: **Process SQL Server Inventory** (created in Agent 3, Step 4)
    - This topic calls the "Read SQL Server Inventory Data" tool, then the LLM analyzes the data and stores results in `Global.consolidatedSQLInstances`

16. Click the **+** (Add node) button below the redirect
17. Select **Redirect to another topic**
18. Select: **Process Web App Inventory** (created in Agent 4, Step 4)
    - This topic calls the "Read Web App Inventory Data" tool, then the LLM analyzes the data and stores results in `Global.consolidatedWebApps`

19. Click the **+** (Add node) button below the redirect
20. Select **Send a message**
21. Type:

```
✅ **All inventory analysis complete!**

📊 Your consolidated data is ready:
• Application inventory analyzed and deduplicated
• SQL Server instances consolidated and grouped by version
• Web applications identified and consolidated

📝 Generating your final report now...
```

> **Note:** After all analysis topics complete, you can optionally redirect to a report generation topic that calls the Agent 5 (Report Generator) flow to create the final Excel spreadsheet and provide a download link. See [Agent 5: Report Generator](#agent-5-report-generator) for details.

##### Step 4.1.8: Configure the FALSE Branch (No Files Uploaded)

1. Click the **+** button below the **FALSE** branch
2. Select **Send a message**
3. Type the following message:

```
⚠️ **No files detected.**

It looks like no files were uploaded. Please try again by:
1. Clicking the attachment/upload button (📎)
2. Selecting your Azure Migrate CSV or Excel file(s)
3. Clicking Send or Upload

**Supported formats:**
• CSV files (.csv)
• Excel files (.xlsx, .xls)

Would you like to try uploading your files again?
```

4. Below this message, click the **+** button
5. Select **Redirect to another topic**
6. Select: **Welcome and Upload Instructions** (this topic - creates a loop back)

##### Step 4.1.9: Save the Topic

1. Click the **Save** button at the top-right of the topic canvas
2. Wait for the "Topic saved" confirmation
3. The topic is now ready for testing

---

#### Step 4.2: Create Topic 2 - Check Processing Status

This topic allows users to check on the status of their file processing. In the LLM-first architecture, processing happens conversationally within the agent's topics (Agents 2, 3, 4 run sequentially in the Welcome topic's TRUE branch). This status topic is useful when:
- The user navigates away and returns to check progress
- The agent is configured with an optional Power Automate Orchestrator for background processing
- The report generation (Agent 5) is running asynchronously

##### Step 4.2.1: Create New Topic

1. Click on **Topics** in the left navigation
2. Click the **+ Add a topic** button
3. Select **From blank**

##### Step 4.2.2: Configure Topic Name

1. Click on **Untitled** at the top
2. Type: `Check Processing Status`
3. Press **Enter** to confirm

##### Step 4.2.3: Add Trigger Phrases

1. In the **Trigger phrases** section, add the following phrases (one at a time, pressing Enter after each):

```
1. Status
2. Check status
3. Is my report ready?
4. Processing status
5. How long will it take?
6. Where is my report?
7. Check progress
8. Report status
9. What's the status
10. Is it done yet
11. Report ready
12. Download report
```

##### Step 4.2.4: Build the Conversation Flow

1. Click the **+** button to add a node
2. Select **Send a message**
3. Type:

```
🔍 **Checking your processing status...**

Let me look up the current status of your Azure Migrate data processing.
```

4. Click the **+** button below this message
5. For now, add a placeholder variable set (the actual flow call will be added in **Step 5.4**):
   - Click **+** and select **Set a variable value**
   - Variable: Select `Global.processingStatus`
   - Value: Click **Formula** and type: `"Processing"` (placeholder)

> **Note:** In Step 5.4 you will replace this placeholder with a real flow call. To add the flow call at that point: click the **+** (Add node) icon, select **Add a tool**, then choose **New Agent flow** or select the **Get Processing Status – Azure Migrate** flow if already created.

6. Click the **+** button
7. Select **Add a condition**
8. Configure the condition:
   - Variable: `Global.processingStatus`
   - Operator: **is equal to**
   - Value: `"Complete"`

##### Step 4.2.5: Configure TRUE Branch (Processing Complete)

1. In the TRUE branch, click **+**
2. Select **Send a message**
3. Type:

```
🎉 **Great news! Your consolidated report is ready!**

📥 **Download your report here:** {Global.downloadUrl}

📊 **Your report includes:**
• **Unique Applications Sheet** - Consolidated application inventory with duplicates removed
• **SQL Server Inventory Sheet** - Unique SQL Server instances across all machines
• **Web App Inventory Sheet** - Consolidated web application inventory

✅ **Processing Summary:**
• Duplicates removed
• Noise filtered (patches, updates, redistributables)
• Data normalized and consolidated

Would you like to process another set of files?
```

4. To insert the variable in the message:
   - Place your cursor where you want the URL
   - Click the **{x}** button (variable insertion)
   - Select **Global.downloadUrl**

##### Step 4.2.6: Configure FALSE Branch (Still Processing)

1. In the FALSE branch, click **+**
2. Select **Add a condition**
3. Configure the condition:
    - Variable: `Global.processingStatus`
    - Operator: **is equal to**
    - Value: `"Error"`

4. **For the Error TRUE branch**, click **+** and select **Send a message**:

```
❌ **Processing Error**

Unfortunately, there was an issue processing your files:

**Error:** {Global.errorMessage}

**Possible causes:**
• File format not recognized
• Missing required sheets (ApplicationInventory, SQL Server, WebApplications)
• File may be corrupted or password protected

**To try again:**
1. Verify your file format (CSV or Excel)
2. Ensure all required sheets are present
3. Upload the files again

Would you like to try uploading your files again?
```

5. **For the Error FALSE branch** (Still Processing), click **+** and select **Send a message**:

```
⏳ **Your files are still being processed.**

**Current Status:** {Global.processingStatus}

📊 **Processing steps:**
1. ✅ Files received
2. ⏳ Parsing data...
3. ⬜ Removing duplicates
4. ⬜ Generating report

⏱️ This typically takes 1-5 minutes depending on file size.

Please check back in a few moments, or I'll notify you when it's complete.

💡 **Tip:** You can say "Check status" anytime to see the latest progress.
```

##### Step 4.2.7: Save the Topic

1. Click **Save** at the top-right
2. Verify the topic structure is complete

---

#### Step 4.3: Create Topic 3 - Help and Supported Formats

##### Step 4.3.1: Create New Topic

1. Click on **Topics** in the left navigation
2. Click **+ Add a topic** → **From blank**

##### Step 4.3.2: Configure Topic Name

1. Click on **Untitled**
2. Type: `Help and Supported Formats`
3. Press **Enter**

##### Step 4.3.3: Add Trigger Phrases

Add the following trigger phrases:

```
1. Help
2. What files do you support?
3. File format
4. How does this work?
5. What is Azure Migrate export?
6. Supported formats
7. What can you do
8. Instructions
9. Guide me
10. How to use
11. File types
12. Accepted formats
```

##### Step 4.3.4: Build the Conversation Flow

1. Click **+** and select **Send a message**
2. Type the following comprehensive help message:

```
📚 **Help & File Format Information**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**🎯 What I Can Do:**
I process Azure Migrate export files and create consolidated reports by:
• Removing duplicate entries
• Filtering out noise (patches, updates, redistributables)
• Creating unique lists of applications, SQL servers, and web apps
• Generating a downloadable Excel report

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**📁 Supported File Formats:**
• **CSV files** (.csv) - Exported directly from Azure Migrate
• **Excel files** (.xlsx, .xls) - With multiple sheets

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**📋 Required Data Sheets:**

**1. ApplicationInventory sheet:**
   | Column | Description |
   |--------|-------------|
   | MachineName | Server name |
   | Application | Application name |
   | Version | Version number |
   | Provider | Software vendor |
   | MachineManagerFqdn | Manager FQDN |

**2. SQL Server sheet:**
   | Column | Description |
   |--------|-------------|
   | MachineName | Server name |
   | Instance Name | SQL instance |
   | Edition | SQL edition |
   | Service Pack | SP level |
   | Version | SQL version |
   | Port | Port number |

**3. Database sheet:**
   | Column | Description |
   |--------|-------------|
   | MachineName | Server name |
   | Database Type | Oracle, MySQL, etc. |
   | Version | Database version |

**4. WebApplications sheet:**
   | Column | Description |
   |--------|-------------|
   | MachineName | Server name |
   | WebServerType | IIS, Apache, Tomcat, etc. |
   | WebAppName | Application name |
   | VirtualDirectory | Virtual path |
   | ApplicationPool | App pool name |
   | FrameworkVersion | Framework/runtime version |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**🚀 Ready to get started?**
Simply upload your Azure Migrate CSV or Excel file(s) and I'll begin processing!
```

3. Click the **+** button below the message
4. Select **Ask a question**
5. Type: `Would you like to upload your files now, or do you have more questions?`
6. Set **Identify** to: **Multiple choice**
7. Add options:
   - Option 1: `Upload files now`
   - Option 2: `I have more questions`
8. Save response as: `helpChoice`

9. Click **+** and add a **Condition**:
   - Variable: `helpChoice`
   - Operator: **is equal to**
   - Value: `Upload files now`

10. TRUE branch: Add **Redirect to another topic** → Select **Welcome and Upload Instructions**

11. FALSE branch: Add **Send a message**:
```
No problem! Feel free to ask me any questions about:
• File formats and requirements
• How the consolidation process works
• Troubleshooting upload issues

What would you like to know?
```

##### Step 4.3.5: Save the Topic

1. Click **Save** at the top-right

---

#### Step 4.4: Create Topic 4 - Handle Off-Topic Requests

##### Step 4.4.1: Create New Topic

1. Click **+ Add a topic** → **From blank**
2. Name: `Handle Off-Topic Requests`

##### Step 4.4.2: Add Trigger Phrases

Add common off-topic phrases that users might say:

```
1. Tell me a joke
2. What's the weather
3. Who are you
4. What time is it
5. Chat with me
6. Let's talk
7. Hi there
8. Good morning
9. Thank you
10. Thanks
11. Goodbye
12. Bye
```

##### Step 4.4.3: Build the Flow

1. Add a **Send a message** node:

```
👋 Thanks for reaching out!

I'm specifically designed to process Azure Migrate CSV files and generate consolidated reports. I'm not a general-purpose chatbot.

**Here's what I can help you with:**
• 📁 Processing Azure Migrate export files
• 🔄 Consolidating application inventories
• 📊 Generating downloadable reports

**Ready to get started?**
Just upload your Azure Migrate CSV or Excel file(s) and I'll begin processing!

Type "Help" if you need more information about supported file formats.
```

2. Click **Save**

---

#### Step 4.5: Customize System Topics

##### Step 4.5.1: Modify the Greeting Topic

1. In the **Topics** list, find **System topics**
2. Click on **Greeting** topic
3. Modify the greeting message to:

```
👋 **Welcome to the Azure Migrate CSV Processor!**

I help you consolidate Azure Migrate export files by:
• 📊 Removing duplicate applications
• 🗄️ Consolidating SQL Server instances
• 💾 Organizing web app inventories
• 📥 Generating downloadable reports

**To get started:**
📎 Upload your Azure Migrate CSV or Excel file(s)

💡 Type "Help" for information about supported file formats.
```

4. Click **Save**

##### Step 4.5.2: Modify the Fallback Topic

1. Click on the **Fallback** topic (also called "Escalate" or "Unknown intent")
2. Modify the message to:

```
🤔 I'm not sure I understood that request.

Remember, I'm specifically designed to:
• Process Azure Migrate CSV files
• Generate consolidated inventory reports

**Common commands:**
• Upload files - to start processing
• Check status - to see processing progress
• Help - for file format information

Would you like to upload your Azure Migrate files now?
```

3. Click **Save**

---

### Step 5: Add Power Automate Flows as Tools

> **Note:** In the current version of Copilot Studio (2025), the former **"Actions"** page has been replaced by the **"Tools"** page. Power Automate flows are now added as **tools** — either at the agent level (available to all topics) or at the topic level (available to a single topic). All instructions below use the current Tools-based workflow. For official reference, see [Add an agent flow to an agent as a tool](https://learn.microsoft.com/en-us/microsoft-copilot-studio/flow-agent).

Tools connect your agent to Power Automate flows (called **agent flows**) that perform the actual file processing. An agent flow must have:
- A **When an agent calls the flow** trigger (also shown as **Run a flow from Copilot**)
- A **Respond to the agent** action (also shown as **Respond to Copilot**)
- The **Asynchronous response** toggle set to **Off** (under **Networking** in the Respond to the agent action settings)

#### Step 5.1: Create the File Upload Agent Flow

##### Option A: Create the Flow from within a Topic (Recommended)

1. Go to the **Topics** page for your agent
2. Open the **Welcome and Upload Instructions** topic
3. In the TRUE branch, find the flow call node placeholder (added in Step 4.1.7 Part B), or click the **+** (Add node) icon below the confirmation message
4. Select **Add a tool**
5. On the **Basic tools** tab, select **New Agent flow**
   - The agent flows designer opens in a new view with a starter template containing the required **When an agent calls the flow** trigger and **Respond to the agent** action
6. Select **Publish** to save the empty flow template before making changes

##### Option B: Create the Flow from the Tools Page

1. In the left navigation panel of Copilot Studio, click on **Tools**
2. The Tools page displays all existing tools (flows, connectors, prompts, etc.)
3. Click **Add a tool**
4. In the **Add tool** panel, select **Flow** to see available flows
5. If no suitable flow exists, go to [Power Automate](https://make.powerautomate.com) to create one (see Step 5.1.1 below), then return here to add it

##### Step 5.1.1: Configure Flow Inputs and Outputs

Whether you created the flow from a topic or from Power Automate directly, open the flow designer and configure:

1. **Name the flow**: On the **Overview** page, under **Details**, set the name to `Handle File Upload - Azure Migrate`

2. **Configure flow inputs** (click on the **When an agent calls the flow** trigger):
   - Click **+ Add an input**
   - Add input: Type: **File**, Name: `uploadedFiles`
   - Add input: Type: **Text**, Name: `userId`
   - Add input: Type: **Text**, Name: `userEmail`

3. **Configure flow outputs** (click on the **Respond to the agent** action):
   - Verify the **Asynchronous response** toggle is set to **Off** under **Networking** in the action settings
   - Add output: Type: **Text**, Name: `sessionId`, Value: type any temporary placeholder (e.g., `temp`) — this will be replaced with a dynamic expression in Step 7
   - Add output: Type: **Text**, Name: `filePath`, Value: type any temporary placeholder (e.g., `temp`) — this will be replaced with the actual storage path in Step 7
   - Add output: Type: **Text**, Name: `status`, Value: `Processing`
   - Add output: Type: **Text**, Name: `message`, Value: `File uploaded successfully. Processing has started.`

> **Important:** The `sessionId` and `filePath` outputs require Compose actions that do not exist yet — they are created in **Step 7**. At this point, enter any non-empty placeholder values to pass validation. In Step 7.4 (Option A) or Step 7.5 (Option B) you will replace them with dynamic expressions. If you try to enter the expressions now, Power Automate will show an error because the referenced actions have not been added yet.

> **Important:** Every output parameter in the **Respond to the agent** action must have a value assigned at runtime. If your flow has conditional branches (e.g., a condition that checks whether processing is complete), ensure **each branch** includes a **Respond to the agent** action with all outputs populated. Leaving any output blank causes a `FlowActionException` error: "output parameter missing from response data."
>
> **Preventing null output errors:** Wrap every dynamic output value in a `coalesce()` expression so that a default is returned when an action produces no value. The table below shows the expected values and safe defaults for each output parameter:
>
> | Output Parameter | Type | Expected Value | Default (when no value) | Safe Expression Example |
> |---|---|---|---|---|
> | `sessionId` | Text | GUID generated by the Compose action named "Generate Session ID" (added in Step 7) | `""` | `coalesce(outputs('Generate_Session_ID'), '')` |
> | `filePath` | Text | Storage path of the uploaded file (added in Step 7) | `""` | `coalesce(outputs('Build_File_Path'), '')` |
> | `status` | Text | `"Processing"` | `"Error"` | `"Processing"` |
> | `message` | Text | `"File uploaded successfully. Processing has started."` | `"No details available."` | `"File uploaded successfully. Processing has started."` |
>
> **Note:** The `status` and `message` outputs use literal string values in this flow, so `coalesce()` is not required for them. Use `coalesce()` only for dynamic expressions that reference action outputs (such as `sessionId` and `filePath`) where a preceding action could return null.
>
> Apply the same pattern to the **Get Processing Status** flow outputs:
>
> | Output Parameter | Type | Expected Value | Default (when no value) |
> |---|---|---|---|
> | `processingStatus` | Text | `"Complete"`, `"Processing"`, or `"Error"` | `"Error"` |
> | `downloadUrl` | Text | Download link for the generated report | `""` |
> | `errorMessage` | Text | Error details or empty when successful | `""` |

4. Click **Publish** to save and publish the flow
5. If you were in the flow designer, click **Go back to agent** to return to Copilot Studio

> **Tip:** The flow must be published before it can be added as a tool. If the flow does not appear in the tool selection list, verify it has the correct trigger (**When an agent calls the flow**) and response action (**Respond to the agent**), and that it has been published.

#### Step 5.2: Add the File Upload Flow as a Tool

If you created the flow from within a topic (Option A above), the flow is automatically added as a topic-level tool and an **Action** node appears in your topic. Skip to mapping inputs below.

If you created the flow separately (Option B above), add it as an agent-level tool:

1. In Copilot Studio, go to the **Tools** page
2. Click **Add a tool**
3. In the **Add tool** panel, select **Flow**
4. Select **Handle File Upload - Azure Migrate** from the list of published flows
5. Click **Add and configure**
6. Update the **Description** to help the agent understand the flow's purpose (e.g., "Handles file uploads for Azure Migrate CSV processing")
7. Click **Save**

The flow now appears in the agent's list of tools.

#### Step 5.3: Map the Flow to the Upload Topic

1. Go to **Topics** → **Welcome and Upload Instructions**
2. In the TRUE branch, locate the **Action** node (if created from a topic) or click the **+** (Add node) icon and select **Add a tool**, then choose **Handle File Upload - Azure Migrate**
3. **Map the inputs** on the Action node:
   - `uploadedFiles` → For file data, click the input field, select **Formula** (fx), and enter the following Power Fx expression:
     ```
     { contentBytes: Topic.uploadedFiles.Content, name: Topic.uploadedFiles.Name }
     ```
     > **Note:** Use `Topic.uploadedFiles` (referencing the variable from the Question node) when mapping within a topic. The variable name must match the one you created in Step 4.1.5.
   - `userId` → `System.User.Id` (System variable)
   - `userEmail` → `System.User.Email` (System variable)
4. **Map the outputs** — click each output field and select the corresponding global variable:
   - `sessionId` → `Global.sessionId`
   - `filePath` → `Global.uploadedFilePath`
   - `status` → `Global.uploadStatus`
   - `message` → (display in a Message node below the Action node)
5. Click **Save**

> **Note:** If you added the flow as an **agent-level tool** (from the Tools page), configure inputs on the tool's **Details** page instead. Go to **Tools**, select the flow, and under **Inputs** use the **Custom value** option with these Power Fx formulas:
> - **contentBytes**: `First(System.Activity.Attachments).Content`
> - **name**: `First(System.Activity.Attachments).Name`
> - **userId**: `System.User.Id`
> - **userEmail**: `System.User.Email`
>
> For a safer pattern that handles cases where no file is attached, use:
> ```
> If(
>     IsEmpty(System.Activity.Attachments),
>     [],
>     [{ contentBytes: First(System.Activity.Attachments).Content, name: First(System.Activity.Attachments).Name }])
> ```
>
> **Important:** On the Tools page, file inputs only work with the **Custom value** (Power Fx) option — the **Dynamically fill with AI** option does not work for file inputs.

#### Step 5.4: Create the Check Status Agent Flow

1. Open the **Check Processing Status** topic (created in Step 4.2)
2. Locate the placeholder variable set node (added in Step 4.2.4)
3. Delete the placeholder node by clicking the **⋮** (menu) icon on it and selecting **Delete**
4. Click the **+** (Add node) icon in its place and select **Add a tool**
5. Select **New Agent flow** to create a new flow from within the topic
6. In the flow designer, configure:
   - **Flow name**: `Get Processing Status - Azure Migrate`
   - **Trigger input**: `sessionId` (Text)
   - **Respond to the agent outputs**: `processingStatus` (Text), `downloadUrl` (Text), `errorMessage` (Text)
7. Click **Publish** to save and publish the flow
8. Click **Go back to agent** to return to your topic — a new **Action** node appears
9. Map the inputs and outputs on the Action node:
   - Input: `sessionId` → `Global.sessionId`
   - Output: `processingStatus` → `Global.processingStatus`
   - Output: `downloadUrl` → `Global.downloadUrl`
   - Output: `errorMessage` → `Global.errorMessage`
10. Click **Save**

> **Tip:** You can also create this flow from the **Tools** page (**Add a tool** → **Flow** → select it) or directly in [Power Automate](https://make.powerautomate.com). Ensure the flow uses the **When an agent calls the flow** trigger and **Respond to the agent** action so it appears in the tool selection list.

---

### Step 6: Test the Agent

> **Prerequisite:** Before testing the file upload flow, complete **Step 7** to configure the flow logic in Power Automate. If you test before Step 7 is completed, the flow will run with only the placeholder output values set in Step 5.1.1. You can still test the welcome message and conversation flow (Steps 6.1–6.2) without completing Step 7.

#### Step 6.1: Open the Test Panel

1. Look at the bottom-left corner of the Copilot Studio interface
2. Click on **Test your copilot** or the **Test** button
3. A test chat panel will open on the right side

#### Step 6.2: Test the Welcome Flow

1. In the test chat, type: `Hello`
2. Press **Enter**
3. Verify the welcome message appears with instructions
4. Verify the file upload prompt appears

#### Step 6.3: Test File Upload

1. Click the **paperclip icon** (📎) in the test chat
2. Select a test CSV or Excel file
3. Click **Send**
4. Verify the processing confirmation message appears

#### Step 6.4: Test Status Check

1. Type: `Check status`
2. Verify the status message appears

#### Step 6.5: Test Help

1. Type: `Help`
2. Verify the help information appears

#### Step 6.6: Review Test Results

Create a testing checklist:
```
Testing Checklist for File Upload Handler:
- [ ] Greeting message displays correctly
- [ ] All trigger phrases activate correct topics
- [ ] File upload prompt works
- [ ] File upload accepts CSV files
- [ ] File upload accepts Excel files
- [ ] Processing confirmation appears after upload
- [ ] Status check topic works
- [ ] Help topic displays complete information
- [ ] Off-topic requests are handled gracefully
- [ ] Fallback message appears for unknown inputs
```

---

### Step 7: Configure the File Upload Flow Logic (Power Automate)

The file upload agent flow stores uploaded files in temporary storage. Choose the appropriate configuration based on your storage choice:

#### Option A: Azure Blob Storage (Recommended - No SharePoint/OneDrive Required)

> **Important — Action naming:** In Power Automate, expressions like `outputs('Generate_Session_ID')` reference actions by their **internal name**, which is the display name with spaces replaced by underscores. If you add a Compose action and leave its default name ("Compose"), the expression `outputs('Generate_Session_ID')` will fail with an "invalid reference" error. You **must** rename actions exactly as shown in the "Rename to:" instructions below so that the internal name matches the expressions used later in the flow.

1. Go to the **Tools** page, find **Handle File Upload - Azure Migrate**, and click **View flow details** to open the flow in Power Automate (or open it from the **Action** node in your topic)
2. In the flow designer, add the following actions between the trigger (**When an agent calls the flow**) and the **Respond to the agent** action:

##### Step 7.1: Add Compose — Generate Session ID

1. Click the **+** (Add an action) icon below the trigger
2. Search for and select **Compose** (under Built-in > Data Operations)
3. In the **Inputs** field, click the **Expression** tab (fx) and type:
   ```
   guid()
   ```
4. Click **OK**
5. Rename the action to: `Generate Session ID`
   - Click on the action title ("Compose") at the top of the action card and type the new name

> **Why rename?** The expression `outputs('Generate_Session_ID')` used later in this flow references this action by its internal name. Power Automate derives the internal name from the display name (spaces become underscores). If the action is still named "Compose", the internal name is `Compose` and the expression will fail.

##### Step 7.2: Add Create blob (V2) — Azure Blob Storage

1. Click the **+** (Add an action) icon below the Compose action
2. Search for **Create blob** and select **Create blob (V2)** from the **Azure Blob Storage** connector
3. Configure:
   - **Storage account name or blob endpoint**: Select your Azure Storage Account connection
   - **Container name**: `uploads`
   - **Blob name**: Click in the field, switch to the **Expression** tab, and enter:
     ```
     @{outputs('Generate_Session_ID')}/@{triggerBody()?['uploadedFiles']?['name']}
     ```
   - **Blob content**: Click in the field, switch to the **Expression** tab, and enter:
     ```
     @{triggerBody()?['uploadedFiles']?['contentBytes']}
     ```

> **Note:** Do not rename this connector action — its default internal name `Create_blob_(V2)` is short enough as-is.

##### Step 7.3: Add Compose — Build File Path

This action builds the full path to the uploaded file in blob storage, which is returned to the agent and stored in `Global.uploadedFilePath` for use by the data extraction tools in Agents 2, 3, and 4.

1. Click the **+** (Add an action) icon below the Create blob action
2. Search for and select **Compose** (under Built-in > Data Operations)
3. In the **Inputs** field, click the **Expression** tab (fx) and enter:
   ```
   concat('uploads/', outputs('Generate_Session_ID'), '/', triggerBody()?['uploadedFiles']?['name'])
   ```
4. Click **OK**
5. Rename the action to: `Build File Path`

> **Why this step?** In the LLM-first architecture, the Copilot agent's topics (Agents 2, 3, 4) coordinate all processing — not a Power Automate Orchestrator. The file path is returned to the agent so it can pass it to each data extraction tool. The HTTP Orchestrator call is no longer needed here.

##### Step 7.4: Configure the Respond to the agent Outputs

1. Click on the existing **Respond to the agent** action at the bottom of the flow
2. Set the output values (these were defined in Step 5.1.1):
   - **sessionId**: Click the value field, switch to the **Expression** tab, and enter:
     ```
     coalesce(outputs('Generate_Session_ID'), '')
     ```
     Then click **OK**
   - **filePath**: Click the value field, switch to the **Expression** tab, and enter:
     ```
     coalesce(outputs('Build_File_Path'), '')
     ```
     Then click **OK**
   - **status**: Type the literal value: `Processing`
   - **message**: Type the literal value: `File uploaded successfully. Processing has started.`
3. Click **Save**

> **Note:** Access the uploaded file data using `triggerBody()?['uploadedFiles']?['name']` and `triggerBody()?['uploadedFiles']?['contentBytes']`. The key names in `triggerBody()` match the input parameter names defined on the trigger. Every output in the **Respond to the agent** action must have a value — use `coalesce()` to wrap any dynamic expression so a default value (e.g., `''`) is returned when the action produces no result. The `filePath` output is stored in `Global.uploadedFilePath` by the agent topic and passed to each data extraction tool (Agents 2, 3, 4).

**Benefits of Azure Blob Storage:**
- Users do NOT need SharePoint or OneDrive access
- Automatic cleanup via lifecycle management policies
- Cost-effective for temporary file storage
- Better suited for large file uploads
- Supports programmatic access via SAS tokens

#### Option B: SharePoint Storage (Alternative)

> **Note**: This option requires users to have SharePoint or OneDrive access.

1. Open the flow designer for **Handle File Upload - Azure Migrate** (same as Option A step 1)
2. Add the following actions between the trigger and the **Respond to the agent** action:

##### Step 7.1 (SharePoint): Add Compose — Generate Session ID

1. Click the **+** (Add an action) icon below the trigger
2. Search for and select **Compose** (under Built-in > Data Operations)
3. In the **Inputs** field, click the **Expression** tab (fx) and type:
   ```
   guid()
   ```
4. Click **OK**
5. Rename the action to: `Generate Session ID`

##### Step 7.2 (SharePoint): Add Create Folder

1. Click the **+** (Add an action) icon below the Compose action
2. Search for **Create new folder** and select it from the **SharePoint** connector
3. Configure:
   - **Site Address**: Select your SharePoint site
   - **List or Library**: Select your document library (e.g., `Shared Documents`)
   - **Folder Path**: Click in the field, switch to the **Expression** tab, and enter:
     ```
     /Uploads/@{outputs('Generate_Session_ID')}
     ```

##### Step 7.3 (SharePoint): Add Create File

1. Click the **+** (Add an action) icon below the Create folder action
2. Search for **Create file** and select it from the **SharePoint** connector
3. Configure:
   - **Site Address**: Select your SharePoint site
   - **Folder Path**: `/Uploads/@{outputs('Generate_Session_ID')}`
   - **File Name**: `@{triggerBody()?['uploadedFiles']?['name']}`
   - **File Content**: `@{triggerBody()?['uploadedFiles']?['contentBytes']}`

##### Step 7.4 (SharePoint): Add Compose — Build File Path

1. Click the **+** (Add an action) icon below the Create file action
2. Search for and select **Compose** (under Built-in > Data Operations)
3. In the **Inputs** field, click the **Expression** tab (fx) and enter:
   ```
   concat('/Uploads/', outputs('Generate_Session_ID'), '/', triggerBody()?['uploadedFiles']?['name'])
   ```
4. Click **OK**
5. Rename the action to: `Build File Path`

> **Why this step?** In the LLM-first architecture, the Copilot agent's topics coordinate processing — not a Power Automate Orchestrator. The file path is returned to the agent so it can pass it to each data extraction tool.

##### Step 7.5 (SharePoint): Configure the Respond to the agent Outputs

1. Click on the existing **Respond to the agent** action at the bottom of the flow
2. Set the output values:
   - **sessionId**: Click the value field, switch to the **Expression** tab, and enter:
     ```
     coalesce(outputs('Generate_Session_ID'), '')
     ```
     Then click **OK**
   - **filePath**: Click the value field, switch to the **Expression** tab, and enter:
     ```
     coalesce(outputs('Build_File_Path'), '')
     ```
     Then click **OK**
   - **status**: Type the literal value: `Processing`
   - **message**: Type the literal value: `File uploaded successfully. Processing has started.`
3. Click **Save**

### Step 4: Test the Upload Agent

1. Use the **Test bot** panel
2. Verify:
   - [ ] Greeting message displays correctly
   - [ ] File upload prompt works
   - [ ] File validation messages are appropriate
   - [ ] Processing confirmation is shown

---

## Agent 2: Application Inventory Processor

### Purpose

This agent uses the Copilot Studio LLM to intelligently analyze and consolidate application inventory data from Azure Migrate exports. Instead of relying on rigid pattern-matching rules in Power Automate loops, the agent's LLM applies reasoning to identify noise, consolidate applications, and produce a comprehensive unique application list. Power Automate is used **only** for reading raw data from the uploaded file.

> **Key Change from Previous Approach:** The analysis and consolidation logic (noise detection, deduplication, dependency identification) is now performed by the Copilot agent's LLM reasoning — not by Power Automate loops and conditions. This makes the agent more intelligent, easier to maintain, and capable of handling edge cases that rigid pattern matching would miss.

---

### How the LLM-Based Approach Works

1. A lightweight Power Automate **tool** reads raw application data from the uploaded file and returns it as structured JSON
2. The Copilot agent's LLM receives the raw data and applies intelligent analysis:
   - **Noise identification**: Uses reasoning to detect patches, software updates, OS-related updates, drivers, redistributables, and other non-business applications
   - **Dependency detection**: Identifies application-dependent drivers and updates (e.g., "SQL Server ODBC Driver", "Oracle Client") and classifies them as noise
   - **Exact name consolidation**: Groups applications by exact name match (case-insensitive, whitespace-trimmed)
   - **Version preservation**: Keeps multiple versions of the same application as **separate entries**
3. The LLM outputs a clean, deduplicated JSON array of unique business applications
4. Results are stored in a global variable for the Report Generator

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Copilot Agent Topic: Process Application Inventory                          │
│                                                                             │
│  1. Call Tool: "Read Application Inventory Data" (Power Automate)           │
│     └─→ Returns raw application rows as JSON                               │
│                                                                             │
│  2. LLM Analysis (performed by the Copilot agent's model):                 │
│     ├─→ Identify and remove noise (patches, updates, drivers, OS items)    │
│     ├─→ Identify application-dependent drivers/updates as noise            │
│     ├─→ Consolidate by exact application name match                        │
│     ├─→ Preserve different versions as separate entries                    │
│     └─→ Output structured JSON array of unique applications                │
│                                                                             │
│  3. Store results in Global.consolidatedApplications                       │
│  4. Report summary to user                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Step 1: Create the Data Extraction Tool (Power Automate Flow)

This minimal flow reads raw application inventory data from the uploaded file and returns it to the Copilot agent. No analysis or processing logic is needed in this flow — the LLM handles all analysis.

#### Step 1.1: Access Power Automate

1. Open your web browser (Microsoft Edge, Chrome, or Firefox recommended)
2. Navigate to the URL: **https://make.powerautomate.com**
3. If prompted, sign in with your Microsoft 365 credentials
4. Ensure you are in the correct environment (same as Copilot Studio)

#### Step 1.2: Create New Agent Flow

1. In the left navigation panel, click on **My flows**
2. Click the **+ New flow** button at the top
3. From the dropdown menu, select **Instant cloud flow**
4. Configure:
   - **Flow name**: `Read Application Inventory Data`
   - **Trigger**: Select **When an agent calls the flow** (Run a flow from Copilot)
5. Click the **Create** button

#### Step 1.3: Configure Flow Inputs

1. Click on the **When an agent calls the flow** trigger to expand it
2. Add the following input parameters:
   - Click **+ Add an input**
   - Select **Text**
   - **Name**: `filePath`
   - **Description**: `Path to the uploaded Azure Migrate file`
3. Add another input:
   - Click **+ Add an input**
   - Select **Text**
   - **Name**: `sessionId`
   - **Description**: `Processing session identifier`

#### Step 1.4: Add Excel Data Retrieval

##### For SharePoint Storage:

1. Click **+** below the trigger → **Add an action**
2. Search for `Excel Online`
3. Select **List rows present in a table** (under Excel Online (Business))
4. If prompted, sign in to create the Excel Online connection
5. Configure:
   - **Location**: Select your SharePoint site
   - **Document Library**: Select `Uploads` (or your upload library)
   - **File**: Click **Dynamic content** → Select `filePath` from trigger
   - **Table Name**: Type `ApplicationInventory`
6. Rename to: `Get ApplicationInventory Rows`

##### For Azure Blob Storage:

1. Add **Azure Blob Storage** connector → **Get blob content (V2)**
2. Then add a **Parse JSON** action to parse the CSV/Excel content
3. Rename to: `Get ApplicationInventory Rows`

#### Step 1.5: Return Data to Agent

1. Click **+** → **Add an action**
2. Search for `Respond to the agent`
3. Select **Respond to the agent** (or "Respond to Copilot")
4. Add output parameters:
   - Click **+ Add an output** → Select **Text**
   - **Name**: `rawApplicationData`
   - **Value**: Click **Expression** → Type:
     ```
     coalesce(string(body('Get_ApplicationInventory_Rows')?['value']), '[]')
     ```
   - Click **+ Add an output** → Select **Text**
   - **Name**: `rowCount`
   - **Value**: Click **Expression** → Type:
     ```
     string(length(body('Get_ApplicationInventory_Rows')?['value']))
     ```
   - Click **+ Add an output** → Select **Text**
   - **Name**: `status`
   - **Value**: `DataReady`

> **Note**: Every output parameter must have a non-null value. The `coalesce()` wrapper ensures a safe default if the data retrieval returns null. The `string()` function serializes the array as text for the agent.

#### Step 1.6: Save the Flow

1. Click **Save** at the top-right
2. Wait for the "Flow saved" confirmation

#### Step 1.7: Verify Flow Diagram

Your completed data extraction flow should look like this:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When an agent calls the flow                                            │
│ (Inputs: filePath, sessionId)                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Get ApplicationInventory Rows                                            │
│ (List rows from Excel ApplicationInventory table)                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Respond to the agent                                                     │
│ • rawApplicationData (JSON string of all rows)                          │
│ • rowCount (total rows read)                                            │
│ • status ("DataReady")                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Step 2: Add the Data Extraction Tool to the Copilot Agent

#### Step 2.1: Open Copilot Studio

1. Navigate to **https://copilotstudio.microsoft.com**
2. Sign in with your Microsoft 365 credentials
3. Open your **Azure Migrate Processing** agent

#### Step 2.2: Add the Flow as a Tool

1. In the left navigation, click **Tools**
2. Click **+ Add a tool**
3. Search for and select your **Read Application Inventory Data** flow
4. Review the tool configuration:
   - Inputs: `filePath` (Text), `sessionId` (Text)
   - Outputs: `rawApplicationData` (Text), `rowCount` (Text), `status` (Text)
5. Click **Add** to confirm

---

### Step 3: Configure Agent Instructions for Application Analysis

The agent's Instructions define the LLM's analysis behavior. This is where you provide the intelligence that replaces the old Power Automate pattern-matching logic.

#### Step 3.1: Open Agent Settings

1. In Copilot Studio, click **Settings** (gear icon) in the top bar
2. Click **Generative AI** (or **AI** depending on your version)
3. Locate the **Instructions** section (sometimes called "How should your copilot act?")

#### Step 3.2: Add Application Analysis Instructions

Add the following to your agent's instructions. If the agent already has instructions, append this section:

```
## Application Inventory Analysis

When the user asks to process application inventory data, or when processing is triggered
as part of the Azure Migrate workflow:

1. Call the "Read Application Inventory Data" tool to retrieve raw data from the file
2. Analyze the returned data using the rules below
3. Return a consolidated unique application list

### Noise Identification Rules
Classify the following as NOISE and EXCLUDE them from the unique application list:
- Windows Updates, Security Updates, Cumulative Updates (names containing "Update", "KB",
  "Hotfix")
- Software patches and service packs (names containing "Patch", "Service Pack")
- Runtime packages and redistributables (names containing "Redistributable", "Runtime")
- .NET Framework installations (names containing ".NET Framework")
- Visual C++ Redistributables (names containing "Visual C++")
- Definition Updates (names containing "Definition Update")
- Device drivers and driver update packages
- OS components, Windows features, and system utilities
- Application-dependent drivers or updates — for example:
  - "SQL Server ODBC Driver" → noise (driver dependency)
  - "Oracle Client" → noise (database client dependency)
  - "MySQL Connector" → noise (connector dependency)
  - "SAP Crystal Reports Runtime" → noise (runtime dependency)

### Consolidation Rules
- Group applications by EXACT name match (case-insensitive, trim whitespace)
- If the same application appears with DIFFERENT versions, keep EACH version as a SEPARATE
  entry
- If the same application name + version appears on multiple machines, keep only ONE entry
  but list all machine names
- Preserve the original casing and formatting from the source data in the output

### Required Output Format
Return the consolidated list as a JSON array:
[
  {
    "Application": "original application name",
    "Version": "version string",
    "Provider": "vendor/provider name",
    "Machines": ["machine1", "machine2"],
    "MachineCount": 2
  }
]
Sort alphabetically by Application name, then by Version.
Include a summary with: total rows processed, unique applications found, noise items removed,
and duplicates consolidated.
```

> **Note**: These instructions guide the LLM's reasoning. The LLM will use these rules to intelligently classify applications rather than relying on rigid substring matching. This means the LLM can also catch edge cases like misspellings, alternate naming patterns, and context-dependent classifications that rigid pattern matching would miss.

---

### Step 4: Create the Application Processing Topic

#### Step 4.1: Create New Topic

1. In Copilot Studio, go to **Topics** in the left navigation
2. Click **+ Add** → **Topic** → **From blank**
3. Name the topic: `Process Application Inventory`

#### Step 4.2: Add Trigger Phrases

1. In the **Trigger** section, add trigger phrases:
   - `Process application inventory`
   - `Analyze applications`
   - `Consolidate application list`
   - `Process apps from file`

#### Step 4.3: Add Message Node - Processing Start

1. Below the trigger, click **+** to add a node
2. Select **Send a message**
3. Enter: `⏳ Reading application inventory data from the uploaded file...`

#### Step 4.4: Add Tool Call Node - Read Application Data

1. Click **+** → **Call an action** → Select **Read Application Inventory Data** (your tool)
2. Map the inputs:
   - **filePath**: Select the file path variable from your upload workflow (e.g., `Global.uploadedFilePath`)
   - **sessionId**: Select the session ID variable (e.g., `Global.sessionId`)

> **Note**: The `Global.uploadedFilePath` and `Global.sessionId` variables should be set by the **Agent 1: File Upload Handler** topic after a successful file upload. If these global variables do not yet exist, create them in the File Upload Handler topic by adding "Set a variable value" nodes and changing the scope from "Topic" to "Global".

3. Store the outputs:
   - `rawApplicationData` → save to `Topic.rawAppData`
   - `rowCount` → save to `Topic.appRowCount`
   - `status` → save to `Topic.readStatus`

#### Step 4.5: Add Message Node - LLM Analysis Prompt

This is the key step where the LLM performs the analysis. Add a **Send a message** node with the following content:

1. Click **+** → **Send a message**
2. Enter the following message (the LLM will process this based on the agent instructions):

```
I have retrieved {Topic.appRowCount} application inventory rows. Now analyzing the data
to identify noise and consolidate unique business applications.

Raw application data:
{Topic.rawAppData}

Please analyze this data following the Application Inventory Analysis rules in your
instructions and return:
1. The consolidated unique application list in JSON format
2. A summary of what was filtered and why
```

> **Note**: The Copilot agent's LLM will apply the analysis rules from the agent Instructions (Step 3) to process this data. The LLM uses reasoning to identify noise, detect dependencies, and consolidate applications — this is more intelligent than pattern-matching.

#### Step 4.6: Add Variable Node - Store Results

1. Click **+** → **Set a variable value**
2. Set variable: **Global.consolidatedApplications**
   - Change scope from "Topic" to "Global" in the variable properties panel
3. Value: Set to the LLM's analysis output

> **Note**: The LLM's response will include both narrative text and the JSON array. In Copilot Studio, you can instruct the LLM (via the analysis prompt in Step 4.5) to output the JSON array in a clearly delimited format. The variable should capture the structured JSON output. If the LLM's response includes both text and JSON, you may need to use a **Parse value** node or an additional **Compose** action in a follow-up Power Automate tool to extract the JSON portion.

> **Tip**: For reliable extraction, add a line to your LLM analysis prompt such as: *"Return ONLY the JSON array as your final message, with no additional text."* This makes it easier to store the entire response as the variable value.

#### Step 4.7: Add Confirmation Message

1. Click **+** → **Send a message**
2. Enter:

```
✅ **Application inventory processed successfully!**

📊 **Summary:**
- Total rows analyzed: {Topic.appRowCount}
- Unique business applications identified
- Noise removed: patches, updates, drivers, OS components, and dependencies

The consolidated application list has been stored and is ready for report generation.
```

---

### Step 5: Test the Application Processing Agent

#### Step 5.1: Test in Copilot Studio

1. Click the **Test** button (chat panel on the right side of Copilot Studio)
2. Type: `Process application inventory`
3. Verify:
   - The agent calls the "Read Application Inventory Data" tool
   - The agent displays the row count
   - The LLM analyzes the data and returns a consolidated list
   - Noise items (updates, patches, drivers) are correctly identified and excluded
   - Different versions of the same application are preserved as separate entries
   - The summary statistics are accurate

#### Step 5.2: Validate Consolidation Rules

Test with sample data that includes:
- Duplicate applications (same name, same version) → should be consolidated
- Same application with different versions → should appear as separate entries
- Windows Updates and KB articles → should be removed as noise
- Application-dependent drivers (e.g., "SQL Server Native Client") → should be removed
- Legitimate business applications → should be preserved

---

### Application Consolidation Rules Reference

The LLM applies these consolidation rules using reasoning:

| Rule | Description | Example |
|------|-------------|---------|
| **Noise: Updates/Patches** | Names indicating updates, patches, hotfixes, KBs | "Security Update for Windows (KB12345)" → removed |
| **Noise: Redistributables** | Runtime libraries and redistributable packages | "Microsoft Visual C++ 2015 Redistributable" → removed |
| **Noise: Drivers/Dependencies** | Application-dependent drivers, connectors, clients | "SQL Server ODBC Driver 17" → removed |
| **Noise: OS Components** | Operating system features and system utilities | ".NET Framework 4.8" → removed |
| **Exact Name Consolidation** | Same name (case-insensitive) with same version | "Adobe Reader" + "adobe reader" → one entry |
| **Version Preservation** | Same application with different versions | "Java 8" and "Java 11" → two separate entries |
| **Multi-Machine Merge** | Same app+version on different machines | Merged with machine list and count |

---

## Agent 3: SQL Server Inventory Processor

### Purpose

This agent uses the Copilot Studio LLM to analyze and consolidate SQL Server inventory data from Azure Migrate exports. The LLM applies reasoning to consolidate SQL Server instances, group by version, remove updates and dependent clients, and generate a unique list of SQL Server versions and entries. Power Automate is used **only** for reading raw data from the uploaded file.

---

### How the LLM-Based Approach Works

1. A lightweight Power Automate **tool** reads raw SQL Server data from the uploaded file and returns it as structured JSON
2. The Copilot agent's LLM receives the raw data and applies intelligent analysis:
   - **Consolidation by version**: Groups SQL Server entries by version, identifying unique SQL Server installations
   - **Update/client removal**: Identifies and removes SQL Server updates, cumulative updates, service packs (as separate noise entries), and dependent client tools (SSMS, SQL Native Client, etc.)
   - **Instance deduplication**: Consolidates duplicate entries for the same instance (MachineName + InstanceName) while preserving distinct version entries
   - **Version grouping**: Organizes results grouped by SQL Server version for clarity
3. The LLM outputs a clean, deduplicated JSON array of unique SQL Server entries
4. Results are stored in a global variable for the Report Generator

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Copilot Agent Topic: Process SQL Server Inventory                           │
│                                                                             │
│  1. Call Tool: "Read SQL Server Inventory Data" (Power Automate)            │
│     └─→ Returns raw SQL Server rows as JSON                                │
│                                                                             │
│  2. LLM Analysis (performed by the Copilot agent's model):                 │
│     ├─→ Consolidate SQL Server instances by version                        │
│     ├─→ Remove updates, cumulative updates, and dependent clients          │
│     ├─→ Deduplicate by MachineName + InstanceName                          │
│     ├─→ Group results by SQL Server version                                │
│     └─→ Output structured JSON array of unique SQL entries                 │
│                                                                             │
│  3. Store results in Global.consolidatedSQLInstances                       │
│  4. Report summary to user                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Step 1: Create the SQL Server Data Extraction Tool (Power Automate Flow)

This minimal flow reads raw SQL Server inventory data from the uploaded file and returns it to the Copilot agent.

#### Step 1.1: Access Power Automate

1. Open your web browser
2. Navigate to: **https://make.powerautomate.com**
3. Sign in with your Microsoft 365 credentials if prompted
4. Ensure you are in the correct environment (same as Copilot Studio)

#### Step 1.2: Create New Agent Flow

1. In the left navigation, click **My flows**
2. Click **+ New flow** → **Instant cloud flow**
3. Configure:
   - **Flow name**: `Read SQL Server Inventory Data`
   - **Trigger**: Select **When an agent calls the flow** (Run a flow from Copilot)
4. Click **Create**

#### Step 1.3: Configure Flow Inputs

1. Click on the trigger to expand it
2. Add input parameters:
   - Click **+ Add an input** → **Text**
   - **Name**: `filePath`
   - **Description**: `Path to the uploaded Azure Migrate file`
3. Add another input:
   - Click **+ Add an input** → **Text**
   - **Name**: `sessionId`
   - **Description**: `Processing session identifier`

#### Step 1.4: Add Excel Data Retrieval

1. Click **+** → **Add an action**
2. Search for `Excel Online`
3. Select **List rows present in a table** (Excel Online Business)
4. If prompted, sign in to create the connection
5. Configure:
   - **Location**: Select your SharePoint site from the dropdown
   - **Document Library**: Select `Uploads`
   - **File**: Click **Dynamic content** → Select `filePath`
   - **Table Name**: Type `SQLServer`
6. Rename action to: `Get SQL Server Rows`

#### Step 1.5: Return Data to Agent

1. Click **+** → **Add an action**
2. Search for and select **Respond to the agent**
3. Add output parameters:
   - **Name**: `rawSQLData` | **Type**: Text | **Value**: Expression: `coalesce(string(body('Get_SQL_Server_Rows')?['value']), '[]')`
   - **Name**: `rowCount` | **Type**: Text | **Value**: Expression: `string(length(body('Get_SQL_Server_Rows')?['value']))`
   - **Name**: `status` | **Type**: Text | **Value**: `DataReady`

> **Note**: Every output parameter must have a non-null value. Use `coalesce()` to provide safe defaults.

#### Step 1.6: Save the Flow

1. Click **Save** at the top-right
2. Wait for the "Flow saved" confirmation

#### Step 1.7: Verify Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When an agent calls the flow                                            │
│ (Inputs: filePath, sessionId)                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Get SQL Server Rows (List rows from Excel SQLServer table)              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Respond to the agent                                                     │
│ • rawSQLData (JSON string of all rows)                                  │
│ • rowCount (total rows read)                                            │
│ • status ("DataReady")                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Step 2: Add the SQL Server Tool to the Copilot Agent

#### Step 2.1: Add the Flow as a Tool

1. In Copilot Studio, click **Tools** in the left navigation
2. Click **+ Add a tool**
3. Search for and select your **Read SQL Server Inventory Data** flow
4. Review the tool configuration:
   - Inputs: `filePath` (Text), `sessionId` (Text)
   - Outputs: `rawSQLData` (Text), `rowCount` (Text), `status` (Text)
5. Click **Add** to confirm

---

### Step 3: Configure Agent Instructions for SQL Server Analysis

#### Step 3.1: Add SQL Server Analysis Instructions

In the agent's Instructions (Settings → Generative AI → Instructions), append the following:

```
## SQL Server Inventory Analysis

When the user asks to process SQL Server inventory data, or when processing is triggered
as part of the Azure Migrate workflow:

1. Call the "Read SQL Server Inventory Data" tool to retrieve raw data
2. Analyze the returned data using the rules below
3. Return a consolidated unique SQL Server list grouped by version

### Noise / Removal Rules for SQL Server
Classify the following as NOISE and EXCLUDE from the unique SQL Server list:
- SQL Server cumulative updates and update entries
- SQL Server service pack entries (when listed as separate rows)
- SQL Server dependent client tools:
  - SQL Server Management Studio (SSMS)
  - SQL Server Native Client
  - SQL Server ODBC Driver
  - SQL Server OLE DB Driver
  - SQL Server Data Tools
  - SQL Server Reporting Services (if listed as a standalone entry)
  - SQL Server Integration Services (if listed as a standalone entry)
  - SQL Server command-line utilities
- SQL Server setup/installer entries
- SQL Server feature packs

### Consolidation Rules for SQL Server
- Consolidate by MachineName + Instance Name combination (case-insensitive)
- If Instance Name is empty or missing, default to "MSSQLSERVER"
- If Port is empty or missing, default to "1433"
- Group the unique SQL Server entries by Version for organized output
- If the same instance appears with different editions, keep each as a separate entry
- Preserve the original Edition, ServicePack, and Version values in the output

### Required Output Format
Return the consolidated list as a JSON array grouped by version:
[
  {
    "MachineName": "server name",
    "InstanceName": "instance name or MSSQLSERVER",
    "Edition": "SQL Server edition",
    "Version": "version string",
    "ServicePack": "service pack level",
    "Port": "port number or 1433",
    "MachineManagerFqdn": "FQDN"
  }
]
Sort by Version (descending, newest first), then by MachineName.
Include a summary with: total rows processed, unique instances found, updates/clients
removed, and version groups identified.
```

---

### Step 4: Create the SQL Server Processing Topic

#### Step 4.1: Create New Topic

1. In Copilot Studio, go to **Topics**
2. Click **+ Add** → **Topic** → **From blank**
3. Name the topic: `Process SQL Server Inventory`

#### Step 4.2: Add Trigger Phrases

1. Add trigger phrases:
   - `Process SQL Server inventory`
   - `Analyze SQL Server instances`
   - `Consolidate SQL Server list`

#### Step 4.3: Add Message Node - Processing Start

1. Click **+** → **Send a message**
2. Enter: `⏳ Reading SQL Server inventory data from the uploaded file...`

#### Step 4.4: Add Tool Call Node - Read SQL Data

1. Click **+** → **Call an action** → Select **Read SQL Server Inventory Data**
2. Map inputs:
   - **filePath**: `Global.uploadedFilePath`
   - **sessionId**: `Global.sessionId`
3. Store outputs:
   - `rawSQLData` → `Topic.rawSQLData`
   - `rowCount` → `Topic.sqlRowCount`
   - `status` → `Topic.readStatus`

#### Step 4.5: Add Message Node - LLM Analysis Prompt

1. Click **+** → **Send a message**
2. Enter:

```
I have retrieved {Topic.sqlRowCount} SQL Server inventory rows. Now analyzing the data
to consolidate SQL Server instances and group by version.

Raw SQL Server data:
{Topic.rawSQLData}

Please analyze this data following the SQL Server Inventory Analysis rules in your
instructions and return:
1. The consolidated unique SQL Server instance list in JSON format, grouped by version
2. A summary of what was removed (updates, dependent clients) and why
3. The distinct SQL Server versions found
```

#### Step 4.6: Add Variable Node - Store Results

1. Click **+** → **Set a variable value**
2. Set variable: **Global.consolidatedSQLInstances**
   - Change scope to "Global"
3. Value: Set to the LLM's analysis output (the JSON array)

#### Step 4.7: Add Confirmation Message

1. Click **+** → **Send a message**
2. Enter:

```
✅ **SQL Server inventory processed successfully!**

📊 **Summary:**
- Total rows analyzed: {Topic.sqlRowCount}
- Unique SQL Server instances identified and grouped by version
- Removed: updates, cumulative updates, dependent clients, and setup entries

The consolidated SQL Server list has been stored and is ready for report generation.
```

---

### Step 5: Test the SQL Server Processing Agent

#### Step 5.1: Test in Copilot Studio

1. Click the **Test** button
2. Type: `Process SQL Server inventory`
3. Verify:
   - The agent calls the "Read SQL Server Inventory Data" tool
   - The LLM correctly consolidates instances by MachineName + InstanceName
   - Updates and dependent clients (SSMS, Native Client, ODBC Driver) are removed
   - Results are grouped by SQL Server version
   - Default values are applied (MSSQLSERVER for empty instance, 1433 for empty port)

---

### SQL Server Consolidation Rules Reference

| Rule | Description | Example |
|------|-------------|---------|
| **Unique Instance** | MachineName + Instance Name combination | Server01_MSSQLSERVER |
| **Default Instance** | Empty Instance Name defaults to "MSSQLSERVER" | Server01 → Server01_MSSQLSERVER |
| **Default Port** | Empty Port defaults to "1433" | Standard SQL Server port |
| **Version Grouping** | Group results by SQL Server version | SQL 2019, SQL 2017, SQL 2016 groups |
| **Noise: Updates** | SQL Server updates and cumulative updates | "SQL Server 2019 CU15" → removed |
| **Noise: Clients** | Dependent client tools and drivers | "SQL Server Native Client 11.0" → removed |
| **Case Normalization** | All comparisons are case-insensitive | SERVER01 = server01 |

---

## Agent 4: Web App Inventory Processor

### Purpose

This agent uses the Copilot Studio LLM to analyze and consolidate web application inventory data from Azure Migrate exports. The LLM applies reasoning to identify unique web applications, remove noise and duplicates, and produce a comprehensive unique web app list. Power Automate is used **only** for reading raw data from the uploaded file.

---

### How the LLM-Based Approach Works

1. A lightweight Power Automate **tool** reads raw web application data from the uploaded file and returns it as structured JSON
2. The Copilot agent's LLM receives the raw data and applies intelligent analysis:
   - **Web app identification**: Identifies distinct web applications by name, server type, and configuration
   - **Noise removal**: Filters out default/system web apps, placeholder entries, and infrastructure components
   - **Consolidation**: Groups identical web apps across machines, preserving unique configurations
   - **Framework detection**: Identifies framework/runtime versions associated with each web app
3. The LLM outputs a clean, deduplicated JSON array of unique web applications
4. Results are stored in a global variable for the Report Generator

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Copilot Agent Topic: Process Web App Inventory                              │
│                                                                             │
│  1. Call Tool: "Read Web App Inventory Data" (Power Automate)               │
│     └─→ Returns raw web application rows as JSON                           │
│                                                                             │
│  2. LLM Analysis (performed by the Copilot agent's model):                 │
│     ├─→ Identify unique web applications by name and configuration         │
│     ├─→ Remove default/system web apps and noise                           │
│     ├─→ Consolidate identical apps across multiple machines                │
│     ├─→ Detect and preserve framework/runtime information                  │
│     └─→ Output structured JSON array of unique web apps                    │
│                                                                             │
│  3. Store results in Global.consolidatedWebApps                            │
│  4. Report summary to user                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Step 1: Create the Web App Data Extraction Tool (Power Automate Flow)

This minimal flow reads raw web application inventory data from the uploaded file and returns it to the Copilot agent.

#### Step 1.1: Access Power Automate

1. Open your web browser
2. Navigate to: **https://make.powerautomate.com**
3. Sign in with your Microsoft 365 credentials if prompted
4. Ensure you are in the correct environment (same as Copilot Studio)

#### Step 1.2: Create New Agent Flow

1. In the left navigation, click **My flows**
2. Click **+ New flow** → **Instant cloud flow**
3. Configure:
   - **Flow name**: `Read Web App Inventory Data`
   - **Trigger**: Select **When an agent calls the flow** (Run a flow from Copilot)
4. Click **Create**

#### Step 1.3: Configure Flow Inputs

1. Click on the trigger to expand it
2. Add input parameters:
   - Click **+ Add an input** → **Text**
   - **Name**: `filePath`
   - **Description**: `Path to the uploaded Azure Migrate file`
3. Add another input:
   - Click **+ Add an input** → **Text**
   - **Name**: `sessionId`
   - **Description**: `Processing session identifier`

#### Step 1.4: Add Excel Data Retrieval

1. Click **+** → **Add an action**
2. Search for `Excel Online`
3. Select **List rows present in a table**
4. Configure:
   - **Location**: Select your SharePoint site
   - **Document Library**: Select `Uploads`
   - **File**: Click **Dynamic content** → Select `filePath`
   - **Table Name**: Type `WebApplications`
5. Rename to: `Get Web App Rows`

#### Step 1.5: Return Data to Agent

1. Click **+** → **Add an action**
2. Search for and select **Respond to the agent**
3. Add output parameters:
   - **Name**: `rawWebAppData` | **Type**: Text | **Value**: Expression: `coalesce(string(body('Get_Web_App_Rows')?['value']), '[]')`
   - **Name**: `rowCount` | **Type**: Text | **Value**: Expression: `string(length(body('Get_Web_App_Rows')?['value']))`
   - **Name**: `status` | **Type**: Text | **Value**: `DataReady`

> **Note**: Every output parameter must have a non-null value. Use `coalesce()` to provide safe defaults.

#### Step 1.6: Save the Flow

1. Click **Save** at the top-right
2. Wait for the "Flow saved" confirmation

#### Step 1.7: Verify Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When an agent calls the flow                                            │
│ (Inputs: filePath, sessionId)                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Get Web App Rows (List rows from Excel WebApplications table)           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Respond to the agent                                                     │
│ • rawWebAppData (JSON string of all rows)                               │
│ • rowCount (total rows read)                                            │
│ • status ("DataReady")                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Step 2: Add the Web App Tool to the Copilot Agent

#### Step 2.1: Add the Flow as a Tool

1. In Copilot Studio, click **Tools** in the left navigation
2. Click **+ Add a tool**
3. Search for and select your **Read Web App Inventory Data** flow
4. Review the tool configuration:
   - Inputs: `filePath` (Text), `sessionId` (Text)
   - Outputs: `rawWebAppData` (Text), `rowCount` (Text), `status` (Text)
5. Click **Add** to confirm

---

### Step 3: Configure Agent Instructions for Web App Analysis

#### Step 3.1: Add Web App Analysis Instructions

In the agent's Instructions (Settings → Generative AI → Instructions), append the following:

```
## Web App Inventory Analysis

When the user asks to process web application inventory data, or when processing is
triggered as part of the Azure Migrate workflow:

1. Call the "Read Web App Inventory Data" tool to retrieve raw data
2. Analyze the returned data using the rules below
3. Return a consolidated unique web application list

### Noise / Removal Rules for Web Apps
Classify the following as NOISE and EXCLUDE from the unique web app list:
- Default web server pages and placeholder sites (e.g., "Default Web Site", "IIS Start Page")
- Web server administration consoles and management interfaces
- Health check endpoints and monitoring agents
- Auto-generated or system-created application pools
- Web server modules and handlers (not actual applications)
- Test or sample applications (e.g., "iisstart", "aspnet_client")

### Consolidation Rules for Web Apps
- Identify unique web applications by WebAppName (case-insensitive, trim whitespace)
- If the same web app runs on different web server types (IIS vs Apache), keep as
  SEPARATE entries
- If the same web app appears on multiple machines, keep only ONE entry but list all
  machine names
- Preserve the original WebServerType, FrameworkVersion, VirtualDirectory, and
  ApplicationPool values in the output
- Group results by WebServerType for organized output

### Required Output Format
Return the consolidated list as a JSON array:
[
  {
    "WebAppName": "application name",
    "WebServerType": "IIS / Apache / Tomcat / Nginx / etc.",
    "VirtualDirectory": "virtual path",
    "ApplicationPool": "app pool name",
    "FrameworkVersion": "framework/runtime version",
    "Machines": ["machine1", "machine2"],
    "MachineCount": 2,
    "MachineManagerFqdn": "FQDN"
  }
]
Sort by WebServerType, then alphabetically by WebAppName.
Include a summary with: total rows processed, unique web apps found, noise items removed,
and web server types identified.
```

---

### Step 4: Create the Web App Processing Topic

#### Step 4.1: Create New Topic

1. In Copilot Studio, go to **Topics**
2. Click **+ Add** → **Topic** → **From blank**
3. Name the topic: `Process Web App Inventory`

#### Step 4.2: Add Trigger Phrases

1. Add trigger phrases:
   - `Process web app inventory`
   - `Analyze web applications`
   - `Consolidate web app list`
   - `Process web servers`

#### Step 4.3: Add Message Node - Processing Start

1. Click **+** → **Send a message**
2. Enter: `⏳ Reading web application inventory data from the uploaded file...`

#### Step 4.4: Add Tool Call Node - Read Web App Data

1. Click **+** → **Call an action** → Select **Read Web App Inventory Data**
2. Map inputs:
   - **filePath**: `Global.uploadedFilePath`
   - **sessionId**: `Global.sessionId`
3. Store outputs:
   - `rawWebAppData` → `Topic.rawWebAppData`
   - `rowCount` → `Topic.webAppRowCount`
   - `status` → `Topic.readStatus`

#### Step 4.5: Add Message Node - LLM Analysis Prompt

1. Click **+** → **Send a message**
2. Enter:

```
I have retrieved {Topic.webAppRowCount} web application inventory rows. Now analyzing
the data to identify unique web applications.

Raw web application data:
{Topic.rawWebAppData}

Please analyze this data following the Web App Inventory Analysis rules in your
instructions and return:
1. The consolidated unique web application list in JSON format
2. A summary of what was filtered and why
3. The distinct web server types found
```

#### Step 4.6: Add Variable Node - Store Results

1. Click **+** → **Set a variable value**
2. Set variable: **Global.consolidatedWebApps**
   - Change scope to "Global"
3. Value: Set to the LLM's analysis output (the JSON array)

#### Step 4.7: Add Confirmation Message

1. Click **+** → **Send a message**
2. Enter:

```
✅ **Web application inventory processed successfully!**

📊 **Summary:**
- Total rows analyzed: {Topic.webAppRowCount}
- Unique web applications identified
- Removed: default sites, system pages, and management interfaces

The consolidated web application list has been stored and is ready for report generation.
```

---

### Step 5: Test the Web App Processing Agent

#### Step 5.1: Test in Copilot Studio

1. Click the **Test** button
2. Type: `Process web app inventory`
3. Verify:
   - The agent calls the "Read Web App Inventory Data" tool
   - The LLM correctly identifies unique web applications
   - Default/system web apps are filtered as noise
   - Same web app on multiple machines is consolidated
   - Results are grouped by web server type
   - Framework/runtime information is preserved

---

### Web App Consolidation Rules Reference

| Rule | Description | Example |
|------|-------------|---------|
| **Unique Web App** | WebAppName + WebServerType combination | "MyApp" on IIS ≠ "MyApp" on Apache |
| **Noise: Default Sites** | Default server pages and placeholders | "Default Web Site" → removed |
| **Noise: System Apps** | Management consoles and system endpoints | "IIS Manager" → removed |
| **Multi-Machine Merge** | Same app on different machines | Merged with machine list and count |
| **Framework Preservation** | Keep framework/runtime version info | ".NET 4.8", "PHP 7.4" preserved |
| **Server Type Grouping** | Group results by web server type | IIS apps, Apache apps, Tomcat apps |

---

## Agent 5: Report Generator

### Purpose
Generate the final consolidated Excel spreadsheet with all unique applications, SQL Server instances, and web applications, and provide a download link to the user. This flow creates a professional multi-sheet Excel report.

> **⚠️ Prerequisite**: Before configuring this flow, you must create an Excel template with predefined tables. See the [Creating the Excel Template](#creating-the-excel-template) section at the end of Agent 5 for detailed instructions. Upload the template to your SharePoint site (e.g., `/Templates/ReportTemplate.xlsx`) before proceeding.

---

### Step 1: Create the Report Generation Flow

#### Step 1.1: Access Power Automate

1. Open your web browser
2. Navigate to: **https://make.powerautomate.com**
3. Sign in with your Microsoft 365 credentials if prompted
4. Ensure you are in the correct environment

#### Step 1.2: Create New Flow

1. In the left navigation, click **My flows**
2. Click **+ New flow** → **Instant cloud flow**
3. Configure:
   - **Flow name**: `Generate Consolidated Report`
   - **Trigger**: Select **When a HTTP request is received**
4. Click **Create**

#### Step 1.3: Configure the HTTP Trigger with Input Schema

1. Click on the trigger to expand it
2. Click **Use sample payload to generate schema**
3. Paste the following comprehensive JSON:

> **EXAMPLE ONLY — replace values with your own.** The machine names, domains, and IDs below are sample data used solely to generate the trigger schema.

```json
{
  "uniqueApplications": [
    {
      "Application": "Microsoft SQL Server 2019",
      "Version": "15.0.2000.5",
      "Provider": "Microsoft",
      "MachineName": "Server01",
      "MachineManagerFqdn": "manager.contoso.com"
    }
  ],
  "uniqueSQLInstances": [
    {
      "MachineName": "SQLServer01",
      "InstanceName": "MSSQLSERVER",
      "Edition": "Enterprise",
      "ServicePack": "SP2",
      "Version": "15.0.2000.5",
      "Port": "1433",
      "MachineManagerFqdn": "manager.contoso.com"
    }
  ],
  "uniqueWebApps": [
    {
      "WebAppName": "ContosoPortal",
      "WebServerType": "IIS",
      "VirtualDirectory": "/portal",
      "ApplicationPool": "ContosoAppPool",
      "FrameworkVersion": ".NET 4.8",
      "MachineName": "WebServer01",
      "MachineManagerFqdn": "manager.contoso.com"
    }
  ],
  "sessionId": "session-123-guid",
  "userEmail": "user@contoso.com",
  "storageType": "SharePoint"
}
```
4. Click **Done**

---

### Step 2: Initialize Flow Variables

#### Step 2.1: Add Variable - reportFileName

1. Click **+** → **Add an action**
2. Search for and select **Initialize variable**
3. Configure:
   - **Name**: `reportFileName`
   - **Type**: **String**
   - **Value**: Click **Expression** → Type:
   ```
   concat('ConsolidatedReport_', formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'), '.xlsx')
   ```
4. Rename to: `Initialize reportFileName`

#### Step 2.2: Add Variable - downloadUrl

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `downloadUrl`
   - **Type**: **String**
   - **Value**: (leave empty for now)
3. Rename to: `Initialize downloadUrl`

#### Step 2.3: Add Variable - reportFilePath

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `reportFilePath`
   - **Type**: **String**
   - **Value**: (leave empty)
3. Rename to: `Initialize reportFilePath`

---

### Step 3: Create the Report Folder

#### Step 3.1: Create Folder in SharePoint (For SharePoint Storage)

1. Click **+** → **Add an action**
2. Search for `SharePoint`
3. Select **Create new folder**
4. Configure:
   - **Site Address**: Select your SharePoint site
   - **List or Library**: Select `Reports`
   - **Folder Path**: Type or use expression:
   ```
   /@{triggerBody()?['sessionId']}
   ```
5. Rename to: `Create Report Folder`

**Note**: If the folder already exists, add a **Scope** with "Configure run after" set to continue on failure.

---

### Step 4: Create the Excel File with Template

#### Step 4.1: Option A - Create from Template (Recommended)

If you have a template Excel file:

1. Click **+** → **Add an action**
2. Search for `SharePoint`
3. Select **Copy file**
4. Configure:
   - **Current Site Address**: Site with template
   - **File to Copy**: `/Templates/ReportTemplate.xlsx`
   - **Destination Site Address**: Your SharePoint site
   - **Destination Folder**: `/Reports/@{triggerBody()?['sessionId']}`
   - **If Another File is Already There**: **Replace**
5. Rename to: `Copy Report Template`

6. Click **+** → **Add an action**
7. Select **Rename file** (SharePoint)
8. Configure:
   - **Site Address**: Your SharePoint site
   - **File Identifier**: Use Dynamic content from Copy file
   - **New Name**: `@{variables('reportFileName')}`
9. Rename to: `Rename Report File`

#### Step 4.2: Option B - Create File Dynamically

If creating from scratch:

1. Click **+** → **Add an action**
2. Search for `SharePoint`
3. Select **Create file**
4. Configure:
   - **Site Address**: Your SharePoint site
   - **Folder Path**: `/Reports/@{triggerBody()?['sessionId']}`
   - **File Name**: `@{variables('reportFileName')}`
   - **File Content**: Leave empty (we'll populate via Excel connector)
5. Rename to: `Create Empty Report File`

**Note**: Creating Excel files from scratch requires additional setup. Using a template is recommended.

---

### Step 5: Set Report File Path

1. Click **+** → **Add an action**
2. Select **Set variable**
3. Configure:
   - **Name**: `reportFilePath`
   - **Value**: Use Dynamic content:
   ```
   /Reports/@{triggerBody()?['sessionId']}/@{variables('reportFileName')}
   ```
4. Rename to: `Set Report File Path`

---

### Step 6: Populate Applications Sheet

#### Step 6.1: Add Header Row to Applications Table

1. Click **+** → **Add an action**
2. Search for `Excel Online (Business)`
3. Select **Add a row into a table**
4. Configure:
   - **Location**: Select your SharePoint site
   - **Document Library**: `Reports`
   - **File**: `@{variables('reportFilePath')}`
   - **Table**: `UniqueApplications` (must exist in template)

**Note**: If using a template, the table should already be defined.

#### Step 6.2: Add Applications Data with Apply to Each

1. Click **+** → **Add an action**
2. Select **Apply to each**
3. Configure:
   - **Select an output**: Click **Dynamic content** → `uniqueApplications` (from trigger)
4. Rename to: `Add Application Rows`

##### Inside the Loop - Add Row

1. Inside the loop, click **Add an action**
2. Search for and select **Add a row into a table** (Excel Online Business)
3. Configure:
   - **Location**: Your SharePoint site
   - **Document Library**: `Reports`
   - **File**: `@{variables('reportFilePath')}`
   - **Table**: `UniqueApplications`
   - **Row data**:
     - **Application**: Click **Dynamic content** → `Application` (from current item)
     - **Version**: Click **Dynamic content** → `Version`
     - **Provider**: Click **Dynamic content** → `Provider`
     - **MachineName**: Click **Dynamic content** → `MachineName`
4. Rename to: `Add Application Row`

---

### Step 7: Populate SQL Server Sheet

#### Step 7.1: Add SQL Instance Rows

1. After the Applications loop, click **+** → **Add an action**
2. Select **Apply to each**
3. Configure:
   - **Select an output**: `uniqueSQLInstances` (from trigger)
4. Rename to: `Add SQL Instance Rows`

##### Inside the Loop - Add SQL Row

1. Inside the loop, click **Add an action**
2. Select **Add a row into a table** (Excel Online Business)
3. Configure:
   - **Location**: Your SharePoint site
   - **Document Library**: `Reports`
   - **File**: `@{variables('reportFilePath')}`
   - **Table**: `UniqueSQLInstances`
   - **Row data**:
     - **MachineName**: `MachineName`
     - **InstanceName**: `InstanceName`
     - **Edition**: `Edition`
     - **ServicePack**: `ServicePack`
     - **Version**: `Version`
     - **Port**: `Port`
4. Rename to: `Add SQL Instance Row`

---

### Step 8: Populate Web Apps Sheet

#### Step 8.1: Add Web App Rows

1. After the SQL loop, click **+** → **Add an action**
2. Select **Apply to each**
3. Configure:
   - **Select an output**: `uniqueWebApps` (from trigger)
4. Rename to: `Add Web App Rows`

##### Inside the Loop - Add Web App Row

1. Inside the loop, click **Add an action**
2. Select **Add a row into a table** (Excel Online Business)
3. Configure:
   - **Location**: Your SharePoint site
   - **Document Library**: `Reports`
   - **File**: `@{variables('reportFilePath')}`
   - **Table**: `UniqueWebApps`
   - **Row data**:
     - **WebAppName**: `WebAppName`
     - **WebServerType**: `WebServerType`
     - **VirtualDirectory**: `VirtualDirectory`
     - **ApplicationPool**: `ApplicationPool`
     - **FrameworkVersion**: `FrameworkVersion`
     - **MachineName**: `MachineName`
4. Rename to: `Add Web App Row`

---

### Step 9: Generate Download Link

#### Step 9.1: Create Sharing Link

1. After all data loops, click **+** → **Add an action**
2. Search for `SharePoint`
3. Select **Create sharing link for a file or folder**
4. Configure:
   - **Site Address**: Your SharePoint site
   - **Item Id**: Use expression to get file identifier:
     - Click **Expression** → Type: This requires the file ID
     - Alternative: Use **Get file metadata using path** first, then use that ID
   - **Link Type**: Select **View**
   - **Link Scope**: Select **People in your organization** (or appropriate scope)

**Better Approach - Get File Metadata First**:

1. Click **+** → **Add an action**
2. Select **Get file metadata using path** (SharePoint)
3. Configure:
   - **Site Address**: Your SharePoint site
   - **File Path**: `@{variables('reportFilePath')}`
4. Rename to: `Get Report File Metadata`

5. Click **+** → **Add an action**
6. Select **Create sharing link for a file or folder**
7. Configure:
   - **Site Address**: Your SharePoint site
   - **Item Id**: Click **Dynamic content** → Select **ItemId** from "Get Report File Metadata"
   - **Link Type**: **View**
   - **Link Scope**: **People in your organization**
8. Rename to: `Create Download Link`

#### Step 9.2: Store the Download URL

1. Click **+** → **Add an action**
2. Select **Set variable**
3. Configure:
   - **Name**: `downloadUrl`
   - **Value**: Click **Dynamic content** → Select **Web URL** from "Create Download Link" output
     - Or use expression: `body('Create_Download_Link')?['link']?['webUrl']`
4. Rename to: `Store Download URL`

---

### Step 10: Send Email Notification (Optional but Recommended)

#### Step 10.1: Add Email Action

1. Click **+** → **Add an action**
2. Search for `Outlook`
3. Select **Send an email (V2)** (Office 365 Outlook)
4. If prompted, sign in to create the connection

#### Step 10.2: Configure the Email

1. **To**: Click **Dynamic content** → Select `userEmail` from trigger
2. **Subject**: Type:
   ```
   ✅ Your Azure Migrate Consolidated Report is Ready
   ```
3. **Body**: Click in the body field and paste:

```html
<html>
<body style="font-family: 'Segoe UI', Arial, sans-serif; color: #333;">
<h2 style="color: #0078d4;">🎉 Your Azure Migrate Report is Ready!</h2>

<p>Hello,</p>

<p>Your Azure Migrate data has been processed successfully. Here's a summary of what was consolidated:</p>

<table style="border-collapse: collapse; margin: 20px 0;">
<tr style="background-color: #f4f4f4;">
<th style="padding: 10px; border: 1px solid #ddd; text-align: left;">Category</th>
<th style="padding: 10px; border: 1px solid #ddd; text-align: right;">Count</th>
</tr>
<tr>
<td style="padding: 10px; border: 1px solid #ddd;">📊 Unique Applications</td>
<td style="padding: 10px; border: 1px solid #ddd; text-align: right;"><strong>@{length(triggerBody()?['uniqueApplications'])}</strong></td>
</tr>
<tr>
<td style="padding: 10px; border: 1px solid #ddd;">🗄️ SQL Server Instances</td>
<td style="padding: 10px; border: 1px solid #ddd; text-align: right;"><strong>@{length(triggerBody()?['uniqueSQLInstances'])}</strong></td>
</tr>
<tr>
<td style="padding: 10px; border: 1px solid #ddd;">🌐 Web Applications</td>
<td style="padding: 10px; border: 1px solid #ddd; text-align: right;"><strong>@{length(triggerBody()?['uniqueWebApps'])}</strong></td>
</tr>
</table>

<p style="margin: 20px 0;">
<a href="@{variables('downloadUrl')}" style="background-color: #0078d4; color: white; padding: 12px 24px; text-decoration: none; border-radius: 4px; display: inline-block;">📥 Download Your Report</a>
</p>

<p><strong>Report Details:</strong></p>
<ul>
<li><strong>File Name:</strong> @{variables('reportFileName')}</li>
<li><strong>Session ID:</strong> @{triggerBody()?['sessionId']}</li>
<li><strong>Generated:</strong> @{utcNow()}</li>
</ul>

<p>The report includes three sheets:</p>
<ol>
<li><strong>Unique Applications</strong> - Consolidated application inventory with duplicates and noise removed</li>
<li><strong>SQL Server Inventory</strong> - Unique SQL Server instances across all machines</li>
<li><strong>Web App Inventory</strong> - Unique web applications across all machines</li>
</ol>

<hr style="margin: 20px 0; border: none; border-top: 1px solid #ddd;">
<p style="color: #666; font-size: 12px;">
This report was generated automatically by the Azure Migrate CSV Processor.<br>
If you have any questions, please contact your IT administrator.
</p>
</body>
</html>
```

5. **Importance**: Select **Normal**
6. Rename to: `Send Report Notification Email`

---

### Step 11: Return Results

#### Step 11.1: Compose Final Output

1. Click **+** → **Add an action** → **Compose**
2. In **Inputs**, paste:
```json
{
  "status": "Complete",
  "reportFileName": "@{variables('reportFileName')}",
  "downloadUrl": "@{variables('downloadUrl')}",
  "reportPath": "@{variables('reportFilePath')}",
  "statistics": {
    "uniqueApplications": @{length(triggerBody()?['uniqueApplications'])},
    "uniqueSQLInstances": @{length(triggerBody()?['uniqueSQLInstances'])},
    "uniqueWebApps": @{length(triggerBody()?['uniqueWebApps'])},
    "totalUniqueItems": @{add(add(length(triggerBody()?['uniqueApplications']), length(triggerBody()?['uniqueSQLInstances'])), length(triggerBody()?['uniqueWebApps']))}
  },
  "generatedAt": "@{utcNow()}",
  "sessionId": "@{triggerBody()?['sessionId']}",
  "emailSent": true
}
```
3. Rename to: `Compose Report Output`

#### Step 11.2: Add HTTP Response

1. Click **+** → **Add an action** → **Response**
2. Configure:
   - **Status Code**: `200`
   - **Headers**: Add `Content-Type`: `application/json`
   - **Body**: Click **Dynamic content** → Select **Outputs** from "Compose Report Output"
3. Rename to: `Return Report Results`

---

### Step 12: Save and Test

#### Step 12.1: Save the Flow

1. Click **Save** at the top-right
2. Wait for confirmation

#### Step 12.2: Copy the Flow URL

1. Click on the HTTP trigger
2. Copy the **HTTP POST URL**
3. Save this URL for the Orchestrator flow

#### Step 12.3: Test the Flow

1. Click **Test** → **Manually** → **Test**
2. Send a test request with sample data
3. Verify:
   - Excel file is created
   - All three sheets are populated
   - Download link is generated
   - Email is sent

---

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When a HTTP request is received                                         │
│ (Inputs: uniqueApplications, uniqueSQLInstances, uniqueWebApps,         │
│  sessionId, userEmail, storageType)                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Initialize Variables                                                     │
│ • reportFileName (String)                                                │
│ • downloadUrl (String)                                                   │
│ • reportFilePath (String)                                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Create Report Folder (SharePoint)                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Copy Report Template / Create Excel File                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Set Report File Path                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│ FOR EACH Application│ │ FOR EACH SQL Instance│ │ FOR EACH Web App   │
│   Add row to        │ │   Add row to         │ │   Add row to       │
│   UniqueApplications│ │   UniqueSQLInstances │ │   UniqueWebApps    │
│   table             │ │   table              │ │   table            │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
              │                     │                     │
              └─────────────────────┼─────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Get Report File Metadata                                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Create Download Link (SharePoint Sharing)                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Store Download URL                                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Send Report Notification Email (Office 365 Outlook)                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Compose Report Output (JSON with status, URL, statistics)               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Return Report Results (HTTP 200 Response)                                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Creating the Excel Template

For the Report Generator to work properly, you need an Excel template with predefined tables.

#### Step-by-Step Template Creation

1. **Open Microsoft Excel** (desktop or online)

2. **Create a new blank workbook**

3. **Sheet 1 - Applications**:
   - Rename the sheet tab to: `Applications`
   - In cell **A1**, type: `Application`
   - In cell **B1**, type: `Version`
   - In cell **C1**, type: `Provider`
   - In cell **D1**, type: `MachineName`
   - Select cells **A1:D1**
   - Go to **Insert** tab → Click **Table**
   - Check "My table has headers" → Click **OK**
   - Right-click the table → Select **Table** → **Rename Table**
   - Name the table: `UniqueApplications`

4. **Sheet 2 - SQL Servers**:
   - Click the **+** button to add a new sheet
   - Rename to: `SQL Servers`
   - Add headers: `MachineName`, `InstanceName`, `Edition`, `ServicePack`, `Version`, `Port`
   - Convert to table named: `UniqueSQLInstances`

5. **Sheet 3 - Web Apps**:
   - Add another new sheet
   - Rename to: `Web Apps`
   - Add headers: `WebAppName`, `WebServerType`, `VirtualDirectory`, `ApplicationPool`, `FrameworkVersion`, `MachineName`
   - Convert to table named: `UniqueWebApps`

6. **Save the template**:
   - Save as: `ReportTemplate.xlsx`
   - Upload to SharePoint: `/Templates/ReportTemplate.xlsx`

---

## Orchestrating the Agents

The Copilot agent orchestrates all processing through its topic flow. The agent's LLM coordinates the sequence: reading data, analyzing each inventory type, and generating the final report. A lightweight Power Automate Orchestrator flow is used only when needed — specifically, when you need to coordinate multiple file processing or provide centralized error handling beyond what the agent topics handle.

> **Note**: In the LLM-first approach, much of the orchestration happens within the Copilot agent's topics. The agent calls each processing topic in sequence, passing results between them via global variables. The Power Automate Orchestrator below is provided for scenarios where you need to process multiple files or require flow-level error handling.

> **⚠️ Important**: The data extraction flows created in Agents 2, 3, and 4 (`Read Application Inventory Data`, `Read SQL Server Inventory Data`, `Read Web App Inventory Data`) use the **"When an agent calls the flow"** trigger and are designed to be called directly by Copilot Studio topics as Tools. If you also need the Power Automate Orchestrator below to call these processors via HTTP, you must create **separate HTTP-triggered versions** of these flows (using the "When a HTTP request is received" trigger). Alternatively, you can rely entirely on the Copilot agent's topic-based orchestration and skip the Power Automate Orchestrator.

---

### Step 1: Create the Orchestrator Flow

#### Step 1.1: Access Power Automate

1. Open your web browser
2. Navigate to: **https://make.powerautomate.com**
3. Sign in with your Microsoft 365 credentials
4. Ensure you are in the correct environment

#### Step 1.2: Create New Flow

1. Click **My flows** in the left navigation
2. Click **+ New flow** → **Instant cloud flow**
3. Configure:
   - **Flow name**: `Azure Migrate Processing Orchestrator`
   - **Trigger**: Select **When a HTTP request is received**
4. Click **Create**

#### Step 1.3: Configure the HTTP Trigger Schema

1. Click on the trigger to expand it
2. Click **Use sample payload to generate schema**
3. Paste the following JSON:

> **EXAMPLE ONLY — replace values with your own.** These sample values are used solely to generate the trigger schema.

```json
{
  "fileUrls": [
    "/uploads/session-123/file1.xlsx",
    "/uploads/session-123/file2.csv"
  ],
  "sessionId": "session-123-guid",
  "userEmail": "user@contoso.com",
  "userId": "user-id-123",
  "storageType": "SharePoint"
}
```

4. Click **Done**

---

### Step 2: Initialize All Required Variables

#### Step 2.1: Initialize allApplications Array

1. Click **+** → **Add an action**
2. Search for and select **Initialize variable**
3. Configure:
   - **Name**: `allApplications`
   - **Type**: **Array**
   - **Value**: `[]`
4. Rename to: `Initialize allApplications`

#### Step 2.2: Initialize allSQLInstances Array

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `allSQLInstances`
   - **Type**: **Array**
   - **Value**: `[]`
3. Rename to: `Initialize allSQLInstances`

#### Step 2.3: Initialize allWebApps Array

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `allWebApps`
   - **Type**: **Array**
   - **Value**: `[]`
3. Rename to: `Initialize allWebApps`

#### Step 2.4: Initialize processingErrors Array

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `processingErrors`
   - **Type**: **Array**
   - **Value**: `[]`
3. Rename to: `Initialize processingErrors`

#### Step 2.5: Initialize filesProcessed Counter

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `filesProcessed`
   - **Type**: **Integer**
   - **Value**: `0`
3. Rename to: `Initialize filesProcessed`

#### Step 2.6: Initialize currentFileUrl Variable

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `currentFileUrl`
   - **Type**: **String**
   - **Value**: (leave empty)
3. Rename to: `Initialize currentFileUrl`

---

### Step 3: Process Each Uploaded File

#### Step 3.1: Add Apply to Each Loop

1. Click **+** → **Add an action**
2. Search for and select **Apply to each**
3. Configure:
   - **Select an output**: Click **Dynamic content** → Select `fileUrls` from trigger
4. Rename to: `Process Each Uploaded File`

#### Step 3.2: Inside Loop - Store Current File URL

1. Inside the loop, click **Add an action**
2. Select **Set variable**
3. Configure:
   - **Name**: `currentFileUrl`
   - **Value**: Click **Dynamic content** → Select **Current item** from the loop
4. Rename to: `Set Current File URL`

#### Step 3.3: Add Scope for Error Handling (Try Block)

1. Inside the loop, click **Add an action**
2. Search for and select **Scope**
3. Rename to: `Try - Process File`

---

### Step 4: Call Application Inventory Processor

Inside the "Try - Process File" scope:

#### Step 4.1: Add HTTP Action to Call App Processor

1. Inside the Scope, click **Add an action**
2. Search for and select **HTTP**
3. Configure:
   - **Method**: **POST**
   - **URI**: Paste the **HTTP POST URL** copied from your saved `Process Application Inventory` flow's trigger (see [Understanding HTTP Action URI Values](#understanding-http-action-uri-values))
   - **Headers**: Add:
     - **Key**: `Content-Type`
     - **Value**: `application/json`
   - **Body**:
   ```json
   {
     "filePath": "@{variables('currentFileUrl')}",
     "sessionId": "@{triggerBody()?['sessionId']}",
     "storageType": "@{triggerBody()?['storageType']}"
   }
   ```
4. Rename to: `Call Application Processor`

#### Step 4.2: Parse the Response

1. Click **Add an action**
2. Select **Parse JSON**
3. Configure:
   - **Content**: Click **Dynamic content** → Select **Body** from "Call Application Processor"
   - **Schema**: Click **Use sample payload to generate schema** and paste:
   ```json
   {
     "uniqueApplications": [],
     "statistics": {
       "totalProcessed": 0,
       "totalUnique": 0,
       "duplicatesRemoved": 0
     },
     "status": "Complete"
   }
   ```
4. Rename to: `Parse App Processor Response`

#### Step 4.3: Merge Application Results

1. Click **Add an action**
2. Select **Compose**
3. Configure:
   - **Inputs**: Click **Expression** → Type:
   ```
   union(variables('allApplications'), body('Parse_App_Processor_Response')?['uniqueApplications'])
   ```
4. Rename to: `Merge Application Results`

5. Click **Add an action**
6. Select **Set variable**
7. Configure:
   - **Name**: `allApplications`
   - **Value**: Click **Dynamic content** → Select **Outputs** from "Merge Application Results"
8. Rename to: `Update allApplications`

---

### Step 5: Call SQL Server Processor

#### Step 5.1: Add HTTP Action

1. Inside the same Scope, click **Add an action**
2. Select **HTTP**
3. Configure:
   - **Method**: **POST**
   - **URI**: Paste the **HTTP POST URL** copied from your saved `Process SQL Server Inventory` flow's trigger (see [Understanding HTTP Action URI Values](#understanding-http-action-uri-values))
   - **Headers**: `Content-Type`: `application/json`
   - **Body**:
   ```json
   {
     "filePath": "@{variables('currentFileUrl')}",
     "sessionId": "@{triggerBody()?['sessionId']}",
     "storageType": "@{triggerBody()?['storageType']}"
   }
   ```
4. Rename to: `Call SQL Processor`

#### Step 5.2: Parse Response

1. Click **Add an action** → **Parse JSON**
2. Configure:
   - **Content**: Select **Body** from "Call SQL Processor"
   - **Schema**: Use appropriate schema for SQL response
3. Rename to: `Parse SQL Processor Response`

#### Step 5.3: Merge SQL Results

1. Click **Add an action** → **Compose**
2. Configure:
   - **Inputs**: Expression:
   ```
   union(variables('allSQLInstances'), body('Parse_SQL_Processor_Response')?['uniqueSQLInstances'])
   ```
3. Rename to: `Merge SQL Results`

4. Click **Add an action** → **Set variable**
5. Configure:
   - **Name**: `allSQLInstances`
   - **Value**: **Outputs** from "Merge SQL Results"
6. Rename to: `Update allSQLInstances`

---

### Step 6: Call Web App Processor

#### Step 6.1: Add HTTP Action

1. Inside the same Scope, click **Add an action**
2. Select **HTTP**
3. Configure:
   - **Method**: **POST**
   - **URI**: Paste the **HTTP POST URL** copied from your saved `Read Web App Inventory Data` flow's trigger (see [Understanding HTTP Action URI Values](#understanding-http-action-uri-values))
   - **Headers**: `Content-Type`: `application/json`
   - **Body**:
   ```json
   {
     "filePath": "@{variables('currentFileUrl')}",
     "sessionId": "@{triggerBody()?['sessionId']}",
     "storageType": "@{triggerBody()?['storageType']}"
   }
   ```
4. Rename to: `Call Web App Processor`

#### Step 6.2: Parse Response

1. Click **Add an action** → **Parse JSON**
2. Configure appropriately
3. Rename to: `Parse Web App Processor Response`

#### Step 6.3: Merge Web App Results

1. Click **Add an action** → **Compose**
2. Configure:
   - **Inputs**: Expression:
   ```
   union(variables('allWebApps'), body('Parse_Web_App_Processor_Response')?['uniqueWebApps'])
   ```
3. Rename to: `Merge Web App Results`

4. Click **Add an action** → **Set variable**
5. Configure:
   - **Name**: `allWebApps`
   - **Value**: **Outputs** from "Merge Web App Results"
6. Rename to: `Update allWebApps`

---

### Step 7: Increment Files Processed Counter

Inside the Scope, after all processors:

1. Click **Add an action**
2. Select **Increment variable**
3. Configure:
   - **Name**: `filesProcessed`
   - **Value**: `1`
4. Rename to: `Increment Files Processed`

---

### Step 8: Add Error Handling (Catch Block)

#### Step 8.1: Add Scope for Catch

1. After the "Try - Process File" scope (but still inside the Apply to each loop)
2. Click **Add an action**
3. Select **Scope**
4. Rename to: `Catch - Handle Error`

#### Step 8.2: Configure Run After for Catch Scope

1. Click the **...** (three dots) on the "Catch - Handle Error" scope
2. Select **Configure run after**
3. Uncheck **is successful**
4. Check **has failed**
5. Check **has timed out**
6. Click **Done**

#### Step 8.3: Add Error Logging Inside Catch

1. Inside the Catch scope, click **Add an action**
2. Select **Append to array variable**
3. Configure:
   - **Name**: `processingErrors`
   - **Value**:
   ```json
   {
     "file": "@{variables('currentFileUrl')}",
     "error": "Processing failed",
     "timestamp": "@{utcNow()}"
   }
   ```
4. Rename to: `Log Processing Error`

---

### Step 9: Check if Data was Processed

After the Apply to each loop:

#### Step 9.1: Add Condition

1. Click **+** → **Add an action**
2. Select **Condition**
3. Configure the condition:
   - Click **Expression** tab and type:
   ```
   or(greater(length(variables('allApplications')), 0), or(greater(length(variables('allSQLInstances')), 0), greater(length(variables('allWebApps')), 0)))
   ```
   - Operator: **is equal to**
   - Value: `true`
4. Rename to: `Check if Data Processed`

---

### Step 10: Generate Report (If Data Exists)

In the **If yes** branch:

#### Step 10.1: Call Report Generator

1. Click **Add an action**
2. Select **HTTP**
3. Configure:
   - **Method**: **POST**
   - **URI**: Paste the **HTTP POST URL** copied from your saved `Generate Consolidated Report` flow's trigger (see [Understanding HTTP Action URI Values](#understanding-http-action-uri-values))
   - **Headers**: `Content-Type`: `application/json`
   - **Body**:
   ```json
   {
     "uniqueApplications": @{variables('allApplications')},
     "uniqueSQLInstances": @{variables('allSQLInstances')},
     "uniqueWebApps": @{variables('allWebApps')},
     "sessionId": "@{triggerBody()?['sessionId']}",
     "userEmail": "@{triggerBody()?['userEmail']}",
     "storageType": "@{triggerBody()?['storageType']}"
   }
   ```
4. Rename to: `Call Report Generator`

#### Step 10.2: Parse Report Response

1. Click **Add an action** → **Parse JSON**
2. Configure with appropriate schema
3. Rename to: `Parse Report Response`

#### Step 10.3: Return Success Response

1. Click **Add an action**
2. Select **Response**
3. Configure:
   - **Status Code**: `200`
   - **Headers**: `Content-Type`: `application/json`
   - **Body**:
   ```json
   {
     "status": "Complete",
     "downloadUrl": "@{body('Parse_Report_Response')?['downloadUrl']}",
     "reportFileName": "@{body('Parse_Report_Response')?['reportFileName']}",
     "statistics": {
       "filesProcessed": @{variables('filesProcessed')},
       "uniqueApplications": @{length(variables('allApplications'))},
       "uniqueSQLInstances": @{length(variables('allSQLInstances'))},
              "uniqueWebApps": @{length(variables('allWebApps'))},
       "totalUniqueItems": @{add(add(length(variables('allApplications')), length(variables('allSQLInstances'))), length(variables('allWebApps')))}
     },
     "errors": @{variables('processingErrors')},
     "completedAt": "@{utcNow()}"
   }
   ```
4. Rename to: `Return Success Response`

---

### Step 11: Handle No Data Case

In the **If no** branch:

1. Click **Add an action**
2. Select **Response**
3. Configure:
   - **Status Code**: `400`
   - **Headers**: `Content-Type`: `application/json`
   - **Body**:
   ```json
   {
     "status": "Error",
     "message": "No valid data found in uploaded files. Please verify the files contain ApplicationInventory, SQL Server, and/or Web Applications sheets.",
     "filesAttempted": @{length(triggerBody()?['fileUrls'])},
     "errors": @{variables('processingErrors')},
     "completedAt": "@{utcNow()}"
   }
   ```
4. Rename to: `Return No Data Error`

---

### Step 12: Save and Test

#### Step 12.1: Save the Flow

1. Click **Save** at the top-right
2. Wait for confirmation

#### Step 12.2: Copy the Flow URL

1. Click on the HTTP trigger
2. Copy the **HTTP POST URL**
3. This URL will be used by the File Upload Handler flow

#### Step 12.3: Test the Flow

1. Click **Test** → **Manually** → **Test**
2. Send a test request with sample file URLs
3. Verify:
   - All processors are called
   - Results are merged correctly
   - Report is generated
   - Success response is returned

---

### Complete Orchestrator Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When a HTTP request is received                                         │
│ (Inputs: fileUrls, sessionId, userEmail, userId, storageType)          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Initialize Variables                                                     │
│ • allApplications (Array: [])                                           │
│ • allSQLInstances (Array: [])                                           │
│ • allWebApps (Array: [])                                                │
│ • processingErrors (Array: [])                                          │
│ • filesProcessed (Integer: 0)                                           │
│ • currentFileUrl (String)                                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ FOR EACH file in fileUrls:                                              │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ Set currentFileUrl                                                  │ │
│ ├─────────────────────────────────────────────────────────────────────┤ │
│ │ TRY SCOPE:                                                          │ │
│ │   1. Call Application Processor (HTTP)                              │ │
│ │   2. Parse App Response                                             │ │
│ │   3. Merge & Update allApplications                                 │ │
│ │   4. Call SQL Processor (HTTP)                                      │ │
│ │   5. Parse SQL Response                                             │ │
│ │   6. Merge & Update allSQLInstances                                 │ │
│ │   7. Call Web App Processor (HTTP)                                   │ │
│ │   8. Parse Web App Response                                          │ │
│ │   9. Merge & Update allWebApps                                       │ │
│ │  10. Increment filesProcessed                                       │ │
│ ├─────────────────────────────────────────────────────────────────────┤ │
│ │ CATCH SCOPE (runs on failure/timeout):                              │ │
│ │   - Append error to processingErrors                                │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ CONDITION: Any data processed?                                          │
│ (allApplications.length > 0 OR allSQLInstances.length > 0              │
│  OR allWebApps.length > 0)                                             │
├──────────────────────────────────┬──────────────────────────────────────┤
│           IF YES                 │              IF NO                    │
├──────────────────────────────────┼──────────────────────────────────────┤
│ 1. Call Report Generator (HTTP)  │ Return 400 Error Response            │
│ 2. Parse Report Response         │ "No valid data found"                │
│ 3. Return 200 Success Response   │                                      │
│    with downloadUrl & statistics │                                      │
└──────────────────────────────────┴──────────────────────────────────────┘
```

---

## Power Automate Flows

### Summary of Required Flows

| Flow Name | Type | Purpose | Storage Option |
|-----------|------|---------|----------------|
| `Azure Migrate Processing Orchestrator` | Parent/Main | Coordinates all processing (optional with LLM-first approach) | Both |
| `Handle File Upload (Blob Storage)` | Agent Tool | Saves files to Azure Blob Storage and returns file path | Option A |
| `Handle File Upload (SharePoint)` | Agent Tool | Saves files to SharePoint and returns file path | Option B |
| `Read Application Inventory Data` | Agent Tool | Reads raw application data for LLM analysis | Both |
| `Read SQL Server Inventory Data` | Agent Tool | Reads raw SQL Server data for LLM analysis | Both |
| `Read Web App Inventory Data` | Agent Tool | Reads raw web app data for LLM analysis | Both |
| `Generate Consolidated Report` | Child | Creates Excel and download link | Both |
| `Get Processing Status` | Utility | Checks processing status | Both |

### Understanding HTTP Action URI Values

Several flows in this solution use the built-in **HTTP** action to call other flows. Each HTTP action requires a **URI** value — this is the **HTTP POST URL** that Power Automate auto-generates when you save a flow that uses the **"When a HTTP request is received"** trigger.

#### How to obtain the URI

1. Open the **target** flow (the flow you want to call) in the Power Automate designer
2. **Save** the flow — the URL is only generated after saving
3. Click on the **When a HTTP request is received** trigger to expand it
4. Copy the **HTTP POST URL** displayed in the trigger card
5. Paste this URL into the **URI** field of the calling flow's HTTP action

#### URL format

The auto-generated URL follows this pattern:

> **PLACEHOLDER — the URL below is a structural example only. Your actual URL is generated by Power Automate and will have different values.**

```
https://<region>.logic.azure.com:443/workflows/<workflow-id>/triggers/manual/paths/invoke?api-version=<api-version>&sp=<permissions>&sv=<sas-version>&sig=<signature>
```

| Segment | Description |
|---------|-------------|
| `<region>` | Azure region where the flow is hosted (e.g., `prod-00.westus`, `prod-28.eastus`) |
| `<workflow-id>` | Unique identifier for the flow (auto-assigned) |
| `<api-version>` | Logic Apps API version (e.g., `2016-10-01`) |
| `<permissions>` (`sp`) | Permissions granted (e.g., `%2Ftriggers%2Fmanual%2Frun` for trigger run permission) |
| `<sas-version>` (`sv`) | SAS protocol version (e.g., `1.0`) |
| `<signature>` (`sig`) | Shared Access Signature — cryptographic hash that authenticates the request |

> **Security Warning:** The `sig` query parameter acts as an authentication key. Treat your actual flow URL as a secret — do not share it publicly, commit real URLs to source control, or log them in full. The placeholder URIs in this guide (e.g., "Paste the HTTP POST URL from your saved flow") are safe because they do not contain real signatures. For production deployments, consider using [Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts) to front the endpoint, or restrict access via [IP address configuration](https://learn.microsoft.com/en-us/power-automate/ip-address-configuration). For more information, see [Secure access and data in Azure Logic Apps](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-securing-a-logic-app#secure-inbound-requests).

#### Flow creation order (dependency chain)

Because each HTTP action's URI comes from the target flow's trigger, you must **create and save the target flows first** before configuring the calling flows. The recommended creation order is:

1. **Agent tool flows** (create and save these first — each is registered as a Copilot Studio tool):
   - `Read Application Inventory Data`
   - `Read SQL Server Inventory Data`
   - `Read Web App Inventory Data`
   - `Generate Consolidated Report`
2. **Orchestrator flow** (optional — `Azure Migrate Processing Orchestrator`) — only needed for multi-file batch processing; paste the tool flow URLs into its HTTP actions, then save to generate its own URL
3. **File Upload Handler flow** (`Handle File Upload`) — in the LLM-first approach, this flow does NOT call the Orchestrator. It stores the file and returns the file path to the agent, which coordinates processing via topic redirects

> **Note**: In the LLM-first approach, the agent tool flows use the "When an agent calls the flow" trigger and are registered directly as Tools in Copilot Studio. The Orchestrator flow is optional and only needed for multi-file batch processing or flow-level error handling.

#### URI value summary per HTTP action

> **Note**: The `Read *` flows created in Agents 2, 3, and 4 use the "When an agent calls the flow" trigger (for Copilot Studio Tool integration). The `Handle File Upload` flows also use this trigger and return the file path to the agent — they do **not** call the Orchestrator in the LLM-first approach. If you use the optional Power Automate Orchestrator for multi-file batch processing, you need separate HTTP-triggered versions of the data extraction flows. If you use only the Copilot agent's topic-based orchestration (recommended), you do not need these HTTP action URIs for the File Upload or data extraction flows.

| Calling Flow | HTTP Action Name | URI Value (paste from) |
|-------------|-----------------|----------------------|
| Azure Migrate Processing Orchestrator (optional) | Call Application Processor | HTTP POST URL from HTTP-triggered version of `Read Application Inventory Data` |
| Azure Migrate Processing Orchestrator (optional) | Call SQL Processor | HTTP POST URL from HTTP-triggered version of `Read SQL Server Inventory Data` |
| Azure Migrate Processing Orchestrator (optional) | Call Web App Processor | HTTP POST URL from HTTP-triggered version of `Read Web App Inventory Data` |
| Azure Migrate Processing Orchestrator (optional) | Call Report Generator | HTTP POST URL from `Generate Consolidated Report` |

> **Note**: In the recommended LLM-first approach, the `Handle File Upload` flow stores the file and returns the path to the Copilot agent. The agent then uses topic redirects to call each processing topic (Agent 2, 3, 4) in sequence. No HTTP action URIs are needed for the File Upload flow.

---

### Flow 1A: File Upload Handler - Azure Blob Storage (Recommended)

This flow stores uploaded files in Azure Blob Storage, which does NOT require users to have SharePoint or OneDrive access. This is the recommended approach for temporary file storage.

```
Name: Handle File Upload (Blob Storage)
Trigger: When an agent calls the flow (Run a flow from Copilot)
Inputs: uploadedFiles (File), userId (Text), userEmail (Text)

Steps:
1. Generate unique session ID
2. Save uploaded file to Azure Blob Storage container
3. Build the file path for the stored blob
4. Return session ID, file path, status, and message to the agent
```

**Flow Definition - Azure Blob Storage:**

```
Trigger: When an agent calls the flow
  Inputs:
    - uploadedFiles (File)
    - userId (Text)
    - userEmail (Text)

Actions:
1. Compose
   Rename to: Generate Session ID
   Expression: guid()

2. Create blob (V2) - Azure Blob Storage
   (Do not rename — keep default connector name)
   Connection: Your Azure Blob Storage connection
   Storage account name: <your-storage-account-name>
   Container name: uploads
   Blob name: @{outputs('Generate_Session_ID')}/@{triggerBody()?['uploadedFiles']?['name']}
   Blob content: @{triggerBody()?['uploadedFiles']?['contentBytes']}

3. Compose
   Rename to: Build File Path
   Expression: concat('uploads/', outputs('Generate_Session_ID'), '/', triggerBody()?['uploadedFiles']?['name'])

4. Respond to the agent:
   - sessionId: @{coalesce(outputs('Generate_Session_ID'), '')}
   - filePath: @{coalesce(outputs('Build_File_Path'), '')}
   - status: "Processing"
   - message: "File uploaded successfully to temporary storage. Processing has started."
```

> **Note:** In the LLM-first architecture, this flow does NOT call the Orchestrator. It stores the file and returns the `filePath` to the Copilot agent, which stores it in `Global.uploadedFilePath` and then coordinates processing by redirecting to each analysis topic (Agent 2, 3, 4) in sequence.
>
> **Important:** Every output in the **Respond to the agent** action must have a value. Wrap dynamic expressions in `coalesce()` (e.g., `coalesce(outputs('Generate_Session_ID'), '')`) so the flow returns an empty string instead of null when an action produces no result.
>
> **Note:** The trigger input names (`uploadedFiles`, `userId`, `userEmail`) define the keys in `triggerBody()`. Access them as `triggerBody()?['uploadedFiles']`, `triggerBody()?['userId']`, and `triggerBody()?['userEmail']` — not via a nested `user` object.

**Azure Blob Storage Connection Setup:**
1. In Power Automate, add a new connection for **Azure Blob Storage**
2. Choose authentication method (listed from most to least secure):
   - **Azure AD / Managed Identity (Recommended)**: Use Azure Active Directory for managed identity — no secrets to rotate
   - **SAS Token**: Use a shared access signature with minimal permissions and short expiry for limited access
   - **Access Key**: Use the storage account access key (least secure — grants full account access; avoid if possible)
3. Test the connection before saving

**Benefits of Azure Blob Storage for Temporary Files:**
- ✅ Users do NOT need SharePoint/OneDrive licenses or access
- ✅ Automatic cleanup via lifecycle management policies
- ✅ Cost-effective (pay only for storage used)
- ✅ Better performance for large file uploads
- ✅ Programmatic SAS token generation for secure, temporary access
- ✅ No impact on SharePoint storage quotas

### Flow 1B: File Upload Handler - SharePoint (Alternative)

This flow stores uploaded files in SharePoint. Use this only if users have SharePoint access and you prefer SharePoint storage.

```
Name: Handle File Upload (SharePoint)
Trigger: When an agent calls the flow (Run a flow from Copilot)
Inputs: uploadedFiles (File), userId (Text), userEmail (Text)

Steps:
1. Generate unique session ID
2. Create session folder in SharePoint
3. Save uploaded file to SharePoint
4. Build the file path for the stored file
5. Return session ID, file path, status, and message to the agent
```

**Flow Definition - SharePoint:**

```
Trigger: When an agent calls the flow
  Inputs:
    - uploadedFiles (File)
    - userId (Text)
    - userEmail (Text)

Actions:
1. Compose
   Rename to: Generate Session ID
   Expression: guid()

2. Create new folder (SharePoint connector)
   (Do not rename — keep default connector name)
   Site: Your SharePoint Site
   Folder Path: /Uploads/@{outputs('Generate_Session_ID')}

3. Create file (SharePoint connector)
   (Do not rename — keep default connector name)
   Site: Your Site
   Folder: /Uploads/@{outputs('Generate_Session_ID')}
   File Name: @{triggerBody()?['uploadedFiles']?['name']}
   Content: @{triggerBody()?['uploadedFiles']?['contentBytes']}

4. Compose
   Rename to: Build File Path
   Expression: concat('/Uploads/', outputs('Generate_Session_ID'), '/', triggerBody()?['uploadedFiles']?['name'])

5. Respond to the agent:
   - sessionId: @{coalesce(outputs('Generate_Session_ID'), '')}
   - filePath: @{coalesce(outputs('Build_File_Path'), '')}
   - status: "Processing"
   - message: "File uploaded successfully. Processing has started."
```

> **Note:** In the LLM-first architecture, this flow does NOT call the Orchestrator. It stores the file and returns the `filePath` to the Copilot agent, which stores it in `Global.uploadedFilePath` and then coordinates processing by redirecting to each analysis topic (Agent 2, 3, 4) in sequence.
>
> **Note:** Access trigger inputs using the names defined on the trigger: `triggerBody()?['uploadedFiles']`, `triggerBody()?['userEmail']`, `triggerBody()?['userId']`. Every output in the **Respond to the agent** action must have a value assigned — wrap dynamic expressions in `coalesce()` to return an empty string `''` when no value is available.

### Flow 2: Get Processing Status

This flow checks for completed reports and returns the download URL. Configure based on your storage choice:

#### Option A: Get Processing Status - Azure Blob Storage

```
Name: Get Processing Status (Blob Storage)
Trigger: When an agent calls the flow (Run a flow from Copilot)
Inputs: sessionId (Text)
Outputs: processingStatus (Text), downloadUrl (Text), errorMessage (Text)

Steps:
1. Receive sessionId from the agent
2. List blobs in reports container for the session
3. Generate SAS URL for download if report exists
4. Return status, download URL, and error message to the agent
```

**Flow Definition - Azure Blob Storage:**

```
Trigger: When an agent calls the flow
  Inputs:
    - sessionId (Text)

Actions:
1. List blobs (V2) - Azure Blob Storage
   (Do not rename — keep default connector name)
   Storage account: <your-storage-account-name>
   Container: reports
   Prefix: @{triggerBody()?['sessionId']}/

2. Filter array
   (Keep default name — no rename needed)
   From: @{body('List_blobs_(V2)')?['value']}
   Condition: endsWith(item()?['Name'], '.xlsx')

3. Condition: Report exists?
   If: length(body('Filter_array')) > 0
   
   Yes branch:
     4. Compose
        Rename to: Get first blob name
        @{first(body('Filter_array'))?['Name']}
     
     5. Create SAS URI by path (V2) - Azure Blob Storage
        (Do not rename — keep default connector name)
        Storage account: <your-storage-account-name>
        Container: reports
        Blob path: @{outputs('Get_first_blob_name')}
        Permissions: Read
        Expiry time: @{addHours(utcNow(), 24)}
     
     6. Respond to the agent:
        - processingStatus: "Complete"
        - downloadUrl: @{coalesce(body('Create_SAS_URI_by_path_(V2)')?['WebUrl'], '')}
        - errorMessage: ""
   
   No branch:
     7. Respond to the agent:
        - processingStatus: "Processing"
        - downloadUrl: ""
        - errorMessage: ""
```

> **Important:** Both branches of the condition must include a **Respond to the agent** action with **all** output parameters populated. Use empty strings `""` for outputs that have no value in a given branch, and wrap dynamic expressions in `coalesce()` (e.g., `coalesce(body('Create_SAS_URI_by_path_(V2)')?['WebUrl'], '')`) to guarantee a value even if the preceding action returns null. Missing output values cause a `FlowActionException` error.
>
> **Tip:** The **Create SAS URI by path (V2)** action is a built-in Azure Blob Storage connector action in Power Automate that generates a SAS URL without custom code. If this action is not available in your environment, use one of the alternatives below.

**Generating SAS URLs for Download:**

For Azure Blob Storage, you have several options to generate secure download URLs:

1. **Azure Function (Recommended)**:
   ```csharp
   // Azure Function to generate SAS URL with input validation
   [FunctionName("GenerateSasUrl")]
   public static async Task<IActionResult> Run(
       [HttpTrigger] HttpRequest req)
   {
       string sessionId = req.Query["sessionId"];
       string fileName = req.Query["fileName"];
       
       // Validate inputs - prevent path traversal attacks
       if (string.IsNullOrEmpty(sessionId) || string.IsNullOrEmpty(fileName))
           return new BadRequestObjectResult("sessionId and fileName are required");
       
       // Validate sessionId is a valid GUID format
       if (!Guid.TryParse(sessionId, out _))
           return new BadRequestObjectResult("Invalid sessionId format");
       
       // Validate fileName doesn't contain path traversal sequences
       if (fileName.Contains("..") || fileName.Contains("/") || fileName.Contains("\\"))
           return new BadRequestObjectResult("Invalid fileName");
       
       // Construct validated blob path
       string blobPath = $"{sessionId}/{fileName}";
       
       var sasBuilder = new BlobSasBuilder
       {
           BlobContainerName = "reports",
           BlobName = blobPath,
           Resource = "b",
           ExpiresOn = DateTimeOffset.UtcNow.AddHours(24)
       };
       // Read-only permission for download
       sasBuilder.SetPermissions(BlobSasPermissions.Read);
       // Generate and return SAS URL
   }
   ```

2. **Pre-configured SAS Token**: Use a service SAS with read-only permissions
3. **Logic App with Managed Identity**: Use Azure Logic Apps with managed identity for blob access

#### Option B: Get Processing Status - SharePoint

```
Name: Get Processing Status (SharePoint)
Trigger: When an agent calls the flow (Run a flow from Copilot)
Inputs: sessionId (Text)
Outputs: processingStatus (Text), downloadUrl (Text), errorMessage (Text)

Steps:
1. Receive sessionId from the agent
2. Check for completed report in SharePoint
3. Return status, download URL, and error message to the agent
```

**Flow Definition - SharePoint:**

```
Trigger: When an agent calls the flow
  Inputs:
    - sessionId (Text)

Actions:
1. Get files (properties only) - SharePoint
   Rename to: Get files (default name "Get_files_(properties_only)" is too long for expressions)
   Site: Your Site
   Library: Reports
   Folder: /@{triggerBody()?['sessionId']}
   Filter: endswith(Name, '.xlsx')

2. Condition: Files exist?
   If: length(body('Get_files')?['value']) > 0
   
   Yes branch:
     3. Create sharing link for a file or folder (SharePoint)
        Rename to: Create sharing link (default name is too long for expressions)
        Site: Your Site
        Item Id: @{first(body('Get_files')?['value'])?['Id']}
        Link type: View
     
     4. Respond to the agent:
        - processingStatus: "Complete"
        - downloadUrl: @{coalesce(body('Create_sharing_link')?['link']?['webUrl'], '')}
        - errorMessage: ""
   
   No branch:
     5. Respond to the agent:
        - processingStatus: "Processing"
        - downloadUrl: ""
        - errorMessage: ""
```

> **Important:** Both branches must include a **Respond to the agent** action with all output parameters populated. Wrap dynamic values in `coalesce()` to return a safe default when no value is available. The output parameter names (`processingStatus`, `downloadUrl`, `errorMessage`) must match those defined in Step 5.4.

### Connecting Flows to Copilot Studio

> **Note:** In the current version of Copilot Studio (2025), the former **"Actions"** page has been replaced by the **"Tools"** page. Power Automate flows are now added as **tools**. For official reference, see [Add an agent flow to an agent as a tool](https://learn.microsoft.com/en-us/microsoft-copilot-studio/flow-agent).

#### Option A: Add from the Tools Page (Agent-Level Tool)

1. In Copilot Studio, select **Agents**, then select your agent
2. Go to the **Tools** page and click **Add a tool**
3. In the **Add tool** panel, select **Flow** to list available agent flows
4. Select the flow (e.g., **Handle File Upload - Azure Migrate**) and click **Add and configure**
5. On the tool's **Details** page, configure **Inputs** using Power Fx formulas (see input mapping below)
6. Click **Save**

#### Option B: Add from within a Topic (Topic-Level Tool)

1. Open the topic where you want to call the flow
2. Click the **+** (Add node) icon below any node, and select **Add a tool**
3. Select the flow from the list — a new **Action** node appears in the topic
4. Map inputs and outputs directly on the Action node

> **Terminology note:** Although Power Automate flows are now added via the **Tools** page (replacing the former "Actions" page), the node that appears on the topic canvas when you add a flow is still labeled **Action** in the current Copilot Studio UI. If your UI shows a different label, use the label you see.

#### Input/Output Mapping

**For File Upload Flow (when added as a tool from the Tools page):**

> **Important:** On the Tools page, file inputs must be set using the **Custom value** (Power Fx formula) option — the **Dynamically fill with AI** option does not work for file inputs.

```
Input Mapping:
  - contentBytes → First(System.Activity.Attachments).Content
  - name         → First(System.Activity.Attachments).Name
  - userId       → System.User.Id
  - userEmail    → System.User.Email

Output Mapping:
  - sessionId   → Global.sessionId
  - status      → Global.uploadStatus
  - message     → (display in a Message node)
```

**For File Upload Flow (when added as a tool via Add a tool within a topic — appears as an Action node on the canvas):**
```
Input Mapping:
  - uploadedFiles → { contentBytes: Topic.uploadedFiles.Content, name: Topic.uploadedFiles.Name }
  - userId        → System.User.Id
  - userEmail     → System.User.Email

Output Mapping:
  - sessionId   → Global.sessionId
  - status      → Global.uploadStatus
  - message     → (display in a Message node)
```

**For Status Check Flow:**
```
Input Mapping:
  - sessionId → Global.sessionId

Output Mapping:
  - processingStatus → Global.processingStatus
  - downloadUrl      → Global.downloadUrl
  - errorMessage     → Global.errorMessage
```

> **Important:** Every output parameter in the **Respond to the agent** action must have a value assigned. Leaving any output value blank causes a `FlowActionException` error with the message "output parameter missing from response data." Always assign a value — use `coalesce()` to wrap dynamic expressions (e.g., `coalesce(outputs('MyAction'), '')`) so a safe default is returned when an action produces no result. Use an empty string `""` for text outputs that may not have data yet.

---

## Testing and Validation

### Test Plan

#### Phase 1: Unit Testing (Individual Agent Topics and Tools)

1. **Test Application Inventory Processor**
   - [ ] Upload CSV with 100 applications
   - [ ] Verify LLM correctly identifies noise (updates, patches, drivers)
   - [ ] Verify exact name match consolidation
   - [ ] Verify version variants are preserved as separate entries
   - [ ] Check output JSON format

2. **Test SQL Server Processor**
   - [ ] Upload CSV with SQL instances
   - [ ] Verify LLM consolidates by version grouping
   - [ ] Verify updates and dependent clients are removed
   - [ ] Check default instance handling (MSSQLSERVER, port 1433)

3. **Test Web App Processor**
   - [ ] Upload CSV with various web application types
   - [ ] Verify LLM-based noise removal (default sites, system pages)
   - [ ] Check web server type grouping
   - [ ] Verify duplicate handling across machines

4. **Test Report Generator**
   - [ ] Provide sample consolidated data
   - [ ] Verify Excel creation
   - [ ] Check sheet formatting
   - [ ] Validate download link

#### Phase 2: Integration Testing

1. **Test Orchestrator Flow**
   - [ ] Submit multiple files
   - [ ] Verify all processors are called
   - [ ] Check data merging
   - [ ] Validate final report

2. **Test Copilot Agent Integration**
   - [ ] Upload files through chat
   - [ ] Check processing initiation
   - [ ] Verify status checking
   - [ ] Download generated report

#### Phase 3: End-to-End Testing

1. **Complete User Journey Test**
   - [ ] Start conversation with agent
   - [ ] Upload Azure Migrate CSV files
   - [ ] Wait for processing
   - [ ] Check status
   - [ ] Download report
   - [ ] Verify report contents

### Sample Test Data

Create test CSV files with the following structure:

**test_application_inventory.csv:**
```csv
MachineName,Application,Version,Provider,MachineManagerFqdn
Server01,Microsoft SQL Server 2019,15.0.2000.5,Microsoft,manager.contoso.com
Server02,Microsoft SQL Server 2019,15.0.2000.5,Microsoft,manager.contoso.com
Server01,Oracle Database,19c,Oracle Corporation,manager.contoso.com
Server03,MySQL Server,8.0.28,Oracle Corporation,manager.contoso.com
Server01,KB5001234 Security Update,1.0,Microsoft,manager.contoso.com
Server02,Visual C++ 2019 Redistributable,14.0,Microsoft,manager.contoso.com
Server04,SAP BusinessObjects,4.3,SAP,manager.contoso.com
Server05,SAP BusinessObjects,4.3,SAP,manager.contoso.com
```

**test_sql_inventory.csv:**
```csv
MachineName,Instance Name,Edition,Service Pack,Version,Port,MachineManagerFqdn
SQLServer01,MSSQLSERVER,Enterprise,SP2,15.0.2000.5,1433,manager.contoso.com
SQLServer01,REPORTDB,Standard,SP1,14.0.3294.2,1434,manager.contoso.com
SQLServer02,MSSQLSERVER,Standard,SP2,15.0.2000.5,1433,manager.contoso.com
SQLServer01,MSSQLSERVER,Enterprise,SP2,15.0.2000.5,1433,manager.contoso.com
```

**test_webapp_inventory.csv:**
```csv
MachineName,WebServerType,WebAppName,VirtualDirectory,ApplicationPool,FrameworkVersion,MachineManagerFqdn
WebServer01,IIS,ContosoPortal,/portal,ContosoAppPool,.NET 4.8,manager.contoso.com
WebServer02,IIS,ContosoPortal,/portal,ContosoAppPool,.NET 4.8,manager.contoso.com
WebServer01,IIS,Default Web Site,/,DefaultAppPool,.NET 4.8,manager.contoso.com
WebServer03,Apache,HRApplication,/hr,,PHP 7.4,manager.contoso.com
WebServer04,Tomcat,InventoryAPI,/api,,Java 11,manager.contoso.com
WebServer01,IIS,iisstart,/,,,,manager.contoso.com
```

### Expected Test Results

After processing the test files above:

**Unique Applications (4 expected - removed updates, redistributables, and duplicates):**
- Microsoft SQL Server 2019
- Oracle Database
- MySQL Server
- SAP BusinessObjects

**Unique SQL Instances (3 expected):**
- SQLServer01/MSSQLSERVER
- SQLServer01/REPORTDB
- SQLServer02/MSSQLSERVER

**Unique Web Apps (3 expected - removed default sites, system pages, and duplicates):**
- ContosoPortal (IIS)
- HRApplication (Apache)
- InventoryAPI (Tomcat)

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: File Upload Fails

**Symptoms:**
- "Unable to upload file" error
- Timeout during upload

**Solutions for Azure Blob Storage (Option A):**
1. Check file size (Azure Blob Storage supports up to 4.75 TB per blob)
2. Verify Azure Blob Storage connection is configured correctly
3. Check storage account firewall settings allow Power Automate access
4. Ensure the container exists and has correct access permissions
5. Verify SAS token hasn't expired (if using SAS authentication)

**Solutions for SharePoint (Option B):**
1. Check file size (SharePoint limit: 250MB)
2. Verify SharePoint permissions
3. Check Power Automate connection status
4. Ensure file format is CSV or XLSX

#### Issue 2: Sheet Not Found

**Symptoms:**
- "Table not found" error
- "Sheet 'ApplicationInventory' not found"

**Solutions:**
1. Verify exact sheet name in uploaded file
2. Check for trailing spaces in sheet names
3. Ensure data is formatted as a table in Excel
4. Use dynamic sheet detection:
   ```
   Action: Get tables (Excel Online)
   Then filter for matching table names
   ```

#### Issue 3: Processing Timeout

**Symptoms:**
- Flow times out during processing
- Partial results

**Solutions:**
1. Increase flow timeout settings
2. Break large files into smaller batches
3. Use chunked processing:
   ```
   - Process 1000 rows at a time
   - Use pagination in Excel connector
   ```

#### Issue 4: Duplicate Detection Not Working

**Symptoms:**
- Duplicates appearing in output
- More entries than expected

**Solutions:**
1. Check case normalization:
   ```
   toLower(trim(value))
   ```
2. Verify key generation logic
3. Check for hidden characters in data:
   ```
   replace(replace(value, char(10), ''), char(13), '')
   ```

#### Issue 5: Download Link Not Working

**Symptoms:**
- 404 error on download
- Access denied

**Solutions for Azure Blob Storage (Option A):**
1. Verify SAS token is valid and not expired
2. Check SAS token has read permissions
3. Ensure the blob path is correct
4. Verify storage account firewall allows access from user's network
5. Check if the blob was deleted by lifecycle management policy

**Solutions for SharePoint (Option B):**
1. Verify sharing link settings
2. Check link expiration
3. Ensure user has SharePoint access
4. Use organization-wide sharing if appropriate

#### Issue 6: Azure Blob Storage Connection Fails

**Symptoms:**
- "Connection failed" error in Power Automate
- "AuthorizationFailure" error
- "ContainerNotFound" error

**Solutions:**
1. Verify storage account name is correct
2. Check access key or SAS token is valid
3. Ensure container name exists (containers are case-sensitive)
4. Verify storage account firewall settings:
   ```
   Allow Azure services on the trusted services list to access this storage account: Enabled
   ```
5. Check if Managed Identity has appropriate RBAC roles:
   - Storage Blob Data Contributor (for read/write)
   - Storage Blob Data Reader (for read-only)

#### Issue 7: SAS Token Issues

**Symptoms:**
- "AuthenticationFailed" error
- Download links expire too quickly
- "Signature did not match" error

**Solutions:**
1. Generate a new SAS token with correct permissions:
   - For **uploads**: Use Read, Write, Create permissions
   - For **download links**: Use Read permission only (principle of least privilege)
2. Ensure the SAS token hasn't expired
3. Check the SAS token is for the correct container/blob
4. Verify the SAS token start time is in the past (account for clock skew)
5. For download links, generate SAS tokens with appropriate expiry (e.g., 24 hours)

#### Issue 8: Flow Output Error — "output parameter missing from response data"

**Symptoms:**
- `FlowActionException` error in Copilot Studio when the agent calls a Power Automate flow
- Error message: "output parameter missing from response data"
- The flow runs successfully in Power Automate but fails when called from the agent

**Causes:**
1. One or more output parameters in the **Respond to the agent** action have no value (null) at runtime
2. A conditional branch is missing a **Respond to the agent** action, so the flow ends without responding
3. A dynamic expression (e.g., `body('Create_SAS_URI_by_path_(V2)')?['WebUrl']`) returns null because the preceding action produced no output

**Solutions:**
1. **Wrap every dynamic output value in `coalesce()`** to guarantee a non-null default:
   ```
   coalesce(body('Create_SAS_URI_by_path_(V2)')?['WebUrl'], '')
   coalesce(outputs('Generate_Session_ID'), '')
   ```
2. **Ensure every conditional branch includes a Respond to the agent action.** If the flow has a Condition with Yes/No branches, both branches must end with a **Respond to the agent** action that populates **all** defined output parameters.
3. **Use empty strings `""` for text outputs** that have no meaningful data in a given branch (e.g., `downloadUrl: ""` in the "No" branch when no report exists yet).
4. **Test the flow directly in Power Automate** with edge-case inputs (empty file name, missing session ID) to verify every path returns all outputs.
5. **Check that output parameter names match exactly** between the flow's **Respond to the agent** action and the tool configuration in Copilot Studio — mismatched names cause the same error.

#### Issue 9: Invalid Reference Error — "invalid reference to 'Generate_Session_ID'"

**Symptoms:**
- Design-time error in Power Automate when saving or validating the flow
- Error message: "The input parameter(s) of action 'Respond_to_the_agent' contain an invalid reference to 'Generate_Session_ID'. Correct to include a valid reference to 'Generate_Session_ID' for the input parameter(s) of action 'Respond_to_the_agent'."

**Causes:**
1. The Compose action was not renamed from its default name ("Compose") to "Generate Session ID"
2. The expression `outputs('Generate_Session_ID')` in the **Respond to the agent** action references an action by internal name, but no action with that internal name exists in the flow

**How Action Names Work in Power Automate:**
In Power Automate, every action has an **internal name** (used in expressions) that is derived from its **display name** by replacing spaces with underscores. For example:
- Display name "Generate Session ID" → Internal name `Generate_Session_ID`
- Display name "Compose" → Internal name `Compose`
- Display name "Create blob (V2)" → Internal name `Create_blob_(V2)`

When you use `outputs('Generate_Session_ID')`, Power Automate looks for an action whose internal name is exactly `Generate_Session_ID`. If the Compose action was never renamed, its internal name is still `Compose`, and the expression fails.

**Solutions:**
1. **Rename the Compose action** to "Generate Session ID":
   - Open the flow in Power Automate
   - Click on the Compose action that contains the `guid()` expression
   - Click on the action title text ("Compose") at the top of the action card
   - Type: `Generate Session ID`
   - Press Enter or click outside the title to confirm
   - Save the flow — the expression `outputs('Generate_Session_ID')` will now resolve correctly
2. **Alternatively**, if you prefer to keep the default name "Compose", change the expression in the **Respond to the agent** action from `outputs('Generate_Session_ID')` to `outputs('Compose')`. However, this is not recommended if you have multiple Compose actions in the flow, as names would conflict.

### Error Logging

Add error logging to your flows:

> **Security Warning:** The `InputData` field below logs the full trigger body, which may contain file content or personally identifiable information (PII). In production, consider logging only the session ID and file name instead of the entire input payload to avoid exposing sensitive data.

```
Scope: Error Handling
  - Create item (SharePoint List: ErrorLog)
    - FlowName: @{workflow().name}
    - ErrorMessage: @{actions('FailedAction')?['error']?['message']}
    - Timestamp: @{utcNow()}
    - SessionId: @{variables('sessionId')}
    - InputData: @{triggerBody()?['uploadedFiles']?['name']}
```

### Performance Optimization

1. **Batch Processing**: Process rows in batches of 100-500
2. **Parallel Processing**: Use parallel branches for independent sheets
3. **Caching**: Cache frequently used lookup data
4. **Indexing**: Use efficient data structures for duplicate checking

---

## Resources

### Microsoft Documentation

- [Microsoft Copilot Studio Documentation](https://learn.microsoft.com/en-us/microsoft-copilot-studio/)
- [Power Automate Documentation](https://learn.microsoft.com/en-us/power-automate/)
- [Azure Blob Storage Connector](https://learn.microsoft.com/en-us/connectors/azureblob/)
- [Azure Blob Storage Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [SharePoint Connectors](https://learn.microsoft.com/en-us/connectors/sharepointonline/)
- [Excel Online Connector](https://learn.microsoft.com/en-us/connectors/excelonlinebusiness/)

### Azure Blob Storage Resources

- [Create a Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)
- [Manage Blob Lifecycle](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-configure)
- [Create SAS Tokens](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Azure Blob Storage Security Best Practices](https://learn.microsoft.com/en-us/azure/storage/blobs/security-recommendations)

### Related Guides

- [Azure Migrate Documentation](https://learn.microsoft.com/en-us/azure/migrate/)
- [Exporting Azure Migrate Data](https://learn.microsoft.com/en-us/azure/migrate/how-to-export-data)

### Community Resources

- [Power Automate Community](https://powerusers.microsoft.com/t5/Power-Automate-Community/ct-p/MPACommunity)
- [Copilot Studio Community](https://powerusers.microsoft.com/t5/Microsoft-Copilot-Studio/ct-p/PVACommunity)

### Video Tutorials

- [Building Agents in Copilot Studio](https://www.youtube.com/results?search_query=copilot+studio+tutorial)
- [Power Automate Excel Processing](https://www.youtube.com/results?search_query=power+automate+excel+tutorial)

---

## Appendix A: Complete Flow Templates

### Orchestrator Flow - Full JSON Export

```json
{
  "definition": {
    "triggers": {
      "When_an_HTTP_request_is_received": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "fileUrls": { "type": "array" },
              "sessionId": { "type": "string" },
              "userEmail": { "type": "string" }
            }
          }
        }
      }
    },
    "actions": {
      "Initialize_allApplications": {
        "type": "InitializeVariable",
        "inputs": {
          "variables": [{ "name": "allApplications", "type": "array", "value": [] }]
        }
      }
    }
  }
}
```

*(Full template would be extensive - export from Power Automate after building)*

---

## Appendix B: Noise Pattern Reference

### Application Noise Patterns

Applications to filter out during consolidation:

```
Update patterns:
- *Update*
- *Patch*
- *Hotfix*
- KB*
- *Security Update*
- *Cumulative Update*

Runtime/Framework patterns:
- Microsoft Visual C++*
- .NET Framework*
- *Redistributable*
- *Runtime*

System utilities:
- Microsoft Update Health Tools
- Windows Update*
- Microsoft Edge Update*
```

### Version / Duplicate Handling Logic

> **Note:** In the LLM-first approach (Agent 2, Steps 3-4), the Copilot agent's LLM uses its reasoning to identify duplicates and consolidate applications. The LLM is instructed to: (1) use exact application name match (case-insensitive) for consolidation, (2) keep different versions of the same application as separate entries, and (3) merge identical application + version combinations from different machines into one entry with a machine list. This is more intelligent than the previous first-occurrence-wins approach, as the LLM can handle edge cases in naming.

```
LLM-based consolidation behavior (as implemented in Agent 2):
For applications:
1. LLM identifies noise using reasoning (updates, patches, drivers, dependencies)
2. LLM groups by exact application name match (case-insensitive)
3. Different versions of the same application → separate entries
4. Same application + version on multiple machines → merged entry with machine list
5. Application-dependent drivers/updates → classified as noise and removed
```

---

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| **Azure Migrate** | Microsoft service for discovering, assessing, and migrating workloads to Azure |
| **Azure Blob Storage** | Microsoft's object storage solution for the cloud, used for temporary file storage without requiring SharePoint/OneDrive access |
| **CSV** | Comma-Separated Values file format |
| **Copilot Studio** | Microsoft's no-code platform for building conversational AI agents |
| **Power Automate** | Microsoft's workflow automation platform |
| **SAS Token** | Shared Access Signature - a URI that grants restricted access to Azure Storage resources |
| **SharePoint** | Microsoft's document management and collaboration platform |
| **Consolidation** | Process of combining and deduplicating data |
| **FQDN** | Fully Qualified Domain Name |
| **Child Flow** | A Power Automate flow called from another flow |
| **Orchestrator** | The main flow that coordinates other flows |
| **Lifecycle Management** | Azure Blob Storage feature to automatically manage blob retention and deletion |

---

**Document Version**: 1.0  
**Last Updated**: February 2026  
**Author**: AZMrepo Project Team

---

*For questions or issues with this guide, please refer to the troubleshooting section or contact the project team.*
