# Maintained Fork Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Превратить `serezha93/go-ts3` в устанавливаемый, проверяемый и обнаруживаемый community-maintained fork с первым релизом `v1.3.0`.

**Architecture:** GitHub fork relationship и история upstream сохраняются, а Go module identity меняется на `github.com/serezha93/go-ts3`. Изменения проходят через отдельную ветку и PR: сначала восстанавливается CI, затем меняются module path и документация, после чего настраиваются metadata, protection и библиотечный release workflow.

**Tech Stack:** Go 1.24+, GitHub Actions, golangci-lint v2.14.0, GitHub CLI/API, Markdown, YAML.

---

## Карта файлов

- `.github/workflows/go.yml` — обязательный CI для pull requests и `master`.
- `.github/workflows/release.yml` — проверка semver-тега и создание GitHub Release.
- `.github/dependabot.yml` — еженедельные обновления Go modules и Actions.
- `.github/ISSUE_TEMPLATE/bug_report.yml` — структурированный bug report без секретов.
- `.github/ISSUE_TEMPLATE/feature_request.yml` — запрос совместимой возможности.
- `.github/ISSUE_TEMPLATE/config.yml` — маршрутизация security reports.
- `.github/pull_request_template.md` — checklist совместимости, тестов и документации.
- `.golangci.yml` — минимальный воспроизводимый набор линтеров v2.
- `go.mod`, `go.sum` — новый module path и согласованные зависимости.
- `README.md` — позиционирование, установка, пример, поддержка и provenance.
- `MIGRATION.md` — миграция со старого import path.
- `CONTRIBUTING.md` — локальный workflow для contributors.
- `SECURITY.md` — поддерживаемая версия и приватное сообщение об уязвимости.
- `docs/testing.md` — локальные и релизные gates.
- `docs/superpowers/specs/2026-09-29-maintained-fork-design.md` — одобренный дизайн.

### Task 1: Влить planning-документы и создать изолированную рабочую ветку

**Files:**
- Existing: `docs/superpowers/specs/2026-09-29-maintained-fork-design.md`
- Existing: `docs/superpowers/plans/2026-09-29-maintained-fork.md`

- [ ] **Step 1: Опубликовать обновлённую документационную ветку**

```bash
git push origin docs/maintained-fork-design
```

Expected: remote branch указывает на commit с одобренной спецификацией и этим планом.

- [ ] **Step 2: Открыть документационный PR**

```bash
doc_pr_url=$(gh pr create \
  --repo serezha93/go-ts3 \
  --base master \
  --head docs/maintained-fork-design \
  --title "docs: зафиксировать план maintained fork" \
  --body "Добавляет одобренный дизайн и пошаговый план превращения go-ts3 в community-maintained fork.")
printf '%s\n' "$doc_pr_url"
```

Expected: создан PR, меняющий только два файла в `docs/superpowers`.

- [ ] **Step 3: Проверить diff документационного PR**

```bash
gh pr diff "$doc_pr_url" --repo serezha93/go-ts3 --name-only
```

Expected:

```text
docs/superpowers/plans/2026-09-29-maintained-fork.md
docs/superpowers/specs/2026-09-29-maintained-fork-design.md
```

- [ ] **Step 4: Влить документационный PR merge-коммитом**

```bash
gh pr merge "$doc_pr_url" --repo serezha93/go-ts3 --merge --delete-branch
```

Expected: PR имеет состояние `MERGED`; `master` содержит спецификацию и план.

- [ ] **Step 5: Создать worktree через обязательный skill**

Invoke `superpowers:using-git-worktrees`, затем из основного клона:

```bash
git fetch origin
git switch master
git pull --ff-only origin master
git worktree add ../go-ts3-maintained-fork -b feat/maintained-fork origin/master
```

Expected: чистый worktree на ветке `feat/maintained-fork`; ветки `update-crypto` и `pr-46` не изменены.

