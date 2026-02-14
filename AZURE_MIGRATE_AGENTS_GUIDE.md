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
│  • Stores files in SharePoint/OneDrive                                      │
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
│  • Store in SharePoint/OneDrive                                             │
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
| App Inventory Processor | Consolidate applications | Power Automate + Excel Connector |
| SQL Server Processor | Consolidate SQL instances | Power Automate + Excel Connector |
| Database Processor | Consolidate other databases | Power Automate + Excel Connector |
| Report Generator | Create final spreadsheet | Power Automate + Excel Connector |
| Orchestrator | Coordinate agent workflow | Power Automate Flow |

---

## Prerequisites

### Required Access

Before building these agents, ensure you have:

1. **Microsoft 365 License** with:
   - Microsoft Copilot Studio access
   - Power Automate access
   - SharePoint or OneDrive access
   - Microsoft Excel Online

2. **Permissions**:
   - Copilot Studio environment creator/maker
   - Power Automate flow creator
   - SharePoint site contributor (for file storage)

3. **Environment Setup**:
   - A dedicated SharePoint site or OneDrive folder for file storage
   - Azure AD application (optional, for advanced authentication)

### Prepare SharePoint Storage

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
This agent handles the initial user interaction, accepts CSV file uploads, validates them, and triggers the processing workflow.

### Step 1: Create the Agent

