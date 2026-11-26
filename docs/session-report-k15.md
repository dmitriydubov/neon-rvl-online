# Отчёт о ходе реализации шага К15 (`contract-service`)

- **Задача:** Шаг К15 — отклонять нестроковые значения необязательных текстовых, GUID- и датовых полей во входящих сообщениях как нарушение контракта (`contract-service`).
- **Ветка задачи:** `task/k15-contract-service-optional-field-types` (создана от `origin/main` `0d90e3a`, итоговый `HEAD = 44a785e` после влития ревизий набора 4.6, 4.7 и 4.7.1).
- **Открытый PR:** https://github.com/dmitriydubov/neon-rvl-online/pull/1 (`main` ← `task/k15-contract-service-optional-field-types`, 15 файлов К15, открыт `2026-11-26T07:43:11Z`).
- **Среда исполнения:** Google Antigravity (облачная Linux VM, Ubuntu 24.04, Docker 29.10.0 `CgroupDriver=systemd`, JDK 21).

---

## 1. Понимание задачи и границы

### Цель шага К15
После шага К13 (выделение записи `OptionalField` из `JsonFields`) методы `OptionalField.text()`, `OptionalField.uuid()` и `OptionalField.date()` проверяли `!field.value().isString()` и возвращали `null` не только при отсутствии ключа или явном JSON `null`, но и при любом нестроковом JSON-узле (число, `boolean`, массив, объект). В результате некорректный тип необязательного поля молча проглатывался как отсутствие значения.

Цель шага К15 — разделить два принципиально разных случая:
- **Отсутствие значения** (ключа нет в JSON-объекте или значение равно JSON `null`) → по-прежнему `null`, штатная обработка (а для условно обязательных полей `program_id` и `relation_type` — доменная ошибка участника `PROGRAM_REQUIRED` / `RELATION_TYPE_REQUIRED` с ответом ДГ-03 `ERROR`).
- **Нестроковый JSON-узел** (число, логическое значение, массив, объект) в необязательном строковом, GUID- или датовом поле → **нарушение контракта** (`ContractViolationException.invalidValue(path)`): окончательный отказ (`FAILED`) с 1-й попытки без повторов, запись уровня `ERROR` с путём поля (без значения узла и без ПДн), рост счётчика `contracts_inbox_failed_total{event_type}`, без применения изменений по заявке и без публикации ответа ДГ-03.

### Что вошло в область изменений
- **Единая точка разбора:** `contract-service/src/main/java/ru/projectx/contracts/service/inbox/parse/OptionalField.java`.
- **Три входящих потока и 10 необязательных полей, читаемых через `OptionalField`:**
  1. **ДГ-01 (`ContractTaskUpserted`, `ContractTaskParser`)** — 8 полей участника:
     - `participants[].program_id` (`uuid()`),
     - `participants[].personal_data.middle_name` (`text()`),
     - `participants[].personal_data.relation_type` (`text()`),
     - `participants[].personal_data.address` (`text()`),
     - `participants[].personal_data.document.series` (`text()`),
     - `participants[].personal_data.document.issued_by` (`text()`),
     - `participants[].personal_data.document.issued_at` (`date()`),
     - `participants[].personal_data.document.division_code` (`text()`).
  2. **ДГ-02 (`ContractTaskCancelled`, `TaskCancellationParser`)** — `data.cancel_comment` (`text()`).
  3. **ДГ-07 (`LmPolicyIssued`, `UskResultParser`)** — `data.policy_usk_number` (`text()`, читается только при `status = "ISSUED"`).
- **Тесты и документация:**
  - Unit-тесты и тестовые билдеры: `FieldPresenceTest`, `OptionalFieldTest`, `ContractTaskParserTest`, `TaskCancellationParserTest`, `UskResultParserTest`, `TaskBodies`, `LmPolicyResultBodies`.
  - Новая спецификация и glue-классы (без правок замороженных фич и существующих классов стенда): `inbox_optional_field_types.feature`, `OptionalFieldTypesSteps`, `OptionalJsonValues`.
  - Документация сервиса: `contract-service/docs/design.md`, `docs/operations.md`, `docs/troubleshooting.md`, `CHANGELOG.md`.

### Что оставлено «Вне фичи» и почему
- **Пустая строка `""` и пробелы `"   "` в необязательных текстовых полях (`OptionalField.text()`, включая `relation_type` и `cancel_comment`):** сохранено прежнее поведение базы (`""` / `"   "` остаются строками, а не `null`; для `relation_type = ""` не возникает `RELATION_TYPE_REQUIRED`). Вынесено в вопрос владельцу до заморозки и утверждено как «Вне фичи» (п. 2). Для `uuid()` и `date()` пустая/пробельная строка по-прежнему отклоняется как нарушение контракта.
- **События оплаты Ordering (`PaymentOrderEvent`, `OrderEventParser`):** не используют `OptionalField`; при пустых/нестроковых полях имеют отдельные правила (`OrderEventEnvelopes` / `OrderEventParser` / `PaidReaction` — пропуск `SKIPPED`, потеря при невалидном `eventId` или отказ при невалидном `updated` на `paid`). Утверждено как «Вне фичи» (п. 3).
- **Конверты (`meta`) и синхронные HTTP-ручки `/internal/v1`:** используют отдельные парсеры, вне `OptionalField`.
- **ДГ-05 (`KmContractRegistered`):** обязательные поля и целочисленный `documents[].file_size` (`OptionalField.longInteger()`) уже валидировали тип.
- **Устаревшая `entity_version`:** отбрасывается до разбора тела — существующий порядок сохранён.
- **README модуля (`contract-service/README.md`) и `project_files/`:** не трогались по правилам набора и сессии; точные пункты обновления редакции 2 «Точек интеграций» вынесены для координатора.

