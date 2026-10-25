# Music Toolbox

Десктопний додаток (Python 3.12 + PySide6) для завантаження треків через yt-dlp,
роботи з тегами (mutagen) та обробки аудіо через ffmpeg. Заміна ручних Python-скриптів
графічним інтерфейсом. Жодних автоматичних ланцюжків: кожна дія — окрема кнопка над
файлами, вибраними в панелі «Файли». Тільки для особистого використання.

## Стан за етапами ТЗ

| # | Етап | Статус |
|---|---|---|
| 1 | Скелет, вікно, `binaries.py`, `workers.py` | ✅ |
| 2 | `downloader.py`: черга, JSON-прогрес, скасування без `.part`, архів дублікатів | ✅ |
| 3 | `library.py` + панель «Файли» (колонки, фільтри, перейменування, видалення) | ✅ |
| 4 | Вкладка «Завантажити»: URL + діапазон + формат, профілі, черга з прогресом | ✅ |
| 5 | `tags.py` + вкладка «Теги» (6 дій; тести на mp3/m4a/opus/flac/wav) | ✅ |
| 6 | `process.py` + вкладка «Обробка» (10 операцій, пресети, пакетний запуск, прослуховування) | ✅ |
| 7 | `search.py` + вкладка «Пошук» (SoundCloud + YouTube паралельно, підказки iTunes) | ✅ |
| 8 | Реліз: іконка, Inno Setup, AppImage + .deb, CI `build.yml`, сповіщення, taskbar-прогрес | ✅ (зібрати й пройти на чистих ОС — див. нижче) |
| 9 | Waveform-редактор (опційний) | — не робився |

## Швидкий старт на Windows (з нуля, без Python)

Потрібен лише PowerShell і доступ до інтернету. Папку проєкту краще тримати поза OneDrive
(наприклад `C:\dev\music-toolbox`), бо збірка створює ~200 МБ у `dist\`.

1. Встановіть uv (менеджер Python-проєктів) — одна команда в PowerShell:
   ```powershell
   powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
   ```
   Закрийте PowerShell і відкрийте знову, щоб команда `uv` стала доступною.
2. Перейдіть у папку проєкту й підготуйте середовище (uv сам завантажить Python 3.12
   і всі залежності, ~250 МБ):
   ```powershell
   cd C:\dev\music-toolbox
   uv python install 3.12
   uv sync --group dev
   ```
3. Запуск із вихідного коду:
   ```powershell
   uv run music-toolbox
   ```
4. Збірка exe:
   ```powershell
   uv run pyinstaller build/pyinstaller.spec
   ```
   Результат — папка `dist\MusicToolbox\` з `MusicToolbox.exe`. Переносити треба
   **всю папку** (поряд лежить `_internal` з Qt і Python); на іншому комп'ютері Python
   не потрібен.
5. (Опційно) інсталятор `setup.exe` — встановіть [Inno Setup 6](https://jrsoftware.org/isdl.php)
   і виконайте:
   ```powershell
   iscc /DAppVersion=0.1.0 build\installer.iss
   ```
   Результат — `dist\MusicToolbox-Setup-0.1.0.exe` (Program Files, ярлик, деінсталятор).

Якщо замість uv у вас уже є Python 3.12 (python.org, з галочкою «Add python.exe to PATH»):

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1      # якщо блокує політика: Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
pip install -e . pyinstaller
python -m app.main                 # запуск
pyinstaller build/pyinstaller.spec # exe
```

Перший запуск: додаток відкриє вкладку «Налаштування» і покаже, яких програм бракує.
Натисніть «Встановити / оновити yt-dlp» і «Встановити / оновити ffmpeg + ffprobe» — вони
завантажаться в `%LOCALAPPDATA%\MusicToolbox\bin` (не вшиті в exe навмисно, бо yt-dlp
оновлюється кожні кілька тижнів). SmartScreen може попередити про непідписаний exe — це
нормально для PyInstaller-збірок.