- [ ] **Step 6: Зафиксировать baseline и SHA защищаемых веток**

```bash
git status --short
git ls-remote origin refs/heads/update-crypto refs/heads/pr-46
go test ./...
go test -race ./...
go vet ./...
```

Expected: дерево чистое; три команды Go завершаются с exit code 0. Сохранить два remote SHA в рабочей заметке для финального сравнения.

### Task 2: Восстановить и модернизировать CI

**Files:**
- Modify: `.github/workflows/go.yml`
- Modify: `.golangci.yml`

- [ ] **Step 1: Проверить воспроизводимость текущей ошибки trigger**

```bash
rg -n 'branches:|main|master' .github/workflows/go.yml
gh run list --repo serezha93/go-ts3 --workflow go.yml --limit 5
```

Expected: workflow слушает `main`, а запусков для текущего `master` нет.

- [ ] **Step 2: Заменить `.github/workflows/go.yml`**

```yaml
name: CI

on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

permissions:
  contents: read

jobs:
  build-and-test:
    name: Build and test (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          cache: true
      - run: go build ./...
      - run: go test ./...

  race:
    name: Race
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          cache: true
      - run: go test -race ./...

  static-analysis:
    name: Static analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          cache: true
      - run: go vet ./...

  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          cache: true
      - uses: golangci/golangci-lint-action@v9
        with:
          version: v2.14.0
          args: --timeout=5m

  module-files:
    name: Module files
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          cache: true
      - run: go mod tidy
      - run: git diff --exit-code -- go.mod go.sum
```

- [ ] **Step 3: Перевести `.golangci.yml` на schema v2**

```yaml
version: "2"

run:
  timeout: 5m

linters:
  default: none
  enable:
    - bodyclose
    - errcheck
    - errorlint
    - govet
    - ineffassign
    - misspell
    - revive
    - staticcheck
    - unparam
    - unused
    - wrapcheck

formatters:
  enable:
    - gofmt
    - gofumpt
    - goimports
```

- [ ] **Step 4: Проверить CI-команды локально**

```bash
go build ./...
go test ./...
go test -race ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.14.0 run --timeout=5m
go mod tidy
git diff --exit-code -- go.mod go.sum
```

Expected: все команды завершаются с exit code 0. При ином результате остановиться, диагностировать конкретную ошибку и не расширять область изменений.

- [ ] **Step 5: Закоммитить CI**

```bash
git add .github/workflows/go.yml .golangci.yml
git commit -m "Настроить CI поддерживаемого форка"
```

### Task 3: Изменить module path и добавить миграцию

**Files:**
- Modify: `go.mod:1`
- Modify: `go.sum` only if `go mod tidy` changes it
- Create: `MIGRATION.md`

- [ ] **Step 1: Зафиксировать старую module identity**

```bash
go list -m
```

Expected: `github.com/multiplay/go-ts3`.

- [ ] **Step 2: Изменить первую строку `go.mod`**

```go
module github.com/serezha93/go-ts3
```

- [ ] **Step 3: Создать `MIGRATION.md`**

````markdown
# Migration to the maintained fork

The maintained fork uses a new Go module path:

```text
github.com/serezha93/go-ts3
```

Replace imports of `github.com/multiplay/go-ts3` with
`github.com/serezha93/go-ts3`, then run:

```sh
go get github.com/serezha93/go-ts3@v1.3.0
go mod tidy
```

The public Go API remains compatible with upstream `v1.2.0`. The import path
change is the required migration step. Report incompatibilities without
including credentials, tokens, or private server addresses.
````

- [ ] **Step 4: Проверить новый путь и отсутствие старых imports**

```bash
go mod tidy
test "$(go list -m)" = "github.com/serezha93/go-ts3"
if rg -n 'github.com/multiplay/go-ts3' --glob '*.go'; then exit 1; fi
go build ./...
go test ./...
go test -race ./...
go vet ./...
```