### Граничные случаи, выявленные сверх текста входного промпта
1. **Нестроковый `data.cancel_comment` в ДГ-02:** в редакции 2 «Точек интеграций» для ДГ-02 явно не был прописан отказ по нестроковому `cancel_comment`, при этом ручной возврат `FAILED → NEW` дежурным не способен вылечить ошибку типа в теле сообщения — вынесено в вопрос №1 владельцу (утверждено) и добавлено явное пояснение в `docs/operations.md` + `docs/troubleshooting.md`.
2. **Нестроковое поле у участника `NONE` (без выпуска) и у уже выпущенного участника `ISSUED` в повторной задаче ДГ-01:** разбор контракта идёт до сверки состава, поэтому сообщение отклоняется целиком, не создавая/не меняя даже валидного соседа. Закреплено двумя отдельными сценариями в спецификации.
3. **ДГ-07 при `status = ERROR` vs `status = ISSUED` и поздний/повторный ДГ-07 по финализированному полису:** при `status = ERROR` поле `policy_usk_number` парсером не читается (нестроковый узел не ломает фиксацию ошибки 1С:УСК); при `status = ISSUED` разбор контракта выполняется до проверки статуса полиса, поэтому даже поздний ответ по `ISSUED`/`CANCELLED`/`ERROR` полису с нестроковым `policy_usk_number` даёт `FAILED`. Вынесено в вопрос №5 владельцу и утверждено.
4. **Безопасность и ПДн при логировании отказов с массивами/объектами:** проверено отдельным шагом приёмки, что маркерное значение нестрокового узла (число `987654`, `true`, маркер `"Тестов-Узел"` внутри массива/объекта) и фамилия участника сохраняются в теле сообщения, но не попадают ни в `error_message` журнала входящих, ни в одну запись лога сервиса (включая MDC и тексты исключений).
5. **Требование `PrometheusSteps` к таблице метрик в `docs/operations.md`:** при проверке роста счётчика `contracts_inbox_failed_total` на `0` с меткой `event_type` шаг сверяет наличие имени типа события в обратных кавычках в строке таблицы `docs/operations.md` (`src/acceptanceTest/README.md:72-77`). Без перечисления типов событий в таблице мутанты `gherkinMutation` выявили расхождение документации со стендом — закрыто в коммите `3c8b22f`.

---

## 2. Чем руководствовался при проектировании и реализации

### Проверенные документы и правила
- **Инструкции среды и процесса:** `GEMINI.md` (разделы 0–6), `gauntlet/CLAUDE.md` целиком (509 строк, все разделы до «Когда застрял» включительно; перечитан повторно после сжатия контекста в `2026-11-26 04:52Z`), `gauntlet/subagents.md` (карточка 4.5 приёмки отчётов сабагентов), `gauntlet/config/gauntlet.properties` и `gauntlet/config/contract-service/*`.
- **Правила модуля и предметная область:** `contract-service/CLAUDE.md`, `contract-service/src/acceptanceTest/README.md`, `project_files/contract-service/edited/Точки интеграций сервис Договоров и Заявок — редакция 2.md` (разделы ДГ-01, ДГ-02 строки 335–355, ДГ-07 строки 736–756), `project_files/contract-service/Журнал решений Договоров.md` (Д-01, Д-02, Д-06, Д-09, Д-10), `docs/gauntlet-plan-contract-service.md` (строка 91 — оценка К15 `4–6 ч`), а также существующие фичи (`requests_task_inbox.feature`, `requests_task_issuance.feature`, `requests_task_cancellation.feature`, `lm_policy_issued.feature`).
- **Жёсткие правила `gauntlet/CLAUDE.md`, определившие форму реализации:**
  - Правило 1 (неприкосновенность набора `gauntlet/`, `.agents/`, workflow и др.).
  - Правило 3 (спецификацию пишет `specifier`, утверждает человек; замороженные фичи прошлых шагов не меняются).
  - Правило 4 (строгий TDD: сначала падающие unit-тесты → затем минимальный код).
  - Правило 5 (маленькие методы, CC ≤ 3, CRAP ≤ 8, ≤ 12 методов в классе).
  - Правило 9 (запрет ПДн в логах, исключениях и текстах ошибок).
  - Правило 12 (сверка API строго с официальной документацией версии сборки, полученной через `./gradlew gauntletInfo` и `./gradlew dependencies`).
  - Правило 15 (ноль комментариев и Javadoc в коде/тестах, ноль следов разработки в переносимых файлах, причины решений — в `docs/design.md`).

