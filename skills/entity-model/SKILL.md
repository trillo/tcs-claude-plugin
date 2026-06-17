---
description: The EntityModel step — generate the app's custom business entities from the SoftwareSpec (and add/edit individual ones), and view the full model including built-in system entities. Use after SoftwareSpec, or when the user wants to add or change an entity.
---

# Step: EntityModel — the data model

Generates the app's **custom** business entities (`ClassM`). The platform's
**system entities** (`User`, `Tenant`, etc.) are built-in — you **reference**
them, never define them. Depends on `SoftwareSpec` being COMPLETED.

## See the full model first

`md_list modelClassName="ClassM"` returns:
- `rows` — this app's **custom** entities (editable here),
- `systemClasses` — the **built-in** platform entities (read-only, from
  `app-classes`; not editable via `md_*`).

The full model = custom + system. Always account for the system entities so you
don't duplicate them.

## Reserved entity names — never create these

The platform owns a set of class names. **Never create a `ClassM` with any of
these names** — the deploy silently drops the collision and your "table" then
resolves to the *platform's* table (e.g. an app `Task` returns background-job
rows; an app `Conversation` writes into the agent-conversation table). If your
domain needs one of these concepts, **qualify the name** (e.g. `ScheduledTask`,
`ClientConversation`, `AppGroup`, `ClientTemplate`, `ClientSecret`).

Reserved (platform) class names:

```
AIUsage, Activity, ActivityOutput, ActivityOutputArchive, Agent,
AgentConversation, AgentMemory, AgentMessage, ApiClient, AppConfig,
AppMDCollection, AppRole, AppSecret, Application, AuditLog, BillingAccount,
CodeGenRequest, Conversation, ConversationMessage, Email, EmailContact,
EmailTemplate, EmailUsage, ExternalInterface, ExternalService, File2,
FileContent, FileRawData, FileToUser, Folder, FolderToUser, Group, GroupToUser,
HostedApp, Invitation, KnowledgeContainer, KnowledgeNode, MessagePair,
MetadataApiKey, OAuthAuthCode, OAuthFlowSession, OAuthProvider,
OAuthRefreshToken, Prompt, RateCard, RecordShare, Secret, SentEmail, Task,
TaskEvent, TaskQueue, Template, Tenant, TestData, TestLastInput, TrilloMD,
TrilloMDArchive, TrilloMDVersion, User, UserPreference, UserToAppRoleToTenant,
UserToTenant, UserToToken, VerificationToken, Workflow
```

Matching is **case-insensitive** — `task`, `Task`, `TASK` all collide. When the
SoftwareSpec names an entity that hits this list, rename it (and update every
reference: function params, agent bindings, relationships) before creating it.

## Generate

1. `step_guide({step:"EntityModel"})` → prompt + `expectedOutputSchema` + the
   live `systemClasses`. Generate to the schema; **reference** system entities
   by name in relationships; do **not** redefine them.
2. Ground on the spec: `md_get modelClassName="SoftwareSpec"`.
3. Write each custom entity (PascalCase names):
   `md_create modelClassName="ClassM" name="<Entity>" content=<def>`.
   Edit one with `md_update`.

## Add a single entity

When the model already exists and the user wants one more, use
`step_guide({step:"EntityModel.add"})` (the `AOSAddEntity` prompt) →
`md_create ClassM`.

## Confirm + hand off

`app_status` → `EntityModel: COMPLETED`, `Functions: READY`. Summarize the
entities (custom + the system ones in play), then ask whether to proceed to
the **functions** step.
