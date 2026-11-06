# Data Model: Облачное хранилище файлов

**Feature**: `specs/001-cloud-storage-requirements` | **Date**: 2026-11-06
**Источники**: `spec.md` (Key Entities, FR-001…FR-104), `research.md` (R2, R3, R5–R9)

Модель технологически конкретна (PostgreSQL), но сущности описаны в терминах требований. Имена таблиц — в нижнем регистре, множественном числе.

---

## Обзор связей

```text
User ─┬─< GroupMember >─ Group
      ├─< Session
      ├─< Device ──< SyncCursor
      ├─< Share (grantee: User|Group) >─ Node
      ├─< Comment >─ Node
      ├─< Notification
      └─< AuditEvent (actor)

Node (file|folder) ─┬─< Node (parent → children)
                    ├─< Version ──> Blob
                    ├─< Share
                    ├─< Link
                    ├─< NodeTag >─ Tag
                    ├─< Property
                    ├─< Comment
                    └─< ExternalMount (folder with kind='external')

UploadSession ──> Node (pending)
Blob ──< Version
ChangeLog ──> Node
FederationPeer ──< Share (scope='federated')
RetentionPolicy ──> (система | пользователь)
```

---

## Технологические сущности

### `users`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `email` | citext UNIQUE NOT NULL | нормализуется к нижнему регистру |
| `display_name` | text NOT NULL | 1…200 символов |
| `password_hash` | text NULL | Argon2id; NULL для OIDC-only учётных записей |
| `status` | enum(`active`,`blocked`,`deleted`) NOT NULL | `deleted` не удаляет строку — см. FR-101…FR-104 |
| `role` | enum(`member`,`admin`) NOT NULL DEFAULT `member` | FR-077 |
| `quota_bytes` | bigint NULL | NULL = квота группы/системы |
| `mfa_secret` | bytea NULL | TOTP; NULL = MFA не настроена |
| `mfa_enabled` | boolean NOT NULL DEFAULT false | |
| `locale` | text NOT NULL DEFAULT `ru` | |
| `created_at` / `updated_at` | timestamptz NOT NULL | |

Инварианты:
- `status='blocked'` инвалидирует все `sessions` и `devices` (FR-007).
- `status='deleted'` требует, чтобы все `nodes.owner_id` были переназначены либо данные удалены явным действием (FR-102…FR-104).

Переходы состояний: `active → blocked → active`, `active|blocked → deleted`. Обратного перехода из `deleted` нет.

### `groups`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `name` | text UNIQUE NOT NULL | 1…200 |
| `quota_bytes` | bigint NULL | |
| `created_at` / `updated_at` | timestamptz | |

### `group_members`

| Поле | Тип | Правила |
|---|---|---|
| `group_id` | uuid FK → groups | PK(часть) |
| `user_id` | uuid FK → users | PK(часть) |
| `added_at` | timestamptz | |

### `sessions`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `user_id` | uuid FK | |
| `token_hash` | bytea UNIQUE | хранится хеш, не токен |
| `user_agent` / `ip` | text | только для отображения |
| `created_at` / `last_seen_at` / `expires_at` | timestamptz | |

### `devices`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `user_id` | uuid FK | |
| `name` | text | имя устройства |
| `token_hash` | bytea UNIQUE | |
| `platform` | enum(`windows`,`macos`,`linux`,`android`,`ios`,`web`,`other`) | FR-052 |
| `last_seen_at` | timestamptz | |
| `revoked_at` | timestamptz NULL | |

---

## Сущности предметной области

### `nodes`

