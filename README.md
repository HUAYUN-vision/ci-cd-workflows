# ci-cd-workflows

Переиспользуемые GitHub Actions workflow и шаблоны конфигов для всех проектов.

| Шаблон | Назначение |
| ------ | ---------- |
| [`node-ci.yml`](.github/workflows/node-ci.yml) | Установка зависимостей через Socket Firewall + формат, линт, типы, тесты, сборка |
| [`version.yml`](.github/workflows/version.yml) | Автоповышение версии по Conventional Commits (только `main`) |
| [`dependabot-sync.yml`](.github/workflows/dependabot-sync.yml) | Синхронизация `.github/dependabot.yml` с шаблоном из [`templates/dependabot/`](templates/dependabot) |

## Быстрое подключение (Node.js / NestJS)

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
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/node-ci.yml@main
    permissions:
      contents: read

  version:
    needs: ci
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/version.yml@main
    permissions:
      contents: write
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
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/dependabot-sync.yml@main
    permissions:
      contents: write
      pull-requests: write
    with:
      template: nestjs # или node
```

### Настройки репозиториев

- Если `ci-cd-workflows` приватный: **Settings → Actions → General → Access** →
  «Accessible from repositories in the organization».
- Для `dependabot-sync` в проекте: **Settings → Actions → General → Workflow permissions** →
  «Allow GitHub Actions to create and approve pull requests».

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
| `templates-ref`        | `main`       | Ветка/тег шаблонов |
| `branch`               | `chore/dependabot-sync` | Ветка для PR |

| Secret            | Описание |
| ----------------- | -------- |
| `templates-token` | Токен на чтение репозитория шаблонов, если он приватный и `GITHUB_TOKEN` к нему не имеет доступа |
