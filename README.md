  - webshot — бэкенд веб‑архива вроде archive.today: принимает URL, ставит задачу на захват страницы через headless Chromium, упаковывает результат в форматах WACZ и WARC, сохраняет артефакт в
    S3‑совместимое хранилище и хранит метаданные и статусы в Postgres (см. schema/webshot.yaml:1, webshotd/src/crawler/runner.cpp:1, webshotd/src/crawler/artifacts.cpp:1, webshotd/sql/schema/
    capture_meta_db.sql:1, webshotd/sql/schema/shared_state_db.sql:1).

  Архитектура

  - Основной сервис: webshotd (C++20, userver), HTTP API и фоновые задачи (webshotd/src/main.cpp:1, webshotd/config/static_config.yaml:1).
  - Хранилища: две базы Postgres — capture_meta_db (таблица capture: UUID→location; индексы для выборок с пагинацией и очистки) и shared_state_db (таблицы crawl_job + host_denylist); используются ltree для
    сопоставления по префиксу и GiST‑индексы (webshotd/sql/schema/*.sql:1).
  - Объектное хранилище: S3 API (в dev — SeaweedFS), опционально STS‑учётные данные с автообновлением (см. webshotd/config/config_vars.*.yaml:1, webshotd/CMakeLists.txt:1).
  - Краулер: управляет Chromium через DevTools Protocol (CDP), скачивает HTML и ресурсы, собирает WARC и упаковывает в WACZ (DTO и форматы описаны в schema/cdp.yaml:1, schema/wacz.yaml:1, schema/
    browsertrix_pages.yaml:1).

  API

  - POST /v1/capture — создать задачу захвата, GET /v1/capture/jobs/{uuid} — статус задачи, GET /v1/capture/{uuid} — 302 на WACZ в хранилище, выборки с пагинацией по точному URL и по префиксу (schema/webshot.yaml:1).
  - Denylist и purge (очистка): POST /v1/denylist/check и POST /v1/denylist/disallow_and_purge (с удалением из S3 и чисткой БД) (schema/webshot.yaml:1, webshotd/src/crud.cpp:760).
  - Документация: RapiDoc на /rapidoc, OpenAPI раздаётся как статика (webshotd/src/docs_handler.cpp:1, webshotd/config/static_config.yaml:1).

  Инженерные темы

  - Надёжность и тестируемость: есть pytest‑testsuite на pytest_userver, плюс chaos‑gates для эмуляции сетевых проблем с Postgres и S3 (webshotd/test/conftest.py:1).
  - Производительность и конкурентность: фоновые задачи, ограничения по ресурсам краулера, период ожидания (cooldown) перед повторными запросами одного URL (webshotd/src/crud.cpp:780, webshotd/config/
    static_config.yaml:1).
  - Dev‑инфра: devenv и Nix tasks + локальный s6‑рантайм, который поднимает Postgres, прокси (mitmproxy), SeaweedFS и test-target‑сервер для функциональных сценариев (devenv/webshot.nix:1,
    s6/runtime.py:1, nginx/nginx_test.conf:1).
