## Table components: AG Grid vs Tanstack Table

AG Grid Community is the default and standard solution across Gridsuite frontends for displaying tabular data: sorting, filtering, inline editing, CSV export, etc.

However, the network modifications table (`NetworkModificationsTable`, in `commons-ui`) is built on `@tanstack/react-table` combined with `@tanstack/react-virtual` instead of AG Grid.

### Why this table is different

This component was originally implemented with AG Grid in gridstudy-app, then rewritten using Tanstack Table ([gridstudy-app#3751](https://github.com/gridsuite/gridstudy-app/pull/3751)). The rationale is not documented in the PR description or review comments, but the rewrite coincides with the introduction of a set of features that the new implementation supports:

- Expand/collapse of **composite modifications**, with sub-modifications lazily fetched from the backend only when a row is expanded ([gridstudy-app#3824](https://github.com/gridsuite/gridstudy-app/pull/3824))
- A custom, tree-aware data model with optimistic updates (and rollback on API error) for drag-and-drop operations
- Drag-and-drop that can move rows across nesting levels (root level ↔ inside a composite ↔ between composites), each relying on a different backend endpoint
- Fine-grained custom styling per row to visually convey nesting depth and the boundaries of composite modifications

Because `@tanstack/react-table` is headless (it manages table/row/column state but leaves all rendering to the application), it gives full control over the DOM and styling needed for this tree-like, highly interactive UI. `@tanstack/react-virtual` is used alongside it to keep the table performant when rendering large, deeply nested modification lists.

### Guidance for new tables

AG Grid remains the default choice for any new tabular UI in Gridsuite. The Tanstack Table + react-virtual combination should only be considered for cases with similarly specific requirements (dynamic/lazy tree data combined with advanced, cross-level drag-and-drop and heavy custom row rendering).


