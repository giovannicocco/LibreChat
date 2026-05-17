# REC Implementation Plan for LibreChat Fork

This document is an implementation guide for Codex and future contributors.

The goal is to evolve this LibreChat fork into an open-source, self-hosted organizational AI workspace with:

- organization/workspace structure
- shared Slack-like AI channels
- organizational RAG with permission-aware retrieval
- organizational memory / REC engine
- agents as first-class workspace/channel participants
- compatibility with existing LibreChat private chats, agents, MCP, permissions, files, and deployment model

REC means shared organizational context: decisions, facts, processes, preferences, customer knowledge, project knowledge, and agent learnings that can be reused safely by humans and agents inside a company.

## Current Architecture Findings

The fork already contains useful primitives:

- `tenantId` exists on `User`, `Conversation`, `Message`, `Agent`, `Group`, `AclEntry`, `Role`, and `AccessRole`.
- `Group` exists and supports `source: 'local' | 'entra'`, `memberIds`, and external IDs.
- `Role` exists and stores broad RBAC permissions.
- `AclEntry` exists and supports granular permissions for principals:
  - user
  - group
  - public
  - role
- `AccessRole` exists and stores named permission bitsets per resource type.
- `accessPermissions.ts` defines principal types, resource types, access role IDs, and permission bits.
- `Agent` already supports tools, skills, MCP server names, subagents, edges, author, and `tenantId`.
- `Conversation` and `Message` still use `user` as a core ownership field.
- `MemoryEntry` is currently personal/user-scoped through `userId`, not organization-scoped.

Important existing files:

```txt
packages/data-schemas/src/schema/convo.ts
packages/data-schemas/src/schema/message.ts
packages/data-schemas/src/schema/user.ts
packages/data-schemas/src/schema/memory.ts
packages/data-schemas/src/schema/agent.ts
packages/data-schemas/src/schema/group.ts
packages/data-schemas/src/schema/role.ts
packages/data-schemas/src/schema/accessRole.ts
packages/data-schemas/src/schema/aclEntry.ts
packages/data-provider/src/permissions.ts
packages/data-provider/src/accessPermissions.ts
```

## Non-Negotiable Implementation Principles

1. Do not break existing private chats.
2. Do not remove `user` from `Conversation` or `Message` in early phases.
3. Add organizational/shared fields as optional first.
4. Keep `tenantId` as technical deployment/tenant isolation.
5. Add `organizationId`, `workspaceId`, and `channelId` as product-level scopes.
6. Use the existing ACL system instead of inventing a separate permission layer.
7. Every organizational RAG and memory lookup must be permission-aware.
8. Agents must never access a workspace/channel/document/memory unless the ACL allows it.
9. Prefer additive migrations over destructive schema changes.
10. Keep the implementation self-host friendly: no required SaaS dependency.

## Target Product Model

The target model is:

```txt
Tenant
  Organization
    Workspace
      Channel
        Conversation / Thread
          Messages from humans and agents
        Knowledge Collections
        Memory Collections / Memory Items
        Agents as members
```

Recommended meaning:

- `tenantId`: technical tenant isolation used by LibreChat.
- `organizationId`: company/account inside the product.
- `workspaceId`: department, project, or internal area.
- `channelId`: shared conversation space, similar to a Slack channel.
- `conversationId`: thread or chat session inside a channel.
- `messageId`: individual message.
- `Group`: identity/permission group, not a collaborative workspace.
- `AclEntry`: canonical way to grant permissions to users, groups, roles, and public access.

## Phase 0: Baseline Validation

Before implementing anything, Codex should inspect the current codebase and confirm paths, exports, model registration, and route conventions.

Tasks:

1. Run the project tests if available.
2. Inspect package exports for `data-schemas` and `data-provider`.
3. Find where Mongoose models are registered.
4. Find where conversations and messages are created, read, updated, and deleted.
5. Find where agents, groups, ACL entries, and access roles are used.
6. Find how the frontend retrieves the conversation list and individual message history.
7. Find how current file search/RAG is wired.
8. Find if there is an existing `Project` model, because `accessRole.ts` already references `project` and `file` resource types.

Deliverable:

- A short internal note or PR description listing exact files touched and discovered conventions.

