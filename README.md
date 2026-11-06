# ci-cd-workflows

Переиспользуемые GitHub Actions workflow и шаблоны конфигов для всех проектов.

| Шаблон | Назначение |
| ------ | ---------- |
| [`node-ci.yml`](.github/workflows/node-ci.yml) | Установка зависимостей через Socket Firewall + формат, линт, типы, тесты, сборка |
| [`secret-scan.yml`](.github/workflows/secret-scan.yml) | Поиск секретов в коммитах (Gitleaks) |
| [`codeql.yml`](.github/workflows/codeql.yml) | Статический анализ кода (CodeQL) |
| [`pr-title.yml`](.github/workflows/pr-title.yml) | Проверка заголовка PR по Conventional Commits |
| [`docker.yml`](.github/workflows/docker.yml) | Сборка Docker-образа, сканирование Trivy, публикация в GHCR |
| [`version.yml`](.github/workflows/version.yml) | Автоповышение версии по Conventional Commits (только `main`) |
| [`dependabot-auto-merge.yml`](.github/workflows/dependabot-auto-merge.yml) | Автомерж patch/minor PR от Dependabot после зелёного CI |
| [`dependabot-sync.yml`](.github/workflows/dependabot-sync.yml) | Синхронизация `.github/dependabot.yml` с шаблоном из [`templates/dependabot/`](templates/dependabot) |

## Версии шаблонов

Репозиторий версионирует сам себя ([`release.yml`](.github/workflows/release.yml)): на каждый
push в `main` ставится тег `vX.Y.Z` по Conventional Commits и переносится плавающий тег `vX`.

- Подключайте шаблоны через **`@v1`** — получаете все совместимые исправления.
- Ломающие изменения (новый обязательный input, переименование, смена поведения) — коммит
  `feat!: ...` → выйдет `v2`, проекты на `@v1` не сломаются.
- Dependabot этого репозитория раз в неделю обновляет экшены внутри шаблонов (`fix(deps)` → patch-релиз),
  и все проекты получают обновления автоматически.

## Подключение (Node.js / NestJS)