### Официальные источники документации версий из сборки (правило 12)
- **Jackson `3.1.4`** (`tools.jackson.core:jackson-databind:3.1.4`): `jackson-databind-3.1.4-javadoc.jar` (Maven Central) + `javap -p -c` по jar из кэша Gradle — проверена семантика `JsonNode#isMissingNode`, `isNull`, `isString`, `path`, `stringValue`, `isIntegralNumber`, `canConvertToLong`, `longValue`.
- **Java SE `21`**: `https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/time/LocalDate.html` + `javap -p -c` — подтверждено поведение `LocalDate#parse` и `UUID#fromString` на пустой и пробельной строках.
- **JUnit Jupiter `6.0.3`** (`docs.junit.org/6.0.3/user-guide/`), **AssertJ `3.27.7`** (`assertj.github.io/doc/`), **Cucumber-JVM `8.0.1`** (`raw.githubusercontent.com/cucumber/cucumber-jvm/v8.0.1/`), **PIT `1.22.1`** (`pitest.org`), **Spring Boot `4.0.7`** (`docs.spring.io/spring-boot/4.0/`).
- **RabbitMQ `v4.1.2` / `v4.1.8` / `v4.2.0–v4.2.9` / `main`** (при расследовании бага `enametoolong`): `raw.githubusercontent.com/rabbitmq/rabbitmq-server/<tag>/deps/rabbit/src/rabbit_variable_queue.erl` и `rabbit_classic_queue_index_v2.erl`.

### Рассмотренные варианты реализации и выбор (правило 14)
1. **Вариант 1 (выбран):** в `OptionalField` (`text()`, `uuid()`, `date()`) заменить проверку `!field.value().isString()` на `field.isAbsent()` и при наличии узла делегировать в `JsonField` (`field.text()`, `field.uuid()`, `field.date()`), оставив `longInteger()` на `isAbsent()` + `notLong()`.
   - *Обоснование:* единая точка инварианта для ДГ-01, ДГ-02 и ДГ-07; минимальный дифф (10 строк в одном классе); CC = 2 на каждый из трёх методов, max CRAP = 3.0; автоматическое переиспользование `ContractViolationException.invalidValue(field.path())` без значения узла и без ПДн.
2. **Вариант 2 (отклонён):** добавить отдельные проверки `isString()` в `ContractTaskParser`, `TaskCancellationParser` и `UskResultParser`.
   - *Почему отклонён:* дублирование в 3 парсерах и 10 точках чтения, рост сложности и риск забыть проверку при появлении нового необязательного поля.
3. **Вариант 3 (отклонён):** приводить нестроковые JSON-узлы к строке (`asString()` / `toString()`).
   - *Почему отклонён:* противоречит постановке К15 и маскирует дефекты контракта у отправителей.

---

## 3. Сабагенты

Маршрут сабагентов (задан владельцем во входном промпте): **`own`** (штатные роли `specifier`, `hardener`, `reviewer`; дополнительные сабагенты через `define_subagent` не создавались).

| # | Роль и ID | Инструмент и параметры | Цель | Краткий итог | Попытка приёмки | Проверка доказательств по карточке 4.5 |
|---|---|---|---|---|---|---|
| 1 | `specifier` (`13393ba27d2161a3364a85d6cd578594`) | `invoke_subagent` (`TypeName: specifier`, `Model: inherit`, `Workspace: inherit`) | Написать спецификацию Gherkin для К15 и таблицу решений / открытых вопросов | Вернул полный текст `inbox_optional_field_types.feature` (11 структур сценариев, 38 примеров) + 6 открытых вопросов и допущений. Из-за read-only прав в облачной VM файл на диск записать не смог — передал текст в ответе | Принят с 1-й попытки (запись на диск выполнена основной сессией дословно после явного разрешения владельца п. 6) | Проверена полнота покрытия всех 10 полей ДГ-01/ДГ-02/ДГ-07, словарь шагов стенда, «дыра-чек» (все спорные правила вынесены в нумерованные вопросы владельцу до заморозки), отсутствие ПДн |
| 2 | `hardener` (`c568351a6374a43ebf4d41984252d992`) | `invoke_subagent` (`TypeName: hardener`, `Model: inherit`, `Workspace: inherit`) + `send_message` | Закалить тесты и проверить мутантов PIT / пустые step-определения | В 1-м сообщении сообщил об отсутствии терминала и инструментов записи в облачной VM (только `view_file`). По правилу 7б владельца/координатора основная сессия прогнала `./gradlew mutationGate` на `940fc98` (`/tmp/mutation-gate-940fc98.log`) и передала сабагенту вывод, `mutations.xml`, `gauntletInfo` и `git grep`. После проверки файлов `hardener` подтвердил: 0 выживших мутантов, 0 правок тестов, 0 рекомендаций по продакшен-коду | Принят после досылки свежего лога `mutationGate` (технический дозапрос из-за среды, не брак отчёта) | Сверены строки `PIT summary: 13 mutant(s) {KILLED=13}`, `Line Coverage 11/11 (100%)`, `0 SURVIVED`, `0 NO_COVERAGE`, `0 RUN_ERROR` с логом `/tmp/mutation-gate-940fc98.log`; `git write-tree` до и после совпал (`d733df48...`) |
| 3 | `reviewer` — Раунд 1 (`7546dd6f9ab9046cc938138dd3890d90`) | `invoke_subagent` (`TypeName: reviewer`, `Model: inherit`, `Workspace: inherit`) | Состязательное ревью диффа `940fc98` против спецификации, SOLID, ПДн, правил документации | Вернул `VERDICT: CHANGES REQUESTED`, `Осмотрено: файлы диффа 15 из 15`. Поймал 3 дефекта формулировок в документации (`docs/design.md:357,365`, `CHANGELOG.md:24`) и сбой `rabbitmq:4.1.2-alpine` `OS monotonic time stepped backwards!` в своём прогоне `task-116` | Принят с 1-й попытки (0 возвратов за доказательствами) | Открыты `docs/design.md:357,365` и `CHANGELOG.md:24`, цитаты сверены с `OrderEventEnvelopes` / `PaidReaction` / `JsonFields` — все 3 дефекта подтверждены и исправлены в `7e19e74`; число файлов `15 из 15` совпало с `git diff --name-only $(git merge-base origin/main HEAD)` (`M = 15`) |
| 4 | `reviewer` — Раунд 2 (`7546dd6f9ab9046cc938138dd3890d90`) | `send_message` в существующего `reviewer` | Повторное ревью после исправлений `7e19e74` и зелёного `gauntletFast` | Проверил исправления и прогнал `gauntletFast` (`task-165`, `22s` — `BUILD SUCCESSFUL`). Вернул **`VERDICT: APPROVED`**, `Осмотрено: файлы диффа 15 из 15` | Принят с 1-й попытки | Сверены `15 из 15` с `M = 15`, 0 следов разработки, `README.md` (3 439 символов), `git write-tree` до и после совпал (`ff73c9f8...`). После `7e19e74` файлы `contract-service/` больше не менялись (`git diff 7e19e74 44a785e -- contract-service/` пуст) |

