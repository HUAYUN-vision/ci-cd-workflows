# ci-cd-workflows

Переиспользуемые GitHub Actions workflow для всех проектов.

## version

`.github/workflows/version.yml` автоматически повышает версию проекта по
[Conventional Commits](https://www.conventionalcommits.org/). **Работает только на ветке `main`.**

| Коммиты с последнего тега                       | Изменение версии |
| ----------------------------------------------- | ---------------- |
| `feat!: ...`, `fix(api)!: ...`, `BREAKING CHANGE:` в теле | major (`1.2.3 → 2.0.0`) |
| `feat: ...`                                     | minor (`1.2.3 → 1.3.0`) |
| `fix: ...`, `perf: ...`                         | patch (`1.2.3 → 1.2.4`) |
| `chore`, `docs`, `refactor`, `test`, ...        | не меняется      |

Что делает workflow:

1. Находит последний тег `vX.Y.Z` (если тегов нет — берёт версию из файла версии).
2. Считает новую версию по коммитам.
3. Обновляет `package.json` + `package-lock.json` (или текстовый файл `VERSION`).
4. Коммитит `chore(release): vX.Y.Z` в `main`, ставит тег `vX.Y.Z`.
5. Создаёт GitHub Release со списком изменений.

### Подключение

В проекте создайте `.github/workflows/version.yml`:

```yaml
name: Version

on:
  push:
    branches: [main]

jobs:
  version:
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/version.yml@main
    permissions:
      contents: write
    # secrets:
    #   token: ${{ secrets.RELEASE_TOKEN }}  # если main защищена от прямого push
```

### Параметры

| Input            | По умолчанию   | Описание |
| ---------------- | -------------- | -------- |
| `version-file`   | `package.json` | `package.json`, текстовый файл (например `VERSION`) или `''` — только теги |
| `tag-prefix`     | `v`            | Префикс тега |
| `create-release` | `true`         | Создавать GitHub Release |
| `git-user-name`  | `github-actions[bot]` | Автор release-коммита |
| `git-user-email` | `41898282+github-actions[bot]@users.noreply.github.com` | Email автора |

| Secret  | Описание |
| ------- | -------- |
| `token` | PAT или GitHub App token. Нужен, если на `main` включена защита ветки, запрещающая прямой push. По умолчанию — `GITHUB_TOKEN` |

| Output     | Описание |
| ---------- | -------- |
| `released` | `true`, если версия была повышена |
| `version`  | Новая версия, например `1.3.0` |
| `tag`      | Новый тег, например `v1.3.0` |

### Использование результата (например, деплой)

Push от `GITHUB_TOKEN` не запускает другие workflow, поэтому события по
новому тегу не будет. Запускайте следующие шаги в том же workflow:

```yaml
jobs:
  version:
    uses: HUAYUN-vision/ci-cd-workflows/.github/workflows/version.yml@main
    permissions:
      contents: write

  deploy:
    needs: version
    if: needs.version.outputs.released == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy ${{ needs.version.outputs.tag }}"
```

### Важно

- Если репозиторий `ci-cd-workflows` приватный: **Settings → Actions → General → Access** →
  «Accessible from repositories in the organization».
- Release-коммиты `chore(release): ...` пропускаются, поэтому зацикливания нет.
