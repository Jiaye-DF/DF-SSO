---
name: setup-cicd
description: 掃描專案技術棧(React / Vue / Jinja2 / Node.js / FastAPI / Java),自動產生對接中央 ci-workflows 的 GitHub Actions caller(.github/workflows/ci-cd.yml,有獨立前端再加 e2e.yml)。當使用者說「設定 CI/CD / 接 Auto-CI-CD / 產生 workflow / setup ci」時觸發。
---

# setup-cicd — 產生 Auto-CI-CD caller

把 User 專案接上公司 Auto-CI-CD。CI/CD 的實際邏輯都在中央私有 repo `ci-workflows` 的 reusable workflow;本 skill 只做兩件事:

1. **掃描**專案技術棧 → 判斷該接哪幾支中央 reusable。
2. 依本資料夾的 `ci-cd.yml` / `e2e.yml` **模板**客製 → 寫進專案 `.github/workflows/`。

產出的 caller 是薄殼,不含任何 build / test / 審查 / 部署邏輯。

## 心法

- **不自由發揮** — 只改模板裡「接哪幾支 reusable」,其餘逐字照抄。
- **判不準就問** — 技術棧無法明確判斷時停下來問使用者,不要猜。
- **輸出語言**:繁體中文。

## 中央 repo 參數(產生前先與使用者確認)

| 項目 | 預設 | 說明 |
| --- | --- | --- |
| 中央 repo | `Dafon-IT/DF-AI-Spec` | reusable workflow 放在中央 repo 的 `.github/workflows/`(GitHub Actions 硬性要求) |
| 版本 ref | `@v1.0.0`(模板預設) | 只用 fixed SemVer tag 或 40-char SHA;**禁用 floating tag(`@v1`)/ 禁用 branch(`@main`)**。完整 SOP 見 [`Github-CI/RELEASE.md`](../RELEASE.md) |

模板內以此為預設;若不同,產生時一併替換。產生後**建議**幫 User 一併加 `.github/dependabot.yml`(`package-ecosystem: github-actions`),讓中央發新 patch tag 時自動 PR 升版。

## 執行步驟

### 1. 偵測前端

| 偵測到 | 判定 | 處理 |
| --- | --- | --- |
| `frontend/package.json`(deps 含 `react` 或 `vue`) | npm 前端(React / Vue) | `frontend-ci` 接 `ci-frontend-node.yml` |
| 無 `frontend/` 目錄 | 無獨立前端 | 刪 `frontend-ci` job、不產 `e2e.yml` |

> **Jinja2 / Thymeleaf 等 server-rendered**:HTML 模板在後端目錄裡,不是獨立可建置前端 → 視為「無獨立前端」。模板品質由後端 CI 的測試涵蓋,不另接前端 reusable。

### 2. 偵測後端

| 偵測到 | 判定 | `backend-ci` 接 | 必填 `with:` |
| --- | --- | --- | --- |
| `backend/pyproject.toml` | Python(FastAPI / Tortoise / Django 等) | `ci-backend-python.yml` | `migration_tool`(見步驟 2.5) |
| `backend/package.json` | Node.js | `ci-backend-node.yml` | `migration_tool`(見步驟 2.5) |
| `backend/pom.xml` | Java / Maven | `ci-backend-java.yml` | `migration_tool`(見步驟 2.5) |
| `backend/build.gradle`(或 `.kts`) | Java / Gradle | `ci-backend-java.yml` | `build_command: "./gradlew build"` + `migration_tool` |

> 專案結構非 `frontend/` + `backend/` 拆分時:掃根目錄找對應 manifest,把判定結果列給使用者確認後再產。

### 2.5 偵測 Migration 工具

中央 `ci-backend-*.yml` 都有 `migration_tool` input,依下表自動填值(對齊 [process-design-v1.1.md § Migration 偵測](../../Architecture/Auto-CI-CD/workflow-v1.1/process-design-v1.1.md#migration-偵測per-stack-分派)):

| 後端棧 | 偵測訊號 | `migration_tool` 值 |
| --- | --- | --- |
| Python | `backend/alembic.ini` + `backend/alembic/` | `alembic`(預設) |
| Python | `backend/pyproject.toml` 含 `tortoise-orm` + `backend/migrations/` | `aerich` |
| Python | `backend/manage.py` 存在 | `django`(僅 forward,無 round-trip) |
| Python | 都偵測不到 | `none` |
| Node | `backend/prisma/schema.prisma` + `backend/prisma/migrations/` | `prisma`(用 reset 替代 down) |
| Node | `backend/package.json` deps 含 `typeorm` + `backend/src/migration/`(或 `migrations/`) | `typeorm` |
| Node | `backend/knexfile.{js,ts}` | `knex` |
| Node | 都偵測不到 | `none`(預設) |
| Java | `pom.xml` 含 `flyway-core` + `src/main/resources/db/migration/` | `flyway`(社群版用 clean+migrate 替代 undo) |
| Java | `pom.xml` 含 `liquibase-core` + `src/main/resources/db/changelog/` | `liquibase` |
| Java | 都偵測不到 / Hibernate auto-ddl | `none`(預設) |

> 偵測不明確(例:同時存在 alembic + django,或 Node 多個 migration 套件)→ 停下問使用者,不要猜。

### 3. 產生 .github/workflows/ci-cd.yml

讀本資料夾的 `ci-cd.yml` 模板,依步驟 1、2、2.5 客製:

- `backend-ci` 的 `uses:` 換成步驟 2 判定的 reusable。
- `backend-ci` 的 `with:` 補 `migration_tool: <步驟 2.5 判定值>`;Gradle 專案 另補 `build_command: "./gradlew build"`。
- **無獨立前端**:刪整個 `frontend-ci` job、`auto-cicd` 的 `needs` 改 `[backend-ci]`、移除 `with.frontend_result` 那行。
- 中央 repo / tag 與「中央 repo 參數」一致。

寫入專案 `.github/workflows/ci-cd.yml`。

### 4. 產生 .github/workflows/e2e.yml(有獨立前端才產)

讀本資料夾的 `e2e.yml` 模板,替換中央 repo / tag 後寫入專案 `.github/workflows/e2e.yml`。預設 `if: false`(停用);啟用條件見 Harness 05-CI/06-e2e。

### 5. 回報使用者

- 列出偵測結果(前端 / 後端各判成什麼)與產生的檔。
- 提醒:caller 不含 CI/CD 細節,實際邏輯在中央 `ci-workflows`,User 無須也無從得知內部。
- 中央團隊須在「組織層級」備妥下列項(本專案無須逐項設):
  - **Variables**:`AGENT_PLATFORM_URL`、`NOTIFIER_URL`、`AUDIT_URL`、`AUTO_MERGE_ENABLED`
  - **Secrets**:`GITLEAKS_LICENSE`、`AGENT_PLATFORM_KEY`、`AUDIT_KEY`、`AUTO_MERGE_TOKEN`、`COOLIFY_DEPLOY_WEBHOOK`、`COOLIFY_TOKEN`
- `main` 分支保護的 required status checks 設定見 Harness 05-CI/07-branch-protection。