**Примечание о шагах, затронутых ограничениями облачной VM:**
- `specifier` и `hardener` вызывались штатно, их работа не подменялась, но из-за read-only прав этих ролей в `.agents/` запись утверждённой спецификации на диск (п. 6) и запуск команды `./gradlew mutationGate` для `hardener` (п. 7б) выполнялись основной сессией по явному разрешению владельца и координатора.

---

## 4. Проведённые проверки и гейты

Все запуски Gradle выполнялись из каталога `contract-service` с `ORG_GRADLE_PROJECT_publicRepositories=true`.

| # | Время завершения (UTC) | Коммит / база | Команда и лог | Итог | Падения и как исправлялись |
|---|---|---|---|---|---|
| 1 | `2026-11-25 12:35:24Z` | `0d90e3a` | `./gradlew gauntletInfo` (`task-55`) | `BUILD SUCCESSFUL in 49s` | Подтверждены версии сборки (`Java 21`, `Spring Boot 4.0.7`, `PIT 1.22.1`, настройки `contract-service`) |
| 2 | `2026-11-25 12:36:11Z` | `0d90e3a` | `./gradlew dependencies` (`task-77`) | OK | Сверены точные версии библиотек (`jackson-databind:3.1.4`, `junit-jupiter:6.0.3`, `assertj:3.27.7`, `cucumber:8.0.1` и др.) |
| 3 | `2026-11-25 12:52:13Z` | `3f17f4b` + новые тесты | `./gradlew test` (TDD-красный, `task-122`) | `BUILD FAILED in 1m 39s` | **Ожидаемый TDD-red:** 29 новых падений в `FieldPresenceTest` (4), `OptionalFieldTest` (9), `ContractTaskParserTest` (8), `TaskCancellationParserTest` (4), `UskResultParserTest` (4) на прежней реализации `OptionalField` |
| 4 | `2026-11-25 12:53:12Z` | `3f17f4b` + правка `OptionalField` | `./gradlew test` (TDD-зелёный, `task-153`) | `BUILD SUCCESSFUL in 18s` | Все 29 новых тестов и весь unit-набор стали зелёными после перевода `OptionalField.text/uuid/date` на `field.isAbsent()` |
| 5 | `2026-11-25 13:05:22Z`, `13:16:59Z` | до `12fe3e0` | `./gradlew gauntletFast --continue` (`task-176`, `task-225`, `/tmp/gauntlet-fast.log`) | `BUILD SUCCESSFUL` (`22s` на повторном) | `integrityCheck`, `checkstyleGate`, `pmdGate`, `test` (335/335), `archTest` (1/1), `acceptanceTest` (521/521), `acceptanceCheck`, `crapGate` (max 3.0), `dryGate` (0 дублей) — все OK → коммит `12fe3e0` |
| 6 | `2026-11-25 13:18:14Z` | `12fe3e0` | `./gradlew mutationGate` (`task-235`, `/tmp/mutation-gate-before.log`) | `BUILD SUCCESSFUL in 52s` | `PIT summary: 13 mutant(s) {KILLED=13}`, 100 % покрытие |
| 7 | `2026-11-25 13:40:11Z` | `12fe3e0` | `./gradlew gherkinMutation` (1-я попытка на 4.5, `/tmp/gherkin-mutation.log`) | Остановлен после обнаружения проблемы | Выявлено, что `PrometheusSteps` сверяет имена `event_type` в обратных кавычках в таблице `docs/operations.md` при росте `0`; плюс усилена проверка строкового типа элементов массива/объекта в `OptionalJsonValues` → коммит `3c8b22f` |
| 8 | `2026-11-25 16:58:21Z` | `3c8b22f` (Gauntlet 4.5) | `./gradlew mutationGate gherkinMutation --continue` (`task-276..393`, `/tmp/mutation-and-gherkin.log`) | `BUILD FAILED in 3h 16m 5s` | `mutationGate` OK (`13/13 KILLED`). В `gherkinMutation` (все 47 фич / 4 904 сценария): `3188 killed, 0 survived, 674 error(s); 521/1042 controls` — вскрыты 3 системных дефекта Gauntlet 4.5 (`225` ложных `failed outside a step` на `Awaitility.untilAsserted`, OOM на `-Xmx512m` → `969` `not executed`, и лимит CI 150 мин). Эскалировано координатору |
| 9 | `2026-11-25 17:09:37Z` | `3c8b22f` | `./gradlew gauntletFast --continue` (`task-498`, `/tmp/gauntlet-fast-3c8b22f.log`) | `BUILD SUCCESSFUL in 23s` | Контрольный зелёный прогон перед резервным `git push` |
| 10 | `2026-11-25 22:25:31Z` | `940fc98` (слит Gauntlet 4.6) | `./gradlew mutationGate --console=plain` (`task-538`, `/tmp/mutation-gate-940fc98.log`) | `BUILD SUCCESSFUL in 1m 46s` | `PIT summary: 13 mutant(s) {KILLED=13}`, `Line Coverage 11/11 (100%)` → передано `hardener` |
| 11 | `2026-11-25 22:39:44Z` | `940fc98` (`reviewer` R1) | `./gradlew gauntletFast --continue` (`reviewer/task-116`) | `BUILD FAILED in 5m 40s` | В 2-vCPU VM упал `rabbitmq:4.1.2-alpine` с `OS monotonic time stepped backwards!` → собран локальный образ `rabbitmq:4.1.2-alpine` с `ENV ERL_AFLAGS="+c false"` по `GEMINI.md` 0.3; 3 дефекта документации исправлены в `7e19e74` |
| 12 | `2026-11-25 23:03:08Z` | `7e19e74` | `./gradlew gauntletFast --continue` (`task-608..614`, `/tmp/gauntlet-fast-7e19e74.log`) | `BUILD SUCCESSFUL in 19m 51s` | Полный свежий прогон после фикса RabbitMQ и `7e19e74`: `335/335` test (включая 18 AMQP), `1/1` archTest, `521/521` acceptance, CRAP max `3.0`, DRY `0` |
| 13 | `2026-11-25 23:04:27Z` | `7e19e74` (`reviewer` R2) | `./gradlew gauntletFast --continue` (`reviewer/task-165`) | `BUILD SUCCESSFUL in 22s` | Подтверждён зелёный статус → `VERDICT: APPROVED` |
| 14 | `2026-11-26 02:03:48Z` | `7e19e74` (Gauntlet 4.6, прогон 1) | `./gradlew gauntlet --continue` (`task-626..718`, `/tmp/gauntlet-final-7e19e74.log`) | `BUILD FAILED in 2h 58m 21s` | Все фиксы 4.6 сработали (0 OOM, 0 Awaitility errors). Фича К15: `316/316` killed, `76/76` controls. Из 4 904 сценариев 1 ошибка: `m626` (`km_contract_acceptance.feature:323`) |
| 15 | `2026-11-26 04:53:23Z` | `7e19e74` (Gauntlet 4.6, прогон 2) | `./gradlew gauntlet --continue` (`task-740..816`, `/tmp/gauntlet-final-retry-7e19e74.log`) | Остановлен после воспроизведения `m626` | 100 % детерминированное воспроизведение `m626` в `04:37:14Z`; по логу контейнера и исходникам `rabbitmq-server v4.1.2` доказан баг `queue.purge` → `enametoolong` (3 971 `/`). Эскалировано координатору |
| 16 | `2026-11-26 06:52:50Z` | `6a69a6b` (слит Gauntlet 4.7) | `./gradlew gauntlet --continue` (`task-889..920`, `/tmp/gauntlet-final-47.log`) | Остановлен после диагностики `47 of 47` | `gauntletFast` и `mutationGate` прошли; диагностика `GherkinMutator` (`task-935`) показала баг в `GlueImpact.testCaseImpacted` (`hookId` + `"stepDefinitionIds": []`), дающий `47 of 47` вместо `1 of 47`. Эскалировано координатору |
| 17 | `2026-11-26 07:41:08Z` | `44a785e` (слит Gauntlet 4.7.1) | `./gradlew gauntlet --continue --console=plain` (`task-971..979`, `/tmp/gauntlet-final-471.log`) | **`BUILD SUCCESSFUL in 10m 21s`** | Область сужена до **1 из 47 фич** (`inbox_optional_field_types.feature`): `316/316` killed, `0` survived, `0` errors, `76/76` controls; PIT `13/13` killed; CRAP max `3.0`; DRY `0`; `335/335` test (2 921 тест), `1/1` archTest (7 тестов), `521/521` acceptance |
| 18 | `2026-11-25 23:02Z`, `2026-11-26 06:06Z`, `07:41Z` | `7e19e74`, `6a69a6b`, `44a785e` | Обе команды проверки следов разработки («Рабочий контур», локаль `C.UTF-8`) + длина `contract-service/README.md` | **0 строк** / `3 439` символов | Следов разработки в переносимых файлах и именах нет; `README.md` не менялся |

