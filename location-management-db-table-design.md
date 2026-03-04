# Location Management Database Table Design

## Scope This Covers
This schema is designed for the wireframe features currently shown in this repo:
- Manage location hierarchy (area -> row -> lane -> container), activate/deactivate, capacity and occupancy.
- Search by element, rack, stack/pre-stack, project, and location grid path.
- View element and rack location details.
- Move/rebook single or multiple elements with conflict checks and full history.
- Scan loadsheets, track loadsheet dates, and act on selected elements.
- Track offline queued operations, retries, and sync results.
- Determine what should be loaded/delivered next.

## Core Design Choices
- Keep **location hierarchy normalized** so each level can be managed cleanly.
- Keep **current state** fast to read (current location on `elements`).
- Keep **history immutable** in separate movement/scan tables.
- Keep **planning** (delivery/load order) separate from physical location state.

## Tables

### 1) `projects`
Purpose: grouping for elements/racks and filtering.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `code` | text unique | Optional short code |
| `name` | text not null | |
| `active` | boolean default true | |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

Indexes:
- unique(`code`)
- btree(`name`)

### 2) `areas`
Purpose: top-level yard/staging regions.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `name` | text not null | e.g., `Yard West`, `Staging West` |
| `active` | boolean default true | Hidden from pickers when false |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

Constraints:
- unique(`name`)

### 3) `rows`
Purpose: row within area.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `area_id` | uuid FK -> areas.id | |
| `code` | text not null | e.g., `A`, `Staging` |
| `active` | boolean default true | |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

Constraints:
- unique(`area_id`, `code`)

### 4) `lanes`
Purpose: lane within row.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `row_id` | uuid FK -> rows.id | |
| `code` | text not null | e.g., `1`, `Main` |
| `active` | boolean default true | |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

Constraints:
- unique(`row_id`, `code`)

### 5) `containers`
Purpose: final assignable location node (rack, pre-stack/stack, staging slot).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `lane_id` | uuid FK -> lanes.id | |
| `name` | text not null | e.g., `Rack R-B3-12`, `Slot 08` |
| `container_type` | text not null | `RACK`, `PRE_STACK`, `STAGING_SLOT`, etc |
| `capacity` | int null | |
| `active` | boolean default true | |
| `tag_code` | text unique null | QR/barcode on location tag |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

Constraints:
- unique(`lane_id`, `name`)

Indexes:
- btree(`container_type`)
- btree(`active`)
- btree(`name`)

### 6) `racks`
Purpose: searchable rack identity and shipping metadata (separate from location container name).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `rack_code` | text unique not null | e.g., `R-B3-12` |
| `barcode` | text unique null | |
| `project_id` | uuid FK -> projects.id | |
| `current_container_id` | uuid FK -> containers.id null | Current physical location |
| `leave_at` | timestamptz null | supports "what leaves next" |
| `status` | text not null default `ACTIVE` | `ACTIVE`, `IN_TRANSIT`, `LOADED`, etc |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

Indexes:
- btree(`project_id`, `leave_at`)
- btree(`current_container_id`)

### 7) `elements`
Purpose: element master record and current location pointer.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `element_code` | text unique not null | e.g., `24-10345-001` |
| `barcode` | text unique null | |
| `description` | text null | |
| `project_id` | uuid FK -> projects.id | |
| `status` | text not null | `CONFIRMED`, `QUEUED`, `CONFLICT`, `MISSING`, `NOT_IN_YARD`, `LOADED`, `DELIVERED` |
| `current_container_id` | uuid FK -> containers.id null | null when not in yard |
| `current_rack_id` | uuid FK -> racks.id null | optional, if tied to rack |
| `last_scanned_at` | timestamptz null | |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

Indexes:
- btree(`element_code`)
- btree(`barcode`)
- btree(`project_id`, `status`)
- btree(`current_container_id`)
- gin(`to_tsvector('simple', coalesce(element_code,'') || ' ' || coalesce(description,''))`)

### 8) `move_batches`
Purpose: one move/rebook action across one or many elements.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `move_type` | text not null | `REBOOK`, `STORE`, `MOVE` |
| `expected_source_container_id` | uuid FK -> containers.id null | used in verify step |
| `target_container_id` | uuid FK -> containers.id null | chosen destination |
| `requested_by_user_id` | uuid null | auth user id |
| `requested_from_device_id` | uuid null | device that queued/submitted |
| `state` | text not null | `DRAFT`, `QUEUED`, `SUBMITTED`, `COMPLETED`, `FAILED`, `PARTIAL_CONFLICT` |
| `conflict_summary` | text null | |
| `submitted_at` | timestamptz null | |
| `completed_at` | timestamptz null | |
| `created_at` | timestamptz not null | |

Indexes:
- btree(`state`, `created_at`)
- btree(`target_container_id`)