Expected: новый module path выводится точно; старых Go imports нет; gates проходят.

- [ ] **Step 5: Закоммитить migration**

```bash
git add go.mod go.sum MIGRATION.md
git commit -m "Перевести модуль на maintained fork"
```

### Task 4: Переписать README и документацию сопровождения

**Files:**
- Replace: `README.md`
- Create: `CONTRIBUTING.md`
- Create: `SECURITY.md`
- Create: `docs/testing.md`

- [ ] **Step 1: Заменить `README.md`**

````markdown
# go-ts3

[![CI](https://github.com/serezha93/go-ts3/actions/workflows/go.yml/badge.svg)](https://github.com/serezha93/go-ts3/actions/workflows/go.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/serezha93/go-ts3.svg)](https://pkg.go.dev/github.com/serezha93/go-ts3)
[![Go Report Card](https://goreportcard.com/badge/github.com/serezha93/go-ts3)](https://goreportcard.com/report/github.com/serezha93/go-ts3)
[![License](https://img.shields.io/badge/license-BSD--2--Clause-blue.svg)](LICENSE)

`go-ts3` is a community-maintained fork of
[`rocketsciencegg/go-ts3`](https://github.com/rocketsciencegg/go-ts3), formerly
[`multiplay/go-ts3`](https://github.com/multiplay/go-ts3). It provides a Go
client for the TeamSpeak 3 ServerQuery protocol.

This project is not an official TeamSpeak product and is not affiliated with
TeamSpeak Systems GmbH.

## Why this fork?

| Area | Upstream | This fork |
| --- | --- | --- |
| Maintenance | Infrequent | Active community maintenance |
| Go module path | `github.com/multiplay/go-ts3` | `github.com/serezha93/go-ts3` |
| CI | Legacy configuration | Build, tests, race, vet and lint |
| Releases | Upstream history through `v1.2.0` | Maintained releases from `v1.3.0` |

## Installation

```sh
go get github.com/serezha93/go-ts3@latest
```

Users of the original module should read [MIGRATION.md](MIGRATION.md).

## Example

```go
package main

import (
	"log"

	ts3 "github.com/serezha93/go-ts3"
)

func main() {
	client, err := ts3.NewClient("127.0.0.1:10011")
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()

	if err := client.Login("serveradmin", "password"); err != nil {
		log.Fatal(err)
	}

	version, err := client.Version()
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("server version: %v", version)
}
```

Never commit ServerQuery credentials. Prefer environment variables or another
secret store in real applications.

## Compatibility

The maintained fork preserves the public API of upstream `v1.2.0` where
possible. Semantic versioning applies to releases from `v1.3.0` onward.

## Documentation and support

- [API reference](https://pkg.go.dev/github.com/serezha93/go-ts3)
- [Testing guide](docs/testing.md)
- [Contributing](CONTRIBUTING.md)
- [Security policy](SECURITY.md)
- [Issues](https://github.com/serezha93/go-ts3/issues)

## Provenance and license

The repository preserves the upstream Git history and authorship. The project
is distributed under the original [BSD 2-Clause License](LICENSE).
````

- [ ] **Step 2: Создать `CONTRIBUTING.md`**

```markdown
# Contributing

Thank you for helping maintain go-ts3.

1. Open an issue for user-visible behavior changes.
2. Create a focused branch from `master`.
3. Preserve API compatibility unless a major-version change is approved.
4. Add or update tests for code changes.
5. Run every command in [docs/testing.md](docs/testing.md).
6. Open a pull request and complete its checklist.

Do not include credentials, tokens, or private server addresses in issues,
tests, logs, or pull requests.
```

- [ ] **Step 3: Создать `SECURITY.md`**

```markdown
# Security policy

## Supported versions

The latest maintained release receives security updates. Upstream releases
through `v1.2.0` are retained for provenance but are not maintained here.

## Reporting a vulnerability

Use GitHub private vulnerability reporting for this repository. Do not open a
public issue for an undisclosed vulnerability and do not include credentials,
tokens, or private server addresses.

Include the affected version, impact, minimal reproduction, and a suggested
remediation when available. Receipt should be acknowledged within seven days.
```

- [ ] **Step 4: Создать `docs/testing.md`**

````markdown
# Testing

## Required local gate

```sh
go mod tidy
git diff --exit-code -- go.mod go.sum
go build ./...
go test ./...
go test -race ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.14.0 run --timeout=5m
```

## CI gate

Pull requests must pass build and tests on Linux, macOS and Windows, plus race,
vet, lint and module-file checks on Linux.

## Release gate

A release requires a semver tag on a commit that passed the complete CI gate.
Never move or reuse a published tag.
````

- [ ] **Step 5: Проверить Markdown и ссылки статически**

```bash
git diff --check
rg -n 'github.com/multiplay/go-ts3' README.md MIGRATION.md CONTRIBUTING.md SECURITY.md docs/testing.md
rg -n 'github.com/serezha93/go-ts3' README.md MIGRATION.md go.mod
```

Expected: старый путь встречается только в provenance и migration; новый путь присутствует в installation, example и `go.mod`; trailing whitespace отсутствует.

- [ ] **Step 6: Закоммитить документацию**

```bash
git add README.md CONTRIBUTING.md SECURITY.md docs/testing.md
git commit -m "Оформить maintained fork и правила сопровождения"
```

### Task 5: Добавить GitHub community templates и Dependabot

**Files:**
- Create: `.github/dependabot.yml`
- Create: `.github/ISSUE_TEMPLATE/bug_report.yml`
- Create: `.github/ISSUE_TEMPLATE/feature_request.yml`
- Create: `.github/ISSUE_TEMPLATE/config.yml`
- Create: `.github/pull_request_template.md`

- [ ] **Step 1: Создать `.github/dependabot.yml`**

```yaml
version: 2
updates:
  - package-ecosystem: gomod
    directory: /
    schedule:
      interval: weekly
    groups:
      go-modules:
        patterns: ["*"]
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      github-actions:
        patterns: ["*"]
```

- [ ] **Step 2: Создать `.github/ISSUE_TEMPLATE/bug_report.yml`**

```yaml
name: Bug report
description: Report reproducible incorrect behavior
title: "bug: "
labels: [bug]
body:
  - type: markdown
    attributes:
      value: Do not include ServerQuery credentials, tokens, or private server addresses.
  - type: input
    id: version
    attributes:
      label: go-ts3 version
      placeholder: v1.3.0
    validations:
      required: true
  - type: input
    id: go-version
    attributes:
      label: Go version
      placeholder: go1.24.0
    validations:
      required: true
  - type: input
    id: teamspeak-version
    attributes:
      label: TeamSpeak Server version
    validations:
      required: true
  - type: dropdown
    id: transport
    attributes:
      label: ServerQuery transport
      options: [TCP, SSH]
    validations:
      required: true
  - type: textarea
    id: reproduction
    attributes:
      label: Minimal reproduction
      render: go
    validations:
      required: true
  - type: textarea
    id: expected
    attributes:
      label: Expected behavior
    validations:
      required: true
  - type: textarea
    id: actual
    attributes:
      label: Actual behavior
    validations:
      required: true
```

- [ ] **Step 3: Создать `.github/ISSUE_TEMPLATE/feature_request.yml`**

```yaml
name: Feature request
description: Propose a compatible ServerQuery capability
title: "feat: "
labels: [enhancement]
body:
  - type: textarea
    id: problem
    attributes:
      label: Problem
    validations:
      required: true
  - type: textarea
    id: proposal
    attributes:
      label: Proposed API and behavior
      render: go
    validations:
      required: true
  - type: textarea
    id: compatibility
    attributes:
      label: Compatibility impact
      description: Explain effects on the v1 public API.
    validations:
      required: true
```

- [ ] **Step 4: Создать config и PR template**

`.github/ISSUE_TEMPLATE/config.yml`:

```yaml
blank_issues_enabled: false
contact_links:
  - name: Security vulnerability
    url: https://github.com/serezha93/go-ts3/security/advisories/new
    about: Report undisclosed vulnerabilities privately
```

`.github/pull_request_template.md`:

```markdown
## Summary

Describe the focused change and its user-visible effect.

## Checklist

- [ ] Tests cover code changes.
- [ ] `docs/testing.md` gates pass locally.
- [ ] Public API compatibility was preserved or explicitly documented.
- [ ] Documentation and migration notes were updated when required.
- [ ] No credentials, tokens, or private server addresses are included.
```

- [ ] **Step 5: Проверить YAML и закоммитить**

```bash
ruby -e 'require "yaml"; Dir[".github/**/*.{yml,yaml}"].each { |f| YAML.load_file(f); puts f }'
git diff --check
git add .github/dependabot.yml .github/ISSUE_TEMPLATE .github/pull_request_template.md
git commit -m "Добавить шаблоны сообщества и Dependabot"
```

Expected: Ruby выводит каждый YAML-файл без исключений.

### Task 6: Заменить бинарный release workflow библиотечным

**Files:**
- Delete: `.github/workflows/release-build.yml`
- Delete: `.goreleaser.yaml`
- Create: `.github/workflows/release.yml`

- [ ] **Step 1: Создать `.github/workflows/release.yml`**

```yaml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read

jobs:
  verify:
    name: Verify release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
          cache: true
      - name: Validate semver tag
        shell: bash
        run: '[[ "$GITHUB_REF_NAME" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]'
      - run: go mod tidy
      - run: git diff --exit-code -- go.mod go.sum
      - run: go build ./...
      - run: go test ./...
      - run: go test -race ./...
      - run: go vet ./...
      - uses: golangci/golangci-lint-action@v9
        with:
          version: v2.14.0
          args: --timeout=5m

  release:
    name: Publish GitHub release
    needs: verify
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - name: Create release
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh release create "$GITHUB_REF_NAME" --verify-tag --generate-notes --title "$GITHUB_REF_NAME"
```

- [ ] **Step 2: Удалить старую release-конфигурацию**

Delete `.github/workflows/release-build.yml` and `.goreleaser.yaml` with `apply_patch`.

- [ ] **Step 3: Проверить release-конфигурацию**

```bash
ruby -e 'require "yaml"; YAML.load_file(".github/workflows/release.yml")'
rg -n 'goreleaser|Release Go Binary' .github . --glob '!docs/superpowers/**' || true
git diff --check
```

Expected: YAML читается; поиск GoReleaser не находит рабочих конфигураций.

- [ ] **Step 4: Закоммитить release workflow**

```bash
git add .github/workflows/release.yml .github/workflows/release-build.yml .goreleaser.yaml
git commit -m "Заменить бинарный release workflow библиотечным"
```

### Task 7: Выполнить полный локальный gate и smoke-test module path

**Files:**
- Verify only; исправления ограничиваются файлами предыдущих задач

- [ ] **Step 1: Запустить полный локальный gate**

```bash
go mod tidy
git diff --exit-code -- go.mod go.sum
go build ./...
go test ./...
go test -race ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.14.0 run --timeout=5m
git diff --check
```

Expected: все команды завершаются с exit code 0.

- [ ] **Step 2: Проверить область старого module path**

```bash
rg -n 'github.com/multiplay/go-ts3'
```

Expected: совпадения есть только в README, MIGRATION и planning-документах; Go files и `go.mod` не совпадают.

- [ ] **Step 3: Опубликовать feature branch**

```bash
git push -u origin feat/maintained-fork
```

Expected: remote branch создан без изменения `master`, `update-crypto` или `pr-46`.

- [ ] **Step 4: Проверить установку форка в чистом модуле по commit SHA**

```bash
feature_sha=$(git rev-parse HEAD)
smoke_dir=$(mktemp -d)
cd "$smoke_dir"
go mod init example.com/go-ts3-smoke
go get "github.com/serezha93/go-ts3@$feature_sha"
go list github.com/serezha93/go-ts3
go list -m all | grep 'github.com/serezha93/go-ts3'
```

Expected: package и module разрешаются по новому пути без `replace`.

- [ ] **Step 5: Сравнить защищаемые ветки с baseline**

```bash
git ls-remote origin refs/heads/update-crypto refs/heads/pr-46
```

Expected: SHA обеих веток совпадают с Task 1 Step 6.

### Task 8: Открыть implementation PR и настроить repository metadata

**Files:**
- GitHub repository settings
- Create outside repository: `/tmp/go-ts3-branch-protection.json`

- [ ] **Step 1: Открыть implementation PR**

```bash
implementation_pr_url=$(gh pr create \
  --repo serezha93/go-ts3 \
  --base master \
  --head feat/maintained-fork \
  --title "Подготовить полноценный maintained fork" \
  --body-file .github/pull_request_template.md)
printf '%s\n' "$implementation_pr_url"
```

Expected: PR содержит только planned files и не меняет публичный Go API.

- [ ] **Step 2: Дождаться всех checks**

```bash
gh pr checks "$implementation_pr_url" --repo serezha93/go-ts3 --watch
```

Expected checks: Build and test на трёх ОС, Race, Static analysis, Lint, Module files — все `pass`.

- [ ] **Step 3: Применить description и topics**

```bash
gh repo edit serezha93/go-ts3 \
  --description "Community-maintained fork of rocketsciencegg/go-ts3 — a Go client for the TeamSpeak 3 ServerQuery protocol." \
  --add-topic go \
  --add-topic golang \
  --add-topic teamspeak \
  --add-topic teamspeak3 \
  --add-topic serverquery \
  --add-topic client-library \
  --add-topic maintained-fork
```

- [ ] **Step 4: Включить private vulnerability reporting**

```bash
gh api --method PUT repos/serezha93/go-ts3/private-vulnerability-reporting
```

Expected: HTTP 204.

- [ ] **Step 5: Создать `/tmp/go-ts3-branch-protection.json` через `apply_patch`**

```json
{
  "required_status_checks": {
    "strict": true,
    "contexts": [
      "Build and test (ubuntu-latest)",
      "Build and test (macos-latest)",
      "Build and test (windows-latest)",
      "Race",
      "Static analysis",
      "Lint",
      "Module files"
    ]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": null,
  "restrictions": null,
  "required_linear_history": false,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "required_conversation_resolution": true,
  "lock_branch": false,
  "allow_fork_syncing": true
}
```

- [ ] **Step 6: Применить защиту `master` после появления check contexts**

```bash
gh api \
  --method PUT \
  -H "Accept: application/vnd.github+json" \
  repos/serezha93/go-ts3/branches/master/protection \
  --input /tmp/go-ts3-branch-protection.json
```

Expected: `allow_force_pushes.enabled=false`, `allow_deletions.enabled=false`, перечислены семь required contexts.

- [ ] **Step 7: Проверить metadata и protection чтением API**

```bash
gh api repos/serezha93/go-ts3 --jq '{description,topics,visibility,fork,parent:.parent.full_name}'
gh api repos/serezha93/go-ts3/branches/master/protection --jq '{required:.required_status_checks.contexts,force:.allow_force_pushes.enabled,deletions:.allow_deletions.enabled}'
```

Expected: repository остаётся public fork `rocketsciencegg/go-ts3`; metadata и protection соответствуют спецификации.

### Task 9: Влить PR, выпустить `v1.3.0` и проверить обнаруживаемость

**Files:**
- GitHub PR, tag, release and profile settings

- [ ] **Step 1: Влить implementation PR без удаления feature branch до проверки**

```bash
gh pr merge "$implementation_pr_url" --repo serezha93/go-ts3 --merge
git fetch origin
git switch master
git pull --ff-only origin master
```

Expected: PR имеет состояние `MERGED`; локальный `master` равен `origin/master`.

- [ ] **Step 2: Повторить полный gate на итоговом merge commit**

```bash
go mod tidy
git diff --exit-code -- go.mod go.sum
go build ./...
go test ./...
go test -race ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.14.0 run --timeout=5m
```

Expected: полный gate проходит; working tree чистый.

- [ ] **Step 3: Создать и отправить annotated tag**

```bash
git tag -a v1.3.0 -m "Первый релиз maintained fork"
git push origin v1.3.0
```

Expected: tag указывает на проверенный merge commit и запускает workflow `Release`.

- [ ] **Step 4: Дождаться release workflow и проверить Release**

```bash
release_run_id=$(gh run list --repo serezha93/go-ts3 --workflow release.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$release_run_id" --repo serezha93/go-ts3 --exit-status
gh release view v1.3.0 --repo serezha93/go-ts3 --json tagName,isDraft,isPrerelease,url,targetCommitish
```

Expected: workflow успешен; Release не draft и не prerelease.

- [ ] **Step 5: Проверить публичную установку `v1.3.0`**

```bash
release_smoke_dir=$(mktemp -d)
cd "$release_smoke_dir"
go mod init example.com/go-ts3-release-smoke
GOPROXY=https://proxy.golang.org,direct go get github.com/serezha93/go-ts3@v1.3.0
go list github.com/serezha93/go-ts3
go list -m all | grep 'github.com/serezha93/go-ts3 v1.3.0'
```

Expected: proxy или direct разрешает опубликованный модуль и package компилируется.

- [ ] **Step 6: Проверить pkg.go.dev и GitHub UI**

Open and verify:

```text
https://pkg.go.dev/github.com/serezha93/go-ts3@v1.3.0
https://github.com/serezha93/go-ts3
```

Expected: pkg.go.dev показывает `v1.3.0`; GitHub показывает maintained description, topics, README и fork provenance.

- [ ] **Step 7: Закрепить репозиторий в профиле**

Через GitHub profile UI открыть “Customize your pins”, выбрать `serezha93/go-ts3`, сохранить и повторно прочитать профиль. Не снимать существующие pins без необходимости; если достигнут лимит, запросить выбор пользователя.

- [ ] **Step 8: Подготовить обращение к upstream и запросить финальное подтверждение текста**

Draft:

```markdown
Hi! I maintain a compatibility-focused community fork at
https://github.com/serezha93/go-ts3. It preserves this repository's history and
BSD-2-Clause license, has CI across Linux/macOS/Windows, and publishes the Go
module as `github.com/serezha93/go-ts3` from v1.3.0 onward.

If this repository is currently in low-maintenance mode, would you consider
adding a neutral “Community-maintained forks” link to the README? I am not
claiming official successor status and will continue to credit this upstream.
```

Expected: сообщение не публикуется до отдельного подтверждения пользователя.

- [ ] **Step 9: Финальная проверка инвариантов**

```bash
gh api repos/serezha93/go-ts3 --jq '{visibility,fork,parent:.parent.full_name,description,topics}'
gh api repos/serezha93/go-ts3/branches/master/protection --jq '{force:.allow_force_pushes.enabled,deletions:.allow_deletions.enabled}'
gh pr view 46 --repo rocketsciencegg/go-ts3 --json state,headRefName,headRepository,url
git ls-remote origin refs/heads/update-crypto refs/heads/pr-46
```

Expected: repository public и остаётся fork; protection активен; upstream PR #46 открыт на `update-crypto`; два SHA совпадают с baseline.
