# GTA VI (GTA 6) — форензика exe-файла Xbox Series X|S и попытка адаптации под Windows x64

> **Документация исследования. Файлов игры здесь нет и не будет.**
> Репозиторий содержит только текстовое описание проведённого анализа: методологию,
> факты и итоги. Никаких исполняемых файлов, дизассемблера, заглушек, патчей и
> прочих производных от кода игры материалов не публикуется — см. [NOTICE.md](NOTICE.md).

---

## Объект исследования (идентификация)

| Поле | Значение |
|---|---|
| Игра | **Grand Theft Auto VI (GTA VI / GTA 6)**, внутренний проект «Americas» (Project Americas) |
| Платформа | Xbox Series X\|S (кодовое имя Scarlett) |
| Файл | `game_scarlett_master.exe` |
| Версия сборки | 1.0.2.1, ноябрь 2026 (сборка от 2026-11-13 21:40:36 UTC, за 6 дней до релиза 19.11.2026) |
| Источник | утечка **GTA6.Xbox.v1.0.2.1-CYBERLEEK** (группа CyberLeek; exe опубликован отдельным релизом после дампа 209 ГБ без исполняемого файла) |
| Размер | 120 181 760 байт (114,6 МБ) |
| Формат | PE32+ (AMD64), подсистема WINDOWS_GUI, неупакованный |
| SHA-256 | `e95020b2fa4a721db3df59fdecf75621730ac559d8835252f6c3b658d5a08993` |
| SHA-1 | `2be54b5e91275a2a8a0ba65b74eab51527933522` |
| MD5 | `8609e31e0b9a2192b3f80771d16ca32c` |

## Итог в одну строку

Файл — подлинная консольная сборка RAGE, корректная и целостная, **но она никогда
не предназначалась для ПК**: максимум, чего удаётся достичь на Windows без
эмулятора уровня консоли, — полностью связанный загрузчик и вход в CRT-инициализацию.

## Что было сделано (этапы)

| Этап | Что сделано | Итог |
|---|---|---|
| **Форензика подлинности** | Статический PE-анализ (pefile), строки, импорты/экспорты, RTTI, .pdata, точка входа (Capstone), OSINT-сверка | Все признаки согласуются с подлинностью консольной сборки Rockstar; вредоносного кода и упаковщиков не найдено — [docs/01](docs/01-object-and-authenticity.md) |
| **Phase 1 — PE-заголовки** | Минимальная правка двух полей заголовка, чтобы образ принимался загрузчиком Windows; код и данные не тронуты, размер не изменился | Образ проходит валидацию загрузчика Windows — [docs/02](docs/02-phase1-pe-header.md) |
| **Phase 2 — заглушки импортов** | 12 DLL-заглушек для консольных зависимостей, 110 экспортов сняты поимённо из import-таблицы | 105 функций — управляемый отказ (return 0), 5 реально работают на Windows — [docs/03](docs/03-phase2-import-stubs.md) |
| **Симуляция загрузчика** | Модель разрешения импортов Windows 10/11 x64 | **68/68 модулей, 587/587 функций связываются**, `STATUS_DLL_NOT_FOUND` устранён — [docs/04](docs/04-loader-simulation.md) |
| **Эмуляция загрузки** | Unicorn 2.1.4: минимальные PEB/TEB, bump-heap, трамплины на все импорты | TLS-колбэки исполняются, затем остановка на чтении нулевого указателя в CRT-инициализации — [docs/05](docs/05-boot-emulation.md) |
| **Анализ предела** | Разбор точки остановки и отсутствующих подсистем | Полный запуск = эмулятор Xbox Scarlett OS + игровые пакеты; на уровне правки файла это недостижимо — [docs/06](docs/06-why-full-boot-impossible.md) |

## Ключевые находки