## Linux

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh     # uv, якщо нема
uv python install 3.12 && uv sync --group dev
uv run music-toolbox                                # запуск
uv run pyinstaller build/pyinstaller.spec           # dist/MusicToolbox/
bash build/build_appimage.sh 0.1.0                  # AppImage + .deb (потрібен dpkg-deb)
```

Бінарники — у `~/.local/share/MusicToolbox/bin`, конфіги — у `~/.config/MusicToolbox`.

## Де що лежить

- Порядок пошуку yt-dlp/ffmpeg/ffprobe: папка даних → `bin/` поряд із exe → `PATH`.
- Конфіги (`settings.json`, `profiles.json`, `presets.json`): Windows —
  `%LOCALAPPDATA%\MusicToolbox`, Linux — `~/.config/MusicToolbox`.
- База SQLite (`library.sqlite3`: архів завантажених треків + кеш ffprobe) і лог
  (`logs\music-toolbox.log`) — у папці даних. Кнопка «Відкрити файл логу» є в «Налаштуваннях».
- Змінні `MUSIC_TOOLBOX_DATA_DIR` / `MUSIC_TOOLBOX_CONFIG_DIR` перевизначають ці шляхи
  (так працюють тести).

## Як користуватися

1. **Налаштування** — робоча папка (за замовчуванням `~/Music/soundcloud`), папка бекапів,
   стратегія результату обробки (новий файл із суфіксом / замінити з бекапом), паралельність
   завантажень (1–4), шаблон імені файлу `{prefix}{title}.{ext}`, шаблон тегів, формат.
2. **Завантажити** — URL треку/плейлиста/профілю + префікс + діапазон (від–до) + формат →
   «Додати в чергу». Перед стартом виконується `yt-dlp -J --flat-playlist`, тому таблиця одразу
   заповнюється назвами. Уже завантажені треки (за `(джерело, id)` в SQLite) позначаються
   «пропущено». **Профілі** — таблиця збережених плейлистів; «Запустити вибрані профілі»
   починає з треку `останній № + 1` і оновлює номер після кожного завантаженого треку.
   Черга: прогрес-бар, швидкість, ETA, розмір; Пауза (нові треки не стартують, поточні
   доходять до кінця) / Продовжити / Скасувати все (kill yt-dlp + видалення `.part`);
   «Повторити помилкові», «Сховати завершені», подвійний клік по помилці — повний текст.
   Після завершення плейлиста — системне сповіщення; на Windows — прогрес на іконці в панелі задач.
3. **Пошук** — «назва — виконавець», галочки SoundCloud/YouTube (`scsearch10:`/`ytsearch10:`
   паралельно), підказки iTunes Search API (подвійний клік підставляє «виконавець - назва» у
   запит), подвійний клік по результату відкриває браузер, «Завантажити вибране» кладе в чергу.
4. **Теги** — над вибраними файлами: очистити всі метадані (ffmpeg `-map_metadata -1 -c copy`,
   fallback mutagen), записати `album` = префікс (до першого `_` включно), записати
   title/artist за шаблоном, видалити/додати обкладинку, переглянути теги.
5. **Обробка** — темп (atempo), тон (rubberband або asetrate+atempo), швидкість+тон
   (Nightcore 1.25 / Slowed 0.85), slowed + reverb, обрізати (`-c copy` або перекодування,
   «▶ Прослухати фрагмент» через QtMultimedia), fade in/out, нормалізація (двопрохідний
   loudnorm, −14 LUFS), конвертація (mp3 128/192/320, m4a, opus, flac, wav), об'єднання
   (порядок = порядок у панелі «Файли»), вирізати тишу. Пресети зберігаються в
   `presets.json`; пакетний запуск із прогресом по кожному файлу.

## Тести й лінт

```bash
uv run pytest        # ~140 тестів, ~40 с
uv run ruff check
```

- `core/` не імпортує Qt (є тест на це). Тести `downloader`/`search` використовують
  `tests/fake_ytdlp.py` замість справжнього yt-dlp; завантаження бінарників мокається.
- Тести `tags`/`process`/`library` і GUI-сценарії (offscreen) потребують `ffmpeg`/`ffprobe`
  в `PATH`; без них вони пропускаються. Аудіофікстури генеруються ffmpeg (`sine`).
- GUI-тести (`tests/test_gui_*.py`, `test_main_window.py`) запускають справжнє вікно з
  `QT_QPA_PLATFORM=offscreen` і клікають кнопки.

## Структура

```
app/
  main.py                точка входу
  core/                  без Qt
    paths.py             папки даних/конфігів/бінарників
    binaries.py          find / version / install / popen / stream_lines / kill_tree
    downloader.py        list_playlist (-J --flat-playlist), download_track (JSON-прогрес)
    archive.py           SQLite-архів завантажених (джерело, id)
    library.py           ffprobe + теги → FileInfo, кеш у SQLite
    tags.py              strip / album / title-artist / cover ± / read_tags
    process.py           10 ffmpeg-операцій, OutputPolicy (суфікс або заміна з бекапом)
    search.py            scsearch / ytsearch паралельно, iTunes Search API
    profiles.py, presets.py, jsonstore.py, settings.py, models.py, logsetup.py
  ui/
    main_window.py       QSplitter: панель «Файли» | вкладки; трей-сповіщення; закриття
    workers.py           Worker(QRunnable) + WorkerContext(progress/log/is_cancelled)
    queue.py             QueueModel + ProgressDelegate + QueueController (пул, пауза, повтор)
    batch.py             BatchPanel: запуск core-функції над файлами з прогресом/результатами
    file_browser.py      панель «Файли»: модель, фільтри, перейменувати/видалити
    taskbar.py           Windows ITaskbarList3 через ctypes (best effort)
    state.py, logbus.py
    tabs/                download, search, tags, process, settings, base