1. Navigate to [https://copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Click **"+ Create"** → **"New copilot"**
3. Configure the agent:
   ```
   Name: Azure Migrate File Handler
   Description: Handles Azure Migrate CSV file uploads and initiates processing
   Language: English
   ```
4. Click **"Create"**

### Step 2: Configure Topics

#### Topic 1: Greeting and Instructions

1. Go to **Topics** → Click **"+ Add a topic"** → **"From blank"**
2. Configure the topic:
   ```
   Name: Welcome and Upload Instructions
   ```

3. **Add Trigger Phrases**:
   ```
   - Start
   - Hello
   - Upload files
   - Process Azure Migrate files
   - I have Azure Migrate export files
   - Help me consolidate applications
   - I need to upload CSV files
   - Azure Migrate CSV processing
   - Consolidate application inventory
   - Process migration assessment
   ```

4. **Build the Conversation Flow**:

   **Node 1: Message**
   ```
   Welcome! I'm your Azure Migrate CSV Processor. I can help you:

   📁 Upload Azure Migrate export CSV files
   📊 Consolidate application inventories
   💾 Process SQL Server and database inventories
   📥 Generate a downloadable consolidated report

   Let's get started! Please upload your Azure Migrate CSV file(s).
   ```

   **Node 2: Question (File Upload)**
   ```
   Question: Please upload your Azure Migrate export file(s). You can upload one or more CSV/Excel files.
   Identify: File upload
   Save response as: uploadedFiles (File type)
   ```

   **Node 3: Condition - Check File Upload**
   ```
   Condition: If uploadedFiles is not empty
   ```

   **Node 4: Action - Call Power Automate Flow**
   ```
   Action: Trigger file processing flow
   Input: uploadedFiles
   Input: User.Id (for session tracking)
   ```

   **Node 5: Message - Processing Started**
   ```
   ✅ Files received successfully!

   I'm now processing your Azure Migrate data. This includes:
   1. Parsing Application Inventory sheet
   2. Processing SQL Server instances
   3. Consolidating database inventory
   4. Generating your report

   This may take a few moments. I'll notify you when the consolidated report is ready for download.
   ```

#### Topic 2: Check Processing Status

1. Create another topic:
   ```
   Name: Check Processing Status
   ```

2. **Add Trigger Phrases**:
   ```
   - Status
   - Check status
   - Is my report ready?
   - Processing status
   - How long will it take?
   - Where is my report?
   ```

3. **Build the Conversation Flow**:

   **Node 1: Action - Check Status Flow**
   ```
   Action: Call Get Processing Status flow
   Input: User.Id
   Output: processingStatus, downloadUrl
   ```

   **Node 2: Condition - Status Check**
   ```
   Condition: If processingStatus equals "Complete"
   ```

   **Node 3 (If Complete): Message**
   ```
   🎉 Great news! Your consolidated report is ready!

   📥 Download your report: {downloadUrl}

   The report includes:
   • Unique Applications Sheet
   • SQL Server Inventory Sheet
   • Database Inventory Sheet

   Would you like to process another set of files?
   ```

   **Node 4 (If Processing): Message**
   ```
   ⏳ Your files are still being processed.

   Current Status: {processingStatus}

   Please check back in a few moments, or I'll notify you when it's complete.
   ```

#### Topic 3: Help and Information

1. Create topic:
   ```
   Name: Help and Supported Formats
   ```

2. **Add Trigger Phrases**:
   ```
   - Help
   - What files do you support?
   - File format
   - How does this work?
   - What is Azure Migrate export?
   - Supported formats
   ```

3. **Build the Conversation Flow**:

   **Node 1: Message**
   ```
   📚 Help & File Format Information

   **Supported File Formats:**
   • CSV files exported from Azure Migrate
   • Excel files (.xlsx) with multiple sheets

   **Expected Sheets:**
   1. **ApplicationInventory** - Contains: MachineName, Application, Version, Provider, MachineManagerFqdn
   2. **SQL Server** - Contains: MachineName, Instance Name, Edition, Service Pack, Version, Port, MachineManagerFqdn
   3. **Database** - Contains: MachineName, Database Type, Version, MachineManagerFqdn

   **What I Do:**
   • Remove duplicate entries
   • Filter out noise (patches, variants, dependencies)
   • Create a consolidated unique list
   • Generate a downloadable Excel report

   Ready to upload your files?
   ```

### Step 3: Configure the File Upload Action

1. Go to **Actions** → **"+ Add an action"**
2. Select **"Create a new flow"**
3. This opens Power Automate - configure the flow (detailed in [Power Automate Flows](#power-automate-flows) section)

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
This agent (implemented primarily as a Power Automate flow) processes the ApplicationInventory sheet to consolidate applications and remove duplicates.

### Step 1: Create Processing Flow in Power Automate

1. Navigate to [https://make.powerautomate.com](https://make.powerautomate.com)
2. Click **"+ Create"** → **"Automated cloud flow"**
3. Configure:
   ```
   Flow Name: Process Application Inventory
   Trigger: When an HTTP request is received (or called from parent flow)
   ```

### Step 2: Define Flow Variables

Add **Initialize variable** actions:

```
Variable 1:
  Name: uniqueApplications
  Type: Array
  Value: []

Variable 2:
  Name: processedApps
  Type: Object
  Value: {}

Variable 3:
  Name: noisePatterns
  Type: Array
  Value: [
    "Update",
    "Patch",
    "Hotfix",
    "KB",
    "Security Update",
    "Service Pack",
    "Redistributable",
    "Runtime",
    ".NET Framework",
    "Visual C++",
    "Microsoft Visual",
    "Microsoft Update"
  ]
```

### Step 3: Parse CSV/Excel Data

**Action: List rows present in a table (Excel Online)**
```
Location: SharePoint Site
Document Library: Uploads
File: @{triggerBody()?['filePath']}
Table Name: ApplicationInventory
```

### Step 4: Consolidation Logic

Add **Apply to each** loop over Excel rows:

```
Apply to each: @{body('List_rows_present_in_a_table')?['value']}

Inside loop:

1. Compose - Normalize Application Name:
   Expression: toLower(trim(items('Apply_to_each')?['Application']))

2. Compose - Check if Noise:
   Expression: 
   if(
     or(
       contains(toLower(items('Apply_to_each')?['Application']), 'update'),
       contains(toLower(items('Apply_to_each')?['Application']), 'patch'),
       contains(toLower(items('Apply_to_each')?['Application']), 'hotfix'),
       contains(toLower(items('Apply_to_each')?['Application']), 'kb'),
       contains(toLower(items('Apply_to_each')?['Application']), 'redistributable')
     ),
     true,
     false
   )

3. Condition - Skip Noise:
   If: outputs('Check_if_Noise') equals false
   
   Yes Branch:
     4. Compose - Create App Key:
        Expression: 
        concat(
          toLower(trim(items('Apply_to_each')?['Application'])),
          '_',
          toLower(trim(items('Apply_to_each')?['Provider']))
        )
     
     5. Condition - Check Duplicate:
        If: not(contains(variables('processedApps'), outputs('Create_App_Key')))
        
        Yes Branch:
          6. Append to array variable:
             Name: uniqueApplications
             Value: {
               "Application": "@{items('Apply_to_each')?['Application']}",
               "Version": "@{items('Apply_to_each')?['Version']}",
               "Provider": "@{items('Apply_to_each')?['Provider']}",
               "MachineCount": 1
             }
          
          7. Set variable:
             Name: processedApps
             Value: @{addProperty(variables('processedApps'), outputs('Create_App_Key'), true)}
```

### Step 5: Output Results

**Action: Compose - Final Output**
```
{
  "uniqueApplications": @{variables('uniqueApplications')},
  "totalUnique": @{length(variables('uniqueApplications'))},
  "totalProcessed": @{length(body('List_rows_present_in_a_table')?['value'])},
  "duplicatesRemoved": @{sub(length(body('List_rows_present_in_a_table')?['value']), length(variables('uniqueApplications')))}
}
```

**Action: Response (HTTP Response)**
```
Status Code: 200
Body: @{outputs('Final_Output')}
```

### Consolidation Rules

The Application Inventory Processor applies these consolidation rules:

| Rule | Description | Example |
|------|-------------|---------|
| **Exact Duplicate** | Same Application + Provider | Remove second occurrence |
| **Version Variant** | Same Application, different version | Keep latest version only |
| **Update/Patch** | Contains "Update", "Patch", "KB" | Filter out as noise |
| **Redistributable** | Runtime or library dependency | Filter out as noise |
| **Case Normalization** | Different casing | Treat as same application |

---

## Agent 3: SQL Server Inventory Processor

### Purpose
Process SQL Server inventory sheets to create a unique list of SQL Server instances.

### Step 1: Create Processing Flow

1. In Power Automate, create a new flow:
   ```
   Flow Name: Process SQL Server Inventory
   Trigger: When called from a flow (child flow)
   ```

### Step 2: Define Variables

```
Variable 1:
  Name: uniqueSQLInstances
  Type: Array
  Value: []

Variable 2:
  Name: processedInstances
  Type: Object
  Value: {}
```

### Step 3: Parse SQL Server Sheet

**Action: List rows present in a table (Excel Online)**
```
Location: SharePoint Site
Document Library: Uploads
File: @{triggerBody()?['filePath']}
Table Name: SQLServer (or sheet containing SQL data)
```

### Step 4: Consolidation Logic

```
Apply to each: @{body('List_SQL_rows')?['value']}

Inside loop:

1. Compose - Create Instance Key:
   Expression:
   concat(
     toLower(trim(items('Apply_to_each')?['MachineName'])),
     '_',
     toLower(trim(coalesce(items('Apply_to_each')?['Instance Name'], 'MSSQLSERVER')))
   )

2. Condition - Check Duplicate:
   If: not(contains(variables('processedInstances'), outputs('Create_Instance_Key')))
   
   Yes Branch:
     3. Append to array variable:
        Name: uniqueSQLInstances
        Value: {
          "MachineName": "@{items('Apply_to_each')?['MachineName']}",
          "InstanceName": "@{coalesce(items('Apply_to_each')?['Instance Name'], 'MSSQLSERVER')}",
          "Edition": "@{items('Apply_to_each')?['Edition']}",
          "ServicePack": "@{items('Apply_to_each')?['Service Pack']}",
          "Version": "@{items('Apply_to_each')?['Version']}",
          "Port": "@{items('Apply_to_each')?['Port']}"
        }
     
     4. Set variable:
        Name: processedInstances
        Value: @{addProperty(variables('processedInstances'), outputs('Create_Instance_Key'), true)}
```

### Step 5: Output Results

```
{
  "uniqueSQLInstances": @{variables('uniqueSQLInstances')},
  "totalUnique": @{length(variables('uniqueSQLInstances'))},
  "totalProcessed": @{length(body('List_SQL_rows')?['value'])}
}
```

### SQL Server Consolidation Rules

| Rule | Description |
|------|-------------|
| **Unique Instance** | MachineName + Instance Name combination |
| **Default Instance** | Treat empty Instance Name as "MSSQLSERVER" |
| **Keep Latest** | When duplicate, keep entry with newest version |

---

## Agent 4: Database Inventory Processor

### Purpose
Process database inventory sheets for non-SQL Server databases (Oracle, MySQL, PostgreSQL, etc.).

### Step 1: Create Processing Flow

1. In Power Automate, create a new flow:
   ```
   Flow Name: Process Database Inventory
   Trigger: When called from a flow (child flow)
   ```

### Step 2: Define Variables

```
Variable 1:
  Name: uniqueDatabases
  Type: Array
  Value: []

Variable 2:
  Name: processedDatabases
  Type: Object
  Value: {}
```

### Step 3: Parse Database Sheet

**Action: List rows present in a table (Excel Online)**
```
Location: SharePoint Site
Document Library: Uploads
File: @{triggerBody()?['filePath']}
Table Name: Database (or sheet containing database data)
```

### Step 4: Consolidation Logic

```
Apply to each: @{body('List_Database_rows')?['value']}

Inside loop:

1. Compose - Create Database Key:
   Expression:
   concat(
     toLower(trim(items('Apply_to_each')?['MachineName'])),
     '_',
     toLower(trim(items('Apply_to_each')?['Database Type']))
   )

2. Condition - Check Duplicate:
   If: not(contains(variables('processedDatabases'), outputs('Create_Database_Key')))
   
   Yes Branch:
     3. Append to array variable:
        Name: uniqueDatabases
        Value: {
          "MachineName": "@{items('Apply_to_each')?['MachineName']}",
          "DatabaseType": "@{items('Apply_to_each')?['Database Type']}",
          "Version": "@{items('Apply_to_each')?['Version']}",
          "MachineManagerFqdn": "@{items('Apply_to_each')?['MachineManagerFqdn']}"
        }
     
     4. Set variable:
        Name: processedDatabases
        Value: @{addProperty(variables('processedDatabases'), outputs('Create_Database_Key'), true)}
```

### Step 5: Output Results

```
{
  "uniqueDatabases": @{variables('uniqueDatabases')},
  "totalUnique": @{length(variables('uniqueDatabases'))},
  "totalProcessed": @{length(body('List_Database_rows')?['value'])}
}
```

### Database Consolidation Rules

| Rule | Description |
|------|-------------|
| **Unique Database** | MachineName + Database Type combination |
| **Version Handling** | Keep highest version for same machine/type |
| **Type Normalization** | Normalize database type names (e.g., "MySQL" = "mysql") |

---

## Agent 5: Report Generator

### Purpose
Generate the final consolidated Excel spreadsheet with all unique applications, SQL instances, and databases, and provide a download link to the user.

### Step 1: Create Report Generation Flow

1. In Power Automate, create a new flow:
   ```
   Flow Name: Generate Consolidated Report
   Trigger: When called from a flow
   ```

### Step 2: Flow Inputs

Configure inputs from the orchestrator:
```
Input 1: uniqueApplications (Array)
Input 2: uniqueSQLInstances (Array)
Input 3: uniqueDatabases (Array)
Input 4: sessionId (String)
Input 5: userEmail (String)
```

### Step 3: Create Excel File

**Action 1: Create file (SharePoint)**
```
Site Address: Your SharePoint Site
Folder Path: /Reports/@{triggerBody()?['sessionId']}
File Name: ConsolidatedReport_@{formatDateTime(utcNow(), 'yyyyMMdd_HHmmss')}.xlsx
File Content: (Use template or create dynamically)
```

### Step 4: Create Tables and Add Data

**For Applications Sheet:**

**Action 2: Create table (Excel Online)**
```
Location: SharePoint Site
Document Library: Reports
File: (Previous action output)
Table Name: UniqueApplications
Table Range: A1:D1
Column Names: Application, Version, Provider, MachineCount
```

**Action 3: Add rows to table (Excel Online) - Apply to each**
```
Apply to each: @{triggerBody()?['uniqueApplications']}

Action: Add a row into a table
Location: SharePoint Site
File: (Report file)
Table: UniqueApplications
Row: {
  "Application": "@{items('Apply_to_each')?['Application']}",
  "Version": "@{items('Apply_to_each')?['Version']}",
  "Provider": "@{items('Apply_to_each')?['Provider']}",
  "MachineCount": "@{items('Apply_to_each')?['MachineCount']}"
}
```

**For SQL Server Sheet:**

**Action 4: Create worksheet (Excel Online)**
```
File: (Report file)
Name: SQLServerInventory
```

**Action 5: Create table and add rows** (similar to applications)
```
Table Name: UniqueSQLInstances
Columns: MachineName, InstanceName, Edition, ServicePack, Version, Port
```

**For Databases Sheet:**

**Action 6: Create worksheet (Excel Online)**
```
File: (Report file)
Name: DatabaseInventory
```

**Action 7: Create table and add rows** (similar pattern)
```
Table Name: UniqueDatabases
Columns: MachineName, DatabaseType, Version, MachineManagerFqdn
```

### Step 5: Generate Download Link

**Action 8: Create sharing link for a file or folder (SharePoint)**
```
Site Address: SharePoint Site
Item Path: /Reports/@{triggerBody()?['sessionId']}/(filename)
Link Type: View
Link Scope: Anyone with the link (or organization-appropriate setting)
```

### Step 6: Return Results

**Action 9: Response**
```
{
  "status": "Complete",
  "reportFileName": "ConsolidatedReport_@{formatDateTime(utcNow(), 'yyyyMMdd_HHmmss')}.xlsx",
  "downloadUrl": "@{body('Create_sharing_link')?['link']?['webUrl']}",
  "statistics": {
    "uniqueApplications": @{length(triggerBody()?['uniqueApplications'])},
    "uniqueSQLInstances": @{length(triggerBody()?['uniqueSQLInstances'])},
    "uniqueDatabases": @{length(triggerBody()?['uniqueDatabases'])}
  }
}
```

### Step 7: Notify User (Optional)

**Action 10: Send an email (V2) - Office 365 Outlook**
```
To: @{triggerBody()?['userEmail']}
Subject: Your Azure Migrate Consolidated Report is Ready
Body:
  Hello,

  Your Azure Migrate data has been processed successfully!

  **Summary:**
  - Unique Applications: @{length(triggerBody()?['uniqueApplications'])}
  - SQL Server Instances: @{length(triggerBody()?['uniqueSQLInstances'])}
  - Database Instances: @{length(triggerBody()?['uniqueDatabases'])}

  📥 Download your report: @{body('Create_sharing_link')?['link']?['webUrl']}

  Thank you for using Azure Migrate CSV Processor!
```

---

## Orchestrating the Agents

### Master Orchestrator Flow

Create a main Power Automate flow that coordinates all processing agents.

### Step 1: Create Orchestrator Flow

1. In Power Automate, create:
   ```
   Flow Name: Azure Migrate Processing Orchestrator
   Trigger: When an HTTP request is received
   ```

### Step 2: Define Trigger Schema

```json
{
  "type": "object",
  "properties": {
    "fileUrls": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "sessionId": {
      "type": "string"
    },
    "userEmail": {
      "type": "string"
    },
    "userId": {
      "type": "string"
    }
  },
  "required": ["fileUrls", "sessionId"]
}
```

### Step 3: Orchestration Logic

```
1. Initialize Variables:
   - allApplications (Array): []
   - allSQLInstances (Array): []
   - allDatabases (Array): []
   - processingErrors (Array): []

2. Apply to each file in fileUrls:
   
   a. Try-Catch Scope: Process Single File
      
      i. Run child flow: Process Application Inventory
         Input: filePath = current file URL
         Output: Store in appResult
      
      ii. Compose: Merge Applications
          Union of allApplications and appResult.uniqueApplications
      
      iii. Set variable: allApplications = merged result
      
      iv. Run child flow: Process SQL Server Inventory
          Input: filePath = current file URL
          Output: Store in sqlResult
      
      v. Set variable: allSQLInstances = merged SQL results
      
      vi. Run child flow: Process Database Inventory
          Input: filePath = current file URL
          Output: Store in dbResult
      
      vii. Set variable: allDatabases = merged DB results
   
   b. Catch Scope: Handle Errors
      - Append error to processingErrors array
      - Continue to next file

3. Condition: Check if any data was processed
   If: length(allApplications) > 0 OR length(allSQLInstances) > 0 OR length(allDatabases) > 0
   
   Yes:
     4. Run child flow: Generate Consolidated Report
        Inputs:
        - uniqueApplications: allApplications
        - uniqueSQLInstances: allSQLInstances
        - uniqueDatabases: allDatabases
        - sessionId: triggerBody()?['sessionId']
        - userEmail: triggerBody()?['userEmail']
        
        Output: reportResult
     
     5. Response - Success
        {
          "status": "Complete",
          "downloadUrl": reportResult.downloadUrl,
          "statistics": {
            "filesProcessed": length(triggerBody()?['fileUrls']),
            "uniqueApplications": length(allApplications),
            "uniqueSQLInstances": length(allSQLInstances),
            "uniqueDatabases": length(allDatabases)
          },
          "errors": processingErrors
        }
   
   No:
     6. Response - No Data
        {
          "status": "Error",
          "message": "No valid data found in uploaded files",
          "errors": processingErrors
        }
```

### Step 4: Error Handling Configuration

Add a **Configure run after** setting for error handling:

```
Scope: Try-Catch for each processor
  - On Success: Continue
  - On Failure: Log error, continue to next
  - On Timeout: Log timeout, continue to next
```

### Orchestration Diagram

```
START
  │
  ▼
┌─────────────────────────────────────────┐
│ Receive File URLs from Copilot Agent    │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│ FOR EACH uploaded file:                 │
│   ├─ Call App Inventory Processor       │
│   ├─ Call SQL Server Processor          │
│   └─ Call Database Processor            │
│   Merge results into master arrays      │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│ Final Deduplication (across all files)  │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│ Call Report Generator                   │
│ - Create Excel with 3 sheets            │
│ - Generate download link                │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│ Return download URL to Copilot Agent    │
└─────────────────────────────────────────┘
  │
  ▼
END
```

---

## Power Automate Flows

### Summary of Required Flows

| Flow Name | Type | Purpose |
|-----------|------|---------|
| `Azure Migrate Processing Orchestrator` | Parent/Main | Coordinates all processing |
| `Process Application Inventory` | Child | Consolidates applications |
| `Process SQL Server Inventory` | Child | Consolidates SQL instances |
| `Process Database Inventory` | Child | Consolidates databases |
| `Generate Consolidated Report` | Child | Creates Excel and download link |
| `Get Processing Status` | Utility | Checks processing status |

### Flow 1: File Upload Handler Action

This flow is called directly from the Copilot Studio agent when files are uploaded.

```
Name: Handle File Upload
Trigger: Power Virtual Agents (Copilot Studio)

Steps:
1. Parse file upload data from Copilot
2. Create session folder in SharePoint
3. Save uploaded files to SharePoint
4. Call Orchestrator flow with file URLs
5. Return status to Copilot agent
```

**Flow Definition:**

```
Trigger: When Power Virtual Agents calls a flow

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
     "userId": "@{triggerBody()?['user']?['id']}"
   }

5. Return value(s) to Power Virtual Agents:
   - sessionId: @{outputs('Generate_Session_ID')}
   - status: "Processing"
   - message: "Files uploaded successfully. Processing has started."
```

### Flow 2: Get Processing Status

```
Name: Get Processing Status
Trigger: Power Virtual Agents

Steps:
1. Receive userId/sessionId from Copilot
2. Check for completed report in SharePoint
3. Return status and download URL if complete
```

**Flow Definition:**

```
Trigger: When Power Virtual Agents calls a flow

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
  - status → Topic.uploadStatus
  - message → Topic.statusMessage
```

**For Status Check Action:**
```
Input Mapping:
  - sessionId → Global.sessionId

Output Mapping:
  - status → Topic.processingStatus
  - downloadUrl → Topic.downloadUrl
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

**Solutions:**
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

**Solutions:**
1. Verify sharing link settings
2. Check link expiration
3. Ensure user has SharePoint access
4. Use organization-wide sharing if appropriate

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
- [SharePoint Connectors](https://learn.microsoft.com/en-us/connectors/sharepointonline/)
- [Excel Online Connector](https://learn.microsoft.com/en-us/connectors/excelonlinebusiness/)

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
| **CSV** | Comma-Separated Values file format |
| **Copilot Studio** | Microsoft's no-code platform for building conversational AI agents |
| **Power Automate** | Microsoft's workflow automation platform |
| **SharePoint** | Microsoft's document management and collaboration platform |
| **Consolidation** | Process of combining and deduplicating data |
| **FQDN** | Fully Qualified Domain Name |
| **Child Flow** | A Power Automate flow called from another flow |
| **Orchestrator** | The main flow that coordinates other flows |

---

**Document Version**: 1.0  
**Last Updated**: February 2026  
**Author**: AZMrepo Project Team

---

*For questions or issues with this guide, please refer to the troubleshooting section or contact the project team.*
