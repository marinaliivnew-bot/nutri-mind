# Архитектура MVP Nutri Intelligence Agent

Дата: 26 июня 2026

## 1. Назначение документа

Локальный прототип ([nutri-agent/index.html](nutri-agent/index.html)) хранит всё состояние в одном объекте `demoState` в `localStorage`. Этот документ описывает, как та же продуктовая механика переносится в MVP с сервером, реальными пользователями, файлами и серверным AI — без изменения логики, собранной на этапах 1–5.

Главные вопросы, на которые отвечает документ:

- какие сущности из `demoState` становятся таблицами базы данных;
- какие действия интерфейса требуют backend API;
- где живёт AI-вызов;
- сколько примерно стоит MVP.

## 2. Принципы переноса

- Модель данных MVP повторяет доменные сущности прототипа один в один. Никаких новых абстракций, пока они не нужны.
- Прототип уже разделён по `caseId` — в MVP это внешний ключ `case_id`.
- Граница «черновик специалиста ↔ клиентская версия» (этап 4) переносится в права доступа (RLS), а не только в UI.
- AI вызывается только на сервере. Браузер никогда не обращается к модели напрямую.
- Любое действие, меняющее общие данные, проходит через backend API; локальный `localStorage` заменяется на запросы к серверу.

## 3. Технологический стек

Опираемся на уже подключённый к проекту Supabase:

| Слой | Технология | Роль |
|------|-----------|------|
| База данных | Supabase Postgres | Все доменные таблицы, целостность через FK |
| Доступы | Postgres RLS + Supabase Auth | Роли нутрициолога/клиента/админа на уровне строк |
| Аутентификация | Supabase Auth (email + magic link) | Пользователи, сессии, инвайты клиентов |
| Файлы | Supabase Storage (private buckets) | PDF/JPG анализов и методичек; метаданные — в таблице `files` |
| AI и логика | Supabase Edge Functions (Deno) | Серверная генерация черновика, валидация переходов статусов |
| Клиент | Тот же SPA, но с API-слоем | `localStorage` заменяется на `supabase-js` |

Альтернатива (если уходить от Supabase): любой Postgres + Node/Go API + S3-совместимое хранилище + отдельный AI-сервис. Структура таблиц и API остаётся той же.

## 4. Маппинг `demoState` → таблицы

| Сущность прототипа (`demoState.*`) | Таблица MVP | Связь |
|-----------------------------------|-------------|-------|
| `cases[]` | `cases` | принадлежит `practice`, ведётся `nutritionist` (users), привязан к `client` (users) |
| (роль нутрициолога/клиента) | `users` | + `practices` для мультиаккаунта |
| `cases[].name/meta/initials` + анкета профиля | `client_profiles` | 1:1 к `cases` |
| `questionnaires[caseId].answers` | `questionnaires` (+ `questionnaire_answers`) | N к `cases` (версии анкеты) |
| `diaryEntries[caseId].days[]` + `clientMaterials[caseId].diaryExtras` | `diary_entries` | N к `cases` (строка = приём пищи / запись воды-добавок) |
| `uploadedFiles[caseId].client` / `.nutritionist` | `files` | N к `cases`, поле `owner_role` |
| `nutritionistSources[caseId].items[]` | `nutritionist_sources` | N к `practice`/`nutritionist`, привязка к кейсу опциональна |
| `agentDialog[caseId].history[]` + `.stage` | `agent_messages` | N к `cases` |
| `draftReports[caseId].hypotheses[]` | `hypotheses` | N к `draft_reports` |
| `draftReports[caseId]` (summary, gaps, drafts, status) | `draft_reports` | 1:1 (текущий) к `cases` |
| `uiState.selectedCaseId` | не хранится в БД | состояние клиента/сессии |

Итого 11 таблиц из ТЗ: `users`, `practices`, `cases`, `client_profiles`, `questionnaires`, `diary_entries`, `files`, `nutritionist_sources`, `agent_messages`, `hypotheses`, `draft_reports`.

## 5. Схема базы данных

DDL дан в стиле Postgres/Supabase. Типы перечислений можно завести как `enum` или `text` + `check`.

