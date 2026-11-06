# Tasks: Облачное хранилище файлов

**Input**: Design documents from `/specs/001-cloud-storage-requirements/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: **TDD обязателен** (решение от 2026-11-06). Для каждой единицы поведения тест пишется **первым** и должен упасть до начала реализации. Тест, прошедший с первого запуска, считается невалидным: он либо ничего не проверяет, либо написан после кода.

**Organization**: Задачи сгруппированы по пользовательским историям. Фазы историй идут **в порядке приоритета** (P1 → P2 → P3), поэтому US11 (P2) стоит перед US6 (P3), хотя в спецификации объявлена позже.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: можно выполнять параллельно (разные файлы, нет зависимостей)
- **[Story]**: к какой истории относится (US1…US12)
- В описании указан точный путь к файлу

## Path Conventions

Монорепозиторий по `plan.md`: `cmd/`, `internal/`, `web/`, `migrations/`, `deploy/`, `tests/`.

> **Ревизия 2026-11-06.** Спецификация получила US11 «Сквозное шифрование папки» и US12 «Выгрузка учётной записи». Соответствующие задачи вынесены из US2 и US1 в собственные фазы; добавлены числовые критерии (глубина 32 уровня, путь 4096 символов, завершение сеансов ≤ 60 секунд, машиночитаемый код причины в отказе).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Инициализация проекта и тестовой инфраструктуры

- [ ] T001 Create monorepo directory skeleton (`cmd/`, `internal/`, `web/`, `migrations/`, `deploy/`, `tests/`) in `cmd/`
- [ ] T002 Initialize Go module with chi, pgx/v5, golang-migrate, x/crypto, pquerna/otp, zerolog and testify in `go.mod`
- [ ] T003 [P] Configure golangci-lint rules in `.golangci.yml`
- [ ] T004 [P] Scaffold React 19 + TypeScript + Vite application in `web/`
- [ ] T005 [P] Create Docker Compose stack (server, postgres, caddy) in `deploy/docker-compose.yml`
- [ ] T006 [P] Create Caddy configuration with automatic TLS and static assets in `deploy/Caddyfile`
- [ ] T007 [P] Add Makefile targets `build`, `test`, `test-unit`, `test-integration`, `test-e2e`, `lint`, `migrate` in `Makefile`
- [ ] T008 [P] Add CI workflow running lint, unit, integration and E2E stages in `.github/workflows/ci.yml`
- [ ] T009 [P] Add ignore, editor and license configuration in `.editorconfig`
- [ ] T010 Create configuration loader with env parsing and validation in `internal/platform/config.go`
- [ ] T011 [P] Create PostgreSQL test harness using testcontainers in `tests/helpers/pg.go`
- [ ] T012 [P] Configure Playwright with web server bootstrap in `web/playwright.config.ts`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Инфраструктура, без которой не работает ни одна история

**⚠️ CRITICAL**: Пока эта фаза не завершена, истории начинать нельзя

- [ ] T013 Create base migration with `users`, `groups`, `group_members`, `sessions`, `devices`; `users.status` is enum `active|blocked|deleted`, `users.role` is enum `member|admin` in `migrations/0001_base.up.sql`
- [ ] T014 [P] Write failing unit tests then implement Argon2id password hashing in `internal/auth/password.go` (tests in `internal/auth/password_test.go`)
- [ ] T015 [P] Write failing unit tests then implement TOTP enrol/verify in `internal/auth/totp.go` (tests in `internal/auth/totp_test.go`)
- [ ] T016 Write failing integration test then implement session middleware with HttpOnly cookie and CSRF in `internal/auth/session.go`
- [ ] T017 Write failing integration test then implement device bearer-token auth with hashed token storage in `internal/auth/device.go`
- [ ] T018 [P] Write failing unit test then implement optional OIDC login in `internal/auth/oidc.go`
- [ ] T019 Write failing unit test then implement ACL evaluator walking ancestors in `internal/acl/acl.go`
- [ ] T020 Create HTTP router with middleware chain (request ID, auth, CSRF, recovery, logging) in `internal/platform/router.go`
- [ ] T021 Define error model with machine codes `forbidden`, `not_found`, `name_conflict`, `invalid_name`, `quota_exceeded`, each response carrying a `hint` with the recommended action, in `internal/platform/errors.go`
- [ ] T022 Write failing integration test then implement append-only audit writer with `UPDATE`/`DELETE` denied at database privilege level in `internal/admin/audit.go`
- [ ] T023 Write failing integration test then implement PostgreSQL-backed job queue (`queued|running|done|failed`, attempts, `run_after`) in `internal/jobs/queue.go`
- [ ] T024 Define storage abstraction interface in `internal/storage/store.go`
- [ ] T025 Write failing integration test then implement content-addressed local storage with `blobs/ab/cd/<hash>` layout in `internal/storage/local.go`
- [ ] T026 Write failing unit tests then implement node repository with NFKC + case-folding normalization, `name` 1…255 chars, no `/` `\`, not `.` or `..`, and per-folder uniqueness, in `internal/nodes/repo.go`
- [ ] T027 Write failing integration test then implement change-log writer assigning monotonic `seq` in the same transaction as node changes in `internal/sync/changelog.go`
- [ ] T028 [P] Add health and metrics endpoints `/healthz` and `/metrics` in `internal/platform/health.go`
- [ ] T029 [P] Add structured request logging with correlation IDs in `internal/platform/logging.go`
- [ ] T030 [P] Create shared test fixtures and builders for users, groups, nodes and shares in `tests/helpers/fixtures.go`

**Checkpoint**: инфраструктура готова — истории можно вести параллельно

---

## Phase 3: User Story 1 - Хранение и упорядочивание файлов (Priority: P1) 🎯 MVP

**Goal**: пользователь загружает файлы и папки, раскладывает по структуре, переименовывает, перемещает, копирует, удаляет и видит занятое место

**Independent Test**: загрузить файлы разного размера, создать вложенную структуру, выполнить переименование/перемещение/копирование/удаление, скачать и побайтово сравнить содержимое

### Tests for User Story 1 (TDD) ⚠️

> Написать до реализации. Все тесты должны упасть.

- [ ] T031 [P] [US1] Contract test for `/uploads` and `/nodes` matching `contracts/openapi.yaml` in `tests/contract/uploads_contract_test.go`
- [ ] T032 [P] [US1] Integration test asserting uploaded and downloaded bytes are identical in `tests/integration/upload_integrity_test.go`
- [ ] T033 [P] [US1] Integration test asserting `state='pending'` nodes return «not ready» and never partial content in `tests/integration/pending_visibility_test.go`
- [ ] T034 [P] [US1] Unit tests for name validation and path limits: `name` 1…255 chars, no `/` `\`, not `.` or `..`, nesting ≤ 32 levels, full path ≤ 4096 chars in `tests/unit/names_test.go`
- [ ] T035 [P] [US1] Unit tests for cycle rejection, copy independence and depth-limit rejection in `tests/unit/nodes_ops_test.go`
- [ ] T036 [P] [US1] Unit test asserting a name changed for platform compatibility is reported with both original and substituted value in `tests/unit/name_warning_test.go`

### Implementation for User Story 1

- [ ] T037 [US1] Implement upload session service (create, offset, chunk, abort, TTL 24h) in `internal/uploads/service.go`
- [ ] T038 [US1] Implement tus-compatible handlers `POST/HEAD/PATCH/DELETE /uploads` in `internal/uploads/handlers.go`
- [ ] T039 [US1] Implement finalize pipeline: incremental SHA-256, temp write, `fsync`, atomic `rename`, blob insert, version insert in `internal/uploads/finalize.go`
- [ ] T040 [P] [US1] Implement expired upload cleanup job in `internal/uploads/gc.go`
- [ ] T041 [US1] Implement node CRUD endpoints in `internal/nodes/handlers.go`
- [ ] T042 [US1] Implement folder listing with stable cursor pagination (limit 1…1000, default 200) in `internal/nodes/list.go`
- [ ] T043 [US1] Implement rename/move/copy rejecting cycles and any move that would exceed 32 levels or 4096 path characters in `internal/nodes/ops.go`
- [ ] T044 [US1] Implement soft delete writing `state='trashed'`, `trashed_at`, `trash_origin_parent_id` in `internal/trash/delete.go`
- [ ] T045 [P] [US1] Implement download with HTTP `Range` support in `internal/nodes/download.go`
- [ ] T046 [US1] Implement bulk move/copy/delete with progress reporting in `internal/nodes/bulk.go`
- [ ] T047 [US1] Implement name validation returning `invalid_name` with the limit and current value in `internal/nodes/names.go`
- [ ] T048 [US1] Implement name-change warning with original and substituted name in the node response in `internal/nodes/name_warning.go`
- [ ] T049 [US1] Build file browser page with folder navigation in `web/src/pages/Files.tsx`
- [ ] T050 [P] [US1] Build uploader with tus client, progress, pause and resume in `web/src/components/Uploader.tsx`
- [ ] T051 [P] [US1] Build breadcrumbs and folder tree sidebar in `web/src/components/Breadcrumbs.tsx`
- [ ] T052 [US1] Build rename/move/copy dialogs in `web/src/components/NodeActions.tsx`
- [ ] T053 [US1] Build storage usage summary (files, versions, trash separately) in `web/src/components/StorageSummary.tsx`
- [ ] T054 [US1] Implement typed API client for nodes and uploads in `web/src/api/nodes.ts`
- [ ] T055 [US1] Implement multi-select and bulk action bar in `web/src/components/BulkBar.tsx`
- [ ] T056 [US1] Write Playwright E2E for upload, rename, move, copy, delete and byte comparison in `web/e2e/files.spec.ts`

**Checkpoint**: US1 полностью функциональна и проверяема независимо — это MVP

---

## Phase 4: User Story 2 - Доступ и права (Priority: P1)

**Goal**: несколько пользователей, группы и уровни доступа; пользователь видит только разрешённое

**Independent Test**: создать двух пользователей и группу, выдать доступ с разными уровнями, проверить, что запрещённые действия отклоняются с кодом `forbidden`, а чужие объекты не видны

### Tests for User Story 2 (TDD) ⚠️

- [ ] T057 [P] [US2] Contract test for `/nodes/{id}/shares` and `/shares/{id}` matching `contracts/openapi.yaml` in `tests/contract/shares_contract_test.go`
- [ ] T058 [P] [US2] Integration test for effective-rights matrix (direct, group, inherited, revoked membership) in `tests/integration/acl_matrix_test.go`
- [ ] T059 [P] [US2] Integration test asserting inaccessible nodes return 404 and never leak existence in `tests/integration/visibility_test.go`
- [ ] T060 [P] [US2] Unit tests for effective-level computation with `level` in `view|comment|edit|own` in `tests/unit/acl_effective_test.go`
- [ ] T061 [P] [US2] Unit tests for inheritance and revocation when group membership is removed in `tests/unit/acl_inherit_test.go`

### Implementation for User Story 2

- [ ] T062 [US2] Implement groups CRUD and membership endpoints in `internal/admin/groups.go`
- [ ] T063 [US2] Implement share endpoints `GET/POST /nodes/{id}/shares`, `DELETE /shares/{id}` in `internal/shares/handlers.go`
- [ ] T064 [US2] Implement effective-rights computation as maximum over direct and group grants in `internal/acl/effective.go`
- [ ] T065 [US2] Implement inheritance up the tree and revocation on membership removal in `internal/acl/inherit.go`
- [ ] T066 [US2] Apply visibility filter so foreign nodes return `not_found` in `internal/nodes/visibility.go`
- [ ] T067 [US2] Implement ownership transfer for a node or subtree in `internal/nodes/owner.go`
- [ ] T068 [US2] Revoke all sessions and device tokens on block or password change within 60 seconds in `internal/auth/revoke.go`
- [ ] T069 [US2] Implement shared-with-me listing in `internal/shares/mine.go`
- [ ] T070 [US2] Build share dialog with user/group search and level selection in `web/src/components/ShareDialog.tsx`
- [ ] T071 [P] [US2] Build shared-with-me page in `web/src/pages/Shared.tsx`
- [ ] T072 [US2] Show effective permission badges on nodes in `web/src/components/PermissionBadge.tsx`
- [ ] T073 [US2] Build admin groups management page in `web/src/pages/admin/Groups.tsx`
- [ ] T074 [US2] Write Playwright E2E for sharing, forbidden actions and membership removal in `web/e2e/sharing.spec.ts`

**Checkpoint**: US1 и US2 работают независимо

---

## Phase 5: User Story 3 - Обмен ссылками (Priority: P2)

**Goal**: публичные ссылки на файлы и папки, с паролем и сроком; ссылки на загрузку без доступа к содержимому

**Independent Test**: создать ссылки, открыть без входа, проверить пароль, срок, отзыв и загрузку по ссылке

### Tests for User Story 3 (TDD) ⚠️

- [ ] T075 [P] [US3] Contract test for `/nodes/{id}/links`, `/links/{id}/revoke` and `/public/*` in `tests/contract/links_contract_test.go`
- [ ] T076 [P] [US3] Integration test for link lifecycle (create, password, expiry, revoke, orphan on delete) in `tests/integration/links_test.go`
- [ ] T077 [P] [US3] Integration test proving a public link exposes no sibling or ancestor metadata in `tests/integration/public_scope_test.go`

### Implementation for User Story 3

- [ ] T078 [US3] Implement link service with hashed token and hashed password, `kind` in `view|upload` in `internal/shares/links.go`
- [ ] T079 [US3] Implement public view and download handlers under `/public/{token}` in `internal/shares/public.go`
- [ ] T080 [US3] Implement upload-link handler under `/public/upload/{token}` in `internal/shares/upload_link.go`
- [ ] T081 [US3] Enforce password, expiry and max-download limits returning 401/410 in `internal/shares/limits.go`
- [ ] T082 [US3] Implement revocation and `orphaned` state on target deletion in `internal/shares/revoke.go`
- [ ] T083 [US3] Isolate public request context in `internal/shares/scope.go`
- [ ] T084 [US3] Build link creation dialog (kind, password, expiry) in `web/src/components/LinkDialog.tsx`
- [ ] T085 [US3] Build public landing page for view and upload links in `web/src/pages/PublicLink.tsx`
- [ ] T086 [US3] Write Playwright E2E for public links in `web/e2e/links.spec.ts`

**Checkpoint**: US3 работает независимо от остальных историй

---

## Phase 6: User Story 4 - Версии и восстановление (Priority: P2)

**Goal**: история версий, восстановление, корзина, понятные сроки хранения

**Independent Test**: создать несколько версий, восстановить раннюю, удалить файл, восстановить из корзины и сверить содержимое

### Tests for User Story 4 (TDD) ⚠️

- [ ] T087 [P] [US4] Contract test for version and trash endpoints in `tests/contract/versions_contract_test.go`
- [ ] T088 [P] [US4] Integration test for version history, restore and trash restore into an occupied path in `tests/integration/versions_test.go`
- [ ] T089 [P] [US4] Unit tests for retention policy: `versions_days=30`, `versions_max_count=100`, `trash_days=30` in `tests/unit/retention_test.go`

### Implementation for User Story 4

- [ ] T090 [US4] Implement version creation on content replacement (previous preserved), `origin` in `upload|sync|restore|collab|external` in `internal/versions/service.go`
- [ ] T091 [US4] Implement history listing with author, time, size and origin in `internal/versions/handlers.go`
- [ ] T092 [US4] Implement restore-version creating a new version with `origin='restore'` in `internal/versions/restore.go`
- [ ] T093 [P] [US4] Implement download of a specific version in `internal/versions/download.go`
- [ ] T094 [US4] Implement retention job applying 30 days / 100 versions and 30 days of trash in `internal/jobs/retention.go`
- [ ] T095 [US4] Implement trash listing, restore and empty with confirmation in `internal/trash/handlers.go`
- [ ] T096 [US4] Implement restore into occupied path creating a distinctly named copy in `internal/trash/restore.go`
- [ ] T097 [P] [US4] Implement separate accounting of versions and trash volume in `internal/admin/quota_calc.go`
- [ ] T098 [US4] Build version history panel with restore and download in `web/src/components/VersionHistory.tsx`
- [ ] T099 [P] [US4] Build trash page with restore and empty actions in `web/src/pages/Trash.tsx`
- [ ] T100 [US4] Write Playwright E2E for versions and trash in `web/e2e/versions.spec.ts`

**Checkpoint**: US4 работает независимо

---

## Phase 7: User Story 5 - Синхронизация между устройствами (Priority: P2)

**Goal**: изменения доезжают до всех устройств, офлайн-работа, явные конфликты без потери данных

**Independent Test**: два устройства, изменение на одном появляется на другом; офлайн-правка одного файла на обоих сохраняет обе версии

### Tests for User Story 5 (TDD) ⚠️

- [ ] T101 [P] [US5] Contract test for `/devices`, `/sync/changes`, `/sync/push`, `/sync/tree` in `tests/contract/sync_contract_test.go`
- [ ] T102 [P] [US5] Integration test proving both versions survive a concurrent offline edit in `tests/integration/conflict_test.go`
- [ ] T103 [P] [US5] Integration test proving ordering follows `seq`, not `mtime`, in `tests/integration/ordering_test.go`
- [ ] T104 [P] [US5] Unit tests for conflict-copy naming and `client_op_id` idempotency in `tests/unit/conflict_naming_test.go`

### Implementation for User Story 5

- [ ] T105 [US5] Implement device registration, listing and revoke endpoints in `internal/sync/devices.go`
- [ ] T106 [US5] Implement `GET /sync/changes?since=` with ordered `seq` and `resync_required` in `internal/sync/pull.go`
- [ ] T107 [US5] Implement `POST /sync/push` with idempotent `client_op_id` in `internal/sync/push.go`
- [ ] T108 [US5] Implement conflict detection by `base_version_id` and conflict copy creation in `internal/sync/conflict.go`
- [ ] T109 [US5] Implement `POST /nodes/{id}/resolve` supporting `keep: current|conflict` in `internal/sync/resolve.go`
- [ ] T110 [US5] Implement full resync snapshot endpoint `/sync/tree` in `internal/sync/tree.go`
- [ ] T111 [P] [US5] Implement per-device cursor persistence in `internal/sync/cursor.go`
- [ ] T112 [P] [US5] Implement change-log retention and compaction (90 days / 1 000 000 records) in `internal/jobs/changelog_gc.go`
- [ ] T113 [US5] Create sync-agent entrypoint with config parsing in `cmd/sync-agent/main.go`
- [ ] T114 [US5] Implement durable local operation journal in `internal/agent/journal.go`
- [ ] T115 [P] [US5] Implement filesystem watcher with debouncing in `internal/agent/watcher.go`
- [ ] T116 [US5] Implement sync loop: pull, apply, push, advance cursor in `internal/agent/sync.go`
- [ ] T117 [US5] Implement offline queue preserving `client_op_id` across restarts in `internal/agent/offline.go`
- [ ] T118 [US5] Implement local name normalization and reserved-name substitution with user warning in `internal/agent/names.go`
- [ ] T119 [US5] Implement local conflict copies and local trash instead of deleting unknown files in `internal/agent/conflict.go`
- [ ] T120 [P] [US5] Add agent build targets for Windows, macOS and Linux in `Makefile`
- [ ] T121 [US5] Build device management page with revoke action in `web/src/pages/Devices.tsx`

**Checkpoint**: US5 работает независимо; US1…US4 не сломаны

---

## Phase 8: User Story 11 - Сквозное шифрование папки (Priority: P2)

**Goal**: содержимое отдельной папки не читается никем, кроме владельца и доверенных лиц, включая администратора; потерянные возможности известны до включения

**Independent Test**: включить шифрование на папке, убедиться что предупреждение показано до подтверждения, поиск по содержимому ничего не находит, серверная выгрузка даёт нечитаемые данные, отключение невозможно

### Tests for User Story 11 (TDD) ⚠️

- [ ] T122 [P] [US11] Contract test for `POST /nodes/{id}/e2ee` and the disabled-capabilities payload matching `contracts/openapi.yaml` in `tests/contract/e2ee_contract_test.go`
- [ ] T123 [P] [US11] Integration test asserting encryption is irreversible, excludes the folder from content search, and rejects mixing with plaintext children in `tests/integration/e2ee_test.go`

### Implementation for User Story 11

- [ ] T124 [US11] Implement `e2ee_folders` model and `POST /nodes/{id}/e2ee` with `key_wrapped`, `kdf_params` and `recovery_mode` in `none|mnemonic` in `internal/e2ee/folders.go`
- [ ] T125 [US11] Return the explicit list of capabilities disabled for encrypted folders (content search, preview without download, co-editing, resumable web upload) in `internal/e2ee/limits.go`
- [ ] T126 [US11] Exclude encrypted folders from text extraction and content search in `internal/e2ee/index_guard.go`
- [ ] T127 [US11] Build encryption setup dialog that lists lost capabilities and requires explicit confirmation in `web/src/components/EncryptionDialog.tsx`
- [ ] T128 [US11] Write Playwright E2E for encryption enablement, blocked search and rejected disable in `web/e2e/encryption.spec.ts`

**Checkpoint**: US11 работает независимо

---

## Phase 9: User Story 6 - Поиск и упорядочивание по смыслу (Priority: P3)

**Goal**: поиск по имени, типу, дате, владельцу, меткам, свойствам и содержимому — только по доступному

**Independent Test**: создать набор файлов с метками и свойствами, проверить поиск по каждому признаку и их комбинации; чужое не находится

### Tests for User Story 6 (TDD) ⚠️

- [ ] T129 [P] [US6] Contract test for `/search`, `/nodes/{id}/tags`, `/nodes/{id}/properties` in `tests/contract/search_contract_test.go`
- [ ] T130 [P] [US6] Integration test asserting search never returns inaccessible objects in `tests/integration/search_isolation_test.go`
- [ ] T131 [P] [US6] Unit tests for combined filter semantics (logical AND) in `tests/unit/search_filters_test.go`

### Implementation for User Story 6

- [ ] T132 [US6] Implement tags service and endpoints, `name` 1…64, unique per owner, in `internal/search/tags.go`
- [ ] T133 [US6] Implement user properties CRUD with `value_type` in `text|number|date|bool`, `key` 1…64, `value` ≤ 4096 in `internal/search/properties.go`
- [ ] T134 [US6] Implement name search with `pg_trgm` and `unaccent` in `internal/search/names.go`
- [ ] T135 [US6] Implement full-text index and query over extracted text in `internal/search/fulltext.go`
- [ ] T136 [US6] Implement asynchronous text extraction worker with isolated converter and size limits in `internal/jobs/extract.go`
- [ ] T137 [US6] Apply ACL filter to all search queries in `internal/search/filter.go`
- [ ] T138 [P] [US6] Implement saved filter conditions (SHOULD) in `internal/search/saved.go`
- [ ] T139 [US6] Build search page with faceted filters in `web/src/pages/Search.tsx`
- [ ] T140 [P] [US6] Build metadata panel for tags and properties in `web/src/components/MetadataPanel.tsx`
- [ ] T141 [US6] Write Playwright E2E for search and metadata in `web/e2e/search.spec.ts`

**Checkpoint**: US6 работает независимо

---

## Phase 10: User Story 7 - Совместная работа (Priority: P3)

**Goal**: комментарии с ветками и упоминаниями, уведомления и непрочитанное; одновременное редактирование (SHOULD)

**Independent Test**: оставить комментарий, ответить, упомянуть пользователя и проверить уведомление и признак непрочитанного

### Tests for User Story 7 (TDD) ⚠️

- [ ] T142 [P] [US7] Contract test for comment and notification endpoints in `tests/contract/comments_contract_test.go`
- [ ] T143 [P] [US7] Integration test for threads, mentions and notification creation in `tests/integration/comments_test.go`
- [ ] T144 [P] [US7] Unit tests for mention parsing and deduplication in `tests/unit/mentions_test.go`

### Implementation for User Story 7

- [ ] T145 [US7] Implement comments service with soft delete and `body` 1…10000 in `internal/comments/service.go`
- [ ] T146 [US7] Implement comment endpoints with thread nesting and mention resolution in `internal/comments/handlers.go`
- [ ] T147 [US7] Implement notification service and unread counters in `internal/notify/service.go`
- [ ] T148 [US7] Implement per-user notification preferences in `internal/notify/prefs.go`
- [ ] T149 [US7] Build comments panel with threads and mention autocomplete in `web/src/components/CommentsPanel.tsx`
- [ ] T150 [P] [US7] Build notifications drawer and preferences page in `web/src/components/Notifications.tsx`
- [ ] T151 [US7] Implement WebSocket channel and Yjs CRDT for concurrent editing (SHOULD) in `internal/collab/hub.go`
- [ ] T152 [US7] Persist collaboration snapshots as new versions in `internal/collab/snapshot.go`
- [ ] T153 [US7] Write Playwright E2E for comments and mentions in `web/e2e/comments.spec.ts`

**Checkpoint**: US7 работает независимо; совместное редактирование может быть отложено

---

## Phase 11: User Story 8 - Администрирование и контроль (Priority: P3)

**Goal**: пользователи, группы, квоты, политики, журнал действий, состояние и надёжность резервного копирования

**Independent Test**: задать квоту, превысить её, проверить отказ; выполнить админ-действие и найти его в журнале; смоделировать неудачу бэкапа и получить уведомление

### Tests for User Story 8 (TDD) ⚠️

- [ ] T154 [P] [US8] Contract test for `/admin/*` endpoints in `tests/contract/admin_contract_test.go`
- [ ] T155 [P] [US8] Integration test for quota enforcement returning `quota_exceeded` with remaining bytes in `tests/integration/quota_test.go`
- [ ] T156 [P] [US8] Integration test for audit completeness and immutability in `tests/integration/audit_test.go`
- [ ] T157 [P] [US8] Integration test for blocked-user data transfer and preservation in `tests/integration/transfer_test.go`
- [ ] T158 [P] [US8] Integration test asserting an admin notification is created when backup creation or verification fails in `tests/integration/backup_alert_test.go`

### Implementation for User Story 8

- [ ] T159 [US8] Implement quota enforcement on upload and copy in `internal/admin/quota.go`
- [ ] T160 [P] [US8] Implement quota reporting (files, versions, trash) in `internal/admin/quota_report.go`
- [ ] T161 [US8] Implement retention policy endpoints at system and user scope in `internal/admin/policies.go`
- [ ] T162 [US8] Implement audit query endpoint with actor/action/time filters in `internal/admin/audit_handlers.go`
- [ ] T163 [US8] Implement admin user management (create, block, role, delete) in `internal/admin/users.go`
- [ ] T164 [US8] Implement blocked-user data transfer as a job in `internal/admin/transfer.go`
- [ ] T165 [US8] Implement backup status endpoint in `internal/admin/backup_status.go`
- [ ] T166 [US8] Implement admin notification on failed backup creation or verification in `internal/admin/backup_alert.go`
- [ ] T167 [P] [US8] Write backup script (`pg_dump` + restic snapshot) in `deploy/backup/backup.sh`
- [ ] T168 [P] [US8] Write restore drill script into isolated namespace in `deploy/backup/restore-drill.sh`
- [ ] T169 [US8] Build admin users page with block, quota and transfer actions in `web/src/pages/admin/Users.tsx`
- [ ] T170 [P] [US8] Build admin audit log page in `web/src/pages/admin/Audit.tsx`
- [ ] T171 [P] [US8] Build admin backup status page in `web/src/pages/admin/Backup.tsx`

**Checkpoint**: US8 работает независимо

---

## Phase 12: User Story 9 - Доступ из сторонних средств и внешние хранилища (Priority: P3)

**Goal**: чтение и запись из сторонних клиентов; внешние хранилища в общей структуре

**Independent Test**: подключить WebDAV-клиент и выполнить чтение/запись; подключить внешнее хранилище и увидеть его содержимое

### Tests for User Story 9 (TDD) ⚠️

- [ ] T172 [P] [US9] Contract test for WebDAV methods (PROPFIND, GET, PUT, MOVE, DELETE) in `tests/contract/dav_contract_test.go`
- [ ] T173 [P] [US9] Integration test for WebDAV rights parity with the HTTP API in `tests/integration/dav_test.go`

### Implementation for User Story 9

- [ ] T174 [US9] Implement WebDAV layer over the shared service layer in `internal/dav/handler.go`
- [ ] T175 [US9] Implement external mount model and endpoints, `kind` in `webdav` only, in `internal/storage/mounts.go`
- [ ] T176 [US9] Implement WebDAV client adapter for external mounts in `internal/storage/webdav_client.go`
- [ ] T177 [US9] Handle unreachable mounts explicitly without showing them empty in `internal/storage/mount_state.go`
- [ ] T178 [P] [US9] Implement outbound event delivery to configured endpoints (SHOULD) in `internal/notify/webhooks.go`
- [ ] T179 [US9] Build external mounts UI in `web/src/pages/ExternalStorage.tsx`

**Checkpoint**: US9 работает независимо

---

## Phase 13: User Story 10 - Обмен между установками (Priority: P3, SHOULD)

**Goal**: две установки делятся папками напрямую

**Independent Test**: связать две установки, выдать доступ к папке, проверить выдачу и отзыв

### Tests for User Story 10 (TDD) ⚠️

- [ ] T180 [P] [US10] Contract test for federation endpoints in `tests/contract/federation_contract_test.go`
- [ ] T181 [P] [US10] Integration test across two instances covering grant and revoke in `tests/integration/federation_test.go`

### Implementation for User Story 10

- [ ] T182 [US10] Implement federation peer model and registration endpoint with `state` in `pending|active|revoked` in `internal/federation/peers.go`
- [ ] T183 [US10] Implement invite token exchange establishing mutual trust in `internal/federation/invite.go`
- [ ] T184 [US10] Implement proxied access to shared objects across peers in `internal/federation/proxy.go`
- [ ] T185 [US10] Implement revocation preserving already-downloaded copies in `internal/federation/revoke.go`

**Checkpoint**: US10 работает независимо; может быть отложена без ущерба для остальных

---

## Phase 14: User Story 12 - Выгрузка учётной записи (Priority: P3)

**Goal**: пользователь забирает все свои данные одним действием — архив со структурой папок и сведениями о файлах

**Independent Test**: запросить выгрузку, дождаться готовности, скачать архив и сверить, что все файлы и папки присутствуют с сохранением путей, а метаданные приложены отдельно

### Tests for User Story 12 (TDD) ⚠️

- [ ] T186 [P] [US12] Contract test for `POST /export` and `GET /export/{jobId}` matching `contracts/openapi.yaml` in `tests/contract/export_contract_test.go`
- [ ] T187 [P] [US12] Integration test asserting account export contains every file and folder with preserved paths and metadata, in `tests/integration/export_test.go`

### Implementation for User Story 12

- [ ] T188 [US12] Implement export job producing a tar archive with a JSON metadata manifest and TTL cleanup in `internal/export/export.go`
- [ ] T189 [US12] Implement export status and download endpoints reporting volume and contents before download in `internal/export/handlers.go`
- [ ] T190 [US12] Build export page with request, progress and download in `web/src/pages/Export.tsx`
- [ ] T191 [US12] Write Playwright E2E for export completeness and encrypted-folder passthrough in `web/e2e/export.spec.ts`

**Checkpoint**: US12 работает независимо

---

## Phase 15: Polish & Cross-Cutting Concerns

**Purpose**: сквозные улучшения, проверка критериев успеха, эксплуатационная готовность

- [ ] T192 [P] Write k6 scenarios for folder listing, search and download start in `tests/perf/`
- [ ] T193 Write scale scenario at 10 users, 1 ТБ and 10 000 objects per folder recording results against SC-016 in `tests/perf/scale.js`
- [ ] T194 Run all 14 scenarios from `quickstart.md` and record results in `tests/quickstart-results.md`
- [ ] T195 [P] Harden HTTP layer: security headers, CSRF everywhere, rate limiting on auth and public links in `internal/platform/security.go`
- [ ] T196 [P] Add loading, empty and error states across all pages in `web/src/components/States.tsx`
- [ ] T197 [P] Accessibility pass: keyboard navigation, focus order, labels, contrast in `web/src/styles/a11y.css`
- [ ] T198 [P] Extract user-facing strings and add localization scaffolding in `web/src/i18n/`
- [ ] T199 [P] Add alerts for failed backup, failed extraction and queue backlog in `deploy/monitoring/alerts.yml`
- [ ] T200 Measure restore time and record RTO evidence in `deploy/backup/RTO.md`
- [ ] T201 Build FR traceability matrix (FR → task → test) in `specs/001-cloud-storage-requirements/traceability.md`
- [ ] T202 Verify TDD discipline: every implementation task has a preceding failing test; record coverage per package in `tests/coverage-report.md`
- [ ] T203 [P] Run dependency and container vulnerability audit, record findings in `docs/security-audit.md`
- [ ] T204 Build release artifacts with signed binaries for all platforms in `Makefile`
- [ ] T205 Final review of `spec.md` non-goals against implemented scope in `specs/001-cloud-storage-requirements/plan.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: без зависимостей
- **Foundational (Phase 2)**: зависит от Phase 1; **блокирует все истории**
- **US1–US2 (Phase 3–4)**: P1
- **US3, US4, US5, US11 (Phase 5–8)**: P2
- **US6–US10, US12 (Phase 9–14)**: P3
- **Polish (Phase 15)**: после нужных историй

### TDD-порядок внутри каждой истории

```text
1. Написать тесты из блока «Tests (TDD)»
2. Запустить — убедиться, что КАЖДЫЙ падает по ожидаемой причине
3. Реализовать минимально достаточно, чтобы тест прошёл
4. Рефакторинг при зелёных тестах
5. Перейти к следующей задаче
```

**Запрещено**: писать реализацию, у которой нет предшествующего падающего теста. T202 проверяет это формально.

### User Story Dependencies

| История | Зависит от | Может идти параллельно с |
|---|---|---|
| US1 | Phase 2 | US2, US7 |
| US2 | Phase 2 | US1, US3 |
| US3 | Phase 2, T026 | US4, US6 |
| US4 | Phase 2, T039 | US3, US5 |
| US5 | Phase 2, T105–T112 (сервер) | US6, US8 |
| US11 | Phase 2, T026, T135 (индексация) | US6, US4 |
| US6 | Phase 2, T026 | US7, US8, US11 |
| US7 | Phase 2, T026 | US6 |
| US8 | Phase 2, T022, T023 | US5, US6 |
| US9 | Phase 2, T063–T066 | US10 |
| US10 | Phase 2, US9 (транспорт) | — |
| US12 | Phase 2, T025 | US11 (общая модель шифрования) |

> **US11 и US12 независимы от остальных историй по функциональности**, но US11 опирается на исключение папок из индексации (T135), а US12 — на модель блобов (T025). Если эти задачи ещё не сделаны, US11 выполняется с заглушкой индексатора, US12 — с прямым обходом хранилища.

### Within Each User Story

- Тесты — **до** реализации (обязательно)
- Модели и миграции — до сервисов
- Сервисы — до обработчиков
- Обработчики — до UI
- UI — до E2E

### Parallel Opportunities

- Phase 1: T003–T009, T011–T012
- Phase 2: T014, T015, T018, T028–T030
- После Phase 2 истории US1…US2 можно вести параллельно
- Внутри истории параллельны тестовые задачи разных файлов и независимые UI-компоненты
- US11 и US12 можно вести параллельно с US6–US10: они не пересекаются по файлам

---

## Parallel Example: User Story 1

```bash
# Сначала — все тесты US1 параллельно (разные файлы):
Task: "Contract test for /uploads and /nodes in tests/contract/uploads_contract_test.go"
Task: "Integration test for byte integrity in tests/integration/upload_integrity_test.go"
Task: "Integration test for pending visibility in tests/integration/pending_visibility_test.go"
Task: "Unit tests for name validation and path limits in tests/unit/names_test.go"

# Убедиться, что все падают. Затем — реализация; UI-компоненты параллельны:
Task: "Build uploader in web/src/components/Uploader.tsx"
Task: "Build breadcrumbs in web/src/components/Breadcrumbs.tsx"
```

## Parallel Example: User Story 11

```bash
Task: "Contract test for POST /nodes/{id}/e2ee in tests/contract/e2ee_contract_test.go"
Task: "Integration test for irreversible encryption and search exclusion in tests/integration/e2ee_test.go"

# Затем реализация и UI параллельны:
Task: "Implement e2ee folder model in internal/e2ee/folders.go"
Task: "Build encryption setup dialog in web/src/components/EncryptionDialog.tsx"
```

## Parallel Example: User Story 12

```bash
Task: "Contract test for POST /export and GET /export/{jobId} in tests/contract/export_contract_test.go"
Task: "Integration test for export completeness in tests/integration/export_test.go"

Task: "Implement export job in internal/export/export.go"
Task: "Build export page in web/src/pages/Export.tsx"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup
2. Phase 2: Foundational
3. Phase 3: US1 — тесты → падение → реализация
4. **STOP and VALIDATE**: загрузить/скачать/переместить/удалить, сверить байты
5. Это рабочее личное хранилище без совместного доступа

### Incremental Delivery

1. Setup + Foundational → фундамент
2. US1 → MVP
3. US2 → многопользовательское хранилище с правами
4. US3 → ссылки
5. US4 → версии и корзина
6. US5 → синхронизация
7. US11 → сквозное шифрование
8. US6…US10, US12 → расширения по приоритету

### Parallel Team Strategy

1. Команда завершает Phase 1 и Phase 2
2. Далее:
   - Разработчик A: US1
   - Разработчик B: US2
   - Разработчик C: US3 + US4
   - Разработчик D: US5 (сервер + агент)
   - Разработчик E: US11 + US12
3. US6…US10 — после стабилизации ядра

---

## Notes

- `[P]` = разные файлы, нет зависимостей
- `[Story]` связывает задачу с историей для трассировки
- **TDD обязателен**: тест → падение → реализация → зелёный → рефакторинг
- Тестовые задачи: 40 отдельных в блоках «Tests (TDD)» + 12 задач Phase 2 в формате «сначала падающий тест, затем реализация» = 52 задачи с тестами из 205
- Проверка соблюдения TDD — T202
- Коммит после каждой задачи или логической группы
- Останавливаться на checkpoint и проверять историю независимо
- Избегать: реализации без теста, расплывчатых задач, конфликтов по одному файлу