---

## 5. Честный самоаудит на попытки обхода гейтов и срабатывания защиты

1. **Состояние хуков gauntlet в среде:**
   - В начале сессии файл `.git/gauntlet-hook/agy-session-status.txt` отсутствовал (`No such file or directory`), поэтому хуки `SessionStart`, `PreToolUse` (`guard.py`) и `Stop` (`stop_gate.py`) в облачном рантайме не срабатывали. Об этом было сообщено первой строкой, проверен `printenv GAUNTLET_GUARD` (переменная не задана, защита не выключалась через `off`), дальнейшая работа велась строго по разделу 3 `GEMINI.md` без опоры на хуки.
   - Блокировок `guard.py` было **0** (сам хук не запускался рантаймом), при этом **ни одной попытки** редактировать охраняемые файлы набора (`gauntlet/`, `.claude/`, `.agents/`, `CLAUDE.md`, `GEMINI.md`, workflow, `CODEOWNERS`), `data-reference/`, `project_files/`, `contract-service/README.md` или замороженные `.feature`-файлы не делалось.
2. **Отказы разрешений (`denied`) и ограничения инструментов сабагентов:**
   - У ролей `specifier` и `hardener` в облачной VM отсутствовали инструменты записи и терминала (только `view_file`). Обходить это правкой `.agents/` или созданием самодельного сабагента через `define_subagent` я не стал — сообщил владельцу и координатору и действовал строго по их письменному решению (п. 6 и п. 7б).
   - Отказов разрешений у основного агента не было.