Готовый пример — [nestjs-template](https://github.com/HUAYUN-vision/payments-service/tree/main/.github/workflows).

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  ci:
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/node-ci.yml@v1
    permissions:
      contents: read

  secret-scan:
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/secret-scan.yml@v1
    permissions:
      contents: read

  docker-check: # PR: сборка + Trivy, без публикации
    if: github.event_name == 'pull_request'
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/docker.yml@v1
    permissions:
      contents: read
      packages: write

  version:
    needs: [ci, secret-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/version.yml@v1
    permissions:
      contents: write

  docker: # main: публикация в GHCR после нового релиза
    needs: version
    if: needs.version.outputs.released == 'true'
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/docker.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      push: true
      ref: ${{ needs.version.outputs.tag }}
      version: ${{ needs.version.outputs.version }}

  dependabot-auto-merge:
    needs: [ci, secret-scan, docker-check]
    if: github.event_name == 'pull_request' && github.event.pull_request.user.login == 'dependabot[bot]'
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/dependabot-auto-merge.yml@v1
    permissions:
      contents: write
      pull-requests: write
```

`.github/workflows/pr-title.yml` — отдельный файл, т.к. нужно событие `edited`
(иначе правка заголовка перезапускала бы весь CI):

```yaml
name: PR title

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

jobs:
  pr-title:
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/pr-title.yml@v1
    permissions: {}
```

`.github/workflows/codeql.yml`:

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 5 * * 1'

jobs:
  codeql:
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/codeql.yml@v1
    permissions:
      contents: read
      security-events: write
      actions: read
```

`.github/workflows/dependabot-sync.yml`:

```yaml
name: Dependabot sync

on:
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:

jobs:
  sync:
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/dependabot-sync.yml@v1
    permissions:
      contents: write
      pull-requests: write
    with:
      template: nestjs # или node
```

### Настройки репозиториев

| Где | Настройка | Зачем |
| --- | --------- | ----- |
| `ci-cd-workflows` (если приватный) | Settings → Actions → General → Access → «Accessible from repositories in the organization» | Чтобы проекты могли вызывать шаблоны |
| Проект | Settings → General → Pull Requests → «Allow squash merging» (и лучше только его) | Заголовок PR становится коммитом → версия |
| Проект | Settings → Actions → General → Workflow permissions → «Allow GitHub Actions to create and approve pull requests» | `dependabot-sync`, approve в `dependabot-auto-merge` |
| Проект | Settings → Rules → правило для `main`: required checks `ci`, `secret-scan`, `pr-title` | Нельзя смержить PR с красным CI |
| Проект (приватный) | Settings → Code security → GitHub Code Security | Нужно для CodeQL |

---

## node-ci

Ставит Node.js (из `.nvmrc`), устанавливает зависимости через
[Socket Firewall](https://github.com/SocketDev/action) — `sfw npm ci` блокирует
известные вредоносные пакеты прямо на установке, — затем запускает npm-скрипты по порядку.

| Input               | По умолчанию | Описание |
| ------------------- | ------------ | -------- |
| `node-version-file` | `.nvmrc`     | Файл с версией Node.js |
| `node-version`      | `''`         | Явная версия Node.js (важнее файла) |
| `working-directory` | `.`          | Папка проекта |
| `scripts`           | `format:check`, `lint:check`, `typecheck`, `test`, `test:e2e`, `build` | npm-скрипты, по одному на строку |
| `socket-firewall`   | `true`       | Ставить зависимости через Socket Firewall |

| Secret         | Описание |
| -------------- | -------- |
| `socket-token` | Socket API token → режим `firewall-enterprise`. Без него — бесплатный `firewall-free` |

Свой набор скриптов:

```yaml
    with:
      scripts: |
        lint:check
        test
        build
```

> Для отчётов о рисках зависимостей в PR дополнительно установите
> [Socket GitHub App](https://github.com/apps/socket-security) на организацию — она работает без workflow.

## version

Повышает версию по [Conventional Commits](https://www.conventionalcommits.org/). **Только на ветке `main`.**

| Коммиты с последнего тега | Изменение версии |
| ------------------------- | ---------------- |
| `feat!: ...`, `fix(api)!: ...`, `BREAKING CHANGE:` в теле | major (`1.2.3 → 2.0.0`) |
| `feat: ...`               | minor (`1.2.3 → 1.3.0`) |
| `fix: ...`, `perf: ...`   | patch (`1.2.3 → 1.2.4`) |
| `chore`, `docs`, `refactor`, `test`, ... | не меняется |

Что делает:

1. Находит последний тег `vX.Y.Z` (если тегов нет — берёт версию из файла версии).
2. Считает новую версию по коммитам.
3. Обновляет `package.json` + `package-lock.json` (или текстовый файл `VERSION`).
4. Коммитит `chore(release): vX.Y.Z` в `main`, ставит тег `vX.Y.Z`.
5. Создаёт GitHub Release со списком изменений.

| Input            | По умолчанию   | Описание |
| ---------------- | -------------- | -------- |
| `version-file`   | `package.json` | `package.json`, текстовый файл (например `VERSION`) или `''` — только теги |
| `tag-prefix`     | `v`            | Префикс тега |
| `create-release` | `true`         | Создавать GitHub Release |
| `git-user-name`  | `github-actions[bot]` | Автор release-коммита |
| `git-user-email` | `41898282+github-actions[bot]@users.noreply.github.com` | Email автора |

| Secret  | Описание |
| ------- | -------- |
| `token` | PAT / GitHub App token, если на `main` запрещён прямой push. По умолчанию — `GITHUB_TOKEN` |

| Output     | Описание |
| ---------- | -------- |
| `released` | `true`, если версия была повышена |
| `version`  | Новая версия, например `1.3.0` |
| `tag`      | Новый тег, например `v1.3.0` |

Push от `GITHUB_TOKEN` не запускает другие workflow, поэтому деплой делайте в том же workflow:

```yaml
  deploy:
    needs: version
    if: needs.version.outputs.released == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy ${{ needs.version.outputs.tag }}"
```

## dependabot-sync

GitHub читает конфиг Dependabot только из самого репозитория — подключить его через `uses:`
нельзя. Поэтому эталонные конфиги лежат в [`templates/dependabot/`](templates/dependabot), а этот
workflow (по расписанию или вручную) открывает в проекте PR, если `.github/dependabot.yml`
отличается от шаблона.

| Input                  | По умолчанию | Описание |
| ---------------------- | ------------ | -------- |
| `template`             | —            | Имя шаблона: `node`, `nestjs` |
| `templates-repository` | `HUAYUN-vision/ci-cd-workflows` | Репозиторий с шаблонами |
| `templates-ref`        | `v1`         | Ветка/тег шаблонов |
| `branch`               | `chore/dependabot-sync` | Ветка для PR |

| Secret            | Описание |
| ----------------- | -------- |
| `templates-token` | Токен на чтение репозитория шаблонов, если он приватный и `GITHUB_TOKEN` к нему не имеет доступа |

## secret-scan

[Gitleaks](https://github.com/gitleaks/gitleaks) ищет токены, ключи и пароли в коммитах.
На PR — только коммиты PR, на push — только новые коммиты (первый запуск — вся история).
Бинарник скачивается напрямую с проверкой SHA-256: `gitleaks-action` требует лицензию для организаций.

| Input              | По умолчанию | Описание |
| ------------------ | ------------ | -------- |
| `gitleaks-version` | `8.30.1`     | Версия gitleaks |
| `gitleaks-sha256`  | `551f6f…70eb` | SHA-256 архива `linux_x64` (обновлять вместе с версией) |
| `full-history`     | `false`      | Всегда сканировать всю историю |

Ложное срабатывание → добавьте fingerprint из лога в `.gitleaksignore` в корне проекта.
Настоящий секрет → **сначала отзовите/смените его**, потом удаляйте из кода (из истории он никуда не денется).

## codeql

Статический анализ [CodeQL](https://codeql.github.com/) (SQL-инъекции, XSS, path traversal,
небезопасные workflow и т.п.). Результаты — Security → Code scanning.
Для приватных репозиториев нужен GitHub Code Security (платно).

| Input       | По умолчанию | Описание |
| ----------- | ------------ | -------- |
| `languages` | `javascript-typescript,actions` | Языки через запятую |
| `queries`   | `security-extended` | `security-extended` или `security-and-quality` |

## pr-title

Проверяет, что заголовок PR — `type(scope)!: описание`. При squash merge заголовок становится
коммитом в `main`, и по нему `version.yml` решает, повышать ли версию.

| Input        | По умолчанию | Описание |
| ------------ | ------------ | -------- |
| `types`      | `feat fix docs style refactor perf test build ci chore revert` | Разрешённые типы |
| `max-length` | `100`        | Максимальная длина |

## docker

Собирает образ, сканирует его [Trivy](https://trivy.dev/) и (при `push: true`) публикует в GHCR
с SBOM и provenance. Теги: `1.3.0`, `1.3`, `1` (для версий ≥ 1.0.0), `latest`, `sha-<short>`.

| Input                  | По умолчанию | Описание |
| ---------------------- | ------------ | -------- |
| `push`                 | `false`      | Публиковать образ |
| `ref`                  | `''`         | Git ref для сборки (тег из `version.yml`, чтобы в образе была новая версия `package.json`) |
| `version`              | `''`         | Версия для тегов образа |
| `image`                | `ghcr.io/<owner>/<repo>` | Имя образа |
| `registry`             | `ghcr.io`    | Registry |
| `context` / `file`     | `.` / `Dockerfile` | Контекст и Dockerfile |
| `platforms`            | `linux/amd64` | Например `linux/amd64,linux/arm64` |
| `trivy`                | `true`       | Сканировать образ |
| `trivy-severity`       | `CRITICAL,HIGH` | При каких уровнях падать |
| `trivy-ignore-unfixed` | `true`       | Не падать на уязвимостях без исправления |

| Secret | Описание |
| ------ | -------- |
| `registry-username` / `registry-password` | Для других registry. По умолчанию — `github.actor` / `GITHUB_TOKEN` (GHCR) |

Принятые риски по Trivy — в `.trivyignore` в корне проекта (по одному CVE на строку).

> `trivy-action` закреплён на SHA `v0.36.0`: версии ≤ 0.34.2 были скомпрометированы
> в марте 2026 ([CVE-2026-33634](https://nvd.nist.gov/vuln/detail/cve-2026-33634)). Не меняйте на старые теги.

## dependabot-auto-merge

Вызывается через `needs:` после CI-джобов, поэтому мержит только при зелёных проверках.
Patch/minor-обновления мержатся автоматически, major остаются на ручное ревью.

| Input          | По умолчанию | Описание |
| -------------- | ------------ | -------- |
| `update-types` | `version-update:semver-patch,version-update:semver-minor` | Что мержить |
| `merge-method` | `squash`     | `squash`, `merge`, `rebase` |

| Secret  | Описание |
| ------- | -------- |
| `token` | PAT / App token. Мерж от `GITHUB_TOKEN` не запускает push-workflow в `main`. Для Dependabot это обычно не важно (`chore(deps)` не меняет версию) |
