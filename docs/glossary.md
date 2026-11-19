# Glossary

Plain-language definitions for readers who are not Aras or AI specialists. Aras terms describe typical Aras Innovator concepts; exact names and behaviour vary by installation and version, and Aras Insight discovers them rather than assuming them.

## Aras and PLM terms

| Term | Meaning |
|---|---|
| Activity | One step in a workflow, such as a review or an approval, assigned to people who complete it, often with a vote and a comment. |
| Affected Item | In a typical change process, a relationship linking a change order to the parts or documents it changes. |
| AML (Adaptive Markup Language) | The XML-based query and command language used to read and write data through the Aras server. Aras Insight generates AML read queries only. |
| Aras Innovator | A product lifecycle management (PLM) platform. "Aras" in these documents refers to it. |
| Change order | A record that authorises a change to parts or documents. Standard examples include Express ECO; installations often have their own. |
| Change request | A record that asks for a change to be considered; in many processes it precedes a change order. |
| Configuration | How an installation is set up: its item types, properties, relationships, lists, workflows, lifecycles, methods, and forms. Distinct from business data. Aras Insight does not read permission definitions. |
| Form | The layout used to display and edit an item in the Aras client. |
| Generation | Aras's internal counter of saved versions of an item; each new version of a versioned item is a new generation. |
| Identity | An Aras user, group, or role to which permissions and workflow assignments are given. |
| Item | One record in Aras, such as a single part or document. |
| Item type | The definition of a kind of record (for example Part, Document, Problem Report), including its properties and relationships. |
| Lifecycle | The set of states an item can be in (for example Preliminary, In Review, Released) and the allowed transitions between them. |
| List | A defined set of allowed values for a property, such as a classification or reason code. |
| Method | Server-side or client-side code configured in Aras, run on events or called directly. |
| Permission | The rules that decide which identities can see or change which items. Aras applies them on every request; Aras Insight relies on that. |
| PLM (product lifecycle management) | Systems and practices for managing product data from design through change, manufacture, and retirement. |
| Problem report | A record describing a product or process issue, often linked to analysis documents and to resulting changes. |
| Property | A field on an item type, such as a name, a date, or a state. |
| Relationship | A link between two items, itself defined as a relationship type (for example a part's BOM lines, or a change order's affected items). |
| Revision | A user-visible version label (for example A, B, C) for a released state of a part or document. Several generations may exist within one revision. |
| Server event | A point at which Aras runs configured methods, such as before or after a read (a typical example is onBeforeGet). Such logic can change what a read returns, which is one reason Aras Insight reads only through the server. |
| Sign-off comment | The comment an assignee leaves when completing a workflow activity. A first-class source for "why" questions. |
| State | The current position of an item in its lifecycle. |
| Vault | The file storage behind Aras. Files are stored there and accessed through the Aras server. |
| Workflow | A defined process of activities and paths that an item moves through, such as a review and approval route. A running instance is often a Workflow Process. |
| Workflow map | The definition of a workflow: its activities, paths, and assignments. |

## AI and product terms

| Term | Meaning |
|---|---|
| Agent | A program in which a language model works in steps, choosing tools to call and reading their results, until it can answer. |
| Agent loop | The cycle of model call, tool call, and result that drives an agent. Aras Insight uses its own thin loop, not an agent framework. |
| Analysis Worker | An agent started by the Main Agent for heavy work such as aggregation, trends, process questions, or reading many files. It computes in a Sandbox and returns a report. |
| Assumption | An interpretation the agent made (for example a definition of "delayed") that the answer states openly. |
| Background job | Long-running work run outside the conversation: a full-text scan across files, which runs as the user after the user confirms, or batched learning over transcripts. |
| Chat App | The web interface where users sign in through Aras, ask questions, watch progress, and read cited answers. It holds no agent logic. |
| Citation | A reference from a statement in an answer to its source: an Aras record, a file passage, or a knowledge entry. |
| Citation Registry | The part of the Tool Gateway that records every Aras record, file passage, and knowledge entry an answer may cite, so that citations point at things that were actually retrieved. |
| Confidence | The high, medium, or low label shown with a knowledge entry. It is derived from the entry's layer, state, and whether someone other than its teacher has confirmed it. |
| Context window | The amount of text a model can consider at once. |
| Coverage | The part of an answer that says what the answer was based on and what it could not see or read. |
| Database | The one PostgreSQL instance that holds sessions and transcripts, knowledge, extracted file text, job state, and audit and usage records. |
| Discovery | Automatic reading of the Aras configuration to build the discovered layer of knowledge. |
| Embedding and vector index | A numeric representation of text used for similarity search, and the index that stores it. Not used in the MVP. |
| File Service | The part of the Tool Gateway that fetches vault files through Aras as the user, extracts their text, stores it keyed by file id, and re-checks the user's access each time the text is used. |
| Fine-tuning | Further training of a model on specific data. Not used. |
| Hallucination | A model stating something that is not supported by its sources. |
| Independently confirmed | A learned entry that someone other than the person who taught it has confirmed. Until then the agent states the entry as an assumption when it uses it. |
| Instance Knowledge | What Aras Insight knows about one installation: the Discovered layer (from configuration) and the Learned layer (from conversations). |
| Knowledge entry | One unit of knowledge, with a description, provenance, and a state (confirmed, uncertain, contested, or retired). A learned entry also records whether someone other than its teacher has confirmed it. |
| Knowledge Service | The component that stores and serves the knowledge layers (Product Skills, Instance Knowledge, User Preferences). It loads them by index and description, not by vector search. |
| Learner | The component that writes and maintains Learned knowledge from conversations. |
| Main Agent | The conversational agent that talks to the user, answers simple lookups, and starts Analysis Workers. |
| MCP (Model Context Protocol) | An open protocol for exposing tools to AI agents. An MCP interface is deferred. |
| Model endpoint | The network address of a language model service. Aras Insight uses endpoints the customer configures. |
| Model Router | The component that sends each model call to the endpoint configured for its slot. |
| Model slot | One of three configurable model roles: main, worker, background. |
| OCR (optical character recognition) | Extracting text from images and scanned pages. Deferred. |
| Product Skills | Guidance shipped with the product, such as how to write AML and how to use the tools. |
| Prompt | The instructions given to a model. |
| Query Analyzer | The component that checks every generated AML query for validity and cost before it runs. |
| Sandbox | An isolated environment, one per conversation, where an Analysis Worker's code runs. It has no network and receives data only as files. |
| Semantic layer | The part of a system that interprets meaning. In Aras Insight, the model plays this role. |
| Throttle | The component that limits how many and how large queries run, per user and system-wide. |
| Token | The unit in which models measure text; model usage is usually billed per token. |
| Tool | A function the agent can call, such as running an AML query or reading a file. |
| Tool Gateway | The only component that talks to Aras. It holds each user's credentials and runs every query as that user. |
| User Preferences | Per-user settings and habits, kept separate from shared knowledge. |