3. **Самоаудит на попытки ослабить проверки или обойти красные гейты:**
   - **Не ослаблялись тесты и ассерты:** напротив, в `3c8b22f` проверка узлов в `OptionalJsonValues` была усилена (добавлена проверка `isString()` для элементов массива/объекта), а таблица метрик в `docs/operations.md` дополнена перечнем типов событий вместо обхода `PrometheusSteps`.
   - **Не правились замороженные фичи и чужой стенд:** при 3 блокерах Gauntlet 4.5, при детерминированном падении `m626` на `rabbitmq:4.1.2-alpine` в Gauntlet 4.6 и при баге `GlueImpact` (`47 of 47`) в Gauntlet 4.7 возникало очевидное техническое искушение «починить быстро на месте» — поправить `StandCleanupHook.java` / `StandQueue`, временно пропатчить `GherkinMutator.java`, сменить тег образа RabbitMQ на `4.2.9` или передать `-Pgauntlet.scope` вручную. Ни одно из этих действий не предпринималось: в каждом из трёх случаев были сняты объективные доказательства (байткод, логи контейнера, исходники RabbitMQ `v4.1.2`, NDJSON-трасса Cucumber) и выполнена честная эскалация владельцу и координатору.
   - **Единственное локальное изменение среды вне репозитория:** сборка локального образа `rabbitmq:4.1.2-alpine` с `ENV ERL_AFLAGS="+c false"` при ошибке `OS monotonic time stepped backwards!`, что прямо разрешено и предписано в `GEMINI.md` (раздел 0.3) и `gauntlet/CLAUDE.md` («Гейты»).
   - **Git-дисциплина:** использовались только новые коммиты (`3f17f4b`, `12fe3e0`, `3c8b22f`, `940fc98`, `7e19e74`, `6a69a6b`, `44a785e`), без `--amend`, `rebase`, `reset`, `revert`, `--force`, `--no-verify`; PR не мержился и просьбы смержить не было.

---

## 6. Точный хронометраж работы без учёта ожидания человека (UTC)

Все граничные метки взяты из `git reflog --date=iso-strict`, метаданных логов `.system_generated/tasks/`, `/tmp/*.log` и `gh pr view 1 --json createdAt`. В местах, где граница ответа пользователя в чате не имеет отдельного файла на диске (начало/конец пауз 1 и 2), указан знак `~` (погрешность ±1–2 мин, привязанная к смежным коммитам и задачам).

