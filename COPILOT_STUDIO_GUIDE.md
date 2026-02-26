# How to Build a Copilot Studio Agent

This guide provides step-by-step instructions for creating and configuring a Microsoft Copilot Studio agent.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Getting Started - Portal Navigation](#getting-started---portal-navigation)
   - [Accessing the Portal](#step-1-access-copilot-studio-portal)
   - [Understanding the Interface](#step-2-understanding-the-portal-interface)
   - [Navigation Reference](#portal-navigation-reference)
3. [Creating Your First Agent](#creating-your-first-agent)
   - [Starting Agent Creation](#step-3-start-creating-a-new-agent)
   - [Agent Creation Wizard](#step-4-complete-the-agent-creation-wizard)
   - [Initial Agent Configuration](#step-5-initial-agent-configuration)
4. [Configuring Agent Capabilities](#configuring-agent-capabilities)
   - [Adding Knowledge Sources](#step-6-add-knowledge-sources)
   - [Defining Topics](#step-7-define-topics)
   - [Creating Tools](#step-8-create-tools-and-integrations)
   - [Configuring Entities](#step-9-configure-entities)
5. [Testing Your Agent](#testing-your-agent)
6. [Publishing Your Agent](#publishing-your-agent)
7. [Best Practices](#best-practices)
8. [Advanced Features](#advanced-features)
9. [Troubleshooting Common Issues](#troubleshooting-common-issues)
10. [Resources](#resources)

## Prerequisites

Before you begin building your Copilot Studio agent, ensure you have:

- **Microsoft Account**: A valid Microsoft 365 account or Azure subscription
- **Copilot Studio Access**: Access to Microsoft Copilot Studio (formerly Power Virtual Agents)
- **Permissions**: Appropriate permissions to create and publish agents in your organization
- **Browser**: A modern web browser (Microsoft Edge, Chrome, Firefox, or Safari)
- **Optional**: Basic understanding of conversational AI concepts

---

## Getting Started - Portal Navigation

This section provides detailed step-by-step instructions for navigating the Copilot Studio portal.

### Step 1: Access Copilot Studio Portal

**Opening the Portal:**

1. **Open your web browser** (Microsoft Edge recommended for best experience)

2. **Navigate to Copilot Studio:**
   - Type `https://copilotstudio.microsoft.com` in the address bar
   - Press **Enter** to load the page

3. **Sign in with your Microsoft 365 credentials:**
   - On the sign-in page, enter your **work email address** in the email field
   - Click the **"Next"** button (blue button on the right)
   - Enter your **password** in the password field
   - Click **"Sign in"** button
   - If prompted for multi-factor authentication (MFA), complete the verification

4. **Select your environment** (if prompted):
   - You will see a dropdown menu labeled **"Environment"** in the top-right corner of the header bar
   - Click the dropdown to see available environments (e.g., "Default", "Production", "Development")
   - Select the appropriate environment for your agent
   - **Note**: If you don't see environment options, your organization may have a default environment configured

5. **Wait for the dashboard to load:**
   - The main Copilot Studio dashboard will appear
   - You'll see the **Home** page with options to create new agents or access existing ones

### Step 2: Understanding the Portal Interface

**Main Portal Layout:**

The Copilot Studio portal has the following main areas:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  HEADER BAR                                                                  │
│  ┌──────────────┬─────────────────────────────────────┬──────────────────┐  │
│  │ Microsoft    │           Search Bar                │ Environment ▼   │  │
│  │ Copilot      │                                     │ User Account    │  │
│  │ Studio Logo  │                                     │ Settings ⚙️     │  │
│  └──────────────┴─────────────────────────────────────┴──────────────────┘  │
├───────────────────┬─────────────────────────────────────────────────────────┤
│  LEFT NAVIGATION  │                MAIN CONTENT AREA                        │
│  (Side Panel)     │                                                         │
│  ┌───────────────┐│  ┌─────────────────────────────────────────────────┐   │
│  │ 🏠 Home       ││  │                                                 │   │
│  │ 🤖 Copilots   ││  │   Dashboard / Agent Editor / Configuration     │   │
│  │ 📚 Knowledge  ││  │                                                 │   │
│  │ 💬 Topics     ││  │                                                 │   │
│  │ 📊 Entities   ││  │                                                 │   │
│  │ ⚡ Tools      ││  │                                                 │   │
│  │ 📈 Analytics  ││  │                                                 │   │
│  │ 🚀 Publish    ││  │                                                 │   │
│  │ ⚙️ Settings   ││  │                                                 │   │
│  └───────────────┘│  └─────────────────────────────────────────────────┘   │
│                   │                                                         │
│                   │  ┌─────────────────────────────────────────────────┐   │
│                   │  │           TEST BOT PANEL (Bottom Right)         │   │
│                   │  │           (Collapsible chat interface)          │   │
│                   │  └─────────────────────────────────────────────────┘   │
└───────────────────┴─────────────────────────────────────────────────────────┘
```

**Navigation Panel Elements (Left Side):**

| Icon | Menu Item | Location | Description |
|------|-----------|----------|-------------|
| 🏠 | **Home** | Top of left panel | Returns to main dashboard; shows recent agents and quick actions |
| 🤖 | **Copilots** | Below Home | Lists all your agents; click to view/manage agents |
| 📚 | **Knowledge** | Below Copilots | Manage knowledge sources (appears when editing an agent) |
| 💬 | **Topics** | Below Knowledge | Create and manage conversation topics (appears when editing an agent) |
| 📊 | **Entities** | Below Topics | Define custom entities for data extraction (appears when editing an agent) |
| ⚡ | **Tools** | Below Entities | Configure Power Automate flows, connectors, and prompts (appears when editing an agent) |
| 📈 | **Analytics** | Below Tools | View performance metrics and usage statistics |
| 🚀 | **Publish** | Below Analytics | Deploy your agent to channels |
| ⚙️ | **Settings** | Bottom of left panel | Configure agent settings, authentication, and integrations |

**Header Bar Elements:**

| Element | Location | Description |
|---------|----------|-------------|
| **Microsoft Copilot Studio Logo** | Top-left corner | Click to return to Home from anywhere |
| **Search Bar** | Top-center | Search for agents, topics, or help content |
| **Environment Dropdown** | Top-right area | Switch between environments (Dev/Test/Prod) |
| **User Account Icon** | Top-right corner | Access account settings, sign out |
| **Settings Gear** | Top-right corner | Access global portal settings |

### Portal Navigation Reference

**Quick Navigation Paths:**

Use these navigation paths to quickly find common features:

| To Access | Navigation Path |
|-----------|-----------------|
| Create New Agent | Home → **"+ Create"** button (top-center) |
| View All Agents | Left Panel → **Copilots** |
| Agent Settings | Left Panel → **Settings** → Select settings category |
| Add Knowledge | Left Panel → **Knowledge** → **"+ Add knowledge"** |
| Create Topic | Left Panel → **Topics** → **"+ Add a topic"** |
| Add Tool | Left Panel → **Tools** → **"Add a tool"** |
| Test Agent | Click **"Test"** button (bottom-left corner) |
| Publish Agent | Left Panel → **Publish** OR Top-right **"Publish"** button |
| View Analytics | Left Panel → **Analytics** |

---

## Creating Your First Agent

This section walks you through creating a new agent with detailed portal navigation.

### Step 3: Start Creating a New Agent

**Initiating Agent Creation:**

1. **From the Home page**, locate the **"+ Create"** button:
   - The button is in the **top-center area** of the Home page
   - It may appear as a **purple/blue button** labeled **"+ Create"** or **"+ New copilot"**
   - **Alternative**: Click **"Copilots"** in the left navigation → then click **"+ Create"** in the top-right

2. **Click the "Create" button** - A creation wizard dialog will open

3. **Choose your creation method** from the options presented:

   | Option | Icon | Description | Best For |
   |--------|------|-------------|----------|
   | **Skip to configure** | ➡️ | Start with minimal configuration and customize later | Experienced users who know what they need |
   | **Start from blank** | 📝 | Build from scratch with full control | Custom agents with specific requirements |
   | **Start from template** | 📋 | Use pre-built templates for common scenarios | Quick starts; common use cases |
   | **Start from website** | 🌐 | Generate topics from existing web content | When you have documentation to leverage |
   | **Start from document** | 📄 | Generate from uploaded documents | When you have existing content |

4. **Select your preferred option** by clicking on the corresponding card

### Step 4: Complete the Agent Creation Wizard

**For "Skip to Configure" or "Start from Blank":**

After selecting your creation method, you'll see a configuration panel:

**Step 4a: Name Your Agent**

1. Look for the **"Name"** field (usually the first field at the top)
2. **Click in the text field** and type your agent's name
   - **Example**: `Azure Resource Helper` or `HR Support Bot`
   - **Tip**: Use a descriptive name that reflects the agent's purpose

**Step 4b: Add a Description (Optional but Recommended)**

1. Below the Name field, find the **"Description"** field
2. **Click in the text field** and type a brief description
   - **Example**: `This agent helps employees find HR policies and submit time-off requests`
   - **Tip**: This description helps users understand what the agent does

**Step 4c: Select the Primary Language**

1. Find the **"Language"** dropdown below the Description field
2. **Click the dropdown** to see available languages
3. **Select your primary language** (e.g., "English (en-US)")
4. **Note**: You can add more languages later in Settings

**Step 4d: Configure Instructions (AI Behavior)**

1. Look for the **"Instructions"** or **"What should your copilot do?"** text area
2. **Click in the text area** and describe how your agent should behave
3. Write clear instructions in natural language
   - **Example Instructions**:
     ```
     You are a helpful HR assistant. You help employees with:
     - Finding company policies
     - Submitting time-off requests  
     - Answering benefits questions
     Always be professional, helpful, and concise.
     If you don't know an answer, direct users to HR.
     ```

**Step 4e: Review and Create**

1. Review all the information you've entered
2. Locate the **"Create"** button (usually in the bottom-right of the dialog)
3. **Click "Create"** to initialize your agent
4. **Wait** for the agent to be created (typically 10-30 seconds)
5. You'll be automatically redirected to the **Agent Editor** view

### Step 5: Initial Agent Configuration

**After your agent is created, you'll see the Agent Editor:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  AGENT EDITOR VIEW                                                          │
├───────────────────┬─────────────────────────────────────────────────────────┤
│  LEFT NAVIGATION  │                AGENT OVERVIEW                           │
│                   │  ┌─────────────────────────────────────────────────┐   │
│  🏠 Home          │  │  Agent Name: [Your Agent Name]                  │   │
│  📋 Overview      │  │  Status: Draft                                  │   │
│  📚 Knowledge     │  │                                                 │   │
│  💬 Topics        │  │  [Quick setup cards showing next steps]         │   │
│  📊 Entities      │  │                                                 │   │
│  ⚡ Tools         │  │  • Add knowledge sources                        │   │
│  📈 Analytics     │  │  • Create topics                                │   │
│  🚀 Publish       │  │  • Configure settings                           │   │
│  ⚙️ Settings      │  │  • Test your agent                              │   │
│                   │  └─────────────────────────────────────────────────┘   │
│                   │                                                         │
│                   │  ┌─────────────────────────────────────────────────┐   │
│                   │  │  TEST BOT    [Click to test your agent]        │   │
│                   │  └─────────────────────────────────────────────────┘   │
└───────────────────┴─────────────────────────────────────────────────────────┘
```

**Navigate to Settings to complete initial configuration:**

1. **Click "Settings"** in the left navigation panel (gear icon ⚙️ at the bottom)

2. **In the Settings panel, you'll see multiple tabs/sections:**

   | Settings Tab | Location | What to Configure |
   |--------------|----------|-------------------|
   | **General** | First tab | Agent name, icon, description |
   | **AI capabilities** | Second tab | Generative AI settings |
   | **Security** | Third tab | Authentication, access control |
   | **Channels** | Fourth tab | Deployment options |
   | **Language** | Fifth tab | Multi-language support |

**Step 5a: Configure General Settings**

1. **Click "General"** tab in Settings
2. Configure the following:

   **Agent Name:**
   - Click in the **"Name"** field
   - Edit if needed (this is the display name users will see)

   **Agent Icon:**
   - Click on the **current icon** or **"Change icon"** link
   - Choose from:
     - **Default icons**: Click an icon from the gallery
     - **Upload custom**: Click **"Upload"** → select an image file (PNG/JPG, max 30KB recommended)
   - Click **"Save"** or **"Apply"** after selecting

   **Agent Description:**
   - Click in the **"Description"** text area
   - Add or edit the agent's description (this appears in agent listings)

3. **Click "Save"** button (top-right of Settings panel) to save changes

**Step 5b: Configure AI Capabilities**

1. **Click "AI capabilities"** tab in Settings (or scroll down in Settings)
2. Configure the following options:

   **Generative AI:**
   - Find the **"Generative AI"** toggle switch
   - **Toggle ON** (switch should turn blue/green) to enable AI-powered responses
   - This allows your agent to generate natural responses using AI

   **Generative Answers:**
   - Find **"Generative answers"** toggle
   - **Toggle ON** to allow agent to answer questions from knowledge sources
   - **Note**: This requires adding knowledge sources (covered in next section)

   **Content Moderation:**
   - Find **"Content moderation"** setting
   - Select moderation level: **"Standard"** (recommended) or **"None"**
   - This filters inappropriate content in conversations

3. **Click "Save"** to apply AI settings

**Step 5c: Configure Security Settings (Optional for Initial Setup)**

1. **Click "Security"** tab in Settings
2. **Authentication** options:
   - **No authentication**: Anyone can use the agent
   - **Only for Teams and Power Apps**: Integrated authentication
   - **Manual (custom)**: Configure OAuth or custom authentication

   **For initial setup**, you can leave authentication as **"No authentication"**
   
3. **Click "Save"** if you made changes

---

## Configuring Agent Capabilities

This section provides detailed navigation for adding knowledge, topics, actions, and entities.

### Step 6: Add Knowledge Sources

Knowledge sources allow your agent to answer questions using existing content.

**Navigation Path:** Left Panel → **Knowledge** → **"+ Add knowledge"**

**Step 6a: Navigate to Knowledge**

1. In the left navigation panel, **click "Knowledge"** (📚 icon)
2. The Knowledge management page will open in the main content area
3. You'll see:
   - A list of existing knowledge sources (if any)
   - An **"+ Add knowledge"** button in the top-right or center of the page

**Step 6b: Add a Knowledge Source**

1. **Click "+ Add knowledge"** button
2. A dialog will appear with knowledge source types:

   | Source Type | Icon | Navigation | Description |
   |-------------|------|------------|-------------|
   | **Public websites** | 🌐 | Click "Public websites" card | Add URLs for web content |
   | **Files** | 📄 | Click "Files" card | Upload PDF, DOCX, XLSX, PPTX |
   | **SharePoint** | 📁 | Click "SharePoint" card | Connect to SharePoint sites |
   | **Dataverse** | 🗃️ | Click "Dataverse" card | Connect to Dataverse tables |
   | **Custom connectors** | 🔌 | Click "Custom connectors" | Connect to custom data sources |

3. **Select your knowledge source type** by clicking on the card

**Step 6c: Configure Public Website Knowledge**

If you selected **"Public websites"**:

1. **In the URL field**, type or paste the website URL
   - **Example**: `https://docs.microsoft.com/azure`
   
2. **Click "Add"** to add the URL to the list

3. **Configure indexing scope** (if available):
   - **Full site**: Index entire website
   - **Specific pages**: Index only specific URLs

4. **Click "Add"** or **"Save"** button to save the knowledge source

5. **Wait for indexing** - A progress indicator will show indexing status
   - Status will change from "Indexing" to "Ready" when complete

**Step 6d: Configure File Upload Knowledge**

If you selected **"Files"**:

1. **Click "Upload file"** or **drag and drop** files onto the upload area
2. **Select files** from your computer (supported: PDF, DOCX, XLSX, PPTX, TXT)
3. **Wait for upload** to complete (progress bar will show)
4. **Click "Add"** to add the files as knowledge

**Step 6e: Verify Knowledge Sources**

1. After adding, you'll return to the Knowledge list
2. **Verify your sources appear** in the list with "Ready" status
3. **To edit/remove**: Click on a knowledge source → use Edit or Delete options

---

### Step 7: Define Topics

Topics define conversation flows - what your agent says and does in response to user inputs.

**Navigation Path:** Left Panel → **Topics** → **"+ Add a topic"**

**Step 7a: Navigate to Topics**

1. In the left navigation panel, **click "Topics"** (💬 icon)
2. The Topics page will open showing:
   - **System topics**: Pre-built topics (Greeting, Goodbye, Fallback, etc.)
   - **Custom topics**: Topics you create
3. You'll see topic status indicators (On/Off)

**Step 7b: Understand Existing System Topics**

Before creating custom topics, review system topics:

| System Topic | Purpose | Location |
|--------------|---------|----------|
| **Greeting** | Welcome message when users start | Topics → System → Greeting |
| **Goodbye** | Farewell message when users leave | Topics → System → Goodbye |
| **Fallback** | Response when agent doesn't understand | Topics → System → Fallback |
| **Escalate** | Transfer to human agent | Topics → System → Escalate |
| **Start over** | Reset conversation | Topics → System → Start over |

**Step 7c: Create a New Topic**

1. **Click "+ Add a topic"** button (top of Topics page)

2. **Choose creation method**:

   | Option | Description | When to Use |
   |--------|-------------|-------------|
   | **From blank** | Start with empty topic | Custom conversation flows |
   | **From description (AI)** | AI generates topic from description | Quick topic creation |

3. **If selecting "From blank"**:
   - Click **"From blank"** card
   - You'll be taken to the **Topic Editor**

**Step 7d: Configure Topic in the Topic Editor**

The Topic Editor has these main areas:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TOPIC EDITOR                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  TOPIC NAME: [Click to edit name]                      Status: Draft  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  TRIGGER PHRASES                                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  + Add phrases that will trigger this topic                    │  │  │
│  │  │  Example: "I need help with billing"                           │  │  │
│  │  │  Example: "billing question"                                   │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  CONVERSATION FLOW (Authoring Canvas)                                 │  │
│  │  ┌─────────┐                                                          │  │
│  │  │ Trigger │ → [+] Add node                                           │  │
│  │  └─────────┘                                                          │  │
│  │                                                                        │  │
│  │  [Visual flow editor with drag-and-drop nodes]                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ TEST BOT PANEL (collapsible, bottom-right)                              ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

**Step 7e: Add Topic Name and Trigger Phrases**

1. **Set Topic Name:**
   - Click on the topic name field at the top (shows "Untitled" or default name)
   - Type a descriptive name (e.g., "Billing Help")
   - Press **Enter** or click outside to save

2. **Add Trigger Phrases:**
   - In the **Trigger Phrases** section, click **"+ Add"** or in the text field
   - Type a phrase that users might say to trigger this topic
   - Press **Enter** to add the phrase
   - **Add 5-10 variations** for better recognition
   
   **Example trigger phrases for "Billing Help" topic:**
   ```
   I need help with billing
   billing question
   can you help me with my bill
   I have a billing issue
   question about my invoice
   help with payment
   billing inquiry
   I need to discuss billing
   ```

**Step 7f: Build the Conversation Flow**

1. **Below the trigger phrases**, you'll see the **Authoring Canvas** with a "Trigger" node

2. **Click the "+" button** below the Trigger node to add your first node

3. **Select a node type** from the menu:

   | Node Type | Icon | Purpose | How to Use |
   |-----------|------|---------|------------|
   | **Send a message** | 💬 | Display text to user | Click → type message |
   | **Ask a question** | ❓ | Get user input | Click → configure question |
   | **Add a condition** | 🔀 | Branch based on logic | Click → set conditions |
   | **Variable management** | 📊 | Set or clear variables | Click → configure variable |
   | **Topic management** | 🔄 | Go to another topic or end | Click → select topic |
   | **Tools (Flow/Action)** | ⚡ | Execute Power Automate flow | Click → hover "Tools" → select "Flow" |
   | **Advanced** | ⚙️ | HTTP requests, authentication | Click → configure |

4. **Add a Message Node (example):**
   - Click the **"+"** button
   - Select **"Send a message"**
   - In the message box that appears, type your message
   - **Example**: `I can help you with billing questions. What would you like to know?`

5. **Add a Question Node (example):**
   - Click the **"+"** button after your message
   - Select **"Ask a question"**
   - Configure:
     - **Question text**: What do you want to ask?
     - **Identify**: Choose response type (Multiple choice, Text, Number, etc.)
     - **Save response as**: Variable name for the answer

6. **Continue building** by adding more nodes as needed

7. **Click "Save"** (top-right) to save your topic

**Example Complete Topic Flow:**
```
Trigger: "billing help" phrases
    ↓
Message: "I can help you with billing questions!"
    ↓
Question: "What type of billing help do you need?"
    - Options: [Payment method] [Invoice inquiry] [Billing dispute]
    ↓
Condition: Check user's selection
    ├── If "Payment method" → Go to: Payment Topic
    ├── If "Invoice inquiry" → Go to: Invoice Topic
    └── If "Billing dispute" → Message: "Let me connect you with support..."
    ↓
Message: "Is there anything else I can help with?"
```

---

### Step 8: Create Tools and Integrations

> **Note:** In the current version of Copilot Studio (2025), the former **"Actions"** page has been replaced by the **"Tools"** page. Power Automate flows are now added as **tools**. For official reference, see [Add an agent flow to an agent as a tool](https://learn.microsoft.com/en-us/microsoft-copilot-studio/flow-agent).

Tools connect your agent to external services and automate tasks using Power Automate flows, connectors, and prompts.

**Navigation Path:** Left Panel → **Tools** → **"Add a tool"**

**Step 8a: Navigate to Tools**

1. In the left navigation panel, **click "Tools"** (⚡ icon)
2. The Tools page will open showing:
   - Existing tools (flows, connectors, prompts)
   - **"Add a tool"** button

**Step 8b: Add a New Tool**

1. **Click "Add a tool"** button

2. **Choose tool type** from the Add tool panel:

   | Tool Type | Description | Navigation |
   |-----------|-------------|------------|
   | **Flow** | Add a Power Automate agent flow | Select "Flow" to list available flows |
   | **Connector** | Pre-built service connectors | Select a connector from the list |
   | **Prompt** | AI-powered custom prompts | Select "Prompt" |

**Step 8c: Create a Power Automate Agent Flow**

You can create an agent flow from within a topic or from the Tools page:

**Option 1 — Create from within a topic (Recommended):**

1. Open the topic where you want to call the flow
2. Click the **+** (Add node) icon below any node, and select **Add a tool**
3. On the **Basic tools** tab, select **New Agent flow**
4. The agent flows designer opens with the required **When an agent calls the flow** trigger and **Respond to the agent** action
5. Configure input parameters on the trigger and output parameters on the response action
6. Add your flow logic (connectors, actions) between the trigger and the response
7. Click **Publish** to save the flow
8. Click **Go back to agent** — a new **Action** node appears in your topic

**Option 2 — Create separately in Power Automate:**

1. Go to [Power Automate](https://make.powerautomate.com) and create a new **Instant cloud flow**
2. Use the **When an agent calls the flow** (Run a flow from Copilot) trigger
3. Add your flow logic
4. End with the **Respond to the agent** action
5. Ensure the **Asynchronous response** toggle is set to **Off**
6. Save and publish the flow
7. Return to Copilot Studio → **Tools** → **Add a tool** → **Flow** → select the flow

**Example Flow Structure:**
   ```
   Trigger: When an agent calls the flow
   ├── Input: userEmail (Text)
   ├── Action: Get user profile (Office 365)
   ├── Action: Send email notification
   └── Respond to the agent:
       └── Output: confirmationMessage (Text)
   ```

> **Important:** Agent flows must respond within 100 seconds. The **Asynchronous response** toggle must be **Off**. Every output parameter in the **Respond to the agent** action must have a value assigned.

**Step 8d: Add a Connector as a Tool**

To add a connector directly as a tool:

1. Go to **Tools** → **Add a tool**
2. **Browse available connectors** in the tool panel

3. **Popular connectors:**
   - **Office 365 Outlook**: Send emails
   - **SharePoint**: Manage files and lists
   - **Microsoft Teams**: Post messages
   - **HTTP**: Call external APIs
   - **Dataverse**: Database operations

4. **Select a connector** by clicking on it

5. **Configure the tool:**
   - Set up authentication (if required)
   - Map input/output parameters
   - Test the connection

6. **Click "Add to agent"** to save the tool

**Step 8e: Use Tools in Topics**

After adding a tool to your agent:

1. **Go to Topics** → open a topic

2. In the conversation flow, click **"+"** (Add node) → select **"Add a tool"**

3. **Select your tool** (flow or connector) from the list

4. **Map inputs**: Connect topic variables or Power Fx formulas to tool inputs

5. **Map outputs**: Store tool outputs in topic or global variables

6. **Continue the flow** with a message using the output

---

### Step 9: Configure Entities

Entities help extract and validate specific information from user inputs.

**Navigation Path:** Left Panel → **Entities** → **"+ Add an entity"**

**Step 9a: Navigate to Entities**

1. In the left navigation panel, **click "Entities"** (📊 icon)
2. You'll see:
   - **System entities**: Pre-built (Age, Email, Phone, etc.)
   - **Custom entities**: Entities you create
   
**Step 9b: View System Entities**

System entities are pre-built and ready to use:

| Entity Name | Extracts | Example Match |
|-------------|----------|---------------|
| **Age** | Age values | "25 years old" → 25 |
| **Boolean** | Yes/No values | "yes", "no", "true" |
| **City** | City names | "Seattle", "New York" |
| **Date and Time** | Date/time | "tomorrow at 3pm" |
| **Email** | Email addresses | "user@example.com" |
| **Money** | Currency amounts | "$50", "100 dollars" |
| **Number** | Numeric values | "42", "one hundred" |
| **Phone Number** | Phone numbers | "(555) 123-4567" |
| **URL** | Web addresses | "https://example.com" |

**Step 9c: Create a Custom Entity**

1. **Click "+ Add an entity"** button

2. **Choose entity type:**

   | Type | Description | Best For |
   |------|-------------|----------|
   | **Closed list** | Predefined values | Product names, categories |
   | **Regular expression** | Pattern matching | Order IDs, codes |
   | **Smart match** | AI-powered extraction | Natural language values |

3. **For Closed List entity:**
   - Enter **Entity name** (e.g., "ProductType")
   - Click **"+ Add"** to add list items
   - Add items: "Standard", "Premium", "Enterprise"
   - Optionally add **synonyms** for each item
   - Click **"Save"**

4. **For Regex entity:**
   - Enter **Entity name** (e.g., "OrderID")
   - Enter **Pattern** (regex): `ORD-[0-9]{6}`
   - Test with sample inputs
   - Click **"Save"**

**Step 9d: Use Entities in Questions**

In a topic, when adding a Question node:

1. In the **"Identify"** dropdown, select your custom entity
2. The agent will extract and validate user input against the entity
3. Store the result in a variable for use in the conversation

---

## Testing Your Agent

### Step 10: Test in the Bot Canvas

**Navigation Path:** Click **"Test"** button (bottom-left corner of any page) OR use Test panel

**Step 10a: Open the Test Panel**

1. **Locate the Test Button:**
   - At the **bottom-left corner** of the screen, look for **"Test"** or **"Test bot"** button
   - The button may show a chat bubble icon 💬
   
2. **Click the Test button:**
   - The Test panel will expand from the bottom-right
   - You'll see a chat interface similar to what users will experience

3. **Alternative access:**
   - While editing a topic, the Test panel may already be visible on the right side
   - Click the **expand arrow** to maximize it

**Step 10b: Conduct Basic Testing**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TEST PANEL                                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  🔄 Reset   📋 Track between topics   👁️ Show all variables       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Agent: Hello! I'm here to help. What can I do for you today?      │   │
│  │  ─────────────────────────────────────────────────────────────────  │   │
│  │  You: [Type your test message here]                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  [Text input field]                         [Send button ➤]        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Test Panel Features:**

| Feature | Icon/Location | Purpose |
|---------|---------------|---------|
| **Reset** | 🔄 Top of panel | Start a new conversation |
| **Track between topics** | 📋 Toggle at top | See topic transitions |
| **Show all variables** | 👁️ Toggle at top | View variable values |
| **Send message** | Text field at bottom | Type test inputs |

**Step 10c: Test Your Topics**

1. **Type a trigger phrase** in the text input at the bottom of the Test panel
   - **Example**: Type `I need help with billing`
   
2. **Press Enter** or click **Send** (➤ button)

3. **Observe the response:**
   - Does the correct topic trigger?
   - Is the message appropriate?
   - Are options/buttons displayed correctly?

4. **Continue the conversation:**
   - Respond to questions
   - Test different paths through the conversation
   - Try edge cases and unexpected inputs

**Step 10d: Use Tracking Features**

To debug your agent:

1. **Enable "Track between topics":**
   - Click the toggle at the top of the Test panel
   - This shows which topic is active during the conversation

2. **Enable "Show all variables":**
   - Click the toggle to view all variable values
   - This helps verify data is being captured correctly

3. **View conversation path:**
   - A visual indicator shows which topic/node is currently active
   - Helpful for debugging complex flows

**Testing Checklist:**

Use this checklist to ensure comprehensive testing:

- [ ] **Trigger phrases**: Test all trigger phrases for each topic
- [ ] **Conversation branches**: Test all possible paths through each topic
- [ ] **Edge cases**: Test with unexpected/invalid inputs
- [ ] **Integration actions**: Test that Power Automate flows execute correctly
- [ ] **Fallback behavior**: Test what happens with unrecognized inputs
- [ ] **Authentication**: Test authentication flow (if enabled)
- [ ] **Variables**: Verify variables capture and store data correctly
- [ ] **Entity recognition**: Test that entities extract data properly
- [ ] **Multi-turn conversations**: Test conversations that span multiple exchanges

---

### Step 11: Review Analytics

**Navigation Path:** Left Panel → **Analytics**

**Step 11a: Access Analytics (After Publishing)**

**Note:** Analytics are only available after your agent has been published and used.

1. **Click "Analytics"** in the left navigation panel (📈 icon)

2. **You'll see the Analytics dashboard with:**
   - Session summary
   - Engagement metrics
   - Resolution rates
   - Topic performance

**Step 11b: Analytics Dashboard Sections**

| Section | Location | What It Shows |
|---------|----------|---------------|
| **Summary** | Top of Analytics page | Overall conversation metrics |
| **Engagement** | Sessions tab | User engagement over time |
| **Topics** | Topics tab | Performance of individual topics |
| **Resolution** | Resolution tab | How often issues are resolved |
| **Escalation** | Escalation tab | Transfers to human agents |
| **Sessions** | Sessions tab | Individual conversation details |

---

## Publishing Your Agent

### Step 12: Publish to Channels

**Navigation Path:** Left Panel → **Publish** OR Top-right **"Publish"** button

**Step 12a: Navigate to Publish**

1. **Option A:** Click **"Publish"** in the left navigation panel (🚀 icon)
2. **Option B:** Click the **"Publish"** button in the top-right corner of the editor

**Step 12b: Review and Publish**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PUBLISH PAGE                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  PUBLISH STATUS                                                     │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐           │   │
│  │  │ Last published│  │ Current draft │  │ Publish now   │           │   │
│  │  │ Feb 15, 2026  │  │ ⚠️ Changes   │  │ [Button]      │           │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CHANGES TO PUBLISH                                                 │   │
│  │  • Topic: Billing Help (modified)                                   │   │
│  │  • Knowledge: website added                                         │   │
│  │  • Action: Send Email (new)                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CHANNELS                                                           │   │
│  │  Configure where your agent is available                            │   │
│  │  [Demo website] [Microsoft Teams] [Custom website] [More ▼]        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Publishing Process:**

1. **Review changes** listed in the "Changes to publish" section
   - Verify all changes are intentional
   
2. **Click "Publish"** button (blue/purple button)

3. **Confirm publication** in the dialog that appears
   - Click **"Publish"** again to confirm

4. **Wait for publishing** to complete
   - A progress indicator shows publishing status
   - Typically takes 1-5 minutes
   - Status changes to "Published" when complete

---

### Step 13: Configure Channels

After publishing, configure where users can access your agent.

**Navigation Path:** Publish page → **Channels** section OR Settings → Channels

**Step 13a: Demo Website (Quick Test)**

The Demo Website is automatically available after publishing:

1. On the **Publish** page, find **"Demo website"** under Channels
2. **Click "Demo website"** 
3. A dialog opens with:
   - **Demo URL**: Direct link to test your agent
   - **QR Code**: For mobile testing
   - **Share options**: Copy link to share with stakeholders
4. **Click "Open"** to test in a new browser tab
5. **Copy the URL** to share with others for testing

**Step 13b: Microsoft Teams Channel**

To deploy your agent to Microsoft Teams:

1. On the Publish page, find **"Microsoft Teams"** under Channels
2. **Click "Microsoft Teams"**
3. **Click "Turn on Teams"** (if not already enabled)
4. Configure Teams settings:

   | Setting | Description | Action |
   |---------|-------------|--------|
   | **App name** | Name shown in Teams | Enter display name |
   | **Short description** | Brief description | Enter description |
   | **Long description** | Detailed description | Enter details |
   | **Icon** | App icon in Teams | Upload or use default |
   | **Privacy URL** | Privacy policy link | Enter URL |
   | **Terms URL** | Terms of service link | Enter URL |

5. **Click "Save"** to save settings

6. **Publish to Teams:**
   - **Option A: Submit to admin** - Click "Submit for approval" to add to your org's app catalog
   - **Option B: Share direct link** - Copy the link to share directly with users
   - **Option C: Download app** - Download the Teams app package (.zip) for manual installation

**Step 13c: Custom Website (Embed)**

To embed your agent on your own website:

1. On the Publish page, find **"Custom website"** under Channels
2. **Click "Custom website"**
3. **Copy the embed code:**

   ```html
   <!-- Copilot Studio embed code example -->
   <iframe
     src="https://web.powerva.microsoft.com/environments/.../bots/.../webchat"
     frameborder="0"
     style="width: 100%; height: 500px;">
   </iframe>
   ```

4. **Add to your website:**
   - Paste the embed code into your HTML page
   - Adjust width/height as needed
   - Test on your website

**Step 13d: Other Channels**

Additional channels available (in Settings → Channels):

| Channel | Purpose | How to Configure |
|---------|---------|------------------|
| **Facebook Messenger** | Facebook integration | Link Facebook page |
| **Azure Bot Service** | Custom integrations | Connect to Azure Bot |
| **Mobile app** | Via Azure Bot Service | Use Bot Framework SDK |
| **Dynamics 365** | Customer service | Connect to Omnichannel |
| **Slack** | Via Azure Bot Service | Configure Bot connector |

---

### Step 14: Configure Handoff to Live Agent

If you want users to be able to escalate to human agents:

**Navigation Path:** Settings → **Transfer to agent** (or Customer Service → Engagement Hub)

**Step 14a: Navigate to Transfer Settings**

1. **Click "Settings"** in the left navigation panel
2. **Click "Transfer to agent"** or **"Customer Service"** (depending on your version)
3. You'll see handoff configuration options

**Step 14b: Configure Handoff**

1. **Enable agent transfers:**
   - Toggle ON the **"Enable agent transfer"** switch

2. **Choose handoff system:**
   - **Dynamics 365 Customer Service**: For Dynamics 365 users
   - **Omnichannel for Customer Service**: Full omnichannel integration
   - **Custom**: Build your own handoff solution

3. **Configure transfer context:**
   - Define what information passes to the human agent
   - Include conversation history, user info, topic context

4. **Set up escalation triggers:**
   - Define when automatic escalation occurs
   - Configure the "Escalate" topic behavior

5. **Click "Save"** to apply settings

---

## Best Practices

### Design Principles

1. **Clear Purpose**: Define specific use cases for your agent
2. **User-Centric**: Design conversations from the user's perspective
3. **Simplicity**: Keep conversations simple and focused
4. **Fallback Strategy**: Always provide options when agent doesn't understand
5. **Progressive Disclosure**: Don't overwhelm users with too many options

### Conversation Design

1. **Use Natural Language**: Write conversational, friendly responses
2. **Short Messages**: Keep messages concise (1-2 sentences)
3. **Provide Options**: Use buttons/quick replies when appropriate
4. **Confirm Understanding**: Echo back important information
5. **Set Expectations**: Let users know what the agent can and cannot do

### Topic Organization

1. **Modular Topics**: Create reusable, focused topics
2. **Trigger Phrases**: Include diverse, natural variations (10-15 per topic)
3. **System Topics**: Customize greeting, goodbye, and fallback topics
4. **Topic Redirection**: Use topic redirects to avoid duplication
5. **Error Handling**: Always handle unexpected inputs gracefully

### Performance Optimization

1. **Regular Testing**: Test after each significant change
2. **Monitor Analytics**: Review metrics weekly
3. **Iterate Based on Data**: Improve low-performing topics
4. **Update Knowledge**: Keep knowledge sources current
5. **User Feedback**: Collect and act on user feedback

### Security Best Practices

1. **Authentication**: Enable authentication for sensitive operations
2. **Data Privacy**: Don't store unnecessary personal information
3. **Compliance**: Follow organizational data handling policies
4. **Access Control**: Limit agent editor access appropriately
5. **Audit Logs**: Regularly review conversation logs

### Maintenance

1. **Version Control**: Document changes between versions
2. **Backup**: Export agent regularly as backup
3. **Testing Environment**: Use separate dev/test/prod environments
4. **Gradual Rollout**: Test with small user groups first
5. **Deprecation Plan**: Plan for end-of-life features

## Advanced Features

### Variables and Context

- **Global Variables**: Available across all topics
- **Topic Variables**: Scoped to specific topics
- **System Variables**: Pre-defined context (user name, date, etc.)

### Conditional Logic

- Use conditions to create dynamic conversations
- Support for complex expressions
- Multiple condition branches

### Multi-Language Support

1. Create separate agents for each language, or
2. Use translation actions with Power Automate
3. Configure language fallback

### Analytics and Insights

- Track conversation metrics
- Identify improvement opportunities
- Monitor customer satisfaction (CSAT)
- Analyze topic performance
- Export data for deeper analysis

## Troubleshooting Common Issues

### Agent Not Responding
- Verify agent is published
- Check trigger phrases match user input
- Review conversation logs
- Ensure channel configuration is correct

### Tools / Flows Failing
- Verify Power Automate agent flow is published and active
- Ensure the flow has the **When an agent calls the flow** trigger and **Respond to the agent** action
- Verify the **Asynchronous response** toggle is set to **Off**
- Check that all output parameters have values assigned
- Check authentication credentials
- Review input/output parameter mapping
- Check error messages in flow run history

### Low Recognition Rate
- Add more diverse trigger phrases
- Use entity extraction for better understanding
- Enable generative answers
- Review and improve topic organization

## Resources

- [Microsoft Copilot Studio Documentation](https://learn.microsoft.com/en-us/microsoft-copilot-studio/)
- [Power Automate Documentation](https://learn.microsoft.com/en-us/power-automate/)
- [Copilot Studio Community](https://powerusers.microsoft.com/t5/Microsoft-Copilot-Studio/ct-p/PVACommunity)
- [Training Modules](https://learn.microsoft.com/en-us/training/browse/?products=power-virtual-agents)

## Next Steps

After building your basic agent:

1. **Enhance with AI**: Enable generative AI features
2. **Add Analytics**: Implement custom tracking
3. **Scale**: Create multiple specialized agents
4. **Integrate**: Connect with enterprise systems
5. **Optimize**: Continuously improve based on usage data

## Support

For additional help:
- Check the [Copilot Studio Community](https://powerusers.microsoft.com/t5/Microsoft-Copilot-Studio/ct-p/PVACommunity)
- Review Microsoft Learn documentation
- Contact your organization's Microsoft support

---

**Last Updated**: February 2026
**Version**: 1.0