### 9) `move_batch_items`
Purpose: per-element result within a move batch.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `move_batch_id` | uuid FK -> move_batches.id | |
| `element_id` | uuid FK -> elements.id | |
| `expected_container_id` | uuid FK -> containers.id null | snapshot at selection time |
| `actual_container_id_at_submit` | uuid FK -> containers.id null | for conflict detection |
| `result` | text not null | `MOVED`, `CONFLICT`, `SKIPPED`, `FAILED` |
| `error_code` | text null | |
| `error_message` | text null | |
| `created_at` | timestamptz not null | |

Constraints:
- unique(`move_batch_id`, `element_id`)

Indexes:
- btree(`element_id`, `result`)

### 10) `element_movements`
Purpose: immutable history of location transitions.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `element_id` | uuid FK -> elements.id | |
| `from_container_id` | uuid FK -> containers.id null | |
| `to_container_id` | uuid FK -> containers.id null | |
| `movement_type` | text not null | `MOVE`, `REBOOK`, `LOAD`, `UNLOAD`, `DELIVER` |
| `move_batch_item_id` | uuid FK -> move_batch_items.id null | |
| `performed_by_user_id` | uuid null | |
| `performed_at` | timestamptz not null | |
| `notes` | text null | |

Indexes:
- btree(`element_id`, `performed_at desc`)
- btree(`to_container_id`, `performed_at desc`)

### 11) `scan_events`
Purpose: audit/scenario support for scanning element, rack, location, or loadsheet code.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `scan_type` | text not null | `ELEMENT`, `RACK`, `LOCATION`, `LOADSHEET` |
| `scan_code` | text not null | raw scanned value |
| `resolved_entity_type` | text null | `ELEMENT`, `RACK`, `CONTAINER`, `LOADSHEET` |
| `resolved_entity_id` | uuid null | |
| `device_id` | uuid null | |
| `user_id` | uuid null | |
| `scan_result` | text not null | `MATCH`, `NOT_FOUND`, `AMBIGUOUS`, `ERROR` |
| `scanned_at` | timestamptz not null | |

Indexes:
- btree(`scan_type`, `scanned_at desc`)
- btree(`scan_code`)

### 12) `loadsheets`
Purpose: track uploaded/scanned loadsheets.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `loadsheet_code` | text unique not null | e.g., `LS-2026-000184` |
| `source_system` | text null | |
| `parsed_at` | timestamptz null | |
| `load_date` | date null | planned load date from sheet |
| `delivery_date` | date null | planned delivery date from sheet |
| `scheduled_departure_at` | timestamptz null | if present in source |
| `status` | text not null | `NEW`, `PARSED`, `PARTIAL`, `FAILED`, `CLOSED` |
| `created_by_user_id` | uuid null | |
| `created_at` | timestamptz not null | |

Indexes:
- btree(`load_date`)
- btree(`delivery_date`)
- btree(`scheduled_departure_at`)

### 13) `loadsheet_items`
Purpose: element lines from loadsheet and yard match status.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `loadsheet_id` | uuid FK -> loadsheets.id | |
| `line_no` | int not null | |
| `element_code_raw` | text not null | as parsed |
| `element_id` | uuid FK -> elements.id null | null when unknown/not found |
| `yard_match_status` | text not null | `OK`, `MISSING`, `NOT_IN_YARD`, `UNKNOWN` |
| `current_container_id` | uuid FK -> containers.id null | snapshot at parse/review |
| `planned_load_at` | timestamptz null | per-line schedule if provided |
| `planned_deliver_at` | timestamptz null | per-line schedule if provided |
| `sequence_no` | int null | load order in the sheet |
| `selected_for_action` | boolean default false | UI selection state |
| `created_at` | timestamptz not null | |

Constraints:
- unique(`loadsheet_id`, `line_no`)

Indexes:
- btree(`loadsheet_id`, `yard_match_status`)
- btree(`element_id`)
- btree(`loadsheet_id`, `sequence_no`)
- btree(`planned_load_at`)
- btree(`planned_deliver_at`)

### 14) `dispatch_plans`
Purpose: outbound delivery/loading plan header.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `project_id` | uuid FK -> projects.id | |
| `plan_code` | text unique not null | |
| `scheduled_departure_at` | timestamptz null | |
| `status` | text not null | `PLANNED`, `READY`, `LOADING`, `DEPARTED`, `CANCELLED` |
| `created_at` | timestamptz not null | |
| `updated_at` | timestamptz not null | |

### 15) `dispatch_plan_items`
Purpose: explicit "what should be loaded/delivered next" ordering.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `dispatch_plan_id` | uuid FK -> dispatch_plans.id | |
| `element_id` | uuid FK -> elements.id null | either element or rack required |
| `rack_id` | uuid FK -> racks.id null | |
| `sequence_no` | int not null | next item is smallest open sequence |
| `required_by_at` | timestamptz null | |
| `load_status` | text not null | `PENDING`, `STAGED`, `LOADED`, `DELIVERED`, `SKIPPED` |
| `loaded_at` | timestamptz null | |
| `delivered_at` | timestamptz null | |
| `notes` | text null | |
| `created_at` | timestamptz not null | |