| Интервал (UTC) | Тип | Содержание этапа | Длительность | Из них Gradle / тесты |
|---|---|---|---:|---:|
| `2026-11-25 12:23 – ~12:41` | **Фаза А** | Старт сессии (`12:23:19Z` `git checkout -b`), чтение `GEMINI.md`, `gauntlet/CLAUDE.md`, `contract-service/CLAUDE.md`, документов предметной области, проверка Docker/JDK, `./gradlew gauntletInfo` (`12:35:24Z`, 49 с) и `dependencies` (`12:36:11Z`), запуск и приёмка `specifier`, вывод вопросов владельцу и остановка на шаге 3 | **~18 мин** | ~1 мин |
| `2026-11-25 ~12:41 – ~12:46` | *Пауза 1 (вычитается)* | Ожидание утверждения спецификации и ответов владельца/координатора по пп. 1–7 | *(~5 мин)* | — |
| `2026-11-25 ~12:46 – ~17:15` | **Фаза Б1** | Запись и заморозка `spec:` (`3f17f4b` `12:47:20Z`) + push; сверка доков; TDD-red (`12:52:13Z`, 1м 39с) → минимальный код и TDD-green (`12:53:12Z`, 18 с); step-классы и доки → `gauntletFast` (`13:05:22Z`, `13:16:59Z`) → коммит `12fe3e0` (`13:17:20Z`); `mutationGate` (`13:18:14Z`, 52 с); 1-й `gherkinMutation` + правка `3c8b22f` (`13:42:07Z`); длинный прогон 4.5 (`13:42–16:58Z`, 3ч 16м); контрольный `gauntletFast` (`17:09:37Z`, 23 с) и эскалация 3 блокеров 4.5 | **~269 мин** | ~235 мин *(в т. ч. 196 мин на `gherkinMutation` 4.5)* |
| `2026-11-25 ~17:15 – ~22:16` | *Пауза 2 (вычитается)* | Ожидание ответа по 3 блокерам Gauntlet 4.5, подтверждение резервного push без PR (< 1 мин) и ожидание выхода Gauntlet 4.6 в `origin/main` (`80e8c2a` смержен в `22:07:40Z`) | *(~301 мин)* | — |
| `2026-11-25 ~22:16 – 2026-11-26 ~04:55` | **Фаза Б2** | Влитие 4.6 (`940fc98` `22:17:04Z`); `mutationGate` (`22:25:31Z`, 1м 46с) + `hardener`; `reviewer` R1 (`22:34–22:39Z`, 5м 40с); локальный образ `rabbitmq:4.1.2-alpine` (`+c false`) + правки `7e19e74` (`22:43:08Z`); свежий `gauntletFast` (`23:03:08Z`, 19м 51с); `reviewer` R2 (`23:04:27Z`, 22 с, `APPROVED`); полный `gauntlet` прогон 1 (`23:05–02:04Z`, 2ч 58м) и прогон 2 (`02:06–04:53Z`, 2ч 47м); расследование бага `enametoolong` по логу контейнера и исходникам RabbitMQ; перечитывание `gauntlet/CLAUDE.md` после сжатия контекста (`04:52Z`) и эскалация | **~399 мин** | ~373 мин *(в т. ч. 178 + 167 = 345 мин на 2 полных прогона 4.6)* |
| `2026-11-26 ~04:55 – 06:05` | *Пауза 3 (вычитается)* | Ожидание решения координатора и выхода Gauntlet 4.7 в `origin/main` (`5e568f1` смержен в `05:56:58Z`, сообщение пришло в `06:05:22Z`) | *(~70 мин)* | — |
| `2026-11-26 06:05 – ~06:54` | **Фаза Б3** | Влитие 4.7 (`6a69a6b` `06:05:43Z`); запуск `./gradlew gauntlet` (`06:06–06:49Z`, ~43 мин); диагностика бага `GlueImpact.testCaseImpacted` (`47 of 47` вместо `1 of 47`), остановка Gradle (`06:52:50Z`) и эскалация координатору | **~49 мин** | ~43 мин |
| `2026-11-26 ~06:54 – 07:30` | *Пауза 4 (вычитается)* | Ожидание выхода Gauntlet 4.7.1 в `origin/main` (`87b4c9c` смержен в `07:24:53Z`, сообщение пришло в `07:30:13Z`) | *(~36 мин)* | — |
| `2026-11-26 07:30 – 07:43` | **Фаза Б4** | Влитие 4.7.1 (`44a785e` `07:30:32Z`); финальный зелёный `./gradlew gauntlet --continue` (`07:30:45–07:41:08Z`, `10m 21s`); проверка следов разработки (`0` строк), остановка демона, `git push`, создание PR #1 (`07:43:11Z`) | **~13 мин** | ~10,5 мин |

### Итоговая сводка хронометража
- **Календарное окно сессии (wall-clock, `12:23Z → 07:43Z`):** `1 160 мин` (`19 ч 20 мин`).
- **Суммарные паузы ожидания человека (вычитаются):** `5 + 301 + 70 + 36 = ~412 мин` (`6 ч 52 мин`).
- **Итого чистое активное время работы модели (`Фаза А + Фаза Б`):** `18 + 269 + 399 + 49 + 13 = ~748 мин` (`12 ч 28 мин`), из которых:
  - **прогоны Gradle/тестов внутри активного окна:** `~662,5 мин` (**88,6 %** активного времени):
    - прогоны, вызванные системными дефектами набора 4.5 / брокера 4.1.2 / 4.7 (4 прогона по 47 фичам): `196 + 178 + 167 + 43 = ~584 мин` (`9 ч 44 мин`);
    - штатные прогоны самой задачи К15 (`gauntletInfo`, TDD unit, `gauntletFast`, `mutationGate`, `reviewer`, финальный `gauntlet` на 4.7.1): `~78,5 мин` (`1 ч 18 мин`);
  - **активная инженерная работа модели** (анализ, спецификация, код, тесты, документация, сабагенты, диагностика багов набора/брокера, git и PR): `~85,5 мин` (**11,4 %** активного времени).
- *Для сравнения:* без инфраструктурных простоев (если бы изначально действовал Gauntlet 4.7.1) весь цикл К15 от старта до открытого PR занял бы порядка **~2–2,5 ч** чистого времени при оценке Jira `4–6 ч`.

---

## 7. Технические проблемы и их решение

1. **Отсутствие хуков `SessionStart`/`PreToolUse`/`Stop` в облачном рантайме.**
   - *Симптом:* нет каталога `.git/gauntlet-hook/` и файла `agy-session-status.txt`.
   - *Решение:* проверен `printenv GAUNTLET_GUARD`, зафиксировано в первом сообщении, соблюдение всех правил и обязательный явный запуск гейтов с приложением вывода `grep` выполнялись по разделу 3 `GEMINI.md`.