Узел дерева — файл или папка. Единая таблица выбрана ради однозначности путей и простоты наследования прав.

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `owner_id` | uuid FK → users NOT NULL | текущий владелец (FR-006) |
| `parent_id` | uuid FK → nodes NULL | NULL только для корня учётной записи |
| `kind` | enum(`folder`,`file`) NOT NULL | |
| `name` | text NOT NULL | 1…255 символов; не `.`/`..`; без `/` и `\` |
| `name_folded` | text NOT NULL | NFKC + case folding; участвует в уникальности (R7) |
| `current_version_id` | uuid FK → versions NULL | NULL для папок и незавершённых файлов |
| `size_bytes` | bigint NOT NULL DEFAULT 0 | для папки — сумма по детям (денормализация, пересчитывается) |
| `mime_type` | text NULL | определяется при загрузке |
| `state` | enum(`active`,`conflict`,`trashed`,`pending`) NOT NULL | см. переходы |
| `trashed_at` | timestamptz NULL | |
| `trash_origin_parent_id` | uuid NULL | исходное место (FR-031) |
| `conflict_of` | uuid FK → nodes NULL | для конфликтной копии (R6) |
| `e2ee_folder_id` | uuid FK → e2ee_folders NULL | NULL = не зашифрован |
| `external_mount_id` | uuid FK → external_mounts NULL | NULL = локальный |
| `created_at` / `updated_at` | timestamptz | |

Инварианты и ограничения:
- **UNIQUE (`parent_id`, `name_folded`) WHERE `state <> 'trashed'`** — реализует FR-012 и R7 (регистронезависимая уникальность) одновременно.
- `kind='folder'` ⇒ `current_version_id IS NULL` и `size_bytes` производна от детей.
- Цикл запрещён: `parent_id` не может быть потомком узла (FR-018).
- `state='pending'` — узел виден только владельцу и только как «загружается»; в листингах другим не показывается (FR-023).
- `state='trashed'` ⇒ `trashed_at IS NOT NULL`.

Переходы состояний:

```text
pending ──(загрузка завершена, блоб зафиксирован)──> active
active  ──(удаление)──> trashed ──(восстановление)──> active
trashed ──(истечение срока / явная очистка)──> (удалён)
active  ──(конфликт при синхронизации)──> conflict ──(выбор основной)──> active
conflict ──(оставлен как копия)──> active (с пометкой conflict_of)
```

### `versions`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `node_id` | uuid FK → nodes NOT NULL | |
| `blob_hash` | bytea FK → blobs.hash NOT NULL | |
| `size_bytes` | bigint NOT NULL | |
| `created_by` | uuid FK → users NULL | NULL для системных операций |
| `created_at` | timestamptz NOT NULL | |
| `origin` | enum(`upload`,`sync`,`restore`,`collab`,`external`) | источник |
| `seq` | bigint | монотонный номер версии в рамках узла |

Инварианты:
- `(node_id, seq)` UNIQUE.
- Восстановление версии не удаляет другие версии (FR-029) — создаётся новая запись с `origin='restore'`.
- Политика хранения удаляет версии, но только если на них не ссылается `nodes.current_version_id`.

### `blobs`

| Поле | Тип | Правила |
|---|---|---|
| `hash` | bytea PK | SHA-256 содержимого |
| `size_bytes` | bigint NOT NULL | |
| `ref_count` | integer NOT NULL DEFAULT 0 | денормализация для GC |
| `created_at` | timestamptz | |
| `encrypted` | boolean NOT NULL DEFAULT false | для E2EE |
| `storage_key` | text NOT NULL | путь в контентно-адресуемом хранилище (R3) |

Инварианты:
- Блоб физически существует до появления ссылки на него: порядок «записать во временный файл → `fsync` → `rename` → вставить `blobs` → вставить `versions`» (FR-023).
- `ref_count = 0` в течение TTL ⇒ кандидат на удаление фоновой задачей.
- Для `encrypted=true` `hash` — хеш **шифротекста**, сервер не знает открытого содержимого.

### `upload_sessions`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | идентификатор сессии tus |
| `user_id` | uuid FK | |
| `target_parent_id` | uuid FK → nodes | |
| `node_id` | uuid FK → nodes NULL | заполняется при создании узла |
| `declared_name` / `declared_size` / `declared_mime` | text / bigint / text | |
| `received_bytes` | bigint NOT NULL DEFAULT 0 | монотонно растёт |
| `expected_hash` | bytea NULL | если клиент передал |
| `temp_path` | text NOT NULL | временный файл |
| `state` | enum(`active`,`completed`,`aborted`,`expired`) | |
| `expires_at` | timestamptz | TTL |
| `created_at` / `updated_at` | timestamptz | |

Инварианты:
- `received_bytes` может только увеличиваться; уменьшение — ошибка (защита от повторной передачи).
- `state='completed'` требует совпадения вычисленного хеша с `expected_hash`, если он был задан (FR-024).

### `shares`

Внутренние права. Субъект — пользователь **или** группа.

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `node_id` | uuid FK → nodes NOT NULL | |
| `grantee_user_id` | uuid FK → users NULL | XOR с `grantee_group_id` |
| `grantee_group_id` | uuid FK → groups NULL | |
| `level` | enum(`view`,`comment`,`edit`,`own`) NOT NULL | FR-003 |
| `granted_by` | uuid FK → users | |
| `created_at` | timestamptz | |
| `expires_at` | timestamptz NULL | |

Инварианты:
- CHECK: ровно одно из `grantee_user_id` / `grantee_group_id` заполнено.
- UNIQUE(`node_id`, `grantee_user_id`) и UNIQUE(`node_id`, `grantee_group_id`).
- Итоговые права = максимум по прямым выдачам и выдачам через группы (FR-004). Наследование — вверх по дереву от узла к корню.
- `level='own'` допустим только для пользователя, не для группы.

### `links`

Публичные ссылки (FR-036…FR-043).

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `node_id` | uuid FK → nodes NOT NULL | |
| `created_by` | uuid FK → users | |
| `kind` | enum(`view`,`upload`) NOT NULL | FR-039 |
| `token_hash` | bytea UNIQUE NOT NULL | в URL — токен, в БД — хеш |
| `password_hash` | text NULL | Argon2id |
| `expires_at` | timestamptz NULL | FR-038 |
| `max_downloads` | integer NULL | FR-042 (SHOULD) |
| `download_count` | integer NOT NULL DEFAULT 0 | |
| `revoked_at` | timestamptz NULL | FR-039, FR-041 |
| `created_at` | timestamptz | |

Инварианты:
- `kind='upload'` разрешает только запись в папку и не даёт списка содержимого (FR-039).
- Ссылка на удалённый узел не работает, но остаётся в списке владельца со статусом «объект удалён» (FR-040, FR-041).

### `tags` и `node_tags`

| `tags` | | |
|---|---|---|
| `id` | uuid PK | |
| `owner_id` | uuid FK → users | метки личные |
| `name` | text | 1…64; UNIQUE(`owner_id`,`name`) |
| `color` | text NULL | |

| `node_tags` | | |
|---|---|---|
| `node_id` | uuid FK | PK(часть) |
| `tag_id` | uuid FK | PK(часть) |

### `properties`

Пользовательские свойства (FR-054).

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `node_id` | uuid FK | |
| `owner_id` | uuid FK | свойства личные для владельца |
| `key` | text | 1…64 |
| `value` | text | до 4096 |
| `value_type` | enum(`text`,`number`,`date`,`bool`) | для отбора |

UNIQUE(`node_id`, `owner_id`, `key`).

### `comments`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `node_id` | uuid FK | |
| `parent_id` | uuid FK → comments NULL | ветки (FR-061) |
| `author_id` | uuid FK → users | |
| `body` | text | 1…10000 |
| `mentions` | uuid[] | FR-062 |
| `created_at` / `edited_at` / `deleted_at` | timestamptz | мягкое удаление |

Инварианты:
- Комментарий существует только при существующем узле; при удалении узла удаляется каскадно (FR-065).
- `parent_id` должен принадлежать тому же `node_id`.

### `notifications`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `user_id` | uuid FK | |
| `kind` | enum(`share`,`comment`,`mention`,`quota`,`backup`,`export`,`conflict`,`federation`) | |
| `payload` | jsonb | |
| `read_at` | timestamptz NULL | FR-063 |
| `created_at` | timestamptz | |

### `trash` — представление

Отдельной таблицы нет: корзина — это `nodes` со `state='trashed'`. Восстановление возвращает `state='active'` и `parent_id = trash_origin_parent_id`; если место занято — создаётся копия с суффиксом (FR-034).

### `change_log`

Журнал изменений для синхронизации (R5).

| Поле | Тип | Правила |
|---|---|---|
| `seq` | bigint PK (монотонный) | курсор устройства |
| `node_id` | uuid FK → nodes | |
| `op` | enum(`upsert`,`delete`,`rename`,`move`,`restore`,`conflict`) | |
| `version_id` | uuid FK → versions NULL | |
| `payload` | jsonb | минимально необходимое |
| `actor_device_id` | uuid FK → devices NULL | |
| `created_at` | timestamptz | |

Инварианты:
- `seq` выдаётся в той же транзакции, что и изменение узла — иначе курсор разъедется.
- Ретенция: 90 дней или 1 000 000 записей (настраивается). Устройство, чей курсор старше горизонта, выполняет полную сверку.

### `sync_cursors`

| Поле | Тип | Правила |
|---|---|---|
| `device_id` | uuid FK → devices | PK |
| `last_seq` | bigint NOT NULL | |
| `updated_at` | timestamptz | |

### `conflicts` — представление

Отдельной таблицы не требуется: конфликт выражается парой узлов, где `conflict_of` указывает на исходный. Состояние снимается действием пользователя.

### `e2ee_folders`

| Поле | Тип | Правила |
|---|---|---|
| `node_id` | uuid FK → nodes PK | папка, включившая шифрование |
| `key_wrapped` | bytea NOT NULL | ключ папки, обёрнутый ключом пользователя |
| `kdf_params` | jsonb NOT NULL | Argon2id: соль, параметры |
| `recovery_mode` | enum(`none`,`mnemonic`) NOT NULL | FR-091 |
| `enabled_at` | timestamptz | |

Инварианты:
- Включение необратимо: `DELETE` запрещён, снятие шифрования не поддерживается (R9).
- Дочерние узлы наследуют `e2ee_folder_id`; смешивание зашифрованных и незашифрованных детей в одной папке запрещено.

### `external_mounts`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `owner_id` | uuid FK | |
| `kind` | enum(`webdav`) | R15 |
| `base_url` | text | |
| `credentials_enc` | bytea | шифруются ключом сервера |
| `state` | enum(`ok`,`unreachable`,`auth_failed`) | FR-080 |
| `last_checked_at` | timestamptz | |

### `federation_peers`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `base_url` | text UNIQUE | |
| `peer_name` | text | |
| `trust_token_hash` | bytea | |
| `state` | enum(`pending`,`active`,`revoked`) | FR-083, FR-084 |

### `audit_events`

| Поле | Тип | Правила |
|---|---|---|
| `id` | bigserial PK | |
| `actor_id` | uuid FK → users NULL | NULL для системных |
| `action` | text NOT NULL | словарь действий |
| `target_type` / `target_id` | text / uuid | |
| `result` | enum(`ok`,`denied`,`error`) | |
| `metadata` | jsonb | без секретов и содержимого |
| `created_at` | timestamptz | |

Инварианты: только вставка; `UPDATE`/`DELETE` запрещены на уровне прав БД (FR-073).

### `retention_policies`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `scope` | enum(`system`,`user`) | |
| `user_id` | uuid FK NULL | заполнен при `scope='user'` |
| `versions_days` | integer | по умолчанию **30** |
| `versions_max_count` | integer | по умолчанию **100** |
| `trash_days` | integer | по умолчанию **30** |
| `updated_by` | uuid FK | |

### `quotas` — вычисляемая величина

Отдельной таблицы нет. Использование считается как `SUM(size_bytes)` по `nodes` владельца со `state IN ('active','conflict')`; объём версий и корзины считается отдельно и в квоту не входит (R8, FR-035).

### `jobs`

| Поле | Тип | Правила |
|---|---|---|
| `id` | uuid PK | |
| `kind` | enum(`gc_blobs`,`retention`,`export`,`backup_check`,`index_text`,`conflict_scan`) | |
| `payload` | jsonb | |
| `state` | enum(`queued`,`running`,`done`,`failed`) | |
| `attempts` | integer | |
| `run_after` | timestamptz | |
| `last_error` | text | |

---

## Правила валидации (сводка)

| Правило | Требование | Где обеспечивается |
|---|---|---|
| Имя 1…255, без `/`, `\`, не `.` и `..` | FR-009, FR-012 | CHECK + слой `nodes` |
| Уникальность по `name_folded` в пределах папки | FR-012, FR-052 | partial UNIQUE index |
| Запрет циклов при перемещении | FR-018 | проверка предков в транзакции |
| Файл не виден до фиксации блоба | FR-023 | `state='pending'` до `rename` |
| Обе версии при конфликте | FR-046 | `conflict_of` + две записи `versions` |
| Квота только на текущие файлы | FR-035 | вычисление `SUM` по `active`/`conflict` |
| Ссылка не даёт доступа к соседям | FR-043 | проверка области видимости в `shares`/`links` |
| Зашифрованная папка не индексируется | FR-059, FR-090 | `e2ee_folder_id IS NOT NULL` исключает из индексации |
| Блокировка инвалидирует сессии | FR-007 | каскадная пометка `sessions`/`devices` |
| Журнал только на добавление | FR-073 | права БД |

## Политики хранения по умолчанию

| Параметр | Значение по умолчанию | Настраивается |
|---|---|---|
| Срок хранения версий | 30 дней | да, системно и на пользователя |
| Максимум версий на файл | 100 | да |
| Срок хранения в корзине | 30 дней | да |
| Ретенция журнала изменений | 90 дней / 1 000 000 записей | да |
| TTL сессий загрузки | 24 часа | да |
| TTL архива выгрузки | 7 дней | да |
| Периодичность резервного копирования | 1 раз в сутки | да |
| Целевые RPO / RTO | 24 ч / 4 ч | нет (требование) |

## Что модель намеренно не содержит

- **Полей, зависящих от технологии передачи** (номера чанков, URL хранилища) — они в `contracts/sync-protocol.md` и `contracts/openapi.yaml`.
- **Денормализованных прав** — права вычисляются, а не хранятся итогом: иначе рассинхронизация при изменении группы.
- **Таблицы конфликтов** — конфликт выражается состоянием узла и ссылкой `conflict_of`; отдельная сущность добавила бы второй источник истины.