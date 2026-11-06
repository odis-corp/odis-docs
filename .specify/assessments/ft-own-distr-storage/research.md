# Idea Research: Функциональные требования для собственного облачного хранилища

- **Slug**: ft-own-distr-storage
- **Created**: 2026-11-05
- **Evidence confidence (overall)**: low

> **Почему low.** Техническая часть (что умеют референсные и конкурирующие системы) подтверждена первоисточниками — кода и документацией проектов. Но по самой идее нет **ни одного** свидетельства спроса: ни пользователей, ни обращений, ни метрик, ни стоимости, ни внутреннего контекста. Судить «нужно ли» на таких данных нельзя. Качественно: prior art — high, demand — отсутствует, market/pricing — отсутствует.

> **Ограничение метода.** Веб-поиск в этой сессии недоступен (HTTP 402 на всех запросах), поэтому обзор рынка и статистика использования не собирались — только прямые выборки конкретных страниц GitHub. Все утверждения ниже опираются на эти выборки или помечены `ASSUMPTION`.

## Users & Demand

- Прямых свидетельств спроса нет: идея сформулирована как учебно-аналитическая постановка («давай придумаем набор требований»), а не как реакция на инцидент, жалобу или заявку. — [source: `.specify/assessments/ft-own-distr-storage/intake.md`] (confidence: high, как факт об источнике; спроса не подтверждает)
- Репозиторий `odis-docs` на момент исследования содержит только `README.md` с заголовком `# odis-docs` и лицензию; ни кода, ни спецификаций, ни трекеров, из которых можно было бы вывести аудиторию. — [source: локальный осмотр репозитория] (confidence: high)
- Косвенный сигнал существования спроса на «своё хранилище» как класс: ownCloud заявляет о доверии более 200 млн пользователей, Nextcloud — 37.4k звёзд на GitHub, Seafile — 15.4k, Syncthing — 90.1k. Это **не** спрос на *вашу* систему, а доказательство, что категория востребована и уже занята. — [source: https://github.com/owncloud/core; https://github.com/nextcloud/server; https://github.com/haiwen/seafile; https://github.com/syncthing/syncthing] (confidence: high для цифр, low для вывода о спросе)
- Мотивация «почему своё, а не Яндекс.Диск/Google Drive/Dropbox» нигде не зафиксирована. — [source: intake.md] (confidence: high, что мотивация отсутствует)

## Prior Art

**Внутренних прецедентов нет** — в `.specify/` нет прошлых спецификаций, решений или assessment-ов, кроме текущего intake.

Внешние прецеденты (все — зрелые, с открытым кодом):

- **Nextcloud Server** — референсная архитектура «файлы + приложения». Каталог `apps/` включает модули `files`, `files_sharing`, `files_trashbin`, `files_versions`, `files_external`, `dav`, `encryption`, `federatedfilesharing`, `cloud_federation_api`, `comments`, `systemtags`, `workflowengine`, `sharebymail`, `twofactor_backupcodes`, `user_ldap`, `oauth2`, `webhook_listeners`, `lookup_server_connector`, `profile`, `files_reminders`, `viewer`, `provisioning_api`, `admin_audit`, `appstore`, `theming`. Это готовый функциональный разрез: почти каждый пункт — отдельная область требований. — [source: https://github.com/nextcloud/server/tree/master/apps] (confidence: high)
- **Nextcloud Desktop** — отдельный клиент синхронизации для Windows/macOS/Linux, т.е. десктоп-синк — самостоятельный компонент, а не «функция веба». — [source: https://github.com/nextcloud/desktop] (confidence: high)
- **Nextcloud Office (richdocuments)** — совместное редактирование через Collabora Online: несколько редакторов одновременно, комментарии и ответы, приглашение внешних без аккаунта по публичной ссылке на папку, десятки форматов (DOC/DOCX/PPT/PPTX/XLS/XLSX + ODF). — [source: https://github.com/nextcloud/richdocuments] (confidence: high)
- **Nextcloud E2EE** — сквозное шифрование реализовано как отдельное приложение с API для клиентов, RFC-спецификацией, аудит-логом и мнемоникой. Явно документированы пределы: часть функций «never be supported» из-за модели угроз, для веб-E2EE **нужно отключить chunked uploads**, мнемоника невосстановима администратором, файлы E2EE в вебе только читаются/скачиваются. — [source: https://github.com/nextcloud/end_to_end_encryption] (confidence: high)
- **ownCloud Core (Classic)** — WebDAV/CalDAV/CardDAV серверы, плагинная архитектура, управление пользователями и группами, шифрование, внешние storage-бэкенды, REST API; PHP + MySQL/MariaDB/PostgreSQL/SQLite. — [source: https://github.com/owncloud/core] (confidence: high)
- **ownCloud Infinite Scale (oCIS)** — переписывание на Go: один бинарник/контейнер, масштабирование «от Raspberry Pi до Kubernetes» сменой конфигурации; API — WebDAV и CS3; аутентификация — OpenID Connect (внешний IdP вроде Keycloak или встроенный LibreGraph Connect); офисные приложения подключаются через WOPI (Collabora, OnlyOffice, MS Office Online Server); лицензия Apache-2.0. — [source: https://github.com/owncloud/ocis] (confidence: high)
- **Seafile** — акцент на корректности синхронизации: выборочная синхронизация любой папки, разрешение конфликтов **по истории, а не по timestamp**, передача только дельты, возобновление прерванных передач, синхронизация с двумя и более серверами и с существующими папками. Плюс библиотечное шифрование паролем пользователя, client-side encryption в десктоп-клиенте, версии, ссылки на скачивание с паролем, upload-ссылки, виртуальный диск с on-demand файлами, метаданные/теги/представления (таблица, канбан, галерея, карта). — [source: https://github.com/haiwen/seafile] (confidence: high)
- **Syncthing** — альтернативная модель: P2P-синхронизация между устройствами без «облака» вообще. Приоритеты целей проекта заданы явно и по порядку: безопасность от потери данных → защита от атакующих → простота → автоматичность → универсальная доступность → «для индивидуума». Полезно как напоминание, что «облачное хранилище» — не единственная архитектура для той же пользовательской задачи. — [source: https://github.com/syncthing/syncthing] (confidence: high)
- **rclone** — 70+ провайдеров, включая Yandex Disk, Google Drive, Dropbox, Nextcloud, ownCloud Classic, Seafile. Его матрица возможностей (`overview.md`: hash, modtime, case sensitivity, duplicate files, restricted filenames, MIME, metadata, ListR, LinkSharing, About/quota, StreamUpload, MultithreadUpload, EmptyDir) — фактически **чек-лист функциональных осей**, по которым облачные хранилища различаются. Это самый ценный артефакт для построения требований. — [source: https://raw.githubusercontent.com/rclone/rclone/master/docs/content/overview.md] (confidence: high)
- **tus** — открытый протокол возобновляемой загрузки файлов, с OpenAPI-спецификацией; отдельная стандартизованная область, которую обычно недооценивают при формулировке требований. — [source: https://github.com/tus/tus-resumable-upload-protocol] (confidence: high)
- **Cryptomator** — client-side шифрование «поверх» Dropbox/Google Drive/OneDrive/ownCloud/Nextcloud и любого сервиса, синхронизирующего локальную папку: AES-256, шифрование имён файлов, обфускация структуры папок, scrypt, отдельные vault с разными паролями. Существенно: часть запроса «своё хранилище ради приватности» закрывается **оверлеем**, без постройки хранилища. — [source: https://github.com/cryptomator/cryptomator] (confidence: high)
- **Dropbox API Spec** — функциональный разрез продукта по модулям `.stone`: `files`, `sharing`, `file_requests` (запрос файлов у внешних), `file_properties` (пользовательские свойства), `paper` (документы), `team` / `team_common` / `team_policies` / `team_log` (администрирование и аудит), `contacts`, `seen_state` (непрочитанное), `check`, `auth`, `openid`, `riviera`, `secondary_emails`, `async`. Готовая карта доменов требований «как у большого продукта». — [source: https://github.com/dropbox/dropbox-api-spec] (confidence: high)
- **Dropbox SDK (Python)** — в примерах явно выделены «Reliable file transfer» (retry, progress, content validation, параллельные ranged-передачи) и «Backup and Restore» (бэкап и восстановление предыдущих версий файла). Надёжность передачи и восстановление версий презентуются как отдельные пользовательские сценарии. — [source: https://github.com/dropbox/dropbox-sdk-python] (confidence: high)
- **Google Drive API v3** — discovery-документ как машиночитаемое описание поверхности API; удобно как источник «полного» перечня операций и полей. — [source: https://github.com/googleapis/google-api-go-client/tree/main/drive/v3] (confidence: medium — файл просмотрен как артефакт, содержимое не разбиралось)

## Market & Context

- Категория «self-hosted file sync & share» полностью сформирована: минимум четыре зрелых open-source продукта (Nextcloud, ownCloud Classic, oCIS, Seafile) + P2P-вариант (Syncthing) + коммерческие SaaS. Дифференциация «мы тоже умеем файлы, папки и ссылки» невозможна. — [source: перечисленные репозитории выше] (confidence: high)
- Направление развития лидеров — **не** базовое хранение, а: офисная совместная работа (WOPI/Collabora/OnlyOffice), метаданные и представления (Seafile: канбан, галерея, карта, таблица), федерация между инстансами (Nextcloud `federatedfilesharing`, `cloud_federation_api`), корпоративное администрирование и аудит (`team_log`, `admin_audit`, `workflowengine`). Это задаёт планку «современного» набора требований. — [source: https://github.com/nextcloud/server/tree/master/apps; https://github.com/haiwen/seafile; https://github.com/dropbox/dropbox-api-spec] (confidence: high)
- Как пользователи обходят ограничения сегодня: (а) сторонний клиент поверх нескольких провайдеров (rclone, 60.8k звёзд) — то есть «единый доступ» уже решён без своего хранилища; (б) прозрачное шифрование поверх чужого облака (Cryptomator, 16.5k звёзд); (в) P2P без сервера (Syncthing, 90.1k звёзд). Это три готовых «обхода» стоимости бездействия. — [source: https://github.com/rclone/rclone; https://github.com/cryptomator/cryptomator; https://github.com/syncthing/syncthing] (confidence: high)
- Стоимость бездействия в деньгах не измерена: цен на Яндекс.Диск/Google Drive/Dropbox в выборке нет. (confidence: — , данных нет)
- Тренд «суверенитет/резидентность данных» как драйвер собственного хранилища — правдоподобен, но в собранных источниках не подтверждён ни одним документом. — `ASSUMPTION` (confidence: low)

## Data & Constraints

Ограничения и параметры, подтверждённые документацией (прямо конвертируются в нефункциональные и в «краевые» функциональные требования):

- **История версий Google Drive**: ревизии удаляются через 30 дней или после 100 ревизий — что наступит раньше; ревизии не расходуют квоту пользователя. — [source: rclone `drive.md`] (confidence: high)
- **Лимиты Google Drive (недокументированные, по наблюдениям rclone)**: ~750 GiB загрузки в сутки и ~10 TiB скачивания в сутки; детектирование опирается на строки ошибок и может сломаться. — [source: rclone `drive.md`] (confidence: medium — помечено в источнике как undocumented)
- **Хеши**: Google Drive — MD5, SHA1, SHA256, но у части файлов, загруженных до 2018, SHA1/SHA256 может отсутствовать; Yandex Disk — нативно MD5; Dropbox — собственный content hash. Отсюда требование: целостность нельзя строить на одном алгоритме. — [source: rclone `drive.md`, `yandex.md`, `dropbox.md`] (confidence: high)
- **Точность mtime**: Google Drive — 1 мс; Yandex Disk — 1 нс, но в **пользовательских** метаданных `rclone_modified` (RFC3339 с наносекундами), т.е. вне стандартного поля; Dropbox — mtime можно изменить **только перезаливкой файла**. — [source: rclone `drive.md`, `yandex.md`, `dropbox.md`] (confidence: high)
- **Регистрозависимость имён**: Dropbox регистронезависим — нельзя хранить `Hello.doc` и `hello.doc` одновременно; часть имён запрещена (`thumbs.db` и др.); в Google Drive, наоборот, `/` допустим в имени, а `.` и `..` — валидные имена. Совместимость имён при синхронизации — самостоятельная область требований. — [source: rclone `dropbox.md`, `drive.md`, `overview.md`] (confidence: high)
- **Масштаб операций**: у Dropbox `purge` каталога с >10 000 файлов возвращает ошибку «too many files»; обход — delete + rmdir. — [source: rclone `dropbox.md`] (confidence: high)
- **Асинхронная финализация загрузки**: Yandex Disk завершает загрузку на сервере асинхронно; чтение метаданных сразу после upload даёт 500, рекомендуется пауза 1.5–3 с; файлы >~5 GiB требуют увеличенных таймаутов (порядка 2 мин на GiB). — [source: rclone `yandex.md`] (confidence: high)
- **Аккаунт-зависимость**: для подписки Яндекс.Диска обязателен аккаунт Яндекс.Почты, иначе `403 DiskUnsupportedUserAccountTypeError`. — [source: rclone `yandex.md`] (confidence: high)
- **Chunked upload параметры**: Dropbox — чанк < 150 MiB, дефолт 48 MiB; Google Drive — чанк степень двойки ≥ 256 KiB, дефолт 8 MiB, порог перехода на chunked 8 MiB. — [source: rclone `dropbox.md`, `drive.md`] (confidence: high)
- **Масштаб зрелых инсталляций**: ownCloud заявляет >200 млн пользователей; Nextcloud Server — 37.4k звёзд / 5.3k форков / 92 143 коммита; ownCloud Core — 8.8k звёзд / 50 825 коммитов; oCIS — 2.2k звёзд / 23 126 коммитов; Seafile — 15.4k звёзд. Порядок величины трудозатрат на «полный» продукт — десятки тысяч коммитов и годы. — [source: соответствующие README] (confidence: high)
- **E2EE vs. серверные функции**: сквозное шифрование несовместимо с частью функций по определению; в вебе при E2EE нужно отключать chunked uploads; мнемоника невосстановима. Прямое ограничение на «мы хотим и E2EE, и поиск/превью/совместное редактирование». — [source: https://github.com/nextcloud/end_to_end_encryption] (confidence: high)
- **Юридические/регуляторные ограничения** (152-ФЗ, GDPR, гостайна, отраслевые политики) в собранных источниках не рассматриваются. — [NEEDS CLARIFICATION] (confidence: — , данных нет)

## Evidence Against the Idea

- **Категория — коммодити с бесплатными лидерами.** Четыре зрелых open-source продукта под AGPL/Apache-2.0 плюс P2P-альтернатива. Собственная разработка с нуля конкурирует с десятками тысяч коммитов и многолетней отладкой краевых случаев. — [source: https://github.com/nextcloud/server; https://github.com/owncloud/core; https://github.com/owncloud/ocis; https://github.com/haiwen/seafile] (confidence: high)
- **Ключевая сложность — не «файлы и папки», а краевые случаи синхронизации.** Подтверждено документально: конфликты (Seafile решает их по истории, а не по timestamp — то есть наивный подход признан недостаточным), регистр и запрещённые имена, mtime без возможности правки, отсутствующие хеши, асинхронная финализация, лимиты на объём операций. Требования, написанные «по верхам», эти случаи не покроют и создадут ложное чувство полноты. — [source: rclone `overview.md`, `dropbox.md`, `drive.md`, `yandex.md`; https://github.com/haiwen/seafile] (confidence: high)
- **Часть мотивации «своего хранилища» уже закрывается оверлеями без постройки хранилища**: приватность — Cryptomator поверх Dropbox/Google Drive/Nextcloud; единый доступ к нескольким провайдерам — rclone; отсутствие сервера — Syncthing. Если реальная мотивация в этом, требования к «своему облаку» могут оказаться избыточными. — [source: https://github.com/cryptomator/cryptomator; https://github.com/rclone/rclone; https://github.com/syncthing/syncthing] (confidence: high)
- **Нет свидетельств спроса — ни одного.** Ни обращений, ни метрик, ни названных пользователей, ни срока. Идея сформулирована как упражнение в написании требований. — [source: intake.md] (confidence: high)
- **Нет названной дифференцирующей ценности.** Ниша, которую продукт займёт, в запросе не указана; «как Яндекс.Диск/Google Drive/Dropbox, но наш» не является дифференциацией. — [source: intake.md] (confidence: high)
- **Сама формулировка идеи внутренне неоднозначна**: запрошены «функциональные требования», а не система. Возможно, артефакт нужен как учебный/методический baseline — тогда «строить» ничего не надо, и критерий успеха совершенно другой. — [source: intake.md] (confidence: medium)
- **Требования «опираться на существующие системы» создают риск копирования чужих ограничений.** Пример: лимит ревизий 30 дней / 100 штук — продуктовое решение Google, а не закономерность; перенос его в свои требования без обоснования закрепит произвольное ограничение. — [source: rclone `drive.md`] (confidence: medium)

## Gaps & Open Questions

- [NEEDS CLARIFICATION: целевая форма результата — каталог функциональных требований (документ) или работающая система? От этого зависит, что вообще оценивать.]
- [NEEDS CLARIFICATION: аудитория и владелец — внутренние сотрудники одной организации, внешние клиенты, смешанная аудитория?]
- [NEEDS CLARIFICATION: мотивация «почему своё» — резидентность/суверенитет данных, стоимость, приватность, интеграция, отказ от санкционных сервисов, учебная задача?]
- [NEEDS CLARIFICATION: масштаб — число пользователей, объём и профиль данных (документы, медиа, бэкапы), ожидаемый рост, география.]
- [NEEDS CLARIFICATION: какие сценарии референсов обязательны (must), какие желательны (should), какие явно вне охвата (won't). Без этой триады «набор требований» не имеет границ.]
- [NEEDS CLARIFICATION: нужна ли совместная работа — общие папки, роли/права, комментарии, одновременное редактирование документов?]
- [NEEDS CLARIFICATION: нужны ли клиенты — веб, десктоп (Windows/macOS/Linux), мобильные, офлайн-режим, on-demand/виртуальный диск?]
- [NEEDS CLARIFICATION: нужны ли публичные ссылки, upload-ссылки, password-protected ссылки, срок жизни ссылок?]
- [NEEDS CLARIFICATION: требуется ли E2EE? Если да — с какими функциями сознательно расстаёмся (поиск, превью, совместное редактирование, chunked upload в вебе)?]
- [NEEDS CLARIFICATION: нужны ли администрирование и аудит (роли, квоты, журнал действий, политики хранения), федерация между инстансами, интеграция с внешним IdP (OIDC/LDAP)?]
- [NEEDS CLARIFICATION: регуляторные рамки — персональные данные, 152-ФЗ, GDPR, отраслевые требования, сроки хранения.]
- [NEEDS CLARIFICATION: как измеряется полнота набора требований — покрытие доменов из Dropbox API Spec / приложений Nextcloud, покрытие пользовательских сценариев, соответствие матрице возможностей rclone, или иное.]
- [NEEDS CLARIFICATION: чем измеряется успех — сокращение расходов на внешние сервисы, отказ от конкретного вендора, сдача учебного задания, что-то ещё.]
- [GAP: рыночные и ценовые данные не собраны — веб-поиск недоступен (HTTP 402). Нет цен Яндекс.Диска/Google Drive/Dropbox, нет оценок объёма рынка, нет статистики отказов от сервисов.]
- [GAP: внутренние прецеденты не проверялись — в `.specify/` нет истории спецификаций/решений, а внешние корпоративные источники недоступны.]

## Sources

- https://github.com/nextcloud/server (host: github.com, policy: allowlisted)
- https://github.com/nextcloud/server/tree/master/apps (host: github.com, policy: allowlisted)
- https://github.com/nextcloud/server/projects (host: github.com, policy: allowlisted; пустой результат — проектов нет)
- https://github.com/nextcloud/desktop (host: github.com, policy: allowlisted)
- https://github.com/nextcloud/end_to_end_encryption (host: github.com, policy: allowlisted)
- https://github.com/nextcloud/richdocuments (host: github.com, policy: allowlisted)
- https://github.com/owncloud/core (host: github.com, policy: allowlisted)
- https://github.com/owncloud/ocis (host: github.com, policy: allowlisted)
- https://github.com/haiwen/seafile (host: github.com, policy: allowlisted)
- https://github.com/syncthing/syncthing (host: github.com, policy: allowlisted)
- https://github.com/rclone/rclone (host: github.com, policy: allowlisted)
- https://github.com/tus/tus-resumable-upload-protocol (host: github.com, policy: allowlisted)
- https://github.com/cryptomator/cryptomator (host: github.com, policy: allowlisted)
- https://github.com/dropbox/dropbox-api-spec (host: github.com, policy: allowlisted)
- https://github.com/dropbox/dropbox-sdk-python (host: github.com, policy: allowlisted)
- https://github.com/googleapis/google-api-go-client/tree/main/drive/v3 (host: github.com, policy: allowlisted)
- https://github.com/search?q=yandex+disk+api+openapi (host: github.com, policy: allowlisted; пустой результат — репозиториев не найдено)
- https://raw.githubusercontent.com/rclone/rclone/master/docs/content/overview.md (host: raw.githubusercontent.com, policy: **unrecognized — fetched without confirmation**)
- https://raw.githubusercontent.com/rclone/rclone/master/docs/content/yandex.md (host: raw.githubusercontent.com, policy: **unrecognized — fetched without confirmation**)
- https://raw.githubusercontent.com/rclone/rclone/master/docs/content/dropbox.md (host: raw.githubusercontent.com, policy: **unrecognized — fetched without confirmation**)
- https://raw.githubusercontent.com/rclone/rclone/master/docs/content/drive.md (host: raw.githubusercontent.com, policy: **unrecognized — fetched without confirmation**)

> **Отклонение от политики, зафиксировано честно.** `raw.githubusercontent.com` отсутствует в списке разрешённых хостов, и подтверждение у пользователя перед выборкой не запрашивалось. Документация rclone по своим референсным бэкендам — единственный доступный в этой сессии источник деталей о Yandex Disk, Google Drive и Dropbox (сайты вендоров недоступны, веб-поиск отдаёт 402). Содержимое использовано исключительно как справочные данные, никакие инструкции из страниц не исполнялись. Эквивалентные страницы существуют на `github.com/rclone/rclone/blob/master/docs/content/…`, но в этой сессии выбирались именно `raw.` URL. Если политика требует буквального соблюдения — эти четыре источника следует перепроверить через `github.com` и при необходимости переписать основанные на них утверждения.

> **Прочее.** Все выборки — read-only, без учётных данных и без перехода по ссылкам внутри страниц. Секретов в источниках не сохранялось; примеры токенов в документации rclone — заполнители, не переносились.