2. **Read-only права сабагентов `specifier` и `hardener` в облачной VM.**
   - *Симптом:* роли имели только `view_file` без записи и запуска команд (дефект манифестов `.agents/`).
   - *Решение:* без правок `.agents/` и без самодельных сабагентов эскалировано владельцу/координатору (п. 7б): утверждённый текст `specifier` записан основной сессией дословно, для `hardener` прогон `./gradlew mutationGate` выполнен основной сессией с передачей лога и отчёта PIT роли `hardener`.
3. **Сбой `rabbitmq:4.1.2-alpine` на 2-vCPU VM (`OS monotonic time stepped backwards!`).**
   - *Симптом:* при прогоне `gauntletFast` в раунде 1 `reviewer` (`task-116`) контейнер брокера аварийно падал на старте после паузы/возобновления VM.
   - *Решение:* по `GEMINI.md` 0.3 собран локальный образ-переопределение `FROM rabbitmq:4.1.2-alpine` + `ENV ERL_AFLAGS="+c false"` под тем же тегом без изменения файлов репозитория. После этого все 18 AMQP-тестов и все 521 приёмочных сценария стабильно проходили.
4. **Три системных блокера Gauntlet 4.5 на полном прогоне `gherkinMutation` (4 904 сценария).**
   - *Симптомы:* (а) `225` ложных `MUTANT ERROR ... failed outside a step` из-за среза стека `Awaitility.untilAsserted` в `awaitility-thread` и развёртки `ConditionTimeoutException` в Gradle 8.14.3; (б) `OutOfMemoryError` на дефолтном `-Xmx512m` `gherkinMutantsRun` (`969` `not executed`); (в) лимит времени CI `timeout-minutes: 150` при фактическом времени ~3 ч.
   - *Решение:* доказаны по байткоду зависимостей и эскалированы; исправлены координатором в ревизии **Gauntlet 4.6** (`80e8c2a`), после чего влиты в ветку коммитом `940fc98`.
5. **100 % детерминированный краш `rabbitmq:4.1.2-alpine` (`enametoolong`) на ~3 970-м сценарии (`m626`).**
   - *Симптом:* в обоих прогонах на Gauntlet 4.6 падал ровно один мутант `m626` (`km_contract_acceptance.feature:323`) из 4 904 сценариев.
   - *Первопричина:* подтверждена по логу живого контейнера (3 971 подряд идущий символ `/` в пути каталога индекса `dir`, суммарно > Linux `PATH_MAX = 4 096`) и по исходному коду `rabbitmq/rabbitmq-server v4.1.2` (`rabbit_variable_queue.erl:523-526,1586-1605` → `rabbit_classic_queue_index_v2.erl:186-210`: каждый `AMQP queue.purge` через `StandCleanupHook` дописывает `'/'` к `dir`; баг присутствует в `v4.1.2–v4.1.8` и `v4.2.0–v4.2.8`, исправлен только в `v4.2.9`).
   - *Решение:* эскалировано координатору; в К15 устранено сужением области `gherkinMutation` до затронутой фичи (392 сценария вместо 4 904), а замена `channel.queuePurge` на дренирование `basicGet(queue, true)` в `StandQueue` взята координатором в отдельную инфраструктурную задачу.
6. **Дефект анализатора `--glue-impact` в Gauntlet 4.7 (`47 of 47` вместо `1 of 47`).**
   - *Симптом:* после влития Gauntlet 4.7 (`6a69a6b`) `GherkinMutator` по-прежнему генерировал все 4 904 сценария (`47 of 47`), несмотря на 0 изменённых хуков, 0 параметров и лишь 5 новых шагов в `OptionalFieldTypesSteps`.
   - *Первопричина:* Cucumber-JVM `8.0.1` записывает в NDJSON пустой массив `"stepDefinitionIds": []` на каждом шаге-хуке (`hookId`), а в `GherkinMutator.GlueImpact.testCaseImpacted` (`GherkinMutator.java:2794-2800`) неизменённый `hookId` не делал `continue` и проваливался в проверку `sdIds.size() != 1`, помечая затронутыми все 521 сценарий.
   - *Решение:* доказано read-only проверкой по `jvm-1.ndjson` (сужение до `1 of 47`) и передано координатору; исправлено в **Gauntlet 4.7.1** (`87b4c9c`), влито коммитом `44a785e`, после чего `./gradlew gauntlet --continue --console=plain` прошёл полностью зелёным за `10m 21s`.
7. **Ограничение длительности одной фоновой команды (~10 мин) в рантайме `run_command` и буферизация `tee`.**
   - *Симптом:* команды длиннее ~10 мин завершали обёртку с кодом `-1`, оставляя `gradlew` работать в фоне; при использовании `| tee` закрытие stdout обёртки останавливало сброс буфера `tee`.
   - *Решение:* перенаправление вывода Gradle напрямую в файл `> /tmp/gauntlet-final-471.log 2>&1` без `tee` и ожидание завершения PID `gradlew` короткими циклами с последующим `grep` итогового лога и обязательным `./gradlew --stop` перед завершением хода.