- PDB-путь сборки указывает на проект `americas` / платформу `Gaming.Xbox.Scarlett.x64` / конфигурацию `master` — соглашение об именах Rockstar (`game_<платформа>_<конфигурация>.exe`).
- Компиляция на внутреннем форке LLVM Rockstar (RSG-LLVM 19.1.7), линкер lld-link (Rich-заголовок отсутствует).
- ~95 МБ кода в `.text`, 299 257 функций по unwind-записям x64, без упаковки, без Denuvo.
- URL-протокол запуска сюжетного контента вида `americas://…`; полный набор консольных зависимостей Xbox Series: `d3d12_xs.dll`, `xg_xs.dll`, `chad.dll`, `AcpHal.dll`, `xapu.dll`, `GameInput.dll`, `XboxBvhBuilder.dll`, `Bink2Scarlett.dll`, `libHttpClient.GDK.dll`, `XCRDAPI.dll`, `amd_fidelityfx_loader_xbox.dll`, `xmem.dll`.
- Подсистема образа — уже WINDOWS_GUI (2): консольные сборки Series X\|S генетически близки к Windows x64, поэтому «перепрошивка» заголовков вообще возможна.
- Из 587 импортов только 12 модулей консольные; 56 — стандартные Windows/api-set/VC++. Именно это позволило довести связывание до 100%.
- Статические инициализаторы CRT — обязательная фаза: без их выполнения движок не стартует (подтверждено эмуляцией).

## FAQ

**Можно ли запустить GTA 6 (Xbox-сборку) на ПК?**
Нет. Не на уровне правки exe. Нужен эмулятор уровня Xbox Scarlett OS (многолетний проект масштаба Xenia) и сами игровые пакеты, которых в утечке не было.

**Почему загрузчик проходит, а игра не стартует?**
Загрузчик Windows связывает все импорты (587/587), но дальше RAGE ожидает аппаратные сервисы Scarlett (D3D12X-рендер, аудио-стек CHAD/APU/ACP, аппаратный BVH, XG) и пакеты данных (`common:/`, `levels/`, `americas://`). Заглушки возвращают управляемый отказ, и движок корректно завершается на инициализации рендера.

**Файл «битый»?**
Нет. CRC32 совпадает с метаданными 7z-контейнера, SHA-256 совпадает с образцом в публичных песочницах. Многотомный 7z просто требует все три тома.

**Опасен ли файл?**
Вредоносного кода не обнаружено; публичная песочница дала низкие статическую и поведенческую оценки без отнесения к семействам малвари. Однако вокруг утечки действуют мемкоин-схемы и распространяются вредоносные подделки под этот файл — проверяйте хэши.

**Где сами файлы?**
Их здесь нет и не будет: исполняемый файл и производные от него материалы — объект авторского права Rockstar Games / Take-Two. См. [NOTICE.md](NOTICE.md).

## English summary

Static forensic analysis and Windows x64 adaptation attempt of the leaked
`game_scarlett_master.exe` — an Xbox Series X|S (Scarlett) build of **Grand Theft
Auto VI** (v1.0.2.1, CYBERLEEK release, built 2026-11-13). Work performed: PE
authenticity forensics (pefile, Capstone, OSINT cross-check), minimal PE-header
adaptation (code/data untouched), 12 import-stub DLLs covering 110 console
exports (xmem shim → VirtualAlloc, d3d12_xs forwarder → system d3d12), full
loader simulation (68/68 modules, 587/587 imports bound — no more
STATUS_DLL_NOT_FOUND) and Unicorn-based boot emulation (TLS callbacks executed,
halt at CRT static-initializer null-pointer read). **Conclusion:** the binary is
authentic and intact, but a full boot requires an Xbox Scarlett OS emulator
(Xenia-scale project) plus the game data packages — unachievable by file patching.
This repository documents methodology and findings only. **No game files,
disassembly, stubs or patches are hosted here**; all copyrights belong to
Rockstar Games / Take-Two Interactive.

## Структура документации

1. [Объект и форензика подлинности](docs/01-object-and-authenticity.md)
2. [Phase 1 — правка PE-заголовков](docs/02-phase1-pe-header.md)
3. [Phase 2 — DLL-заглушки импортов](docs/03-phase2-import-stubs.md)
4. [Симуляция загрузчика Windows](docs/04-loader-simulation.md)
5. [Эмуляционная загрузка (Unicorn)](docs/05-boot-emulation.md)
6. [Почему полный запуск невозможен без эмулятора](docs/06-why-full-boot-impossible.md)
7. [Методология и инструментарий](docs/07-methodology.md)
8. [NOTICE — правовое положение](NOTICE.md)

---

*Исследование выполнено в образовательных и исследовательских целях, без
распространения защищённых материалов. Все права на Grand Theft Auto VI
принадлежат Rockstar Games и Take-Two Interactive.*