```sql
-- 5.1 Практики (мультиаккаунт: кабинет нутрициолога/клиники)
create table practices (
    id          uuid primary key default gen_random_uuid(),
    name        text not null,
    created_at  timestamptz not null default now()
);

-- 5.2 Пользователи (расширяет auth.users из Supabase Auth)
create table users (
    id          uuid primary key references auth.users(id) on delete cascade,
    practice_id uuid references practices(id) on delete set null,
    role        text not null check (role in ('nutritionist','client','admin')),
    full_name   text,
    created_at  timestamptz not null default now()
);

-- 5.3 Кейсы (== cases[] в demoState)
create table cases (
    id              uuid primary key default gen_random_uuid(),
    practice_id     uuid not null references practices(id) on delete cascade,
    nutritionist_id uuid not null references users(id),
    client_id       uuid references users(id),          -- клиент может быть ещё не приглашён
    title           text not null,                       -- demoState: name (анонимно «Кейс #1001»)
    meta            text,                                -- demoState: meta («26–35 · Снижение веса»)
    status          text not null default 'active'
                    check (status in ('active','pending','completed')),
    created_at      timestamptz not null default now(),
    updated_at      timestamptz not null default now()
);

-- 5.4 Профиль клиента (1:1 к кейсу)
create table client_profiles (
    case_id     uuid primary key references cases(id) on delete cascade,
    age_range   text,
    goals       text,
    notes       text,
    updated_at  timestamptz not null default now()
);

-- 5.5 Анкета (шапка) и ответы
create table questionnaires (
    id          uuid primary key default gen_random_uuid(),
    case_id     uuid not null references cases(id) on delete cascade,
    template    text not null default 'vera_v2',         -- версия опросника
    completed   boolean not null default false,
    updated_at  timestamptz not null default now()
);

create table questionnaire_answers (
    questionnaire_id uuid not null references questionnaires(id) on delete cascade,
    question_index   int  not null,                      -- demoState: ключ questionScores
    section_index    int  not null,
    value            int  not null,                      -- балл 0..3
    primary key (questionnaire_id, question_index)
);

-- 5.6 Дневник питания (строка = приём пищи; вода/добавки — отдельным типом)
create table diary_entries (
    id          uuid primary key default gen_random_uuid(),
    case_id     uuid not null references cases(id) on delete cascade,
    day_index   int  not null,                           -- 0..6 (Пн..Вс)
    kind        text not null default 'meal'
                check (kind in ('meal','water','supplement')),
    time        text,
    food        text,
    method      text,
    portion     text,
    feeling     text,
    note        text,                                    -- для water/supplement (diaryExtras)
    created_at  timestamptz not null default now()
);

-- 5.7 Файлы (метаданные; содержимое — в Storage)
create table files (
    id           uuid primary key default gen_random_uuid(),
    case_id      uuid not null references cases(id) on delete cascade,
    owner_role   text not null check (owner_role in ('client','nutritionist')),
    uploaded_by  uuid references users(id),
    name         text not null,
    size_bytes   bigint,
    mime_type    text,
    storage_path text not null,                          -- путь в bucket
    created_at   timestamptz not null default now()
);

-- 5.8 Источники/методички нутрициолога
create table nutritionist_sources (
    id           uuid primary key default gen_random_uuid(),
    practice_id  uuid not null references practices(id) on delete cascade,
    owner_id     uuid not null references users(id),
    case_id      uuid references cases(id) on delete set null,  -- null = общая методичка
    name         text not null,
    meta         text,                                   -- «PDF · 124 стр.»
    pages        text,
    storage_path text,
    created_at   timestamptz not null default now()
);

-- 5.9 Сообщения диалога с агентом (== agentDialog.history)
create table agent_messages (
    id          uuid primary key default gen_random_uuid(),
    case_id     uuid not null references cases(id) on delete cascade,
    role        text not null check (role in ('agent','user')),
    text        text not null,
    actions     jsonb,                                   -- кнопки [1][2]
    stage       int,                                     -- этап диалога
    used        boolean not null default false,
    created_at  timestamptz not null default now()
);

-- 5.10 Черновик отчёта (1:1 к кейсу — текущая версия)
create table draft_reports (
    case_id             uuid primary key references cases(id) on delete cascade,
    summary             text,
    data_gaps           jsonb not null default '[]',
    nutritionist_notes  text,                            -- НЕ видно клиенту
    recommendation_draft text,                           -- НЕ видно клиенту
    client_version      text,                            -- видно клиенту после одобрения
    stage_status        text,
    status              text not null default 'draft'
                        check (status in ('draft','needs_review','approved_for_client','sent_to_client')),
    updated_at          timestamptz not null default now()
);

-- 5.11 Гипотезы и решения по ним (== draftReport.hypotheses)
create table hypotheses (
    id          uuid primary key default gen_random_uuid(),
    case_id     uuid not null references cases(id) on delete cascade,
    label       text,                                    -- «Гипотеза A: ...»
    action      text,                                    -- «Принять/Уточнить/Отклонить»
    decision    text check (decision in ('accepted','clarify','rejected','noted')),
    confidence  int,                                     -- уровень уверенности %, опционально
    source_ref  text,                                    -- «Орлинская, стр. 18»
    created_at  timestamptz not null default now()
);
```

