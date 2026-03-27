# Linear Sync: Parent and Dependency Analysis

**Issue:** [steveyegge/beads#1528](https://github.com/steveyegge/beads/issues/1528)
**Related:** [#1191](https://github.com/steveyegge/beads/issues/1191) (bidirectional label sync), [#1192](https://github.com/steveyegge/beads/issues/1192) (store beads ID in Linear)

## Problem Statement

When syncing beads issues to Linear via `bd linear sync --push`, parent-child
relationships and issue dependencies are not propagated to Linear. An epic with
4 children syncs as 5 independent, unrelated issues in Linear. The hierarchy
visible in beads is invisible in Linear's UI.

Pull (Linear -> beads) correctly imports parent and relation data and creates
local dependency records. Push (beads -> Linear) only syncs scalar fields
(title, description, priority, state) and ignores all relationships.

## Current Architecture

### Data Flow Overview

```
Pull (Linear -> Beads):
  Linear API (issuesQuery) -> TrackerIssue -> IssueToBeads() -> DependencyInfo[] -> createDependencies()
    - parent{id,identifier} -> DependencyInfo{Type:"parent-child"}
    - relations.nodes[]     -> DependencyInfo{Type:"blocks"|"duplicates"|"related"}

Push (Beads -> Linear):
  Store.SearchIssues() -> shouldPushIssue() -> IssueToTracker() -> CreateIssue()/UpdateIssue()
    - Only syncs: title, description, priority, stateId
    - NO parent, NO relations, NO dependencies
```

### Key Files

| File | Role |
|------|------|
| `internal/linear/client.go` | GraphQL queries and mutations |
| `internal/linear/mapping.go` | Beads <-> Linear field conversion, `IssueToBeads()` |
| `internal/linear/fieldmapper.go` | `IssueToTracker()` — builds push update map |
| `internal/linear/tracker.go` | `IssueTracker` implementation: `CreateIssue()`, `UpdateIssue()` |
| `internal/linear/types.go` | Linear data types (`Issue`, `Parent`, `Relation`) |
| `internal/tracker/engine.go` | Sync engine: `doPull()`, `doPush()`, `createDependencies()` |
| `internal/tracker/tracker.go` | `IssueTracker` / `FieldMapper` interfaces |
| `internal/tracker/types.go` | `TrackerIssue`, `SyncOptions`, `DependencyInfo` |
| `internal/types/types.go` | Beads `Dependency`, `DependencyType` constants |

### Pull Path (Working)

1. **GraphQL query** (`client.go:48-103`) fetches `parent{id, identifier}` and
   `relations{nodes{id, type, relatedIssue{id, identifier}}}` for every issue.

2. **`IssueToBeads()`** (`mapping.go:391-492`) converts the Linear `Issue` to a
   beads `Issue` plus a `[]DependencyInfo` slice:
   - If `li.Parent != nil`, appends `DependencyInfo{Type: "parent-child"}`.
   - For each relation, maps via `RelationToBeadsDep()` using `MappingConfig.RelationMap`.
   - Direction is correctly handled: `blocks` inverts from/to, `blockedBy` preserves.

3. **`createDependencies()`** (`engine.go:593-615`) resolves external IDs to local
   issue IDs via `GetIssueByExternalRef()` and calls `Store.AddDependency()`.

4. **`buildDescendantSet()`** (`engine.go:617-637`) uses `DepParentChild`
   dependencies for BFS traversal when `--parent` flag is used during push.

### Push Path (Gaps)

1. **`doPush()`** (`engine.go:388-533`) iterates local issues. For each:
   - Calls `IssueToTracker()` to build an update map.
   - Calls `Tracker.CreateIssue()` or `Tracker.UpdateIssue()`.
   - **Never reads or syncs dependency/relationship data.**

2. **`IssueToTracker()`** (`fieldmapper.go:77-84`) returns only:
   ```go
   map[string]interface{}{
       "title":       issue.Title,
       "description": issue.Description,
       "priority":    PriorityToLinear(issue.Priority, m.config),
   }
   ```
   No `parentId`, no relation data.

3. **`Tracker.CreateIssue()`** (`tracker.go:108-123`) calls `client.CreateIssue()`
   which builds an `IssueCreateInput` with: `teamId`, `title`, `description`,
   `priority`, `stateId`, `labelIds`. **No `parentId` field.**

4. **`Tracker.UpdateIssue()`** (`tracker.go:125-145`) calls `client.UpdateIssue()`
   with the map from `IssueToTracker()` plus `stateId`. **No `parentId` or relation mutations.**

5. **Linear client** (`client.go`) has no `issueRelationCreate` mutation at all.

## Gap Summary

| Feature | Pull (Import) | Push (Export) | Status |
|---------|:---:|:---:|--------|
| Parent-child (`parentId`) | **Yes** | No | **GAP** |
| `blocks` / `blockedBy` relations | **Yes** | No | **GAP** |
| `duplicate` relations | **Yes** | No | **GAP** |
| `related` relations | **Yes** | No | **GAP** |
| `--parent` descendant filtering | N/A | **Yes** (local only) | Works |
| Project inheritance for sub-issues | No | No | **GAP** (raised in #1528 comments) |

## Proposed Code Changes

### Phase 1: Push Parent-Child Relationships

**Scope:** When pushing a beads issue that has a `parent-child` dependency, set
`parentId` on the Linear issue.

#### 1a. Add `parentId` to `CreateIssue` mutation (`client.go`)

The `CreateIssue` method's input map needs a `parentId` field. Linear's
`IssueCreateInput` accepts `parentId: ID` to establish sub-issue hierarchy.

```go
// In client.go CreateIssue(), after building the input map:
func (c *Client) CreateIssue(ctx context.Context, title, description string,
    priority int, stateID string, labelIDs []string,
    parentID string) (*Issue, error) {
    // ... existing input building ...
    if parentID != "" {
        input["parentId"] = parentID
    }
    // ...
}
```

**Signature change propagation:** `Tracker.CreateIssue()` and the engine need
to pass the parent's Linear internal ID.

#### 1b. Add `parentId` to `UpdateIssue` flow (`fieldmapper.go`, `tracker.go`)

When `IssueToTracker()` builds the update map, include `parentId` if the issue
has a parent-child dependency.

The challenge: `IssueToTracker()` currently only receives a `*types.Issue` which
has no dependency information. Options:

**Option A (minimal):** Extend `IssueToTracker()` signature or add a
`DependenciesToTracker()` method to `FieldMapper`. The engine would query
dependencies before calling the mapper.

**Option B (context-passing):** Add a `PushContext` struct that carries
dependencies alongside the issue through the push pipeline.

**Recommended: Option A** — Add a new optional method to the push flow:

```go
// New engine method to collect relationship data for push
func (e *Engine) getParentExternalID(ctx context.Context, issueID string) (string, error) {
    deps, err := e.Store.GetDependencies(ctx, issueID)
    // Find parent-child dep where this issue depends on (is child of) a parent
    // Look up parent's external_ref, extract Linear internal ID
}
```

Then in `doPush()`, before creating/updating, resolve the parent's Linear ID
and include it in the mutation.

#### 1c. Ensure parent is synced before child

The push loop must guarantee that parent issues are created in Linear before
their children, so `parentId` references are valid. Two approaches:

**Topological sort (recommended):** Before the push loop, sort issues so parents
appear before children. This is a simple BFS from root issues along `DepParentChild`
edges.

**Two-pass approach:** First pass creates all issues without `parentId`. Second
pass updates children with `parentId` once all parents have Linear IDs. Simpler
but doubles API calls for child issues.

### Phase 2: Push Issue Relations (blocks, related, duplicate)

**Scope:** After pushing issues, create Linear relations for beads dependencies
that map to Linear relation types.

#### 2a. Add `issueRelationCreate` mutation (`client.go`)

```go
func (c *Client) CreateRelation(ctx context.Context, issueID, relatedIssueID, relationType string) error {
    query := `
        mutation CreateRelation($input: IssueRelationCreateInput!) {
            issueRelationCreate(input: $input) {
                success
                issueRelation {
                    id
                    type
                }
            }
        }
    `
    input := map[string]interface{}{
        "issueId":        issueID,
        "relatedIssueId": relatedIssueID,
        "type":           relationType,
    }
    // ...
}
```

Linear relation types: `"blocks"`, `"blockedBy"`, `"duplicate"`, `"related"`.

#### 2b. Add relation push to engine (`engine.go`)

After the push loop creates/updates all issues, add a relation sync phase:

```go
// Phase 3.5: Sync relations (after all issues have external refs)
func (e *Engine) pushDependencies(ctx context.Context, opts SyncOptions) error {
    // 1. Fetch all local dependencies
    // 2. Filter to types that map to Linear relations
    // 3. For each, look up both issues' Linear internal IDs via external_ref
    // 4. Call CreateRelation() (idempotent — Linear deduplicates)
}
```

#### 2c. Bidirectional relation mapping in `MappingConfig`

The existing `RelationMap` maps Linear -> beads. For push, we need the reverse:

```go
// DefaultMappingConfig addition:
ReverseRelationMap: map[string]string{
    "blocks":     "blocks",     // beads blocks -> Linear blocks
    "duplicates": "duplicate",  // beads duplicates -> Linear duplicate
    "related":    "related",    // beads related -> Linear related
}
```

Note the singular/plural difference: beads uses `"duplicates"`, Linear uses
`"duplicate"`.

#### 2d. Avoid duplicate relations

Linear may already have relations (from a previous pull or manual creation).
Before creating, check if the relation already exists. Options:
- Fetch existing relations during the push (already available from `FetchIssue`)
- Rely on Linear's built-in dedup (it returns success even if relation exists)
- Track synced relations locally (adds complexity)

**Recommended:** Rely on Linear's dedup behavior for initial implementation.

### Phase 3: Project Inheritance for Sub-Issues

As noted in #1528 comments, child issues should inherit their parent's Linear
project assignment. When creating a child issue:

```go
if parentID != "" && c.ProjectID == "" {
    // Fetch parent to get its project assignment
    parent, _ := c.FetchIssueByIdentifier(ctx, parentIdentifier)
    if parent != nil && parent.Project != nil {
        input["projectId"] = parent.Project.ID
    }
}
```

This requires adding `project{id}` to the `parent` field in the issues query,
or fetching the parent separately.

### Phase 4: Pull-Side Improvements

#### 4a. Resolve `DependencyInfo` external IDs correctly

Currently `createDependencies()` (`engine.go:593-615`) resolves dependencies
using `GetIssueByExternalRef()` with `dep.FromExternalID` and `dep.ToExternalID`.
However, `IssueToBeads()` (`mapping.go:445-451`) populates these with Linear
*identifiers* (e.g., `"TEAM-123"`), not full URLs.

The engine's `createDependencies` passes these identifiers directly to
`GetIssueByExternalRef()`, which looks up by URL. This means dependency creation
silently fails for issues whose `external_ref` is a full URL (the common case).

**Fix:** In `createDependencies()`, build a lookup map from Linear identifier to
local issue ID using the already-imported issues, rather than relying on
`GetIssueByExternalRef()` with raw identifiers.

## Implementation Order and Effort Estimates

| Phase | Description | Complexity | Files Changed |
|-------|-------------|:---:|---|
| **1a** | `parentId` in `CreateIssue` | Low | `client.go`, `tracker.go` |
| **1b** | `parentId` in update flow | Medium | `fieldmapper.go`, `engine.go`, `tracker.go` |
| **1c** | Topological sort for push | Medium | `engine.go` |
| **2a** | `CreateRelation` mutation | Low | `client.go` |
| **2b** | Relation push in engine | Medium | `engine.go` |
| **2c** | Reverse relation mapping | Low | `mapping.go` |
| **2d** | Dedup handling | Low | Design decision (use Linear's built-in dedup) |
| **3** | Project inheritance | Low | `client.go` or `tracker.go` |
| **4a** | Fix pull dependency resolution | Medium | `engine.go`, `mapping.go` |

**Recommended implementation order:** 1a -> 1b -> 1c -> 4a -> 2a -> 2b -> 2c -> 3

Phase 1 (parent-child push) delivers the most user-visible value and addresses
the core complaint in #1528. Phase 4a fixes a latent bug in the existing pull
path. Phase 2 adds relation sync. Phase 3 is a quality-of-life improvement.

## Linear API Reference

### Relevant Mutations

**`issueCreate`** — accepts `parentId` in `IssueCreateInput`:
```graphql
mutation {
  issueCreate(input: {
    teamId: "...",
    title: "Child issue",
    parentId: "<parent-linear-uuid>"
  }) {
    success
    issue { id identifier }
  }
}
```

**`issueUpdate`** — accepts `parentId` in `IssueUpdateInput`:
```graphql
mutation {
  issueUpdate(id: "<issue-uuid>", input: {
    parentId: "<parent-linear-uuid>"
  }) {
    success
    issue { id identifier }
  }
}
```

**`issueRelationCreate`** — creates a relation between issues:
```graphql
mutation {
  issueRelationCreate(input: {
    issueId: "<issue-uuid>",
    relatedIssueId: "<related-uuid>",
    type: blocks  # blocks | blockedBy | duplicate | related
  }) {
    success
    issueRelation { id type }
  }
}
```

### Relevant Query Fields (Already Fetched)

```graphql
issues {
  nodes {
    parent { id identifier }
    relations { nodes { id type relatedIssue { id identifier } } }
  }
}
```

## Relationship to Other Issues

- **#1191 (Bidirectional label sync):** Independent but shares the push pipeline.
  Label sync would add `labelIds` to the push flow in `IssueToTracker()`.
  Can be implemented in parallel.

- **#1192 (Store beads ID in Linear):** Complementary. Storing beads IDs in
  Linear's description or custom fields would aid cross-referencing and could
  help with dependency resolution during push (mapping beads IDs to Linear IDs).
  Not a prerequisite but would simplify the lookup in Phase 2b.