Constraints:
- check: exactly one of (`element_id`, `rack_id`) is non-null
- unique(`dispatch_plan_id`, `sequence_no`)

Indexes:
- btree(`dispatch_plan_id`, `load_status`, `sequence_no`)
- btree(`required_by_at`)

### 16) `sync_queue`
Purpose: offline-safe operations queued on device before server submit.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `device_id` | uuid not null | |
| `operation_type` | text not null | `MOVE_BATCH_SUBMIT`, `SCAN_UPLOAD`, etc |
| `payload_json` | jsonb not null | full request payload |
| `dedupe_key` | text null | prevent duplicates |
| `status` | text not null | `QUEUED`, `SENDING`, `ACKED`, `FAILED`, `DEAD_LETTER` |
| `retry_count` | int default 0 | |
| `next_retry_at` | timestamptz null | |
| `last_error` | text null | |
| `created_at` | timestamptz not null | |
| `acked_at` | timestamptz null | |

Indexes:
- btree(`device_id`, `status`, `created_at`)
- unique(`dedupe_key`) where `dedupe_key is not null`

### 17) `sync_attempts`
Purpose: per-attempt telemetry for offline queue retries and diagnostics.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `sync_queue_id` | uuid FK -> sync_queue.id | |
| `attempt_no` | int not null | |
| `started_at` | timestamptz not null | |
| `finished_at` | timestamptz null | |
| `result` | text not null | `SUCCESS`, `RETRYABLE_ERROR`, `FATAL_ERROR` |
| `http_status` | int null | |
| `error_code` | text null | |
| `error_message` | text null | |

Constraints:
- unique(`sync_queue_id`, `attempt_no`)

Indexes:
- btree(`sync_queue_id`, `attempt_no`)
- btree(`started_at desc`)

## Relationship Summary
- `areas -> rows -> lanes -> containers` is the managed hierarchy.
- `elements.current_container_id` gives fast "where is it now".
- `element_movements` stores complete movement history.
- `move_batches` + `move_batch_items` model multi-select rebook/move flow and conflicts.
- `loadsheets` + `loadsheet_items` support batch scan/review/selection.
- `loadsheets` + `loadsheet_items` also hold load/delivery dates and line sequence.
- `dispatch_plan_items.sequence_no` is the source of truth for "load/deliver next".
- `sync_queue` captures offline operations until acknowledged.
- `sync_attempts` stores retry history for offline/debug visibility.

## High-Value Queries This Enables
- **Find element by scan**: `barcode` or `element_code`, include current location path.
- **Find rack and leave order**: rack by code/barcode, ordered by `leave_at`.
- **Find stack/pre-stack quickly**: filter `containers` by `container_type='PRE_STACK'` + `name`.
- **Grid path search**: join `areas -> rows -> lanes -> containers -> elements` and search by any path token.
- **Project search**: list element/rack locations filtered by `project_id`.
- **Location occupancy**: count active elements grouped by container vs `capacity`.
- **Move conflict detection**: compare `expected_container_id` with element current container at submit.
- **Loadsheet triage**: group lines by `yard_match_status` (`OK`, `MISSING`, `NOT_IN_YARD`).
- **Loadsheet date views**: sort by `planned_load_at` / `planned_deliver_at` or header `load_date` / `delivery_date`.
- **What loads next**: first `dispatch_plan_items` row with `load_status='PENDING'` ordered by `sequence_no`.

## Suggested Enums (or controlled text)
- `element_status`: `CONFIRMED`, `QUEUED`, `CONFLICT`, `MISSING`, `NOT_IN_YARD`, `LOADED`, `DELIVERED`
- `move_state`: `DRAFT`, `QUEUED`, `SUBMITTED`, `COMPLETED`, `FAILED`, `PARTIAL_CONFLICT`
- `batch_item_result`: `MOVED`, `CONFLICT`, `SKIPPED`, `FAILED`
- `yard_match_status`: `OK`, `MISSING`, `NOT_IN_YARD`, `UNKNOWN`
- `dispatch_item_status`: `PENDING`, `STAGED`, `LOADED`, `DELIVERED`, `SKIPPED`
- `sync_attempt_result`: `SUCCESS`, `RETRYABLE_ERROR`, `FATAL_ERROR`

## Minimal MVP Cut (if you want to phase delivery)
Phase 1 tables:
- `projects`, `areas`, `rows`, `lanes`, `containers`, `elements`, `element_movements`

Phase 2 tables:
- `move_batches`, `move_batch_items`, `loadsheets`, `loadsheet_items`

Phase 3 tables:
- `racks`, `dispatch_plans`, `dispatch_plan_items`, `scan_events`, `sync_queue`