Замечания:
- `draft_reports` сделан 1:1 к кейсу (как `draftReports[caseId]` в прототипе). Если потребуется история версий черновика — добавить `id` и `version`, убрать `primary key (case_id)`.
- `questionnaire_answers` нормализует объект `answers`; для MVP допустимо хранить и как `jsonb` в `questionnaires.answers` — это упрощает перенос один в один.
- `diary_entries.note` + `kind in ('water','supplement')` заменяют `clientMaterials.diaryExtras`.

## 6. Роли и доступы (RLS)

Три роли, как в ТЗ. Доступ строится на уровне строк (Row Level Security) от `auth.uid()` и `users.role`.

### Нутрициолог
- Видит и редактирует только кейсы своей практики, где он `nutritionist_id`.
- Полный доступ к `draft_reports` (включая `nutritionist_notes`, `recommendation_draft`), `hypotheses`, `nutritionist_sources`, всем `files` кейса, `agent_messages`.
- Может менять статус черновика и переводить кейс в `approved_for_client` / `sent_to_client`.

### Клиент
- Видит только свои кейсы (`cases.client_id = auth.uid()`).
- Может писать: `questionnaires`/`questionnaire_answers`, `diary_entries`, загружать `files` с `owner_role = 'client'`.
- Из `draft_reports` читает **только** `client_version`, и **только** если `status in ('approved_for_client','sent_to_client')`. Поля `nutritionist_notes`, `recommendation_draft`, таблица `hypotheses` ему недоступны вообще.
- Это серверная реализация продуктового правила этапа 4.

### Администратор демо/практики
- Управляет `practices`, приглашает пользователей, назначает роли.
- Доступ к данным практики для поддержки; не выступает медицинским специалистом.

Пример политики (концептуально), закрывающей клиентский доступ к черновику:

```sql
alter table draft_reports enable row level security;

-- Нутрициолог практики видит черновик целиком
create policy draft_nutritionist on draft_reports for all
using (exists (
    select 1 from cases c
    where c.id = draft_reports.case_id
      and c.nutritionist_id = auth.uid()
));

-- Клиент видит строку черновика только когда она одобрена;
-- закрытые поля прячем через VIEW (см. ниже), а не через прямой select по таблице
create policy draft_client_read on draft_reports for select
using (
    status in ('approved_for_client','sent_to_client')
    and exists (
        select 1 from cases c
        where c.id = draft_reports.case_id
          and c.client_id = auth.uid()
    )
);
```

Чтобы клиент физически не получал `nutritionist_notes`/`recommendation_draft`, клиент работает не с таблицей, а с витриной:

```sql
create view client_recommendations as
select case_id, client_version, status
from draft_reports
where status in ('approved_for_client','sent_to_client');
```

## 7. Хранение файлов

- Два приватных bucket'а Supabase Storage: `client-files` и `nutritionist-sources`.
- Путь: `{practice_id}/{case_id}/{file_id}.{ext}`.
- В БД хранятся только метаданные (`files`, `nutritionist_sources.storage_path`); сами PDF/JPG — в Storage.
- Загрузка клиента: подписанный upload-URL выдаёт backend после проверки прав; после загрузки создаётся строка в `files` с `owner_role='client'`.
- Доступ на чтение — через подписанные (signed) URL с коротким TTL, выдаются только тем, кому разрешает RLS.
- В прототипе файлы хранятся как метаданные (`{name,size,type,savedAt}`) — переносится напрямую, добавляется `storage_path`.

## 8. Серверная AI-функция

AI живёт в Edge Function `generateDraft` (Deno). Браузер её не вызывает напрямую — только через backend, который проверяет права нутрициолога на кейс.

Вход (собирается на сервере из БД по `case_id`):
- профиль клиента и ответы анкеты;
- дневник питания, вода, добавки;
- метаданные (и при необходимости извлечённый текст) файлов;
- источники/методички нутрициолога;
- ранее принятые решения по гипотезам.

Выход — структурированный JSON, который сохраняется в `draft_reports`/`hypotheses` как **черновик**:
- `gaps` → `draft_reports.data_gaps`;
- `observations`, `recommendationDraft` → `recommendation_draft`;
- `clientSummaryDraft` → предзаполнение `client_version` (требует одобрения);
- `hypotheses` (+ `sourceRefs`, `confidence`) → таблица `hypotheses`;
- `safetyNotes` → `nutritionist_notes`.

