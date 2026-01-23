# Neo NCE UI User Guide

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Dashboard Overview](#dashboard-overview)
3. [Managing Agents](#managing-agents)
4. [Configuring Subagents](#configuring-subagents)
5. [Managing Plugins and Functions](#managing-plugins-and-functions)
6. [Working with Indexes](#working-with-indexes)
7. [Knowledge Management](#knowledge-management)
8. [Templates and MCP](#templates-and-mcp)
9. [Troubleshooting](#troubleshooting)

---

## Getting Started

### Accessing the Application

1. Navigate to your Neo NCE UI URL (e.g., `https://your-domain.com/nce`)
2. You will be redirected to the Microsoft Azure AD login page
3. Sign in with your organizational credentials
4. The system will verify your group membership
5. Upon successful authentication, you'll be directed to the Home dashboard

### Authentication & Session Management

- **Session Duration**: Your session can remain active indefinitely (days/weeks) as long as the refresh token is valid
- **Token Refresh**: The application automatically refreshes your authentication token every 45 minutes
- **Re-authentication Required When**:
  - Your refresh token expires (typically 90 days)
  - Conditional Access policies enforce re-authentication
  - You explicitly log out
  - Browser cache is cleared

### Navigation

The left sidebar provides access to all major sections:

- **Home** - Dashboard with statistics and overview
- **Agents** - Manage AI agents and their configurations
  - Each agent expands to show its Coordinator and Subagents
- **Indexes** - Manage search indexes for knowledge retrieval
- **Knowledge** - Document and data connection management
  - Document Upload
  - Data Connections (SQL databases)
  - Document Connections (Confluence, SharePoint, Blob Storage)
- **Logout** - Sign out of the application

**Tip**: The sidebar can be minimized by clicking the chevron icon for a more compact view.

---

## Dashboard Overview

The Home dashboard provides a high-level overview of your system:

### Key Metrics Displayed

- **Total Agents**: Number of configured AI agents
- **Active Agents**: Currently running agents
- **Recent Activity**: Agents created this week
- **Total Indexes**: Number of search indexes
- **Documents**: Total indexed documents with type breakdown
- **Storage Usage**: Total storage consumed by documents
- **Last Activity**: Timestamp of most recent activity

### Quick Actions

From the dashboard, you can quickly:
- Navigate to any agent by clicking its card
- Access index details
- View recent document uploads
- Monitor system health

---

## Managing Agents

### Understanding Agents

An **Agent** is an AI-powered entity that can:
- Process user requests
- Execute functions via plugins
- Search knowledge indexes
- Coordinate with specialized subagents
- Generate responses based on configured instructions

### Viewing All Agents

1. Click **Agents** in the left sidebar
2. View the agents page showing:
   - Statistics cards (Total, Active, Recent)
   - Grid of agent cards with key information

### Viewing Agent Details

1. From the Agents page, click **View Details** on any agent card
2. The agent detail page displays:
   - **Agent Overview**: Name, description, model configuration
   - **Tabbed Interface**: Access to different configuration areas

### Agent Configuration Tabs

#### Overview Section

The overview card shows:
- **Agent Name**: Identifier for the agent
- **Description**: Purpose and capabilities
- **Welcome Message**: Initial greeting to users
- **Model Configuration**:
  - Model (e.g., gpt-4o, o1-mini)
  - Deployment name
  - Temperature (0.00 to 2.00)
  - Reasoning Level (for o1 models: minimal, low, medium, high)
- **Token Limits**:
  - Max Input Tokens
  - Max Output Tokens
  - Max Search Tokens
- **Created Date**: When the agent was created

**Actions Available**:
- **Edit Configuration**: Click the edit button to modify agent settings
- **Restart Workload**: Restart the agent's pod (for applying changes)

### Editing Agent Configuration

1. Click the **Edit** button (pencil icon) in the Agent Overview
2. The Edit Configuration modal opens with fields:
   - **Description**: Update the agent's purpose
   - **Welcome Message**: Customize the greeting message
   - **Model**: Select from available AI models
   - **Deployment**: Choose the deployment endpoint
   - **Temperature**: Adjust creativity (0.00-2.00)
     - Lower values (0.0-0.5): More focused and deterministic
     - Higher values (0.5-2.0): More creative and varied
   - **Max Input Tokens**: Maximum tokens for input context
   - **Max Output Tokens**: Maximum tokens for responses
   - **Max Search Tokens**: Maximum tokens for search results
3. Click **Save** to apply changes
4. Consider restarting the agent workload for changes to take effect

### Editing Agent Instructions

1. Navigate to the **Instructions** tab
2. Click the **Edit** button
3. Modify the system instructions that guide the agent's behavior
4. Instructions should include:
   - Agent's role and purpose
   - Behavioral guidelines
   - Response format preferences
   - Constraints and limitations
5. Click **Save** to apply changes

**Best Practice**: Be specific and clear in instructions. Include examples of desired behavior when possible.

---

## Configuring Subagents

### What are Subagents?

**Subagents** are specialized AI agents that work under a main agent (Coordinator) to handle specific tasks or domains. The Coordinator can delegate work to subagents based on the user's request.

### Understanding the Agent Hierarchy

- **Coordinator**: The main agent that receives user requests and coordinates responses
- **Subagents**: Specialized agents for specific tasks (e.g., code generation, data analysis)

### Viewing Subagents

1. Navigate to an agent's detail page
2. Click the **Subagents** tab
3. View the list of subagents with:
   - Name
   - Foundry Deployment
   - Model Settings (Temperature or Reasoning Level)
   - Token Limits (Input/Output/Search)
   - Created Date

**Note**: The Coordinator is not listed here as it's the parent agent.

### Creating a Subagent

1. On the Subagents tab, click **Create Subagent**
2. Fill in the required fields:
   - **Name**: Unique identifier (must follow Python function naming rules)
     - Start with a letter or underscore
     - Only letters, numbers, and underscores
     - Cannot be "coordinator" (reserved)
     - Cannot be a Python keyword
   - **Description**: Purpose of the subagent
   - **Instructions**: Detailed guidelines for this subagent's behavior
   - **Deployment**: Select the AI model deployment
   - **Model Settings**:
     - For standard models: Set Temperature (0.00-2.00)
     - For reasoning models (o1): Set Reasoning Level (minimal, low, medium, high)
   - **Token Limits**:
     - Max Input Tokens
     - Max Output Tokens
     - Max Search Tokens (optional)
3. Click **Create** to save the subagent

**Naming Best Practices**:
- Use descriptive names: `code_reviewer`, `data_analyst`, `content_writer`
- Keep names concise but meaningful
- Use lowercase with underscores for readability

### Editing a Subagent

#### Viewing Subagent Details

1. Click on a subagent name in the navigation sidebar (under the agent)
   - Or navigate to `/agents/{agentId}/subagent/{subagentId}`
2. View the subagent overview and configuration

#### Editing Subagent Configuration

1. On the subagent detail page, click **Edit** in the overview section
2. Modify any of the following:
   - Description
   - Instructions
   - Deployment
   - Temperature or Reasoning Level
   - Token limits
3. Click **Save**

**Important**: Changing the deployment may require restarting the agent workload.

### Deleting a Subagent

1. On the Subagents tab, find the subagent you want to remove
2. Click the **Delete** button (trash icon)
3. Confirm the deletion in the dialog
4. The subagent is permanently removed

**Warning**: This action cannot be undone. Ensure the subagent is no longer needed before deleting.

### Managing Subagent Plugins

Each subagent can have its own set of plugins and functions:

1. Navigate to the subagent detail page
2. Click the **Plugins** tab
3. Follow the same process as agent plugins (see [Managing Plugins](#managing-plugins-and-functions))

---

## Managing Plugins and Functions

### What are Plugins?

**Plugins** extend agent capabilities by providing functions that can:
- Query databases
- Call external APIs
- Process data
- Perform calculations
- Access external systems

Each plugin contains one or more **functions** that the agent can invoke.

### Viewing Agent Plugins

1. Navigate to an agent detail page
2. Click the **Plugins** tab
3. View the list of available plugins

### Expanding Plugin Details

1. Click on a plugin row to expand it
2. View all functions within that plugin
3. Each function shows:
   - Function name
   - Description
   - Active/Inactive status

### Activating/Deactivating Plugins

**To toggle an entire plugin**:
1. Locate the plugin in the list
2. Click the **toggle switch** on the right side of the plugin row
3. Green = Active (agent can use this plugin)
4. Gray = Inactive (plugin is disabled)

**Effect**: When you deactivate a plugin, all its functions are also deactivated.

### Activating/Deactivating Functions

**To toggle individual functions**:
1. Expand a plugin by clicking on it
2. Locate the specific function
3. Click the **toggle switch** next to the function
4. Green = Active, Gray = Inactive

**Best Practice**: Only enable functions that the agent needs to reduce token usage and improve performance.

### Editing Plugin Descriptions

1. Expand a plugin to view its details
2. Click the **Edit** button (pencil icon) next to the description
3. The description field becomes editable
4. Modify the description to clarify the plugin's purpose
5. Click the **Check** button to save or **X** to cancel

### Editing Function Descriptions

1. Expand a plugin and locate the function
2. Click the **Edit** button next to the function description
3. Update the description
4. Click **Check** to save or **X** to cancel

**Why edit descriptions?** Clear descriptions help the AI agent understand when and how to use each function effectively.

---

## Working with Indexes

### What are Indexes?

**Indexes** are search-enabled knowledge bases that allow agents to:
- Retrieve relevant information from documents
- Answer questions based on uploaded content
- Provide contextual responses with citations

### Viewing All Indexes

1. Click **Indexes** in the left sidebar
2. View the indexes page showing:
   - Total Indexes count
   - Total Documents count
   - Storage usage
   - List of all indexes with statistics

### Creating an Index

**Note**: Index creation may be restricted based on your permissions. Contact your administrator if the Create Index button is not available.

### Viewing Index Details

1. From the Indexes page, click on an index name or "View Details"
2. The index detail page opens with several tabs:
   - **Configuration**: Index settings
   - **Files**: Manage documents in the index
   - **Image Extraction**: Configure image-to-text processing
   - **Connections**: Automated data connections

### Index Configuration Tab

#### Viewing Index Settings

The Configuration tab displays:
- **Index Name**: Unique identifier
- **Description**: Purpose of the index
- **Search Service**: Azure AI Search service name
- **Storage Account**: Azure storage location
- **Chunking Style**: How documents are split
- **Top Results**: Number of results to return in searches
- **Language Extract Settings**:
  - Instructions for text extraction
  - Examples for the extraction model
  - Model used for extraction
- **Vision Model**: Model for image processing

#### Editing Index Configuration

1. Click the **Edit** button in the configuration section
2. Modify available fields:
   - Description
   - Language extraction instructions and examples
   - Image-to-text prompt
   - Image-to-text max tokens
   - Top results count
3. Click **Save** to apply changes

### Managing Index Files

#### Viewing Files

1. Navigate to the **Files** tab
2. View all documents in the index with:
   - File name
   - File size
   - Status (Draft, Ingesting, Indexed, Failed)
   - Last modified date
   - Last ingested date

#### Uploading Files to an Index

1. On the Files tab, click **Upload Files**
2. Click **Choose Files** or drag and drop files
3. Select one or multiple files to upload
4. Supported formats include:
   - Documents: PDF, DOCX, XLSX, PPTX, TXT
   - Images: JPG, PNG, GIF
   - Other formats (check with your administrator)
5. Click **Upload** to begin the upload
6. Monitor upload progress via the progress bar
7. Files appear in the list with "Draft" status

**Note**: All file types are now accepted. The system will process supported formats.

#### Ingesting Files

After uploading, files must be **ingested** to be searchable:

**To ingest all files**:
1. Click the **Ingest All Files** button
2. Confirm the action
3. File statuses change to "Ingesting"
4. Wait for status to change to "Indexed"

**To ingest selected files**:
1. Check the boxes next to specific files
2. Click **Ingest Selected** button
3. Confirm the action
4. Selected files begin processing

**Ingestion Status**:
- **Draft**: Uploaded but not processed
- **Ingesting**: Currently being processed
- **Indexed**: Successfully processed and searchable
- **Failed**: Processing error occurred

#### Deleting Files

**To delete files**:
1. Select files by checking their boxes
2. Click the **Delete** button (trash icon)
3. Confirm the deletion
4. Files are permanently removed from the index

**Warning**: Deleted files cannot be recovered. Ensure backups exist if needed.

#### Restarting Index Workload

If files are stuck in "Ingesting" status or the index is unresponsive:
1. Click the **Restart** button in the index overview
2. Confirm the restart
3. The index processing pod restarts
4. Processing resumes after restart

### Image Extraction Configuration

1. Navigate to the **Image Extraction** tab
2. Configure settings for extracting text from images:
   - **Image-to-Text Prompt**: Instructions for the vision model
   - **Max Tokens**: Maximum tokens for image processing (limits cost)
3. Click **Save** to apply changes

**Use Case**: When your documents contain important diagrams, charts, or images with text.

### Assigning Indexes to Agents

#### Viewing Agent Indexes

1. Navigate to an agent detail page
2. Click the **Indexes** tab
3. View currently connected indexes

#### Connecting an Index

1. On the Indexes tab, click **Connect Index**
2. A modal opens showing available indexes
3. Select an index from the dropdown
4. Click **Connect**
5. The index appears in the agent's index list

**Effect**: The agent can now search this index to retrieve relevant information.

#### Disconnecting an Index

1. Locate the index in the agent's index list
2. Click the **Disconnect** button
3. Confirm the action
4. The agent can no longer access this index

---

## Knowledge Management

The Knowledge section provides tools for managing documents and data connections.

### Document Upload

#### Uploading Documents (General)

1. Click **Knowledge** → **Document Upload** in the sidebar
2. Click **Choose Files** or drag-and-drop files
3. Select one or multiple files
4. Click **Upload**
5. Monitor upload progress
6. Files are stored and ready for index assignment

**Note**: Documents uploaded here are not automatically indexed. Assign them to an index for searchability.

### Data Connections (SQL Databases)

Data Connections allow agents to query SQL databases directly.

#### Viewing Data Connections

1. Click **Knowledge** → **Data Connections**
2. View all configured SQL connections with:
   - Connection name
   - SQL endpoint
   - Database name
   - Authentication type
   - Created date

#### Creating a SQL Connection

1. Click **Create Connection**
2. Fill in the connection details:
   - **SQL Endpoint**: Server address (e.g., `server.database.windows.net`)
   - **Database Name**: Target database
   - **Authentication Type**:
     - **Managed Identity (MI)**: Uses Azure Managed Identity (recommended)
     - **Connection String (CS)**: Uses explicit credentials
   - **Connection String** (if CS selected): Full connection string
   - **Description**: Optional notes about the connection
   - **Data Dictionary**: Optional schema information to help the AI understand the database structure
3. Click **Test Connection** to verify connectivity
4. If successful, click **Create**

**Best Practice**: Use Managed Identity authentication for enhanced security.

#### Testing a Connection

1. Locate the connection in the list
2. Click the **Test** button
3. View test results:
   - Success: Connection is valid
   - Failure: Review error message and check credentials/network

#### Editing a Connection

1. Click the **Edit** button on a connection
2. Modify available fields:
   - Description
   - SQL Endpoint
   - Connection String (if applicable)
   - Data Dictionary
3. Click **Save**

**Note**: Some fields like Database Name and Auth Type cannot be changed after creation.

#### Deleting a Connection

1. Click the **Delete** button on a connection
2. Confirm the deletion
3. The connection is removed

**Warning**: Agents using this connection will lose access to the database.

### Document Connections

Document Connections enable automatic synchronization of documents from external sources.

#### Viewing Document Connections

1. Click **Knowledge** → **Document Connections**
2. View all configured connections with:
   - Connection name
   - Type (Confluence, SharePoint, Blob Storage)
   - Connected index
   - Status
   - Last synced date

#### Creating a Document Connection

**Connection Types**:
- **Confluence**: Sync Confluence spaces and pages
- **SharePoint**: Sync SharePoint sites and documents
- **Blob Storage**: Sync Azure Blob containers

**General Steps**:
1. Click **Create Connection**
2. Select connection type
3. Enter configuration details (specific to each type)
4. Assign to an index
5. Click **Create**

#### Confluence Connection

Required fields:
- **Name**: Connection identifier
- **Description**: Purpose of the connection
- **Base URL**: Confluence instance URL
- **Space Keys**: Comma-separated list of spaces to sync
- **User Email**: Confluence user email
- **Token Secret Name**: Azure Key Vault secret containing API token
- **Index**: Target index for documents

Optional filters:
- **Timeframe**: Only sync pages modified in the last X months
- **Label Filter**: Only sync pages with specific labels

#### SharePoint Connection

Required fields:
- **Name**: Connection identifier
- **Description**: Purpose
- **SharePoint URL**: Site URL
- **Client ID**: Azure AD application ID
- **Tenant ID**: Azure AD tenant ID
- **Token Secret Name**: Key Vault secret for access token
- **Index**: Target index

#### Blob Storage Connection

Required fields:
- **Name**: Connection identifier
- **Description**: Purpose
- **Storage URL**: Azure Storage account URL
- **Container Name**: Blob container name
- **SAS Token Secret Name**: Key Vault secret with SAS token
- **Index**: Target index

#### Syncing Connections

1. Locate the connection in the list
2. Click the **Sync Now** button
3. The system fetches and ingests new/updated documents
4. Monitor the "Last Synced" timestamp

**Automatic Syncing**: Connections may sync automatically on a schedule (check with administrator).

#### Editing/Deleting Connections

Similar to SQL connections:
1. Click **Edit** to modify configuration
2. Click **Delete** to remove the connection

---

## Templates and MCP

### Templates Tab

Templates define structured data extraction workflows.

#### Viewing Templates

1. Navigate to an agent detail page
2. Click the **Templates** tab
3. View configured templates for this agent

**Note**: Template functionality may be limited based on your agent's configuration.

### MCP (Model Context Protocol) Tab

The MCP tab manages external context providers.

#### Understanding MCP

MCP allows agents to access external data sources and tools through a standardized protocol.

#### Viewing MCP Connections

1. Navigate to an agent detail page
2. Click the **MCP** tab
3. View configured MCP providers

**Note**: MCP configuration is an advanced feature. Consult your administrator for setup.

---

## Troubleshooting

### Common Issues and Solutions

#### "Failed to load agents/indexes"

**Cause**: Network issue or API unavailable  
**Solution**:
1. Refresh the page
2. Check your network connection
3. Contact your administrator if the issue persists

#### Files Stuck in "Ingesting" Status

**Cause**: Processing error or workload issue  
**Solution**:
1. Wait 5-10 minutes (large files take time)
2. Click the **Restart** button on the index
3. If the issue persists, delete and re-upload the file

#### "Token expired" or Unexpected Logout

**Cause**: Session timeout or authentication issue  
**Solution**:
1. Log back in
2. If the issue repeats, clear browser cache
3. Contact your administrator if problems continue

#### Plugin/Function Toggle Not Working

**Cause**: Temporary connection issue  
**Solution**:
1. Refresh the page
2. Try toggling again
3. Verify the change persisted by refreshing

#### Cannot Upload Files

**Cause**: File size limit or unsupported format  
**Solution**:
1. Check file size (large files may timeout)
2. Try uploading smaller batches
3. Contact administrator for file size limits

#### Subagent Name Validation Error

**Cause**: Invalid Python function name  
**Solution**:
- Use only letters, numbers, and underscores
- Start with a letter or underscore
- Avoid Python keywords (if, for, while, etc.)
- Don't use "coordinator" (reserved)

### Getting Help

If you encounter issues not covered in this guide:

1. **Check the logs**: Look for error messages on the page
2. **Refresh the browser**: Many issues resolve with a refresh
3. **Contact your administrator**: Provide error messages and steps to reproduce
4. **Check Azure Status**: Some issues may be related to Azure service availability

---

## Best Practices

### Agent Configuration

1. **Be specific in instructions**: Clear, detailed instructions yield better results
2. **Use appropriate temperature settings**:
   - Creative tasks: 0.7-1.0
   - Factual tasks: 0.0-0.3
   - Balanced: 0.5
3. **Set reasonable token limits**: Balance between response quality and cost
4. **Test changes gradually**: Modify one setting at a time to understand impact

### Plugin Management

1. **Enable only necessary plugins**: Reduces token usage and improves focus
2. **Write clear descriptions**: Helps the AI understand when to use each function
3. **Test plugin functionality**: Verify plugins work as expected after activation
4. **Review periodically**: Disable unused plugins to optimize performance

### Index Management

1. **Organize by topic**: Create separate indexes for different knowledge domains
2. **Keep files updated**: Remove outdated documents, add new ones regularly
3. **Use meaningful names**: Name indexes and files descriptively
4. **Monitor storage**: Large indexes consume more resources and cost more
5. **Ingest regularly**: Don't let too many files accumulate in Draft status

### Security

1. **Use Managed Identity**: Preferred for SQL connections
2. **Rotate secrets regularly**: Update Key Vault secrets periodically
3. **Review access logs**: Monitor who accesses what data
4. **Limit connection scope**: Only grant access to necessary databases/sites

---

## Glossary

- **Agent**: An AI entity that processes requests and generates responses
- **Coordinator**: The main agent that receives user requests and may delegate to subagents
- **Subagent**: A specialized agent focused on specific tasks
- **Plugin**: A collection of functions that extend agent capabilities
- **Function**: A specific capability within a plugin (e.g., query database, call API)
- **Index**: A search-enabled knowledge base containing documents
- **Ingestion**: The process of analyzing and indexing documents for search
- **Token**: A unit of text (roughly 4 characters or 0.75 words)
- **Temperature**: A setting controlling randomness in AI responses (0.0 = deterministic, 2.0 = very random)
- **Reasoning Level**: For o1 models, controls the depth of reasoning (minimal, low, medium, high)
- **Managed Identity**: Azure AD authentication without storing credentials
- **MCP**: Model Context Protocol for external data sources
- **Chunking**: Splitting documents into smaller pieces for efficient searching

---