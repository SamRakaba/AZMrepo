# Azure Migrate CSV Processing Agents - Copilot Studio Guide

A comprehensive guide for building no-code Copilot Studio agents to process Azure Migrate export CSV files, consolidate application and database inventories, and generate downloadable reports.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Agent 1: File Upload Handler](#agent-1-file-upload-handler)
5. [Agent 2: Application Inventory Processor](#agent-2-application-inventory-processor)
6. [Agent 3: SQL Server Inventory Processor](#agent-3-sql-server-inventory-processor)
7. [Agent 4: Database Inventory Processor](#agent-4-database-inventory-processor)
8. [Agent 5: Report Generator](#agent-5-report-generator)
9. [Orchestrating the Agents](#orchestrating-the-agents)
10. [Power Automate Flows](#power-automate-flows)
11. [Testing and Validation](#testing-and-validation)
12. [Troubleshooting](#troubleshooting)
13. [Resources](#resources)

---

## Overview

### Purpose

This guide provides step-by-step instructions for building a suite of Microsoft Copilot Studio agents that:

1. **Accept file uploads** - Allow users to upload one or more Azure Migrate export CSV files
2. **Process Application Inventory** - Parse the `ApplicationInventory` sheet and consolidate applications by removing duplicates and noise (variants, dependencies, patches)
3. **Process SQL Server Inventory** - Parse SQL Server sheets and create a unique list of SQL instances
4. **Process Database Inventory** - Parse other database sheets and create a unique database list
5. **Generate Reports** - Create a new consolidated spreadsheet with 2-3 sheets containing unique applications, SQL databases, and other databases
6. **Enable Download** - Provide users with a downloadable link to the generated spreadsheet

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
│  • Triggers processing workflow                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────────┐
│ AGENT 2: App Inventory│ │ AGENT 3: SQL Server   │ │ AGENT 4: Database     │
│ • Parse AppInventory  │ │ • Parse SQL sheets    │ │ • Parse DB sheets     │
│ • Remove duplicates   │ │ • Remove duplicates   │ │ • Remove duplicates   │
│ • Filter noise/variants│ │ • Consolidate instances│ │ • Consolidate DBs    │
│ • Generate unique list│ │ • Generate unique list│ │ • Generate unique list│
└───────────────────────┘ └───────────────────────┘ └───────────────────────┘
                    │                 │                 │
                    └─────────────────┼─────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AGENT 5: Report Generator                               │
│  • Combine processed data                                                   │
│  • Create multi-sheet Excel file                                            │
│  • Store in temporary storage (Azure Blob Storage or SharePoint/OneDrive)   │
│  • Generate download link                                                   │
│  • Notify user with download URL                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Power Automate Orchestrator                             │
│  • Coordinates agent sequence                                               │
│  • Handles data flow between agents                                         │
│  • Error handling and notifications                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| File Upload Handler | Accept and validate CSV files | Copilot Studio + Power Automate |
| Temporary Storage | Store uploaded files temporarily | Azure Blob Storage (recommended) or SharePoint |
| App Inventory Processor | Consolidate applications | Power Automate + Excel Connector |
| SQL Server Processor | Consolidate SQL instances | Power Automate + Excel Connector |
| Database Processor | Consolidate other databases | Power Automate + Excel Connector |
| Report Generator | Create final spreadsheet | Power Automate + Excel Connector |
| Orchestrator | Coordinate agent workflow | Power Automate Flow |

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
           └── databases_consolidated.json
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
           └── databases_consolidated.json
   /Reports/
       └── {SessionID}/
           └── ConsolidatedReport_{timestamp}.xlsx
   ```

---

## Agent 1: File Upload Handler

### Purpose
This agent is the **starting point of the Azure Migrate processing flow**. It provides users with instructions to upload Azure Migrate extracted CSV files, accepts the file uploads, validates them, stores files in temporary storage (Azure Blob Storage recommended), and triggers the processing workflow.

> **Important**: This agent is:
> - **NOT conversational** - It follows a structured flow without general chat capabilities
> - **Triggered on file(s) upload** - The main flow activates when users upload files
> - **The start of the processing flow** - It initiates the entire Azure Migrate CSV processing pipeline
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
   - Type exactly: `Handles Azure Migrate CSV file uploads and initiates processing workflow for application and database inventory consolidation`

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
   Description: Handles Azure Migrate CSV file uploads and initiates processing workflow for application and database inventory consolidation
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
4. TRIGGER the processing workflow upon successful upload

IMPORTANT BEHAVIOR:
- You are NOT a general-purpose conversational agent
- You should ONLY handle Azure Migrate CSV file upload requests
- Always start by providing clear instructions for file upload
- Do NOT engage in off-topic conversations
- If users ask unrelated questions, redirect them to upload their files

EXPECTED FILE FORMATS:
- CSV files exported from Azure Migrate
- Excel files (.xlsx) with sheets: ApplicationInventory, SQL Server, Database

WORKFLOW:
1. Greet the user and explain the purpose
2. Provide instructions for uploading Azure Migrate CSV files
3. Wait for file upload
4. Validate and process uploaded files
5. Confirm processing has started

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

##### Step 3.1.9: Add End Conversation Node

1. Click the **+** button below the last variable node
2. Select **Topic management** → **End current topic**
3. This ensures the topic ends cleanly after initialization

##### Step 3.1.10: Save the Topic

1. Click the **Save** button at the top-right of the canvas
2. Wait for the "Topic saved" confirmation

#### Step 3.2: Verify Global Variables

After saving the topic, verify your global variables are created:

1. Open any topic in your agent (or create a test topic)
2. Add a "Send a message" node
3. Click the **{x}** icon (Insert variable) in the message text
4. In the variable picker, you should see your global variables listed with the `Global.` prefix:

```
┌──────────────────────────┬─────────┬───────────────────────────────────────────┐
│ Variable Name            │ Type    │ Purpose                                   │
├──────────────────────────┼─────────┼───────────────────────────────────────────┤
│ Global.sessionId         │ String  │ Unique identifier for the current session │
│ Global.uploadStatus      │ String  │ Current status of file upload             │
│ Global.downloadUrl       │ String  │ URL for downloading the report            │
│ Global.processingStatus  │ String  │ Current status of processing workflow     │
│ Global.errorMessage      │ String  │ Error message if processing fails         │
└──────────────────────────┴─────────┴───────────────────────────────────────────┘
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
   • Database (Database types, Versions)
3. Files can be in CSV or Excel (.xlsx) format
4. You can upload one or multiple files at once

📁 **What I'll do with your files:**
• Parse and validate the data
• Remove duplicates and noise
• Consolidate application and database inventories
• Generate a downloadable report

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

I'm now processing your Azure Migrate data. This includes:
1. 📊 Parsing Application Inventory sheet
2. 🗄️ Processing SQL Server instances
3. 💾 Consolidating database inventory
4. 📝 Generating your report

⏳ This may take a few moments depending on file size.

I'll notify you when the consolidated report is ready for download. You can also ask me "Check status" at any time.
```

5. Click outside the text area to save the message

**Part B: Add a Power Automate Flow Call Node (Placeholder)**

> **Note:** The Power Automate agent flow that processes the uploaded files will be fully configured in **Step 5**. The steps below add the flow tool node to the canvas now so the topic structure is complete. You will return here to configure the inputs and outputs once the flow is ready.

6. Click the **+** (Add node) button below the message node you just added (still inside the TRUE branch)
7. From the dropdown menu, select **Add a tool**
8. You have two options in the tool selection panel:
   - **New Agent flow** – Select this to create a new agent flow template with the required trigger and response action already configured. You will build the processing logic in Step 5. For now, click **Publish** to save the empty template, then click **Go back to agent**.
   - **Select an existing flow** – If you have already created and published the **Handle File Upload – Azure Migrate** flow (from Step 5), select it here.
9. If you created a new agent flow template and returned to the topic, an **Action** node will appear in the canvas — this is expected and will be fully configured in Step 5.

> **Tip:** If you prefer to skip adding the flow tool for now, you can come back after completing Step 5. Locate the TRUE branch in this topic, click **+**, select **Add a tool**, and choose the flow to add and configure it at that point.

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

This topic allows users to check on the status of their file processing.

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
• **Database Inventory Sheet** - Consolidated database inventory (Oracle, MySQL, PostgreSQL, etc.)

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
• Missing required sheets (ApplicationInventory, SQL Server, Database)
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
• Creating unique lists of applications, SQL servers, and databases
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
• 💾 Organizing database inventories
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

##### Step 5.1.1: Configure Flow Inputs and Actions

Whether you created the flow from a topic or from Power Automate directly, open the flow designer and configure:

1. **Name the flow**: On the **Overview** page, under **Details**, set the name to `Handle File Upload - Azure Migrate`

2. **Configure flow inputs** (click on the **When an agent calls the flow** trigger):
   - Click **+ Add an input**
   - Add input: Type: **File**, Name: `uploadedFiles`
   - Add input: Type: **Text**, Name: `userId`
   - Add input: Type: **Text**, Name: `userEmail`

3. **Configure flow outputs** (click on the **Respond to the agent** action):
   - Verify the **Asynchronous response** toggle is set to **Off** under **Networking** in the action settings
   - Add output: Type: **Text**, Name: `sessionId`
   - Add output: Type: **Text**, Name: `status`
   - Add output: Type: **Text**, Name: `message`

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
3. Map the inputs on the Action node:
   - `uploadedFiles` → For file data, click the input field, select **Formula** (fx), and enter the following Power Fx expression:
     ```
     { contentBytes: First(System.Activity.Attachments).Content, name: First(System.Activity.Attachments).Name }
     ```
   - `userId` → `System.User.Id` (System variable)
   - `userEmail` → `System.User.Email` (System variable)
4. Map the outputs:
   - `sessionId` → `Global.sessionId`
   - `status` → `Global.uploadStatus`
   - `message` → (display in a Message node below the Action node)
5. Click **Save**

> **Note:** If you added the flow as an agent-level tool (from the Tools page), you can also configure its inputs on the tool's **Details** page. Go to **Tools**, select the flow, and set the **Inputs** section with Power Fx formulas or variable references.

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

1. Go to the **Tools** page, find **Handle File Upload - Azure Migrate**, and click **View flow details** to open the flow in Power Automate (or open it from the **Action** node in your topic)
2. In the flow designer, add the following actions between the trigger and the **Respond to the agent** action:
3. Configure the flow for Azure Blob Storage:

**Flow Configuration:**
```
Trigger: When an agent calls the flow (Run a flow from Copilot)

Flow Actions:
1. Compose - Generate Session ID
   Expression: guid()

2. Create blob (V2) - Azure Blob Storage
   Storage Account: Your Azure Storage Account
   Container: uploads
   Blob name: @{outputs('Generate_Session_ID')}/@{item()?['name']}
   Blob content: @{item()?['contentBytes']}

3. HTTP - Call Orchestrator Flow
   (Detailed in Power Automate Flows section)

4. Respond to the agent:
   - sessionId: @{outputs('Generate_Session_ID')}
   - status: "Processing"
   - message: "Files uploaded successfully. Processing has started."
```

**Benefits of Azure Blob Storage:**
- Users do NOT need SharePoint or OneDrive access
- Automatic cleanup via lifecycle management policies
- Cost-effective for temporary file storage
- Better suited for large file uploads
- Supports programmatic access via SAS tokens

#### Option B: SharePoint Storage (Alternative)

1. In the flow designer for **Handle File Upload - Azure Migrate**, configure the actions for SharePoint instead of Azure Blob Storage
2. Add SharePoint connector actions between the trigger and the **Respond to the agent** action
3. Configure the flow for SharePoint (detailed in [Power Automate Flows](#power-automate-flows) section)

> **Note**: This option requires users to have SharePoint or OneDrive access.

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
This agent (implemented primarily as a Power Automate flow) processes the ApplicationInventory sheet to consolidate applications and remove duplicates. It filters out noise (updates, patches, redistributables) and creates a unique list of business applications.

---

### Step 1: Access Power Automate

#### Step 1.1: Navigate to Power Automate

1. Open your web browser (Microsoft Edge, Chrome, or Firefox recommended)
2. Navigate to the URL: **https://make.powerautomate.com**
3. If prompted, sign in with your Microsoft 365 credentials:
   - Enter your email address in the **Sign in** field
   - Click the **Next** button
   - Enter your password
   - Click the **Sign in** button
4. If prompted for multi-factor authentication, complete the verification
5. Wait for the Power Automate home page to load

#### Step 1.2: Select Your Environment

1. Look at the top-right corner of the Power Automate interface
2. Click on the **Environment selector** dropdown
3. Select the same environment you used for Copilot Studio
4. Wait for the environment to switch

---

### Step 2: Create the Flow

#### Step 2.1: Start New Flow Creation

1. In the left navigation panel, click on **My flows**
2. Click the **+ New flow** button at the top
3. From the dropdown menu, select **Instant cloud flow**
4. A dialog box will appear asking you to configure the flow

#### Step 2.2: Configure Flow Name and Trigger

1. In the **Flow name** field, type: `Process Application Inventory`
2. Under **Choose how to trigger this flow**, scroll down
3. Select **When a HTTP request is received**
   - This allows the flow to be called from other flows
4. Click the **Create** button
5. The flow designer will open with the trigger already added

#### Step 2.3: Configure the HTTP Trigger

1. Click on the **When a HTTP request is received** trigger to expand it
2. Click on **Use sample payload to generate schema**
3. In the dialog that appears, paste the following JSON sample:

```json
{
  "filePath": "/uploads/session-123/AzureMigrate_Export.xlsx",
  "sessionId": "session-123-guid",
  "storageType": "AzureBlob"
}
```

4. Click **Done**
5. The schema will be automatically generated

---

### Step 3: Add Flow Variables

Variables store data during flow execution. You need to initialize all variables at the beginning of the flow.

#### Step 3.1: Add Variable 1 - uniqueApplications

1. Below the trigger, click the **+** button
2. Click **Add an action**
3. In the search box, type: `Initialize variable`
4. Select **Initialize variable** from the results (under Built-in > Variables)

5. Configure the action:
   - Click in the **Name** field
   - Type: `uniqueApplications`
   - Click the **Type** dropdown
   - Select: **Array**
   - Click in the **Value** field
   - Type: `[]` (empty array)

6. Click on the action title "Initialize variable" and rename it to:
   - Type: `Initialize uniqueApplications`

#### Step 3.2: Add Variable 2 - processedApps

1. Below the first variable action, click the **+** button
2. Click **Add an action**
3. Search for and select **Initialize variable**

4. Configure the action:
   - **Name**: `processedApps`
   - **Type**: **Object**
   - **Value**: `{}`

5. Rename the action to: `Initialize processedApps`

#### Step 3.3: Add Variable 3 - noisePatterns

1. Click **+** → **Add an action** → **Initialize variable**

2. Configure the action:
   - **Name**: `noisePatterns`
   - **Type**: **Array**
   - **Value**: 
   ```json
   ["Update", "Patch", "Hotfix", "KB", "Security Update", "Service Pack", "Redistributable", "Runtime", ".NET Framework", "Visual C++", "Microsoft Visual", "Microsoft Update", "Cumulative Update", "Definition Update"]
   ```

3. Rename to: `Initialize noisePatterns`

#### Step 3.4: Add Variable 4 - totalProcessed

1. Click **+** → **Add an action** → **Initialize variable**

2. Configure:
   - **Name**: `totalProcessed`
   - **Type**: **Integer**
   - **Value**: `0`

3. Rename to: `Initialize totalProcessed`

#### Step 3.5: Verify All Variables

Your flow should now have these actions in order:
```
┌─────────────────────────────────────────────────────────────┐
│ When a HTTP request is received                             │
├─────────────────────────────────────────────────────────────┤
│ Initialize uniqueApplications (Array: [])                   │
├─────────────────────────────────────────────────────────────┤
│ Initialize processedApps (Object: {})                       │
├─────────────────────────────────────────────────────────────┤
│ Initialize noisePatterns (Array: ["Update", "Patch", ...])  │
├─────────────────────────────────────────────────────────────┤
│ Initialize totalProcessed (Integer: 0)                      │
└─────────────────────────────────────────────────────────────┘
```

---

### Step 4: Add Excel Data Retrieval Action

#### Step 4.1: Add Excel Connector

1. Below the last variable, click **+** → **Add an action**
2. In the search box, type: `Excel Online`
3. Select **List rows present in a table** (under Excel Online (Business))
4. If prompted, sign in to create the Excel Online connection

#### Step 4.2: Configure Excel Connection (For SharePoint Storage)

If using SharePoint storage:

1. **Location**: Click the dropdown and select your SharePoint site
2. **Document Library**: Select `Uploads` (or your upload library)
3. **File**: Click in the field and select **Dynamic content**
   - Click **Dynamic content** tab
   - Select `filePath` from the HTTP trigger
4. **Table Name**: Type: `ApplicationInventory`
   - Or click the dropdown to select from available tables

#### Step 4.3: Configure Excel Connection (For Azure Blob Storage)

If using Azure Blob Storage, you need a different approach:

1. Add **Azure Blob Storage** connector instead
2. Action: **Get blob content (V2)**
3. Then add a **Parse JSON** action to parse the CSV/Excel content

Alternative approach using Azure Functions or custom connector for Excel parsing.

#### Step 4.4: Rename the Action

1. Click on the action title
2. Rename to: `Get ApplicationInventory Rows`

---

### Step 5: Add Row Processing Loop

#### Step 5.1: Add Apply to Each Loop

1. Click **+** → **Add an action**
2. Search for: `Apply to each`
3. Select **Apply to each** (under Built-in > Control)

4. Configure the loop:
   - Click in the **Select an output from previous steps** field
   - Click **Dynamic content**
   - Select **value** from the "Get ApplicationInventory Rows" action
   - This will be: `body('Get_ApplicationInventory_Rows')?['value']`

5. Rename the action to: `Process Each Application Row`

---

### Step 6: Add Processing Logic Inside the Loop

All the following actions go INSIDE the "Apply to each" loop.

> **Important Note About Expression Names**: The expressions below reference the loop action name `Process_Each_Application_Row`. If you renamed your "Apply to each" action to something different in Step 5.1, you must update the action name in all expressions to match. For example, if you named your loop `Loop_Through_Apps`, change `items('Process_Each_Application_Row')` to `items('Loop_Through_Apps')` in all expressions.

#### Step 6.1: Add Compose - Normalize Application Name

1. Inside the loop, click **Add an action**
2. Search for and select **Compose** (under Built-in > Data Operations)

3. Configure:
   - Click in the **Inputs** field
   - Click **Expression** tab (not Dynamic content)
   - Type the following expression:
   ```
   toLower(trim(items('Process_Each_Application_Row')?['Application']))
   ```
   - Click **OK**

4. Rename to: `Normalize Application Name`

#### Step 6.2: Add Compose - Check if Noise

1. Inside the loop, click **Add an action** → **Compose**

2. Click in **Inputs** → Click **Expression** tab
3. Type the following expression:
```
if(
  or(
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'update'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'patch'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'hotfix'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'kb'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'redistributable'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'runtime'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), '.net framework'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'visual c++'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'cumulative update'),
    contains(toLower(coalesce(items('Process_Each_Application_Row')?['Application'], '')), 'security update')
  ),
  true,
  false
)
```
4. Click **OK**
5. Rename to: `Check if Noise`

#### Step 6.3: Add Condition - Skip Noise Applications

1. Inside the loop, click **Add an action**
2. Search for and select **Condition** (under Built-in > Control)

3. Configure the condition:
   - In the left field, click **Dynamic content**
   - Select **Outputs** from "Check if Noise"
   - Click the operator dropdown and select: **is equal to**
   - In the right field, type: `false`

4. Rename to: `Skip if Noise Application`

#### Step 6.4: Configure the "If yes" Branch (Not Noise - Process)

Inside the **If yes** branch (when the application is NOT noise):

##### Step 6.4.1: Add Compose - Create Application Key

1. In the "If yes" branch, click **Add an action** → **Compose**

2. Click **Inputs** → **Expression** tab
3. Type:
```
concat(
  toLower(trim(coalesce(items('Process_Each_Application_Row')?['Application'], ''))),
  '_',
  toLower(trim(coalesce(items('Process_Each_Application_Row')?['Provider'], '')))
)
```
4. Click **OK**
5. Rename to: `Create Application Key`

##### Step 6.4.2: Add Condition - Check for Duplicate

1. In the "If yes" branch, click **Add an action** → **Condition**

2. Configure:
   - Left field: Click **Expression** tab and type:
   ```
   contains(string(variables('processedApps')), outputs('Create_Application_Key'))
   ```
   - Operator: **is equal to**
   - Right field: `false`

3. Rename to: `Check if Not Duplicate`

##### Step 6.4.3: Add Actions in "If yes" of Duplicate Check (New Application)

In the "If yes" branch of the duplicate check (when it's a new unique application):

###### Add Append to Array Variable

1. Click **Add an action**
2. Search for: `Append to array variable`
3. Select **Append to array variable** (under Built-in > Variables)

4. Configure:
   - **Name**: Select `uniqueApplications` from dropdown
   - **Value**: Click in the field and paste:
   ```json
   {
     "Application": "@{items('Process_Each_Application_Row')?['Application']}",
     "Version": "@{items('Process_Each_Application_Row')?['Version']}",
     "Provider": "@{items('Process_Each_Application_Row')?['Provider']}",
     "MachineName": "@{items('Process_Each_Application_Row')?['MachineName']}",
     "MachineManagerFqdn": "@{items('Process_Each_Application_Row')?['MachineManagerFqdn']}"
   }
   ```

5. Rename to: `Add Unique Application`

###### Add Set Variable for Tracking

1. Click **Add an action**
2. Search for: `Set variable`
3. Select **Set variable** (under Built-in > Variables)

4. Configure:
   - **Name**: Select `processedApps`
   - **Value**: Click **Expression** tab and type:
   ```
   addProperty(variables('processedApps'), outputs('Create_Application_Key'), true)
   ```

5. Rename to: `Mark Application as Processed`

#### Step 6.5: Update Total Processed Counter

After the loop (not inside it):

1. Click **+** after the "Apply to each" loop ends
2. Add **Set variable**
3. Configure:
   - **Name**: `totalProcessed`
   - **Value**: Click **Expression**:
   ```
   length(body('Get_ApplicationInventory_Rows')?['value'])
   ```

4. Rename to: `Set Total Processed Count`

---

### Step 7: Add Output Response

#### Step 7.1: Add Compose for Final Output

1. Click **+** → **Add an action** → **Compose**

2. In **Inputs**, paste:
```json
{
  "uniqueApplications": @{variables('uniqueApplications')},
  "statistics": {
    "totalProcessed": @{variables('totalProcessed')},
    "totalUnique": @{length(variables('uniqueApplications'))},
    "duplicatesRemoved": @{sub(variables('totalProcessed'), length(variables('uniqueApplications')))},
    "noiseFiltered": "See noise patterns for filtered items"
  },
  "status": "Complete",
  "processedAt": "@{utcNow()}"
}
```

3. Rename to: `Compose Final Output`

#### Step 7.2: Add HTTP Response

1. Click **+** → **Add an action**
2. Search for: `Response`
3. Select **Response** (under Built-in > Request)

4. Configure:
   - **Status Code**: `200`
   - **Headers**: Click **+ Add new parameter** and add:
     - **Key**: `Content-Type`
     - **Value**: `application/json`
   - **Body**: Click **Dynamic content** → Select **Outputs** from "Compose Final Output"

5. Rename to: `Return Results`

---

### Step 8: Save and Test the Flow

#### Step 8.1: Save the Flow

1. Click the **Save** button at the top-right of the flow designer
2. Wait for the "Flow saved" confirmation message

#### Step 8.2: Get the Flow URL

1. Click on the **When a HTTP request is received** trigger
2. Copy the **HTTP POST URL** that appears
3. Save this URL - you'll need it for the Orchestrator flow

#### Step 8.3: Test the Flow

1. Click **Test** at the top-right
2. Select **Manually**
3. Click **Test**
4. In another tab, use a tool like Postman to send a test request:
   ```json
   POST [Your Flow URL]
   Content-Type: application/json
   
   {
     "filePath": "/Uploads/test-session/test_application_inventory.xlsx",
     "sessionId": "test-session-123"
   }
   ```
5. Verify the flow runs successfully
6. Check the output contains consolidated unique applications

---

### Step 9: Complete Flow Diagram

Your completed flow should look like this:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When a HTTP request is received                                         │
│ (Trigger with filePath, sessionId inputs)                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Initialize Variables                                                     │
│ • uniqueApplications (Array: [])                                        │
│ • processedApps (Object: {})                                            │
│ • noisePatterns (Array)                                                  │
│ • totalProcessed (Integer: 0)                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Get ApplicationInventory Rows                                            │
│ (List rows from Excel table)                                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ FOR EACH Application Row:                                           │ │
│ │                                                                     │ │
│ │   1. Normalize Application Name (toLower, trim)                     │ │
│ │                        │                                            │ │
│ │                        ▼                                            │ │
│ │   2. Check if Noise (contains update, patch, etc.)                  │ │
│ │                        │                                            │ │
│ │                        ▼                                            │ │
│ │   3. IF NOT Noise:                                                  │ │
│ │      │                                                              │ │
│ │      ├─→ Create Application Key (app_provider)                      │ │
│ │      │                                                              │ │
│ │      └─→ IF NOT Duplicate:                                          │ │
│ │          │                                                          │ │
│ │          ├─→ Append to uniqueApplications array                     │ │
│ │          │                                                          │ │
│ │          └─→ Mark as processed in processedApps                     │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Set Total Processed Count                                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Compose Final Output                                                     │
│ (JSON with uniqueApplications and statistics)                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Return Results (HTTP 200 Response)                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Consolidation Rules Reference

The Application Inventory Processor applies these consolidation rules:

| Rule | Description | Example |
|------|-------------|---------|
| **Exact Duplicate** | Same Application + Provider | Remove second occurrence |
| **Version Variant** | Same Application, different version | Keep first version encountered |
| **Update/Patch** | Contains "Update", "Patch", "KB" | Filter out as noise |
| **Redistributable** | Runtime or library dependency | Filter out as noise |
| **Case Normalization** | Different casing | Treat as same application |
| **Trim Whitespace** | Leading/trailing spaces | Normalize before comparison |

### Noise Patterns Filtered

The following patterns are automatically filtered out:
- Windows Updates (KB numbers)
- Security Updates
- Cumulative Updates
- Microsoft Visual C++ Redistributables
- .NET Framework installations
- Runtime packages
- Hotfixes and Patches
- Service Packs

---

## Agent 3: SQL Server Inventory Processor

### Purpose
Process SQL Server inventory sheets to create a unique list of SQL Server instances. This flow identifies unique SQL Server installations across all machines and consolidates them by machine name and instance name.

---

### Step 1: Create the SQL Server Processing Flow

#### Step 1.1: Access Power Automate

1. Open your web browser
2. Navigate to: **https://make.powerautomate.com**
3. Sign in with your Microsoft 365 credentials if prompted
4. Ensure you are in the correct environment (same as Copilot Studio)

#### Step 1.2: Create New Flow

1. In the left navigation, click **My flows**
2. Click **+ New flow** button
3. Select **Instant cloud flow**
4. Configure the flow:
   - **Flow name**: Type `Process SQL Server Inventory`
   - **Trigger**: Select **When a HTTP request is received**
5. Click **Create**

#### Step 1.3: Configure the HTTP Trigger

1. Click on the **When a HTTP request is received** trigger to expand it
2. Click **Use sample payload to generate schema**
3. Paste the following JSON:
```json
{
  "filePath": "/uploads/session-123/AzureMigrate_Export.xlsx",
  "sessionId": "session-123-guid",
  "storageType": "AzureBlob"
}
```
4. Click **Done**

---

### Step 2: Initialize Flow Variables

#### Step 2.1: Add Variable - uniqueSQLInstances

1. Click **+** below the trigger → **Add an action**
2. Search for `Initialize variable` and select it
3. Configure:
   - **Name**: `uniqueSQLInstances`
   - **Type**: **Array**
   - **Value**: `[]`
4. Rename action to: `Initialize uniqueSQLInstances`

#### Step 2.2: Add Variable - processedInstances

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `processedInstances`
   - **Type**: **Object**
   - **Value**: `{}`
3. Rename to: `Initialize processedInstances`

#### Step 2.3: Add Variable - totalProcessed

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `totalProcessed`
   - **Type**: **Integer**
   - **Value**: `0`
3. Rename to: `Initialize totalProcessed`

---

### Step 3: Retrieve SQL Server Data

#### Step 3.1: Add Excel Connector

1. Click **+** → **Add an action**
2. Search for `Excel Online`
3. Select **List rows present in a table** (Excel Online Business)
4. If prompted, sign in to create the connection

#### Step 3.2: Configure the Excel Action

1. **Location**: Select your SharePoint site from the dropdown
2. **Document Library**: Select `Uploads`
3. **File**: Click in the field → **Dynamic content** → Select `filePath`
4. **Table Name**: Type `SQLServer`
   - Note: This should match the exact name of the SQL Server sheet/table in the Excel file
5. Rename action to: `Get SQL Server Rows`

---

### Step 4: Process Each SQL Server Row

#### Step 4.1: Add Apply to Each Loop

1. Click **+** → **Add an action**
2. Search for `Apply to each` and select it
3. In **Select an output from previous steps**:
   - Click **Dynamic content**
   - Select **value** from "Get SQL Server Rows"
4. Rename to: `Process Each SQL Instance`

> **Important Note About Expression Names**: The expressions below reference the loop action name `Process_Each_SQL_Instance`. If you renamed your "Apply to each" action to something different, you must update the action name in all expressions to match.

#### Step 4.2: Inside the Loop - Create Instance Key

1. Inside the loop, click **Add an action**
2. Search for and select **Compose**
3. Click **Inputs** → **Expression** tab
4. Type the following expression:
```
concat(
  toLower(trim(coalesce(items('Process_Each_SQL_Instance')?['MachineName'], ''))),
  '_',
  toLower(trim(coalesce(items('Process_Each_SQL_Instance')?['Instance Name'], 'MSSQLSERVER')))
)
```
5. Click **OK**
6. Rename to: `Create Instance Key`

**Note**: The `coalesce` function handles cases where Instance Name is empty, defaulting to 'MSSQLSERVER' (the default SQL Server instance name).

#### Step 4.3: Add Condition - Check for Duplicate

1. Inside the loop, click **Add an action**
2. Search for and select **Condition**
3. Configure the condition:
   - Left field: Click **Expression** → Type:
   ```
   contains(string(variables('processedInstances')), outputs('Create_Instance_Key'))
   ```
   - Operator: **is equal to**
   - Right field: `false`
4. Rename to: `Check if Not Duplicate Instance`

#### Step 4.4: Configure "If yes" Branch (New Unique Instance)

##### Add Append to Array Variable

1. In the "If yes" branch, click **Add an action**
2. Search for and select **Append to array variable**
3. Configure:
   - **Name**: Select `uniqueSQLInstances`
   - **Value**: Paste the following:
```json
{
  "MachineName": "@{items('Process_Each_SQL_Instance')?['MachineName']}",
  "InstanceName": "@{coalesce(items('Process_Each_SQL_Instance')?['Instance Name'], 'MSSQLSERVER')}",
  "Edition": "@{items('Process_Each_SQL_Instance')?['Edition']}",
  "ServicePack": "@{coalesce(items('Process_Each_SQL_Instance')?['Service Pack'], '')}",
  "Version": "@{items('Process_Each_SQL_Instance')?['Version']}",
  "Port": "@{coalesce(items('Process_Each_SQL_Instance')?['Port'], '1433')}",
  "MachineManagerFqdn": "@{items('Process_Each_SQL_Instance')?['MachineManagerFqdn']}"
}
```
4. Rename to: `Add Unique SQL Instance`

##### Add Set Variable

1. Click **Add an action**
2. Search for and select **Set variable**
3. Configure:
   - **Name**: Select `processedInstances`
   - **Value**: Click **Expression** → Type:
   ```
   addProperty(variables('processedInstances'), outputs('Create_Instance_Key'), true)
   ```
4. Rename to: `Mark Instance as Processed`

---

### Step 5: Add Output Response

#### Step 5.1: Set Total Processed Count

1. After the loop (not inside it), click **+** → **Add an action**
2. Select **Set variable**
3. Configure:
   - **Name**: `totalProcessed`
   - **Value**: Click **Expression**:
   ```
   length(body('Get_SQL_Server_Rows')?['value'])
   ```
4. Rename to: `Set Total Processed Count`

#### Step 5.2: Compose Final Output

1. Click **+** → **Add an action** → **Compose**
2. In **Inputs**, paste:
```json
{
  "uniqueSQLInstances": @{variables('uniqueSQLInstances')},
  "statistics": {
    "totalProcessed": @{variables('totalProcessed')},
    "totalUnique": @{length(variables('uniqueSQLInstances'))},
    "duplicatesRemoved": @{sub(variables('totalProcessed'), length(variables('uniqueSQLInstances')))}
  },
  "status": "Complete",
  "processedAt": "@{utcNow()}"
}
```
3. Rename to: `Compose SQL Output`

#### Step 5.3: Add HTTP Response

1. Click **+** → **Add an action**
2. Search for and select **Response**
3. Configure:
   - **Status Code**: `200`
   - **Headers**: Add `Content-Type`: `application/json`
   - **Body**: Click **Dynamic content** → Select **Outputs** from "Compose SQL Output"
4. Rename to: `Return SQL Results`

---

### Step 6: Save and Test

#### Step 6.1: Save the Flow

1. Click **Save** at the top-right
2. Wait for confirmation

#### Step 6.2: Copy the Flow URL

1. Click on the HTTP trigger
2. Copy the **HTTP POST URL**
3. Save this URL for the Orchestrator flow

#### Step 6.3: Test the Flow

1. Click **Test** → Select **Manually** → Click **Test**
2. Send a test request with sample SQL Server data
3. Verify unique instances are returned correctly

---

### SQL Server Consolidation Rules

| Rule | Description | Example |
|------|-------------|---------|
| **Unique Instance** | MachineName + Instance Name combination | Server01_MSSQLSERVER |
| **Default Instance** | Empty Instance Name defaults to "MSSQLSERVER" | Server01 → Server01_MSSQLSERVER |
| **Default Port** | Empty Port defaults to "1433" | Standard SQL Server port |
| **Case Normalization** | All comparisons are case-insensitive | SERVER01 = server01 |

---

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When a HTTP request is received                                         │
│ (Trigger with filePath, sessionId)                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Initialize Variables                                                     │
│ • uniqueSQLInstances (Array: [])                                        │
│ • processedInstances (Object: {})                                       │
│ • totalProcessed (Integer: 0)                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Get SQL Server Rows (List rows from Excel SQLServer table)              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ FOR EACH SQL Instance Row:                                              │
│                                                                         │
│   1. Create Instance Key (MachineName_InstanceName)                     │
│                        │                                                │
│                        ▼                                                │
│   2. IF NOT Duplicate:                                                  │
│      ├─→ Append to uniqueSQLInstances array                             │
│      └─→ Mark as processed                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Set Total Processed Count                                               │
├─────────────────────────────────────────────────────────────────────────┤
│ Compose SQL Output (JSON with instances and statistics)                 │
├─────────────────────────────────────────────────────────────────────────┤
│ Return SQL Results (HTTP 200 Response)                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Agent 4: Database Inventory Processor

### Purpose
Process database inventory sheets for non-SQL Server databases (Oracle, MySQL, PostgreSQL, MongoDB, etc.). This flow identifies unique database installations across all machines.

---

### Step 1: Create the Database Processing Flow

#### Step 1.1: Access Power Automate

1. Open your web browser
2. Navigate to: **https://make.powerautomate.com**
3. Sign in with your Microsoft 365 credentials if prompted
4. Ensure you are in the correct environment

#### Step 1.2: Create New Flow

1. In the left navigation, click **My flows**
2. Click **+ New flow** → **Instant cloud flow**
3. Configure:
   - **Flow name**: `Process Database Inventory`
   - **Trigger**: Select **When a HTTP request is received**
4. Click **Create**

#### Step 1.3: Configure the HTTP Trigger

1. Click on the trigger to expand it
2. Click **Use sample payload to generate schema**
3. Paste:
```json
{
  "filePath": "/uploads/session-123/AzureMigrate_Export.xlsx",
  "sessionId": "session-123-guid",
  "storageType": "AzureBlob"
}
```
4. Click **Done**

---

### Step 2: Initialize Flow Variables

#### Step 2.1: Add Variable - uniqueDatabases

1. Click **+** → **Add an action**
2. Search for and select **Initialize variable**
3. Configure:
   - **Name**: `uniqueDatabases`
   - **Type**: **Array**
   - **Value**: `[]`
4. Rename to: `Initialize uniqueDatabases`

#### Step 2.2: Add Variable - processedDatabases

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `processedDatabases`
   - **Type**: **Object**
   - **Value**: `{}`
3. Rename to: `Initialize processedDatabases`

#### Step 2.3: Add Variable - totalProcessed

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `totalProcessed`
   - **Type**: **Integer**
   - **Value**: `0`
3. Rename to: `Initialize totalProcessed`

---

### Step 3: Retrieve Database Data

#### Step 3.1: Add Excel Connector

1. Click **+** → **Add an action**
2. Search for `Excel Online`
3. Select **List rows present in a table**

#### Step 3.2: Configure the Excel Action

1. **Location**: Select your SharePoint site
2. **Document Library**: Select `Uploads`
3. **File**: Click **Dynamic content** → Select `filePath`
4. **Table Name**: Type `Database`
5. Rename to: `Get Database Rows`

---

### Step 4: Process Each Database Row

#### Step 4.1: Add Apply to Each Loop

1. Click **+** → **Add an action**
2. Select **Apply to each**
3. Configure:
   - **Select an output**: Click **Dynamic content** → Select **value** from "Get Database Rows"
4. Rename to: `Process Each Database Row`

> **Important Note About Expression Names**: The expressions below reference the loop action name `Process_Each_Database_Row`. If you renamed your "Apply to each" action to something different, you must update the action name in all expressions to match.

#### Step 4.2: Inside the Loop - Normalize Database Type

1. Inside the loop, click **Add an action** → **Compose**
2. Click **Inputs** → **Expression** tab
3. Type:
```
toLower(trim(coalesce(items('Process_Each_Database_Row')?['Database Type'], 'Unknown')))
```
4. Click **OK**
5. Rename to: `Normalize Database Type`

#### Step 4.3: Create Database Key

1. Inside the loop, click **Add an action** → **Compose**
2. Click **Inputs** → **Expression** tab
3. Type:
```
concat(
  toLower(trim(coalesce(items('Process_Each_Database_Row')?['MachineName'], ''))),
  '_',
  outputs('Normalize_Database_Type')
)
```
4. Click **OK**
5. Rename to: `Create Database Key`

#### Step 4.4: Add Condition - Check for Duplicate

1. Inside the loop, click **Add an action** → **Condition**
2. Configure:
   - Left field: Click **Expression** → Type:
   ```
   contains(string(variables('processedDatabases')), outputs('Create_Database_Key'))
   ```
   - Operator: **is equal to**
   - Right field: `false`
3. Rename to: `Check if Not Duplicate Database`

#### Step 4.5: Configure "If yes" Branch (New Unique Database)

##### Add Append to Array Variable

1. In the "If yes" branch, click **Add an action**
2. Select **Append to array variable**
3. Configure:
   - **Name**: Select `uniqueDatabases`
   - **Value**: Paste:
```json
{
  "MachineName": "@{items('Process_Each_Database_Row')?['MachineName']}",
  "DatabaseType": "@{items('Process_Each_Database_Row')?['Database Type']}",
  "NormalizedType": "@{outputs('Normalize_Database_Type')}",
  "Version": "@{coalesce(items('Process_Each_Database_Row')?['Version'], '')}",
  "MachineManagerFqdn": "@{items('Process_Each_Database_Row')?['MachineManagerFqdn']}"
}
```
4. Rename to: `Add Unique Database`

##### Add Set Variable

1. Click **Add an action** → **Set variable**
2. Configure:
   - **Name**: Select `processedDatabases`
   - **Value**: Click **Expression** → Type:
   ```
   addProperty(variables('processedDatabases'), outputs('Create_Database_Key'), true)
   ```
3. Rename to: `Mark Database as Processed`

---

### Step 5: Add Output Response

#### Step 5.1: Set Total Processed Count

1. After the loop, click **+** → **Add an action**
2. Select **Set variable**
3. Configure:
   - **Name**: `totalProcessed`
   - **Value**: Click **Expression**:
   ```
   length(body('Get_Database_Rows')?['value'])
   ```
4. Rename to: `Set Total Processed Count`

#### Step 5.2: Compose Final Output

1. Click **+** → **Add an action** → **Compose**
2. In **Inputs**, paste:
```json
{
  "uniqueDatabases": @{variables('uniqueDatabases')},
  "statistics": {
    "totalProcessed": @{variables('totalProcessed')},
    "totalUnique": @{length(variables('uniqueDatabases'))},
    "duplicatesRemoved": @{sub(variables('totalProcessed'), length(variables('uniqueDatabases')))}
  },
  "status": "Complete",
  "processedAt": "@{utcNow()}"
}
```

3. Rename to: `Compose Database Output`

#### Step 5.3: Add HTTP Response

1. Click **+** → **Add an action** → **Response**
2. Configure:
   - **Status Code**: `200`
   - **Headers**: Add `Content-Type`: `application/json`
   - **Body**: Click **Dynamic content** → Select **Outputs** from "Compose Database Output"
3. Rename to: `Return Database Results`

---

### Step 6: Save and Test

#### Step 6.1: Save the Flow

1. Click **Save** at the top-right
2. Wait for confirmation

#### Step 6.2: Copy the Flow URL

1. Click on the HTTP trigger
2. Copy the **HTTP POST URL**
3. Save this URL for the Orchestrator flow

#### Step 6.3: Test the Flow

1. Click **Test** → **Manually** → **Test**
2. Send a test request with sample database data
3. Verify unique databases are returned correctly

---

### Database Consolidation Rules

| Rule | Description | Example |
|------|-------------|---------|
| **Unique Database** | MachineName + Database Type combination | DBServer01_oracle |
| **Type Normalization** | Lowercase comparison | "Oracle" = "oracle" |
| **Version Handling** | Keeps first encountered version | Not version-compared |
| **Unknown Type** | Empty type defaults to "Unknown" | Handled gracefully |

---

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ When a HTTP request is received                                         │
│ (Trigger with filePath, sessionId)                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Initialize Variables                                                     │
│ • uniqueDatabases (Array: [])                                           │
│ • processedDatabases (Object: {})                                       │
│ • totalProcessed (Integer: 0)                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Get Database Rows (List rows from Excel Database table)                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ FOR EACH Database Row:                                                  │
│                                                                         │
│   1. Normalize Database Type (toLower, trim)                            │
│                        │                                                │
│                        ▼                                                │
│   2. Create Database Key (MachineName_DatabaseType)                     │
│                        │                                                │
│                        ▼                                                │
│   3. IF NOT Duplicate:                                                  │
│      ├─→ Append to uniqueDatabases array                                │
│      └─→ Mark as processed                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Set Total Processed Count                                               │
├─────────────────────────────────────────────────────────────────────────┤
│ Compose Database Output (JSON with databases and statistics)            │
├─────────────────────────────────────────────────────────────────────────┤
│ Return Database Results (HTTP 200 Response)                             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Agent 5: Report Generator

### Purpose
Generate the final consolidated Excel spreadsheet with all unique applications, SQL instances, and databases, and provide a download link to the user. This flow creates a professional multi-sheet Excel report.

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
  "uniqueDatabases": [
    {
      "MachineName": "DBServer01",
      "DatabaseType": "Oracle",
      "Version": "19c",
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

### Step 8: Populate Database Sheet

#### Step 8.1: Add Database Rows

1. After the SQL loop, click **+** → **Add an action**
2. Select **Apply to each**
3. Configure:
   - **Select an output**: `uniqueDatabases` (from trigger)
4. Rename to: `Add Database Rows`

##### Inside the Loop - Add Database Row

1. Inside the loop, click **Add an action**
2. Select **Add a row into a table** (Excel Online Business)
3. Configure:
   - **Location**: Your SharePoint site
   - **Document Library**: `Reports`
   - **File**: `@{variables('reportFilePath')}`
   - **Table**: `UniqueDatabases`
   - **Row data**:
     - **MachineName**: `MachineName`
     - **DatabaseType**: `DatabaseType`
     - **Version**: `Version`
     - **MachineManagerFqdn**: `MachineManagerFqdn`
4. Rename to: `Add Database Row`

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
<td style="padding: 10px; border: 1px solid #ddd;">💾 Database Instances</td>
<td style="padding: 10px; border: 1px solid #ddd; text-align: right;"><strong>@{length(triggerBody()?['uniqueDatabases'])}</strong></td>
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
<li><strong>Database Inventory</strong> - Non-SQL databases (Oracle, MySQL, PostgreSQL, etc.)</li>
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
    "uniqueDatabases": @{length(triggerBody()?['uniqueDatabases'])},
    "totalUniqueItems": @{add(add(length(triggerBody()?['uniqueApplications']), length(triggerBody()?['uniqueSQLInstances'])), length(triggerBody()?['uniqueDatabases']))}
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
│ (Inputs: uniqueApplications, uniqueSQLInstances, uniqueDatabases,       │
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
│ FOR EACH Application│ │ FOR EACH SQL Instance│ │ FOR EACH Database  │
│   Add row to        │ │   Add row to         │ │   Add row to       │
│   UniqueApplications│ │   UniqueSQLInstances │ │   UniqueDatabases  │
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

5. **Sheet 3 - Databases**:
   - Add another new sheet
   - Rename to: `Databases`
   - Add headers: `MachineName`, `DatabaseType`, `Version`, `MachineManagerFqdn`
   - Convert to table named: `UniqueDatabases`

6. **Save the template**:
   - Save as: `ReportTemplate.xlsx`
   - Upload to SharePoint: `/Templates/ReportTemplate.xlsx`

---

## Orchestrating the Agents

The Orchestrator is the central Power Automate flow that coordinates all the processing agents. It receives file uploads, calls each processor in sequence, merges results, and triggers report generation.

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

#### Step 2.3: Initialize allDatabases Array

1. Click **+** → **Add an action** → **Initialize variable**
2. Configure:
   - **Name**: `allDatabases`
   - **Type**: **Array**
   - **Value**: `[]`
3. Rename to: `Initialize allDatabases`

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
   - **URI**: Paste the URL from your "Process Application Inventory" flow
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
   - **URI**: Paste the URL from your "Process SQL Server Inventory" flow
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

### Step 6: Call Database Processor

#### Step 6.1: Add HTTP Action

1. Inside the same Scope, click **Add an action**
2. Select **HTTP**
3. Configure:
   - **Method**: **POST**
   - **URI**: Paste the URL from your "Process Database Inventory" flow
   - **Headers**: `Content-Type`: `application/json`
   - **Body**:
   ```json
   {
     "filePath": "@{variables('currentFileUrl')}",
     "sessionId": "@{triggerBody()?['sessionId']}",
     "storageType": "@{triggerBody()?['storageType']}"
   }
   ```
4. Rename to: `Call Database Processor`

#### Step 6.2: Parse Response

1. Click **Add an action** → **Parse JSON**
2. Configure appropriately
3. Rename to: `Parse Database Processor Response`

#### Step 6.3: Merge Database Results

1. Click **Add an action** → **Compose**
2. Configure:
   - **Inputs**: Expression:
   ```
   union(variables('allDatabases'), body('Parse_Database_Processor_Response')?['uniqueDatabases'])
   ```
3. Rename to: `Merge Database Results`

4. Click **Add an action** → **Set variable**
5. Configure:
   - **Name**: `allDatabases`
   - **Value**: **Outputs** from "Merge Database Results"
6. Rename to: `Update allDatabases`

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
   or(greater(length(variables('allApplications')), 0), or(greater(length(variables('allSQLInstances')), 0), greater(length(variables('allDatabases')), 0)))
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
   - **URI**: Paste the URL from your "Generate Consolidated Report" flow
   - **Headers**: `Content-Type`: `application/json`
   - **Body**:
   ```json
   {
     "uniqueApplications": @{variables('allApplications')},
     "uniqueSQLInstances": @{variables('allSQLInstances')},
     "uniqueDatabases": @{variables('allDatabases')},
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
       "uniqueDatabases": @{length(variables('allDatabases'))},
       "totalUniqueItems": @{add(add(length(variables('allApplications')), length(variables('allSQLInstances'))), length(variables('allDatabases')))}
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
     "message": "No valid data found in uploaded files. Please verify the files contain ApplicationInventory, SQL Server, and/or Database sheets.",
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
│ • allDatabases (Array: [])                                              │
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
│ │   7. Call Database Processor (HTTP)                                 │ │
│ │   8. Parse DB Response                                              │ │
│ │   9. Merge & Update allDatabases                                    │ │
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
│  OR allDatabases.length > 0)                                           │
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
| `Azure Migrate Processing Orchestrator` | Parent/Main | Coordinates all processing | Both |
| `Handle File Upload (Blob Storage)` | Handler | Saves files to Azure Blob Storage | Option A |
| `Handle File Upload (SharePoint)` | Handler | Saves files to SharePoint | Option B |
| `Process Application Inventory` | Child | Consolidates applications | Both |
| `Process SQL Server Inventory` | Child | Consolidates SQL instances | Both |
| `Process Database Inventory` | Child | Consolidates databases | Both |
| `Generate Consolidated Report` | Child | Creates Excel and download link | Both |
| `Get Processing Status` | Utility | Checks processing status | Both |

### Flow 1A: File Upload Handler - Azure Blob Storage (Recommended)

This flow stores uploaded files in Azure Blob Storage, which does NOT require users to have SharePoint or OneDrive access. This is the recommended approach for temporary file storage.

```
Name: Handle File Upload (Blob Storage)
Trigger: Run a flow from Copilot

Steps:
1. Parse file upload data from Copilot
2. Generate unique session ID
3. Save uploaded files to Azure Blob Storage container
4. Call Orchestrator flow with blob URLs
5. Return status to Copilot agent
```

**Flow Definition - Azure Blob Storage:**

```
Trigger: Run a flow from Copilot

Actions:
1. Compose - Generate Session ID
   Expression: guid()

2. Initialize variable - uploadedBlobUrls
   Name: uploadedBlobUrls
   Type: Array
   Value: []

3. Apply to each uploaded file:
   
   a. Create blob (V2) - Azure Blob Storage
      Connection: Your Azure Blob Storage connection
      Storage account name: <your-storage-account-name>
      Container name: uploads
      Blob name: @{outputs('Generate_Session_ID')}/@{item()?['name']}
      Blob content: @{item()?['contentBytes']}
   
   b. Append to array variable
      Name: uploadedBlobUrls
      Value: @{body('Create_blob_(V2)')?['Path']}

4. HTTP - Call Orchestrator Flow
   Method: POST
   URI: (Orchestrator flow HTTP trigger URL)
   Headers:
     Content-Type: application/json
   Body: {
     "fileUrls": @{variables('uploadedBlobUrls')},
     "sessionId": "@{outputs('Generate_Session_ID')}",
     "userEmail": "@{triggerBody()?['user']?['email']}",
     "userId": "@{triggerBody()?['user']?['id']}",
     "storageType": "AzureBlob"
   }

5. Respond to the agent:
   - sessionId: @{outputs('Generate_Session_ID')}
   - status: "Processing"
   - message: "Files uploaded successfully to temporary storage. Processing has started."
```

**Azure Blob Storage Connection Setup:**
1. In Power Automate, add a new connection for **Azure Blob Storage**
2. Choose authentication method:
   - **Access Key**: Use storage account access key
   - **SAS Token**: Use shared access signature for limited access
   - **Azure AD**: Use Azure Active Directory for managed identity
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
Trigger: Run a flow from Copilot

Steps:
1. Parse file upload data from Copilot
2. Create session folder in SharePoint
3. Save uploaded files to SharePoint
4. Call Orchestrator flow with file URLs
5. Return status to Copilot agent
```

**Flow Definition - SharePoint:**

```
Trigger: Run a flow from Copilot

Actions:
1. Compose - Generate Session ID
   Expression: guid()

2. Create folder (SharePoint)
   Site: Your SharePoint Site
   Folder Path: /Uploads/@{outputs('Generate_Session_ID')}

3. Apply to each uploaded file:
   - Create file (SharePoint)
     Site: Your Site
     Folder: /Uploads/@{outputs('Generate_Session_ID')}
     File Name: @{item()?['name']}
     Content: @{item()?['contentBytes']}
   
   - Append file URL to array

4. HTTP - Call Orchestrator Flow
   Method: POST
   URI: (Orchestrator flow HTTP trigger URL)
   Body: {
     "fileUrls": @{variables('uploadedFileUrls')},
     "sessionId": "@{outputs('Generate_Session_ID')}",
     "userEmail": "@{triggerBody()?['user']?['email']}",
     "userId": "@{triggerBody()?['user']?['id']}",
     "storageType": "SharePoint"
   }

5. Respond to the agent:
   - sessionId: @{outputs('Generate_Session_ID')}
   - status: "Processing"
   - message: "Files uploaded successfully. Processing has started."
```

### Flow 2: Get Processing Status

This flow checks for completed reports and returns the download URL. Configure based on your storage choice:

#### Option A: Get Processing Status - Azure Blob Storage

```
Name: Get Processing Status (Blob Storage)
Trigger: Run a flow from Copilot

Steps:
1. Receive sessionId from Copilot
2. List blobs in reports container for the session
3. Generate SAS URL for download if report exists
4. Return status and download URL
```

**Flow Definition - Azure Blob Storage:**

```
Trigger: Run a flow from Copilot

Actions:
1. List blobs (V2) - Azure Blob Storage
   Storage account: <your-storage-account-name>
   Container: reports
   Prefix: @{triggerBody()?['sessionId']}/

2. Filter array - Find Excel files
   From: @{body('List_blobs_(V2)')?['value']}
   Condition: endsWith(item()?['Name'], '.xlsx')

3. Condition: Report exists?
   If: length(body('Filter_array')) > 0
   
   Yes:
     4. Compose - Get first blob name
        @{first(body('Filter_array'))?['Name']}
     
     5. Get blob content using path (V2) - Azure Blob Storage
        Container: reports
        Blob: @{outputs('Get_first_blob_name')}
     
     6. Compose - Generate SAS URL (or use Azure Function for SAS generation)
        Expression: Create a SAS token with read permission, valid for 24 hours
     
     7. Return to Copilot:
        - status: "Complete"
        - downloadUrl: @{outputs('Generate_SAS_URL')}
        - message: "Your report is ready for download!"
   
   No:
     8. Return to Copilot:
        - status: "Processing"
        - downloadUrl: ""
        - message: "Your files are still being processed. Please check back shortly."
```

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
Trigger: Run a flow from Copilot

Steps:
1. Receive userId/sessionId from Copilot
2. Check for completed report in SharePoint
3. Return status and download URL if complete
```

**Flow Definition - SharePoint:**

```
Trigger: Run a flow from Copilot

Actions:
1. Get files (properties only) - SharePoint
   Site: Your Site
   Library: Reports
   Folder: /@{triggerBody()?['sessionId']}
   Filter: endswith(Name, '.xlsx')

2. Condition: Files exist?
   If: length(body('Get_files')?['value']) > 0
   
   Yes:
     3. Create sharing link for a file
        File: First file in results
     
     4. Return to Copilot:
        - status: "Complete"
        - downloadUrl: @{body('Create_sharing_link')?['link']?['webUrl']}
   
   No:
     5. Return to Copilot:
        - status: "Processing"
        - downloadUrl: ""
```

### Connecting Flows to Copilot Studio

1. In Copilot Studio, go to your agent
2. Navigate to **Actions** → **"+ Add an action"**
3. Select **"Create a flow"** or **"Add existing flow"**
4. Map the flow inputs/outputs to Copilot variables:

**For File Upload Action:**
```
Input Mapping:
  - uploadedFiles → (From Copilot file upload question)
  - user → (System variable: User)

Output Mapping:
  - sessionId → Global.sessionId
  - status → Global.uploadStatus
  - message → Topic.statusMessage
```

**For Status Check Action:**
```
Input Mapping:
  - sessionId → Global.sessionId

Output Mapping:
  - status → Global.processingStatus
  - downloadUrl → Global.downloadUrl
```

---

## Testing and Validation

### Test Plan

#### Phase 1: Unit Testing (Individual Flows)

1. **Test Application Inventory Processor**
   - [ ] Upload CSV with 100 applications
   - [ ] Verify duplicates are removed
   - [ ] Verify noise filtering works
   - [ ] Check output format

2. **Test SQL Server Processor**
   - [ ] Upload CSV with SQL instances
   - [ ] Verify instance consolidation
   - [ ] Check default instance handling

3. **Test Database Processor**
   - [ ] Upload CSV with various database types
   - [ ] Verify type normalization
   - [ ] Check duplicate handling

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

**test_database_inventory.csv:**
```csv
MachineName,Database Type,Version,MachineManagerFqdn
DBServer01,Oracle,19c,manager.contoso.com
DBServer02,MySQL,8.0.28,manager.contoso.com
DBServer03,PostgreSQL,14.1,manager.contoso.com
DBServer01,Oracle,19c,manager.contoso.com
DBServer04,MongoDB,5.0.6,manager.contoso.com
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

**Unique Databases (4 expected):**
- DBServer01/Oracle
- DBServer02/MySQL
- DBServer03/PostgreSQL
- DBServer04/MongoDB

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

### Error Logging

Add error logging to your flows:

```
Scope: Error Handling
  - Create item (SharePoint List: ErrorLog)
    - FlowName: @{workflow().name}
    - ErrorMessage: @{actions('FailedAction')?['error']?['message']}
    - Timestamp: @{utcNow()}
    - SessionId: @{variables('sessionId')}
    - InputData: @{string(triggerBody())}
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

### Version Comparison Logic

```
For applications with same name but different versions:
1. Parse version strings
2. Compare major.minor.build.revision
3. Keep entry with highest version
4. If versions equal, keep first occurrence
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