Правила безопасности (повторяют этап 7 плана):
- любой AI-вывод помечается как черновик (`status='draft'`/`needs_review`), не уходит клиенту без действия специалиста;
- модель не выдумывает значений — опирается на загруженные источники; при недостатке данных возвращает «нет данных»;
- никаких диагнозов/назначений в формулировках.

Модель: **OpenAI GPT-4o** (зафиксировано, согласовано с планом этапа 7); вызов через серверный OpenAI SDK с ключом в секретах Edge Function.

## 9. Действия интерфейса, которым нужен backend API

| Действие в прототипе | Сейчас (локально) | В MVP (API) |
|----------------------|-------------------|-------------|
| Выбор/создание кейса | `selectClient`, мутация `demoState` | `GET/POST /cases` |
| Заполнение анкеты | `questionScores` → `localStorage` | `PUT /cases/:id/questionnaire` |
| Запись дневника | `addDiaryEntry` | `POST /cases/:id/diary` |
| Загрузка файла клиента/методички | метаданные в `uploadedFiles` | signed upload URL + `POST /files` |
| Диалог с агентом | `agentDialog.history` в памяти | `POST /cases/:id/messages`, `GET` истории |
| Решения по гипотезам | `recordHypothesisDecision` | `POST /cases/:id/hypotheses` |
| Редактирование черновика/заметок | `handleDraftEdit` | `PUT /cases/:id/draft` |
| Смена статуса / одобрение / отправка | `setDraftStatus`, `approveDraftForClient`, `sendDraftToClient` | `POST /cases/:id/draft/transition` |
| Просмотр рекомендаций клиентом | `renderClientRecommendations` | `GET /cases/:id/client-recommendations` (через витрину) |
| Генерация AI-черновика | имитация в JS | Edge Function `generateDraft` |
| Экспорт/импорт демо | JSON в `localStorage` | в MVP не нужен (данные на сервере); остаётся как админ-инструмент |

Остальное (рендеринг, переключение вкладок, подсчёт баллов) остаётся на клиенте.

## 10. Путь миграции из локального прототипа

1. **Завести схему**: применить DDL из раздела 5 как Supabase-миграцию; включить RLS и политики из раздела 6.
2. **Auth и роли**: подключить Supabase Auth, создать `practices`, посадить пользователей в `users` с ролями.
3. **API-слой**: заменить прямые мутации `demoState` на вызовы `supabase-js`; функции из раздела 9 — точки подключения. `getCurrentCaseData(caseId)` становится загрузкой кейса с сервера.
4. **Файлы**: подключить Storage, заменить метаданные-заглушки на реальную загрузку (signed URL).
5. **Импортер демо**: одноразовый скрипт принимает экспорт `demoState` (этап 5) и раскладывает по таблицам — удобно для переноса наработанных демо-кейсов и для сидинга.
6. **AI**: вынести имитацию диалога в Edge Function `generateDraft`; UI диалога остаётся, источник данных меняется.
7. **Отключить localStorage** как источник истины (оставить только как кэш/офлайн, если потребуется).

Граница «черновик ↔ клиентская версия», история диалога и решения по гипотезам уже собраны в прототипе (этапы 3–4) — миграция переносит их в таблицы и RLS, а не переписывает логику.

## 11. Оценка трудоёмкости MVP (ориентир)

| Блок | Объём | Оценка |
|------|-------|--------|
| Схема БД + RLS + Auth/роли | 11 таблиц, политики, практики | 4–6 дней |
| API-слой (CRUD кейсов, анкета, дневник, файлы) | ~10 эндпоинтов | 5–7 дней |
| Storage и загрузка файлов | bucket'ы, signed URL, метаданные | 2–3 дня |
| Диалог агента на сервере (история, статусы) | messages + переходы статусов | 3–4 дня |
| Edge Function `generateDraft` + интеграция модели | вход/выход JSON, safety-правила | 4–6 дней |
| Переключение клиента с `localStorage` на API | рефакторинг вызовов | 4–5 дней |
| Импортер демо-состояния | одноразовый скрипт | 1–2 дня |
| **Итого** | | **≈ 4–6 недель** одного разработчика |

Реальный AI-черновик (раздел 9 «генерация» + Edge Function) — это этап 7 плана; в этой оценке учтён каркас вызова, а доводка качества промптов и извлечения текста из PDF выносится отдельно.

## 12. Definition of Done этапа 6

- Понятно, какие сущности `demoState` становятся таблицами (раздел 4).
- Понятно, какие действия требуют backend API (раздел 9).
- Понятно, где живёт AI-вызов (раздел 8 — Edge Function `generateDraft`, только сервер).
- Есть схема БД, роли/доступы, хранение файлов и путь миграции — достаточно, чтобы оценить трудоёмкость MVP (раздел 11).