## Phase 1: Extend Permission Resource Types

Objective: extend the existing ACL/resource permission system so it can govern REC resources.

Add or verify these resource types:

```ts
workspace
channel
conversation
knowledgeCollection
knowledgeDocument
memoryCollection
memoryItem
```

Likely files:

```txt
packages/data-provider/src/accessPermissions.ts
packages/data-schemas/src/schema/accessRole.ts
```

Required updates:

1. Extend `ResourceType` enum in `accessPermissions.ts`.
2. Extend `AccessRoleIds` with viewer/editor/owner variants for new resource types.
3. Update `accessRoleToPermBits` to map new access roles.
4. Align `accessRole.ts` `resourceType` enum with `accessPermissions.ts`.
5. Add tests for permission bit conversion.

Suggested access roles:

```ts
WORKSPACE_VIEWER
WORKSPACE_EDITOR
WORKSPACE_OWNER
CHANNEL_VIEWER
CHANNEL_EDITOR
CHANNEL_OWNER
CONVERSATION_VIEWER
CONVERSATION_EDITOR
CONVERSATION_OWNER
KNOWLEDGE_COLLECTION_VIEWER
KNOWLEDGE_COLLECTION_EDITOR
KNOWLEDGE_COLLECTION_OWNER
MEMORY_COLLECTION_VIEWER
MEMORY_COLLECTION_EDITOR
MEMORY_COLLECTION_OWNER
MEMORY_ITEM_VIEWER
MEMORY_ITEM_EDITOR
MEMORY_ITEM_OWNER
```

Acceptance criteria:

- Existing resource permissions still work for agents, prompt groups, MCP servers, remote agents, and skills.
- New resource types compile and validate.
- Permission bits still use the current model:
  - `VIEW = 1`
  - `EDIT = 2`
  - `DELETE = 4`
  - `SHARE = 8`

## Phase 2: Add Organization, Workspace, and Channel Schemas

Objective: introduce organizational structure while reusing the existing ACL system.

Create schemas:

```txt
packages/data-schemas/src/schema/organization.ts
packages/data-schemas/src/schema/workspace.ts
packages/data-schemas/src/schema/channel.ts
```

Recommended schemas:

```ts
Organization {
  organizationId: string;
  name: string;
  slug: string;
  description?: string;
  createdBy: ObjectId | string;
  defaultWorkspaceId?: string;
  settings?: Record<string, unknown>;
  tenantId?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

```ts
Workspace {
  workspaceId: string;
  organizationId: string;
  name: string;
  slug: string;
  description?: string;
  visibility: 'private' | 'organization';
  createdBy: ObjectId | string;
  settings?: Record<string, unknown>;
  tenantId?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

```ts
Channel {
  channelId: string;
  organizationId: string;
  workspaceId: string;
  name: string;
  slug: string;
  description?: string;
  type: 'public' | 'private' | 'direct' | 'agent';
  createdBy: ObjectId | string;
  defaultAgentIds?: string[];
  settings?: Record<string, unknown>;
  tenantId?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

Indexes:

```ts
organization: { organizationId, tenantId } unique
organization: { slug, tenantId } unique
workspace: { workspaceId, tenantId } unique
workspace: { organizationId, slug, tenantId } unique
channel: { channelId, tenantId } unique
channel: { workspaceId, slug, tenantId } unique
```

Do not create separate member tables initially unless needed. Prefer ACL entries:

```txt
principal user/group/role -> workspace/channel -> VIEW/EDIT/DELETE/SHARE
```

Acceptance criteria:

- Models load without breaking existing app startup.
- Organization, Workspace, and Channel can be created directly in tests/seed scripts.
- ACL can grant a group viewer access to a workspace or channel.

## Phase 3: Add Organizational Fields to Conversation and Message

Objective: support shared conversations while preserving private chats.

Update `packages/data-schemas/src/schema/convo.ts` with optional fields:

```ts
organizationId?: string;
workspaceId?: string;
channelId?: string;
visibility?: 'private' | 'channel' | 'workspace' | 'organization';
createdBy?: string;
participantIds?: string[];
participantAgentIds?: string[];
```

Update `packages/data-schemas/src/schema/message.ts` with optional fields:

```ts
organizationId?: string;
workspaceId?: string;
channelId?: string;
authorType?: 'user' | 'agent' | 'system';
authorId?: string;
```

Keep existing fields:

```ts
user
sender
isCreatedByUser
```

During transition:

- For private chats: `visibility = 'private'`, `user` remains the owner.
- For shared channel chats: `visibility = 'channel'`, `user` may be the creator/original owner, while `authorId` identifies the actual message author.

Important index considerations:

Current conversation index likely includes:

```ts
{ conversationId: 1, user: 1, tenantId: 1 }
```

Current message index likely includes:

```ts
{ messageId: 1, user: 1, tenantId: 1 }
```

Do not immediately remove these. Add new indexes:

```ts
Conversation: { channelId: 1, updatedAt: -1, tenantId: 1 }
Conversation: { organizationId: 1, workspaceId: 1, tenantId: 1 }
Message: { channelId: 1, createdAt: 1, tenantId: 1 }
Message: { conversationId: 1, createdAt: 1, tenantId: 1 }
```

Acceptance criteria:

- Existing private chat creation and retrieval still work.
- New messages can store author metadata.
- A shared conversation can be associated with a channel without schema validation errors.

## Phase 4: Centralize Permission Checks

Objective: avoid scattering authorization logic throughout routes.

Create a permission service, for example:

```txt
api/server/services/PermissionsService.js
```

or follow existing project conventions if another permission service exists.

Required API:

```ts
canAccessResource({
  user,
  resourceType,
  resourceId,
  requiredBits,
  tenantId,
}): Promise<boolean>
```

```ts
getEffectivePermissionBits({
  user,
  resourceType,
  resourceId,
  tenantId,
}): Promise<number>
```

```ts
assertCanAccessResource(args): Promise<void>
```

Permission resolution must consider:

1. direct user ACL
2. group ACL through `Group.memberIds`
3. role ACL through user role
4. public ACL
5. owner/creator shortcut only when safe and explicit

Acceptance criteria:

- Existing agent sharing still works.
- New workspace/channel ACL checks work.
- Unit tests cover user, group, role, and public principals.

## Phase 5: Workspace and Channel API

Objective: expose minimal backend APIs for organizations, workspaces, and channels.

Suggested routes:

```txt
GET    /api/organizations
POST   /api/organizations
GET    /api/organizations/:organizationId/workspaces
POST   /api/organizations/:organizationId/workspaces
GET    /api/workspaces/:workspaceId/channels
POST   /api/workspaces/:workspaceId/channels
GET    /api/channels/:channelId
PATCH  /api/channels/:channelId
DELETE /api/channels/:channelId
```

Rules:

- Creating an organization grants owner permissions to creator.
- Creating a workspace grants owner permissions to creator.
- Creating a channel grants owner permissions to creator.
- Public channels inside a workspace should still require workspace access.
- Private channels require explicit channel ACL.

Acceptance criteria:

- User can create an organization, workspace, and channel.
- User cannot access a workspace/channel without ACL.
- Admin/owner can grant access to a group.

## Phase 6: Shared Channel Conversations

Objective: enable multiple humans to write into the same channel conversation.

Backend requirements:

1. Create channel conversation endpoint:

```txt
POST /api/channels/:channelId/conversations
```

2. List channel conversations:

```txt
GET /api/channels/:channelId/conversations
```

3. Read channel messages:

```txt
GET /api/channels/:channelId/conversations/:conversationId/messages
```

4. Send message into channel conversation:

```txt
POST /api/channels/:channelId/conversations/:conversationId/messages
```

Message write flow:

1. Resolve user from auth.
2. Load channel.
3. Assert user can write to channel.
4. Create message with:
   - `authorType = 'user'`
   - `authorId = user._id`
   - `channelId`
   - `workspaceId`
   - `organizationId`
   - `tenantId`
5. If an agent is mentioned or default channel agent is active, invoke agent.
6. Store agent response with:
   - `authorType = 'agent'`
   - `authorId = agent.id`

Frontend requirements:

- Add a workspace/channel sidebar or minimal route.
- Add channel message list.
- Render message author by `authorType`/`authorId` when present.
- Preserve legacy rendering for private chats.

Realtime strategy:

MVP can use polling, like the community workaround mentioned in the upstream issue.

Preferred later:

- WebSocket or Socket.IO.
- Redis pub/sub optional for multi-instance deployments.

Acceptance criteria:

- Two different users with channel access can post into the same conversation.
- Both users can see the shared message history.
- A user without channel ACL cannot read or write messages.
- Existing private chats remain unaffected.

## Phase 7: Agents as Channel Members

Objective: make agents first-class participants in shared channels.

Create schema:

```txt
packages/data-schemas/src/schema/agentMembership.ts
```

Recommended fields:

```ts
AgentMembership {
  agentId: string;
  organizationId: string;
  workspaceId?: string;
  channelId?: string;
  role: 'member' | 'assistant' | 'admin';
  memoryScope: 'none' | 'channel' | 'workspace' | 'organization';
  toolScope: 'none' | 'channel' | 'workspace' | 'organization';
  createdBy: ObjectId | string;
  tenantId?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

Rules:

- The agent must have ACL access to the channel or be assigned through `AgentMembership` by an authorized user.
- Agent tool usage must respect both agent permissions and channel/workspace permissions.
- If `memoryScope = channel`, the agent can read only approved channel memory.
- If `memoryScope = workspace`, the agent can read approved workspace memory plus channel memory.
- If `memoryScope = organization`, the agent can read approved organization memory, subject to ACL.

Acceptance criteria:

- A channel can include one or more agents.
- A user can mention an agent in a channel message.
- Agent response is stored as a normal channel message with `authorType = 'agent'`.
- Agent cannot access unauthorized memory/documents/tools.

## Phase 8: Organizational RAG

Objective: implement permission-aware knowledge collections.

Create schemas:

```txt
packages/data-schemas/src/schema/knowledgeCollection.ts
packages/data-schemas/src/schema/knowledgeDocument.ts
packages/data-schemas/src/schema/knowledgeChunk.ts
```

Recommended fields:

```ts
KnowledgeCollection {
  collectionId: string;
  organizationId: string;
  workspaceId?: string;
  channelId?: string;
  name: string;
  description?: string;
  scope: 'user' | 'channel' | 'workspace' | 'organization' | 'restricted';
  createdBy: ObjectId | string;
  tenantId?: string;
}
```

```ts
KnowledgeDocument {
  documentId: string;
  collectionId: string;
  organizationId: string;
  workspaceId?: string;
  channelId?: string;
  sourceType: 'upload' | 'url' | 'integration' | 'activepieces' | 'mcp' | 'manual';
  title: string;
  fileId?: string;
  metadata?: Record<string, unknown>;
  uploadedBy: ObjectId | string;
  tenantId?: string;
}
```

```ts
KnowledgeChunk {
  chunkId: string;
  documentId: string;
  collectionId: string;
  organizationId: string;
  workspaceId?: string;
  channelId?: string;
  text: string;
  embedding?: number[];
  metadata?: Record<string, unknown>;
  tenantId?: string;
}
```

Retrieval rule:

Every retrieval must filter by:

```txt
tenantId
organizationId
collection ACL
workspace/channel scope
```

Never run vector search across organizational documents without permission filters.

MVP retrieval options:

1. Reuse existing LibreChat file/RAG pipeline if practical.
2. Wrap existing retrieval with a permission filter.
3. If current RAG abstractions are too user-scoped, create a parallel organizational RAG service first.

Acceptance criteria:

- User can create a knowledge collection in a workspace/channel.
- User can upload/add documents to the collection.
- Agent retrieves only documents from collections it can access.
- Cross-workspace leakage tests fail closed.

## Phase 9: Organizational Memory / REC Engine MVP

Objective: implement memory items that can be proposed, approved, and reused.

Do not replace existing `MemoryEntry` initially. Keep personal memory separate.

Create schemas:

```txt
packages/data-schemas/src/schema/memoryCollection.ts
packages/data-schemas/src/schema/memoryItem.ts
packages/data-schemas/src/schema/memoryRelation.ts
```

Recommended fields:

```ts
MemoryCollection {
  collectionId: string;
  organizationId: string;
  workspaceId?: string;
  channelId?: string;
  name: string;
  scope: 'channel' | 'workspace' | 'organization' | 'restricted';
  createdBy: ObjectId | string;
  tenantId?: string;
}
```

```ts
MemoryItem {
  memoryId: string;
  collectionId?: string;
  organizationId: string;
  workspaceId?: string;
  channelId?: string;
  type:
    | 'decision'
    | 'policy'
    | 'customer_fact'
    | 'project_fact'
    | 'team_preference'
    | 'sales_objection'
    | 'process'
    | 'agent_learning'
    | 'meeting_summary'
    | 'task_context';
  content: string;
  summary?: string;
  sourceType: 'message' | 'document' | 'integration' | 'manual' | 'agent';
  sourceId?: string;
  subjectType?: 'customer' | 'project' | 'team' | 'user' | 'agent' | 'organization';
  subjectId?: string;
  status: 'proposed' | 'approved' | 'rejected' | 'archived';
  confidence?: number;
  createdByType: 'user' | 'agent' | 'system';
  createdById: string;
  validatedBy?: ObjectId | string;
  visibility: 'channel' | 'workspace' | 'organization' | 'restricted';
  expiresAt?: Date;
  tenantId?: string;
}
```

```ts
MemoryRelation {
  fromMemoryId: string;
  toMemoryId: string;
  relationType: 'supports' | 'contradicts' | 'updates' | 'depends_on' | 'belongs_to';
  tenantId?: string;
}
```

Memory extraction MVP:

1. After a channel conversation has enough new messages, run a memory extraction job.
2. Extract candidate memories as `status = 'proposed'`.
3. Show proposed memories in a review UI.
4. Human admin/editor approves or rejects.
5. Agents can use only `approved` memories.

Important:

- Never auto-approve organizational memory in the MVP.
- Every memory item must include source references.
- Every memory retrieval must respect ACL.
- Add auditability from day one.

Acceptance criteria:

- Channel messages can generate proposed memory items.
- Approved memory items can be injected into agent context.
- Rejected memory items are not used.
- Unauthorized users/agents cannot retrieve restricted memory.

## Phase 10: REC Context Injection

Objective: use approved organizational memory and knowledge in agent prompts safely.

Create a context builder service:

```txt
api/server/services/RECContextService.js
```

Suggested API:

```ts
buildRECContext({
  user,
  agent,
  organizationId,
  workspaceId,
  channelId,
  conversationId,
  messageText,
  tenantId,
}): Promise<{
  memories: MemoryItem[];
  knowledgeChunks: KnowledgeChunk[];
  citations: unknown[];
}>
```

Context priority:

1. current thread/conversation context
2. channel approved memories
3. workspace approved memories
4. organization approved memories
5. permissioned knowledge chunks
6. personal memory only when explicitly allowed

Prompt format suggestion:

```txt
You are operating inside an organizational workspace.
Use only the REC context below if it is relevant.
Do not reveal memories or documents that are not included.
If context is insufficient, say so.

REC_CONTEXT:
- Decisions:
- Policies:
- Customer facts:
- Project facts:
- Team preferences:

KNOWLEDGE_CONTEXT:
- Chunk citations...
```

Acceptance criteria:

- Agent responses can reference approved memory and permissioned RAG.
- Citations/source references are available for knowledge and memory.
- Unauthorized context is not injected.

## Phase 11: UI MVP

Objective: provide minimal usable UI without rebuilding the entire LibreChat frontend.

MVP UI screens:

1. Workspace switcher
2. Channel list
3. Shared channel conversation view
4. Agent assignment to channel
5. Knowledge collection manager
6. Proposed memory review panel

UI principles:

- Keep existing private chat UI intact.
- Add shared channels as a new mode, not a replacement.
- Make shared chat visually distinct from private chat.
- Show author identity clearly:
  - human user
  - agent
  - system
- Show channel/workspace context in the header.
- Show memory/RAG indicators when agent used organizational context.

Acceptance criteria:

- User can switch between private chats and workspace channels.
- Shared channel messages show all authors correctly.
- Proposed memories can be approved/rejected from UI.
- Agent membership in channel is visible.

## Phase 12: Integration Layer with Activepieces and MCP

Objective: let organizations connect external systems without building every integration directly.

Recommended approach:

- Activepieces handles workflows and integrations.
- MCP handles tool exposure to agents.
- LibreChat fork owns identity, ACL, memory, RAG, context, and collaboration.

Potential entities:

```ts
IntegrationConnection {
  connectionId: string;
  organizationId: string;
  workspaceId?: string;
  provider: 'activepieces' | 'mcp' | 'custom';
  name: string;
  config: Record<string, unknown>;
  createdBy: ObjectId | string;
  tenantId?: string;
}
```

Rules:

- Integration access must be ACL-controlled.
- Agent tool access must be scoped by workspace/channel.
- External data imported into RAG/memory must record source and permission scope.

Acceptance criteria:

- A workspace can register an integration connection.
- A channel agent can use only integrations allowed in that channel/workspace.
- Imported integration data can be added to knowledge collections or memory proposals.

## Security Requirements

Codex must treat these as blocking requirements:

1. No cross-tenant data leakage.
2. No cross-organization data leakage.
3. No cross-workspace/channel leakage without ACL.
4. No RAG retrieval without permission filters.
5. No memory retrieval without permission filters.
6. No agent tool execution without tool permission checks.
7. No silent organizational memory auto-approval in MVP.
8. Keep source references for every organizational memory.
9. Add tests for negative permission cases.
10. Prefer fail-closed behavior.

## Suggested Test Cases

### Permissions

- User with direct ACL can view channel.
- User without ACL cannot view channel.
- User in group with ACL can view channel.
- User not in group cannot view channel.
- Public ACL works only when explicitly granted.
- Role ACL works for assigned user role.

### Shared Chat

- User A creates channel conversation.
- User B with access posts a message.
- User A sees User B message.
- User C without access cannot read messages.
- Agent response stores `authorType = 'agent'`.

### RAG

- Workspace A document is not returned in Workspace B search.
- Channel-private document is not returned to workspace-only users.
- Agent cannot retrieve collection without ACL.

### Memory

- Proposed memory is not injected into context.
- Approved memory is injected into context.
- Rejected memory is never injected.
- Restricted memory is visible only to authorized principals.

### Backward Compatibility

- Existing private chat still works.
- Existing agent use still works.
- Existing ACL sharing for agents still works.
- Existing user login still works.

## Recommended Branching Plan

Use separate PRs/branches:

```txt
rec/01-resource-types
rec/02-org-workspace-channel-schemas
rec/03-conversation-message-scopes
rec/04-permission-service
rec/05-channel-api
rec/06-shared-chat-mvp
rec/07-agent-channel-membership
rec/08-org-rag
rec/09-memory-engine
rec/10-context-injection
rec/11-ui-mvp
rec/12-activepieces-mcp-scoping
```

Do not implement all phases in one PR.

## Codex Working Instructions

When Codex works on this repo:

1. First inspect existing conventions before writing code.
2. Do not assume file paths; verify them.
3. Prefer TypeScript types and Zod schemas where the project already uses them.
4. Keep schema changes additive.
5. Add tests with every permission-sensitive change.
6. Keep private chats backwards compatible.
7. Reuse existing ACL and access role abstractions.
8. Do not create a second unrelated permission system.
9. Avoid large rewrites of the frontend until backend primitives are stable.
10. Make small, reviewable commits.
11. Document any migration assumptions in the PR body.
12. If a decision is ambiguous, choose the safer permission model.

## MVP Scope Definition

The first credible MVP is not the full REC engine.

The first credible MVP is:

1. workspace + channel schemas
2. ACL-controlled channels
3. shared channel conversations
4. multi-human message authorship
5. agent as channel participant
6. proposed/approved channel memory
7. permission-aware context injection for approved memory

Do not include in MVP:

- full Slack replacement
- full Notion replacement
- mobile native app
- graph database
- autonomous workflow marketplace
- hundreds of integrations
- automatic enterprise-grade data sync

## Strategic Product Boundary

LibreChat already provides:

- LLM providers
- agents
- MCP/tooling
- private chat UX
- auth foundation
- self-host deployment
- partial group/ACL sharing primitives

This fork should add:

- collaborative workspace model
- shared AI channels
- permission-aware organizational RAG
- organizational memory / REC
- agents participating in team context

The differentiation is not another ChatGPT UI. The differentiation is contextual continuity across humans and agents inside a self-hosted organization.