build/
  pyinstaller.spec       one-folder, без консолі, іконка
  installer.iss          Inno Setup → MusicToolbox-Setup-{version}.exe
  build_appimage.sh      AppImage + .deb (dpkg-deb)
  make_icon.py, icon.png, icon.ico
.github/workflows/
  ci.yml                 ruff + pytest на windows-latest / ubuntu-latest
  build.yml              на тег v*: PyInstaller → Inno Setup / AppImage+deb → GitHub Release
tests/
legacy/                  місце для старих скриптів (не імпортуються)
```

## Реліз через GitHub Actions

Push тегу `v0.1.0` запускає `build.yml`: тести → PyInstaller → Inno Setup (Windows) і
AppImage + .deb (Linux) → файли прикріплюються до GitHub Release. Деінсталятор не видаляє
папку даних користувача. Оновлення yt-dlp/ffmpeg — з додатка; оновлення самого додатка —
новим інсталятором.

## Контракти

- `core/tags.py`, `core/process.py`: `fn(paths: list[Path], **params, progress=None) -> list[OpResult]`;
  у `process` додатково `is_cancelled=None, policy: OutputPolicy | None = None`.
- Довгі операції в UI — тільки через `ui/workers.py`; задача отримує `WorkerContext`
  з `progress(**payload)`, `log(msg)`, `is_cancelled()`.
- Усі зовнішні процеси — через `binaries.popen()` / `binaries.run()`: повна команда в лозі,
  utf-8, `CREATE_NO_WINDOW` на Windows, своя група процесів на Linux для `kill_tree`.
- yt-dlp: `--newline --progress-template "download:%(progress)j"` (+ `postprocess:`),
  `--print after_move:...` для шляху результату, `--windows-filenames`, `--ffmpeg-location`.
- ffmpeg для обробки: `-progress pipe:1 -nostats`, відсоток = `out_time_us / тривалість`.
- Оновлення рядків черги — не частіше 10 разів/с (таймер 100 мс).

## Відомі обмеження

- Пауза не зупиняє вже запущений yt-dlp (у нього немає паузи) — поточні треки доходять до
  кінця, нові не стартують. Скасування вбиває процес і прибирає `.part`.
- Прогрес на іконці панелі задач — лише Windows, best effort (COM через ctypes); якщо
  інтерфейс недоступний, функція мовчки вимикається.
- Збірки на чистих Windows/Linux (критерій етапу 8) виконуються через `build.yml`; у
  середовищі розробки перевірено PyInstaller-збірку та запуск на Linux.
