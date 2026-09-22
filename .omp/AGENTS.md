# Exiled Exchange 2 — личный форк (mttzzz)

## Git: форк и синк

- `origin` = **наш форк** `git@github.com:mttzzz/Exiled-Exchange-2.git` — сюда push.
- `upstream` = апстрим `https://github.com/Kvan7/Exiled-Exchange-2.git` — только fetch, не push.
- Основная ветка — `master` (default апстрима; не переименовывать в `main`, только усложняет sync/PR).
  `master` = upstream master + **наши патчи** (сейчас: RU Exceptional fix, b291f13, cherry-pick по-над upstream).
- `fix_ru_exceptional_prefix` — ветка открытого PR #1062 в Kvan7 — НЕ удалять (upstream может вмержить).
- **Синк с апстримом:**
  ```bash
  git fetch upstream
  git checkout master && git rebase upstream/master   # наши коммиты едут сверху
  git push --force-with-lease origin master
  ```
  Если PR #1062 вмержат — наш коммит выпадет сам по patch-id; конфликт в тех же строках = upstream поправил иначе: гнать их версию, нашу правку дорабатывать поверх.
- Новые фичи: ветка от `master` → push в `origin`. Апстриму шлём только PRs; merges не ожидаем — форк живёт самостоятельной жизнью.

## Сборка (всё через `lane exec`, на хосте проект не исполнять)

| Шаг | Команда |
|---|---|
| deps renderer | `lane exec -- bash -lc 'cd renderer && npm ci'` |
| build renderer | `lane exec -- bash -lc 'cd renderer && npm run make-index-files && npm run build'` |
| deps+build main | `lane exec -- bash -lc 'cd main && npm ci && npm run build'` |
| тесты renderer | `lane exec -- bash -lc 'cd renderer && npx vitest run'` |
| Linux AppImage | `lane exec -- bash -lc 'cd main && npx electron-builder build --publish never'` → `main/dist/Exiled Exchange 2-<ver>.AppImage` |
| Windows portable | `lane exec -- bash -lc 'cd main && npx electron-builder build --win --dir --publish never'` |

### Внимания заслуживает
- **CI форка ВКЛЮЧЕНА (01.09.2026).** `push master` → run `Build` (matrix win/ubuntu/macos) + `test.yml` (lint+vitest). Артефакты run'ов живут 1 день — не копировать как источник.
- **Релиз форка = форс-пуш тега `vX.Y.Z`** (тег на коммите бампа, после push master).
- **Бамп версии — 5 файлов** (hook `versionCheck.py`): `main/package.json`, `main/package-lock.json` (2 места), `README.md` (Setup-badge), `bug-report.yml` (верх списка), docs-конфиг (`appVersion`).
- Локальная проверка перед коммитом: `python3 versionCheck.py`.
-electron-builder-публикация явная: `publish` в `main/electron-builder.yml` (блок описан ниже). Токен загрузки — `GH_TOKEN: RELEASE_TOKEN || GITHUB_TOKEN` в `main.yml`, флаг `--publish onTagOrDraft` (публикуется только с тега). Инсталляторы: `https://github.com/mttzzz/Exiled-Exchange-2/releases`.
- **Auto-updater шьётся на форк:** `main/electron-builder.yml` → `publish: {provider: github, owner: mttzzz, repo: Exiled-Exchange-2}` (поля — **owner/repo**, не `name` — иначе electron-builder роняет весь build навалидацией). `AppUpdater.ts` логирует и показывает в UI реальный текст ошибки (`UpdateInfo.message` в `ipc/types.ts`); ссылки «О программе» идут на форк.
- **Windows-build падает на последнем шаге «updating asar integrity executable resource»** — нужен wine (rcedit), в runner-поде uid 1000 без root/sudo: не поставить. Каталог `main/dist/win-unpacked/` несмотря на ошибку полный и рабочий (asar + prebuilds win32-x64 на месте) — asar-integrity это optional hardening-ресурс Electron, на запуск не влияет. Упаковывать: `tar -czf ee2-win.tar.gz win-unpacked`. Передача на Windows-бокс — через `/mnt/poe2` (RW, owner mttzzz).
- **Данных у проекта нет** (не web-сервер): lane = только runner; `lane db pull`/`reset` здесь не используются.
- `main/package.json: version` — версия релиза; `testUpdate.sh` — скрипт апстрима, дублирует renderer+main build.
- Временные файлы — только `/.tmp/` (в .gitignore).
