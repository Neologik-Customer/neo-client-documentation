# Neologik Platform User Guide

How to operate the Neologik Platform from its admin tool (the NCE): create agents, build
knowledge, connect the two, give users a way in, and keep an eye on everything. Written
against the v5 platform; every page, tab and button named below is what you will see on
screen.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [The Home Dashboard](#the-home-dashboard)
3. [Understanding the Building Blocks](#understanding-the-building-blocks)
4. [Managing Agents](#managing-agents)
5. [Sub-agents](#sub-agents)
6. [Apps and MCP Servers](#apps-and-mcp-servers)
7. [Working with Indexes](#working-with-indexes)
8. [Knowledge Management](#knowledge-management)
9. [Channels: Giving Users Access](#channels-giving-users-access)
10. [Agent Memory](#agent-memory)
11. [Analytics](#analytics)
12. [Administrator Surfaces](#administrator-surfaces)
13. [Troubleshooting](#troubleshooting)
14. [Glossary](#glossary)

---

## Getting Started

### Accessing the Application

1. Navigate to your platform URL: `https://<your-environment-host>/nce`
2. You are redirected to the Microsoft sign in page
3. Sign in with your organisational credentials
4. The platform verifies your group membership and grants your role
5. On success you land on the Home dashboard

If you see **Access denied**, your account is not in a group with platform access; contact
your administrator. If sign in completes at Microsoft but the page shows **Sign-in failed**
with a Try again button, retry once and then contact your administrator.

### Your Role

Every user holds exactly one effective role, the highest their group memberships grant:

| Role | Can | Cannot |
|---|---|---|
| **Admin** | Everything a Contributor can, plus the Blueprints, Audit and Admin pages | |
| **Contributor** | Create and change all platform configuration: agents, connections, indexes, knowledge, apps, channels, loops, models | See Blueprints, Audit or Admin |
| **Reader** | See and read everything a Contributor sees | Change anything; every save, create or delete is refused with "This operation requires the NCE contributor role" |

### Navigation

The left sidebar gives every user:

* **Home** - dashboard with statistics and the platform health panel
* **Agents** - the agent directory and everything about each agent
* **Apps & MCP** - registered applications and tool servers
* **Indexes** - searchable knowledge stores, and the environment's ingest model settings
* **Knowledge** - document upload, document connections and SQL data connections
* **Analytics** - usage across agents
* **Logout**

Administrators additionally see **Blueprints**, **Audit** and **Admin**. These appear once
your admin role resolves after sign in.

---

## The Home Dashboard

Home is your situational view:

* **Activity numbers**: agents, indexes, documents, recent activity
* **Platform health panel**: anything needing a decision surfaces here: an expiring TLS
  certificate, expiring Key Vault secrets, **model slots with no model selected**, and
  channels awaiting admin consent. Checks the platform cannot evaluate are named in a
  "Checks unavailable" footnote rather than silently skipped.
* **Shortcut cards** into the other areas.

A warning such as "N model slot(s) have no model selected" counts the **environment
level** model slots: the three ingest slots (see
[Ingest model settings](#ingest-model-settings)) plus a slot for each platform feature in
your environment that generates content with AI (for example an email summariser or an
evaluation judge). The ingest slots matter immediately, because document ingestion stops
until they are set; a feature slot matters only when that feature is used, and its
consumer reports a clear error naming the slot until then. Per agent slots (compaction,
loop judging) are not part of this count and are safe to leave empty.

---

## Understanding the Building Blocks

The mental model in one paragraph: you curate **content** (Indexes and Knowledge), you
configure **behaviour** (Agents), and you grant **reach** (Channels on an agent). An agent
with no channel is a Worker; only other agents can call it. An agent with a Teams or web
chat channel is a user facing Agent. An agent with no knowledge attached answers only from
its model and instructions; attach an index and it searches your documents and cites them.

### Agents and their roles

An **agent** is a model given a job: instructions, tools, knowledge, memory and limits.
Every agent shows a role chip in the directory. The role is always derived, never chosen:

* **Agent** - user facing: it has one or more channels (Teams, Copilot, web chat)
* **Sub-agent** - a helper created under a parent agent; users never talk to it directly
* **Worker** - a standalone agent with no channels, called by other agents

### Apps and MCP servers

An **App** is a bespoke application registered on the platform: at most one Interface (a
web front end) plus one or more Services (business logic). An **MCP server** is a tool
service offering typed operations that agents can call. Both appear on the Apps & MCP page.
Neither is an agent: they are conventional software that agents may use.

### Connections are deny by default

No agent can call another agent, an MCP server or an app tool unless an operator has
created a connection permitting exactly that. No connection, no access, always.

---

## Managing Agents

### The Agents directory

**Agents** in the sidebar shows every agent with its role chip, status badge and key
details. Use **Create agent** (top right) to add one.

### Creating an agent

One form, one required field:

| Field | Required | Notes |
|---|---|---|
| Name | yes | The display name users will see |
| Description | no | Shows on the directory and the agent's Overview |
| Data Classification | no | General / Confidential / Restricted (default General). Gates what the agent may connect to; downgrading later needs explicit confirmation |
| Parent Agent | no | Set a parent to create a Sub-agent. Leave "No parent" for a normal agent |
| Memory Scope | no | "Own" (its own memory partition) or "Inherit parent" (shares the parent's memory pool; needs a parent) |
| Model Deployment | no | The model it runs on, picked from the deployments that exist on your environment's Azure AI Foundry account. Use **Refresh models** after a new model is deployed in Azure. Can be set later on the Configuration tab |
| Max Input / Output / Search Tokens | no | Starting token budgets; fine to leave empty and tune later |

Click **Create Agent**. You land on the agent's detail page, which refreshes itself while
provisioning runs; the status badge walks from Pending or Deploying to **Active**, usually
within a few minutes. A failed setup shows **Retry provisioning** in the header.

Two things people expect to set but do not:

* **There is no role field.** Every agent starts as a Worker. Attach a channel and it
  becomes a user facing Agent; create it with a parent and it is a Sub-agent.
* **Instructions are not on the create form.** Set them after creation on the Instructions
  tab.

**Creating an agent with no model:** the platform accepts it and defers provisioning with a
clear reason and an unset model badge rather than creating a broken agent. Pick a model on
the Configuration tab to release it.

### The agent detail page

Up to eleven tabs, depending on the agent's role and connections: **Overview,
Configuration, Instructions, Tools, Connections, Channels, Access, Knowledge, Memory,
Document Templates, Loops**. Channels, Access, Knowledge, Memory and Loops belong to user
facing agents, so sub-agents do not show them. Document Templates appears on any agent
connected to the document creation tool.

#### Overview

Name, description, status, role, hierarchy (parent and children), and the headline
configuration. **Restart workload** in the header restarts the agent's runtime; some
changes note that they apply on the next restart.

#### Configuration

* **Model Deployment**: a dropdown of the models actually deployed on your environment's
  Foundry account, never free text, with a **Refresh models** link that re-queries Azure
  live. The selector names the account it draws from.
* **Temperature** (standard models) or **Reasoning level** (reasoning models).
* **Token budgets**: Max Input, Max Output and Max Search Tokens.
* **Model Slots**: optional cheaper models for background work (conversation compaction,
  loop judging, memory extraction). Unset means the work runs on the agent's own model,
  which is a valid configuration.
* **Search Tuning**: how knowledge search behaves for this agent. Defaults suit most cases.

Sensible values are set when your environment is built. Tune only when you see a symptom.

#### Instructions

* **Agent Instructions**: the system prompt - who the agent is, what it does, tone, what it
  must not do. Markdown supported.
* **Welcome Message**: the greeting users get on a new conversation.

Be specific and clear. Include examples of desired behaviour where possible.

#### Tools

Built in platform capabilities switched on per agent: document search, memory, web search
and others. Enable only what the agent needs; every enabled tool costs the agent attention
and tokens.

#### Connections

The agent's permissions to call other agents, MCP servers and app tools. Deny by default:
create a connection to grant access, remove it to revoke. Each connection names exactly one
target.

#### Access

The agent's identity and access contract, shared by all its channels: the **user groups**
whose members may use the agent (with an optional All users switch), and the **sign in
permission packs** the agent requests when users sign in. Changes to packs are applied
from the channel cards, where re-provisioning and admin consent happen.

#### Knowledge

* **Attach index**: the agent's search now covers that index and answers ground themselves
  in the indexed content with citations. Several indexes can be attached; detaching is the
  same place.
* If the attach toast says database access lands on the agent's next deployment, use
  **Restart workload** to apply it immediately.
* **SQL data connections**: associate a registered database connection here to let the
  agent answer questions from structured data (natural language to SQL).

#### Channels, Memory, Document Templates, Loops

Covered in their own sections: [Channels](#channels-giving-users-access),
[Agent Memory](#agent-memory). Document Templates manages the templates the document
creation tool fills for this agent. Loops are recurring, evaluated agent tasks with
approval gates; your administrator will advise if your solution uses them.

### Deleting an agent

Open the agent and use the delete action in the header. Deletion asks for confirmation and
removes the agent's full footprint: its runtime, identity, permissions, routing and
configuration. The audit trail keeps the record of the deletion.

---

## Sub-agents

A sub-agent is one of an agent's skills, with its own instructions, model choice and
tools. Users never talk to it; its parent calls it.

* **Create**: use **Create child agent** on the parent's Overview tab (Hierarchy card); it
  opens the create form with the parent pre-selected. Or set a Parent Agent on the normal
  create form.
* **Memory scope**: a sub-agent either keeps its own memory partition or inherits its
  parent's pool; chosen at creation.
* **Configure**: same tabs as any agent, minus Channels, Access, Knowledge, Memory and
  Loops.
* **Delete**: same as any agent, from its detail page.

---

## Apps and MCP Servers

The **Apps & MCP** page lists the bespoke applications and tool servers registered in your
environment, each with a status badge, its components (Interface and Services) and version.

* Apps are provisioned by the platform the same zero hands way agents are: identity,
  permissions and routing are created automatically.
* An app that generates content with AI has its own **model setting** in the app's
  editor. If it is unset the app reports a clear configuration error when asked to
  generate.
* An app may expose tools to agents through its own MCP server. Agents still need a
  connection to use them.

Most day to day operation of apps is done inside the app itself; this page is where you
check status, versions and model configuration.

---

## Working with Indexes

### What an index is

A searchable knowledge store: one Azure AI Search index plus a storage container,
provisioned as a pair. Agents attached to an index can retrieve relevant passages and cite
them in answers.

### Creating an index

**Indexes** -> **Create index**:

| Field | Required | Notes |
|---|---|---|
| Name | yes | e.g. "Product Documentation". The Azure resource name is derived from it and previewed in the dialog |
| Description | yes | What this index is for |
| Top results | yes | Maximum results returned per query; the default is fine |
| Chunking type | yes | See below |
| Image description model | Conceptual only, optional | Leave as **Environment default** to use the shared image description model; an override picks from the Content Understanding catalogue |

Chunking type decides how documents are broken up and enriched:

* **Conceptual (recommended)**: semantic chunks, tables, OCR, page accurate citations.
* **Standard**: fixed size chunks, images described, lower cost.
* **Basic**: text only, near zero cost.

Chunk size, image descriptions and classification rules can be adjusted later on the
index's **Ingestion** tab. Changes apply to files ingested from then on; already indexed
content keeps its chunks until re-ingested.

### Ingest model settings

At the bottom of the Indexes page, the **Ingest Models** section holds three environment
wide model slots (the "Ingest models" tile at the top of the page scrolls to it and turns
red when any is unset):

| Slot | While unset |
|---|---|
| Embeddings | Document ingestion and agent knowledge search fail with an error naming the slot |
| Ingest enrichment | Chunk classification during ingestion fails (Conceptual and Standard chunking; Basic does no enrichment) |
| Image description | Conceptual ingestion with image descriptions on fails, unless the index has its own override |

These fail fast **by design**: a silent fallback to the wrong model would be worse. Set
each from its dropdown of actually deployed models. Agent conversations keep working while
they are unset; agent memory falls back to keyword retrieval so a half configured
environment never breaks chats.

### Managing files in an index

The index's **Files** tab lists every document with size, status and dates.

* **Statuses**: Draft (uploaded, not processed), Ingesting, Indexed (searchable, shows a
  chunk count), Failed.
* New uploads are picked up automatically within a few minutes, or press **Ingest all
  files** (or select rows and **Ingest selected**) to start straight away. While it runs
  the tab shows "Ingestion in progress: X of Y files processed" and refreshes itself.
* Delete files by selecting rows and confirming; deleted files cannot be recovered.

### Deleting an index

Open the index and use **Delete index** (top right). Deletion is permanent: documents,
search index, storage container and configuration all go. Agents that used the index lose
it as a knowledge source.

---

## Knowledge Management

### Document Upload

**Knowledge** -> **Document Upload** (or the **Upload documents** link on an index's empty
Files tab, which pre-selects the index):

1. Pick the **Target index**
2. Drag files onto the dropzone or **Browse files** (multiple at once)
3. Click **Upload N file(s)** and watch the queue (Queued -> Uploaded)

Uploading stores the files; indexing follows automatically or from the index's Files tab.
Supported formats include PDF, Word, Excel, PowerPoint, text and common image formats.

### Document Connections

**Knowledge** -> **Document Connections** syncs documents from external sources. Every
connection targets exactly one index.

* **Source types**: SharePoint, Blob Storage, Confluence.
* Each type asks for its own details and credentials; secrets are stored in Azure Key
  Vault and never shown again.
* SharePoint and Confluence connections have an **Auto-ingest** toggle so new and updated
  documents flow in automatically; SharePoint rows also offer **Sync now** for an
  immediate pull.
* Optional filters (subfolder paths, label filters, space keys) narrow what is synced.

### Data Connections (SQL)

**Knowledge** -> **Data Connections** registers databases that agents may query with
natural language:

* **SQL endpoint** and **database name**
* **Authentication**: Managed Identity (recommended) or connection string
* An optional **data dictionary** describing the schema, which markedly improves the
  quality of generated queries
* **Test Connection** before saving

Registering a connection grants nothing by itself: associate it with an agent on that
agent's Knowledge tab.

---

## Channels: Giving Users Access

Until it has a channel, an agent is a Worker; nothing user facing exists. Open the agent
-> **Channels** tab -> **Attach channel** (the agent must be Active first).

### Microsoft Teams

Always signed in. After provisioning:

1. A tenant Global Administrator must approve consent; the channel card carries the
   consent URL with Copy and Open buttons and a **Verify consent** flow.
2. Assign at least one **user group** on the agent's **Access** tab; members of the
   assigned Entra security groups are who can use the agent. No group, no access; that is
   the "Awaiting user group" state.
3. Generate the Teams app package from the card (**Teams app package** button) and upload
   it in the Teams admin centre. Once installed, the app is available in both Microsoft
   Teams and Microsoft 365 Copilot.

The amber "awaiting" states on the card are tasks for you, not failures.

### Web chat

* **Signed in** web chat needs a user group, like Teams.
* **Anonymous** web chat is open to anyone with the link. The channel card offers a
  **Preview** button to try the chat without leaving the page.

### Session commands

In any chat, users can send `/clear` to clear the current session (the reply starts
"Session cleared." and confirms the history was reset), and `login`, `logout` and `me` are
handled by the platform directly.

---

## Agent Memory

An agent remembers in three layers: the current conversation (short term), episodes (one
short summary per past conversation, expiring automatically), and a durable knowledge
graph of facts. Facts stay until contradicted, deleted, or the user is forgotten.

* Every memory entry is either **shared** (organisational knowledge any user of the agent
  may benefit from) or **private to one user**. Private memory is never shown to other
  users.
* **Consolidation** runs in the background shortly after a conversation goes quiet,
  distilling anything durable. It is on by default.
* The agent's **Memory tab** lets operators inspect what the agent has learned, correct or
  delete individual entries, and erase a user's memory entirely. Erasure asks for
  confirmation and is permanent; it supports right to be forgotten requests.

---

## Analytics

**Analytics** in the sidebar shows conversations, active users, token usage and tool
calls, filterable per agent and time range, with a per agent breakdown table and CSV
export. Data appears shortly after activity; a brand new agent shows once it has traffic.

---

## Administrator Surfaces

Visible to Admins only.

### Blueprints

Install packaged capabilities into the environment (a blueprint registers and provisions a
component in one step) and export an agent as an anonymised blueprint file for reuse.
Exported blueprints contain configuration only: no secrets, identities or environment
specific values.

### Audit

Every configuration change on the platform is recorded: who, what, when. The trail is
append only; entries cannot be edited or removed, and the page offers integrity
verification. Deletions of agents and other objects remain on the trail after the object
is gone.

### Admin

Role management: the Entra security groups (and optional "All users" switch) behind the
Reader, Contributor and Admin roles. Groups show a **Verified** chip once the platform has
confirmed them in Entra using your signed in session; a mistyped group is refused on the
spot. Note that a role group must be assigned to the environment's Neologik Platform
Sign-in enterprise application by your Entra administrator, or its members will not get
access.

---

## Troubleshooting

#### "I uploaded files but the agent doesn't know them"

Check the three links in the chain:

1. The files show as **Indexed** with a chunk count on the index's Files tab
2. The index is attached on the agent's Knowledge tab
3. The agent was restarted if the attach toast said access lands on next deploy

#### Files stuck in "Ingesting"

Large files take time; wait several minutes first. Check the Home health panel for failed
indexing. If the environment's embeddings or enrichment model slot is unset, ingestion
fails with an error naming the slot; set the slot and re-ingest.

#### "The Attach channel button is greyed out"

The agent must be Active with a provisioned workload first. Watch the status badge, or use
Retry provisioning if it failed.

#### "N model slot(s) have no model selected" on Home

See [Ingest model settings](#ingest-model-settings) for the slots that matter
immediately and what each does while unset. Feature slots (such as an email summariser or
evaluation judge) matter only when that feature is used. Per agent slots on the
Configuration tab are not part of this count and are safe to leave empty.

#### An agent's answers cite nothing

The agent has no index attached, the index has no indexed files, or its instructions do
not ask it to ground answers. Check the Knowledge tab first.

#### "Access denied" or missing sidebar items

Your role comes from group membership. Blueprints, Audit and Admin require the Admin role.
If a colleague sees pages you do not, compare roles with your administrator.

#### Changes refused with "requires the NCE contributor role"

You hold the Reader role. Ask your administrator for Contributor if you need to make
changes.

### Getting help

1. Check the Home health panel; most platform level problems surface there with a
   description
2. Refresh the browser; transient load failures usually clear
3. Contact your administrator with the error message and the steps to reproduce

---

## Glossary

* **Agent**: a model given a job - instructions, tools, knowledge, memory and limits
* **Agent (role chip)**: a user facing agent, i.e. one with at least one channel
* **Sub-agent**: a helper agent owned by one parent; no channels
* **Worker**: a standalone agent with no channels, called by other agents
* **App**: a bespoke application registered on the platform (at most one Interface plus
  one or more Services)
* **MCP server**: a tool service offering typed operations to agents; not an agent
* **Connection**: an operator created permission for one agent to use another agent or
  tool; deny by default
* **Channel**: a way users reach an agent (Teams, Copilot, web chat)
* **Index**: a searchable knowledge store of ingested documents
* **Ingestion**: processing documents into searchable, citable chunks
* **Chunking**: how documents are split (Conceptual, Standard, Basic)
* **Model Deployment**: the specific AI model an agent runs on, chosen from what is
  deployed in your environment
* **Model slot**: a named place where the platform consumes a model, set from a dropdown
  of deployed models; the Home panel flags unset slots
* **Blueprint**: a packaged, anonymised capability that installs or exports in one step
* **Data Classification**: General / Confidential / Restricted; gates what an agent may
  connect to
* **Managed Identity**: Azure authentication without stored credentials
