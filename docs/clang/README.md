# ClangVideoAnalyticsServer

REST-совместимый с `PythonVideoAnalyticsServer` сервер видеоаналитики на Clang/C++20 без Qt. Он управляется существующим `JsVideoAnalyticsViewer` и предоставляет те же маршруты `/api/v1/*`, JPEG preview и HLS `live.m3u8` на порту 8888.

Текущая версия — **0.1.19**. История изменений ведётся в
[ReleaseNotes.txt](ReleaseNotes.txt); единственный источник номера версии —
`project(... VERSION ...)` в `CMakeLists.txt`.

## Возможности

- Bearer-авторизация с PBKDF2-SHA256 и ролями `admin`/`viewer`;
- RTSP через OpenCV/FFmpeg, автоматическое переподключение и метрики потока;
- motion detection, трекинг с постоянными ID и траекториями;
- загрузка ONNX и YOLO inference через ONNX Runtime CPU;
- история tracking/YOLO в PostgreSQL;
- HLS через отдельный FFmpeg-процесс;
- запись необработанных кадров единственного RTSP-сеанса в H.264/MP4;
- ONVIF PTZ для основной камеры;
- Docker-окружение и одинаковый benchmark для Python и Clang серверов.

## Структура проекта

- `include/clang_video_analytics/app.hpp` — классы подсистем и их потоковые
  контракты;
- `src/main.cpp`, `src/server.cpp` — композиция приложения и REST-маршруты
  Drogon;
- `src/runtime_state.cpp`, `src/authentication.cpp` — общее состояние,
  конфигурация, пользователи и Bearer-сессии;
- `src/video_pipeline.cpp` — последовательность RTSP → Motion → Tracking → YOLO
  → JPEG/HLS;
- `src/motion_detector.cpp`, `src/motion_tracker.cpp`,
  `src/object_detector.cpp` — алгоритмы аналитики OpenCV и ONNX Runtime;
- `src/hls_encoder.cpp` — асинхронная передача последнего кадра процессу FFmpeg;
- `src/history_repository.cpp`, `src/ptz_controller.cpp` — PostgreSQL и ONVIF;
- `db`, `benchmark`, `tests`, `scripts` — схема базы, сравнение, контрактные
  тесты и сценарии Windows;
- `compose.yaml` — основной Docker-стек, `compose.compare.yaml` — изолированное
  сравнение C++ и Python.

## C++20 и управление ресурсами

Проект собирается со строгим `CMAKE_CXX_STANDARD 20` без расширений
компилятора. Возможности именно C++20 — `std::string::starts_with` для
Bearer-заголовка и `std::erase_if` для жизненного цикла треков. Вместе с ними
используются structured bindings, вывод типов у `std::lock_guard`/
`std::unique_lock`, `std::filesystem`, `std::optional`, `std::jthread`,
`std::stop_token` и атомарные счётчики поколений. `std::string_view` в обработке `--version` не владеет строкой и
используется только пока действителен соответствующий элемент `argv`.

Ресурсы освобождаются по RAII: `std::unique_ptr` с custom deleter закрывает
`PGconn`/`PGresult`, деструкторы потоковых классов выполняют `join`, а локальные
контейнеры OpenCV владеют памятью кадров. В `HlsEncoder::submit` кадр копируется,
поскольку `cv::Mat` обычно разделяет буфер по счётчику ссылок и исходный кадр
продолжает изменяться видеопотоком.

REST-лямбды Drogon захватывают компоненты по ссылке. Это безопасно благодаря
порядку жизни в `main`: все компоненты создаются до регистрации маршрутов,
`app.run()` блокирует основной поток, а разрушение начинается только после
остановки event loop. Передавать такие обработчики за пределы времени жизни
компонентов нельзя.

## Многопоточность

Во время работы существуют следующие независимые исполнители:

1. пул Drogon (`SERVER_THREADS`, минимум 2) обслуживает REST-запросы;
2. постоянный RTSP-reader владеет `cv::VideoCapture` и публикует последний кадр;
3. постоянные Motion/Tracking- и YOLO-worker-ы независимо обрабатывают задания;
4. координатор рисует последние результаты и наполняет ограниченный буфер;
5. display-worker равномерно публикует JPEG-preview и HLS с выбранными 15/30 FPS;
6. `HlsEncoder` передаёт последний готовый кадр отдельному процессу FFmpeg.

Слоты камеры, Motion/Tracking, YOLO и HLS имеют глубину один. Только показ
использует ограниченную очередь на 12 кадров и начинает выдачу после накопления
6 кадров. Это сглаживает сетевой джиттер с запасом 0,2 с при 30 FPS или 0,4 с
при 15 FPS.
Внутренний пул OpenCV ограничен двумя потоками, оставляя CPU для RTSP и HLS.
Полный и tracking-детекторы имеют отдельные ONNX Runtime-сессии; каждая сессия
по умолчанию использует два intra-op потока и один inter-op поток. Лимит сессии
можно изменить через `CVA_ONNXRUNTIME_THREADS`, не меняя пул OpenCV.

Изменяемое состояние между REST и видео защищает `RuntimeState::mutex`.
Пользователи и сессии имеют отдельный mutex, HLS — собственные mutex и
`condition_variable_any`. Постоянные workers используют `std::jthread` и
прерываемые `std::stop_token` ожидания. Поколения конфигурации атомарны: REST
не трогает `VideoCapture`, а только публикует настройки и просит видеопоток
переоткрыть источник. Очередь HLS имеет глубину один, поэтому медленное
кодирование отбрасывает устаревшие кадры вместо неограниченного роста памяти.
По умолчанию HLS работает в 720p/30 FPS. `POST /api/v1/hls/config` принимает
`{"height":720,"fps":30}`; допустимы 15 и 30 FPS, а запрос только с `height`
сохраняет текущую частоту. Масштабирование выполняет фильтр FFmpeg, поэтому
поток подачи не блокируется на синхронном `cv::resize`.

## Запуск Clang-сервера

Сначала запустите общий PostgreSQL-контейнер Windows, которым пользуются обе
версии Server:

```powershell
docker compose -f C:\work\Docker\PostgreSQL\docker-compose.yml up --detach --wait
```

Он публикует PostgreSQL на `127.0.0.1:5432`. Контейнерный Clang Server
подключается к той же базе через `host.docker.internal:5432`; отдельный
PostgreSQL основной Clang-стек больше не создаёт. Затем задайте пароль
администратора и запустите HTTP Server. Ключ `-Viewer` дополнительно запускает
контейнер JS Viewer, а `-Build` предварительно собирает образ:

```powershell
.\scripts\run-docker.ps1 -AdminPassword 'change-me' -Viewer
```

HTTPS-профиль по умолчанию использует сертификат и ключ соседнего Python Server
и публикует REST на порту 8443:

```powershell
.\scripts\run-docker-https.ps1 -AdminPassword 'change-me'
```

Свои PEM-файлы можно передать явно:

```powershell
.\scripts\run-docker-https.ps1 `
  -AdminPassword 'change-me' `
  -CertificatePath 'C:\certs\server-cert.pem' `
  -PrivateKeyPath 'C:\certs\server-key.pem'
```

Контейнер собирается и работает на Ubuntu 24.04. Windows- и Linux-цели требуют
Clang 21.1.8, OpenCV 5.0.0 и ONNX Runtime 1.23.2. При первой Docker-сборке
официальные архивы OpenCV и ONNX Runtime проверяются по SHA-256. OpenCV
компилируется отдельным кэшируемым слоем, ONNX Runtime используется как готовый
CPU SDK. FFmpeg не компилируется: пакет Ubuntu запускается как внешний
`/usr/bin/ffmpeg`.

Docker-образ по умолчанию задаёт `CVA_OPENCV_THREADS=2` и
`CVA_ONNXRUNTIME_THREADS=2`. В `compose.yaml` и `compose.compare.yaml` значения
можно переопределить одноимёнными переменными окружения; изменение лимита ONNX
Runtime применяется отдельно к каждой из двух сессий YOLO.

HTTP REST будет доступен на `http://127.0.0.1:8080`, HTTPS REST — на
`https://127.0.0.1:8443`, HLS — на `http://127.0.0.1:8888/live.m3u8`, а Viewer
при ключе `-Viewer` — на `http://127.0.0.1:5173`. Если пароль не задан, сервер
создаст случайный и один раз выведет его в `docker compose logs clang-server`.
В health-ответе и заголовке Viewer контейнерная версия отображается как
`ClangVideoAnalyticsServer Docker`; нативная Windows-версия сохраняет имя
`ClangVideoAnalyticsServer`.

Основной Docker-стек сохраняет журнал Server вне контейнера, в каталоге
`C:\work\settings\logs\clang-server`. Drogon держит не более пяти файлов по
10 МБ. Администратор может посмотреть до 2000 последних строк через защищённый
`GET /api/v1/diagnostics/server-log?lines=500`; роль `viewer` получает HTTP 403.
Статус потока также возвращает список активных за последние 30 секунд сессий
только администратору — без логинов и Bearer-токенов.

Камеры хранятся в массиве `stream.cameras` файла `settings.json`. Каждая запись
содержит числовой `id` от 0 до 9, `name`, `host`, `user`, `password` и
`channel`. `stream.activeCameraIndex` задаёт позицию камеры, открываемой при
старте. Администратор может изменить эти записи в диалоге Viewer; Server
атомарно заменяет внешний файл, а пароль не возвращает через REST. За основу
можно взять `settings.example.json`. Рабочий `settings.json` исключён из Git,
поскольку содержит реквизиты камеры. Если массив пуст, Server запускается без
RTSP и ожидает добавления камеры через административный API.

В контейнерном Viewer укажите адрес `http://clang-server:8080`: запросы выполняет его Node-прокси внутри compose-сети. Если Viewer запущен непосредственно на Windows, используйте `http://127.0.0.1:8080`. Viewer сам использует порт 8888 того же узла для HLS.

## Нативная сборка Windows с Clang

Требуется Visual Studio 2022 с компонентом `Desktop development with C++` и
Windows SDK. Отдельно устанавливать Clang, CMake и Ninja не нужно: скрипт
загружает официальный LLVM 21.1.8 с проверкой SHA-512, использует Ninja из
Visual Studio, а при слишком старом CMake — проверенный переносимый архив
Kitware. vcpkg загружается в `.tools/vcpkg` автоматически и сохраняет
зависимости на диске.

Первая сборка долгая, поскольку vcpkg компилирует Drogon, PostgreSQL client,
OpenSSL и curl. OpenCV 5 берётся из локального SDK. Build-script загружает
официальный ONNX Runtime CPU SDK 1.23.2 и проверяет его SHA-256; FFmpeg остаётся
внешней программой:

```powershell
cd C:\work\Clang\ClangVideoAnalyticsServer
.\scripts\build-windows.ps1
```

Готовый файл будет находиться в
`C:\work\build\ClangVideoAnalyticsServer-windows-clang21-release\ClangVideoAnalyticsServer.exe`.
Запуск HTTP-версии:

```powershell
.\scripts\run-windows.ps1 `
  -AdminPassword 'change-me' `
  -DatabaseHost 127.0.0.1 `
  -DatabasePort 5432 `
  -DatabasePassword 'postgres-password'
```

Она слушает `http://127.0.0.1:8080`. HTTPS-версия запускается отдельно:

```powershell
.\scripts\run-windows-https.ps1 `
  -AdminPassword 'change-me' `
  -DatabasePassword 'postgres-password'
```

HTTPS-профиль слушает `https://127.0.0.1:8443` и по умолчанию использует
`server-cert.pem` и `server-key.pem` из соседнего проекта
`PythonVideoAnalyticsServer`. Профили рассчитаны на поочерёдный запуск и
используют одни настройки, учётную запись и runtime-каталог.
Общие сценарии и Docker передают Qt, Clang и Python один каталог
`C:\work\settings\runtime\shared\data`, поэтому области и параметры
Motion/Tracking/YOLO сохраняются для всех реализаций. HLS, модели, uploads и
auth остаются в отдельных служебных каталогах внутри `C:\work\settings`.

Кнопка «Запись» во Viewer включает запись исходных кадров камеры на Server.
Доступны отдельные файлы длительностью 10, 15, 30 или 60 минут; по умолчанию
выбран один час. Имена имеют вид `2026-08-25_14-30-00_camera_ptz.mp4` или
`2026-08-25_14-30-00_panoramic.mp4`, без миллисекунд. Камера имеет один
RTSP-сеанс через `VideoCapture`: тот же необработанный кадр до рамок и аналитики
передаётся неблокирующей очередью глубины один в FFmpeg и кодируется в H.264.
Отдельного подключения рекордера к камере нет. Звуковая дорожка не записывается.
Обычный MP4 окончательно формируется после команды остановки и затем открывается
в VLC. REST сразу подтверждает остановку, а worker закрывает stdin FFmpeg и
дожидается формирования индексных таблиц, не останавливая Viewer.
Нативные скрипты сохраняют записи в `C:\work\settings\files`. Путь можно
переопределить параметром `-RecordingsPath` или переменной
`RECORDINGS_PATH`. Для записи требуется FFmpeg и роль `admin`.

Viewer получает список поддерживаемых видео из этого каталога запросом
`GET /api/v1/recordings/files`. В этом ответе полный путь не передаётся.
Команда `POST /api/v1/stream/file` принимает имя выбранной записи, повторно
проверяет, что файл находится непосредственно внутри `RECORDINGS_PATH`, и
подаёт его кадры в общий `VideoPipeline`. Motion, Tracking, YOLO, JPEG и HLS
работают так же, как для камеры. Файл воспроизводится с записанным FPS, а при
достижении конца источник штатно останавливается без RTSP-переподключения.

Проверить встроенную версию без запуска сервера:

```powershell
C:\work\build\ClangVideoAnalyticsServer-windows-clang21-release\ClangVideoAnalyticsServer.exe --version
```

HTTP-сервер запускается нативно, без Docker, на REST-порту 8080; HTTPS-профиль
использует REST-порт 8443. HLS в обоих случаях доступен на HTTP-порту 8888.
DLL из `C:\work\build\ClangVideoAnalyticsServer-windows-clang21-release\vcpkg_installed\x64-windows-release\bin` автоматически добавляются скриптом в
`PATH`. Рабочие файлы сохраняются в `runtime`, а зависимости и build cache — в
`.tools` и `C:\work\build\ClangVideoAnalyticsServer-windows-clang21-release`.
При наличии `C:\work\tools\ffmpeg.exe` launcher использует этот общий автономный
FFmpeg; экземпляр из `vcpkg_installed` остаётся только резервом Release-сборки.

PostgreSQL должен быть доступен как Windows-служба или на отдельном компьютере.
Для пустой базы один раз выполните `db\init.sql`, например:

```powershell
psql -h 127.0.0.1 -U qtvideoanalytics -d qtvideoanalytics -f .\db\init.sql
```

Если PostgreSQL недоступен, сервер всё равно запустится, но история tracking и
YOLO работать не будет.

Оба `GET`-маршрута истории принимают одинаковые необязательные параметры:
`region`, `date`, `time`, `timezoneOffset`, `sort` (`region`, `date`, `time` или
`duration`) и `order` (`asc` или `desc`). Дата передаётся как `YYYY-MM-DD`, а начало
выбранного часа — как `HH:00`; `timezoneOffset` соответствует
`Date.getTimezoneOffset()` браузера. Например:

```text
/api/v1/tracking/history?region=Вход&date=2026-08-26&time=07:00&timezoneOffset=-180&sort=time&order=asc
```

Подробное учебное описание процессов, потоков, видеоконвейера и PostgreSQL
находится в [документе по архитектуре](Architecture.md). Версия для чтения и
печати: [PDF](output/pdf/ClangVideoAnalyticsServer-Architecture.pdf). То же
содержание, распределённое по разделам ГОСТ 19.402-78 без форматки ЕСПД:
[описание программы](output/pdf/ClangVideoAnalyticsServer-Program-Description-GOST-19.402-78.pdf).

## Проверка REST

```powershell
$env:TEST_PASSWORD = 'change-me'
python tests/rest_contract.py
```

## Сравнение Python, Clang и Rust

`compose.compare.yaml` собирает и поднимает три сервера в Docker, PostgreSQL и запускает для них один k6-сценарий:

```powershell
./scripts/compare.ps1 -VirtualUsers 20 -Duration 30s
```

Clang: REST `48080`, HLS `48888`. Python: REST `28443`, HLS `28888`.
Rust (Ubuntu 24.04): REST `38080`, HLS `38888`. Viewer сравнения доступен на
`http://127.0.0.1:15173`; внутри compose-сети реализации доступны как
`http://clang-server:8080`, `http://python-server:8443` и
`http://rust-server:8080`. JSON-результаты и `summary.csv` сохраняются в
`results`.

Qt-сервер в Docker-окружение не входит и проект Qt не изменяется.

## Профилирование Release

Нативная Windows Release-сборка сохраняет оптимизацию `/O2 /Ob2 /DNDEBUG`,
добавляет PDB через `/Zi /DEBUG:FULL` и оставляет оптимизации линкера
`/OPT:REF /OPT:ICF`. Символы предназначены для Windows Performance Analyzer и
не включают поведение Debug-сборки.

Защищённый маршрут `GET /api/v1/diagnostics/profiling` возвращает память
процесса, счётчики и `mean/p50/p95/p99/max` последних 2048 измерений. Для
нового прогона роль `admin` очищает накопитель запросом `DELETE` того же
маршрута. Основные длительности: `capture.read`, `motion.detect`,
`tracking.update`, `motion.total`, `yolo.full2304.total`,
`yolo.tracking256.taskTotal` и `hls.submit`.

Clang, Python и Rust используют одинаковый контракт из 34 счётчиков
прохождения кадров. Счётчики всегда присутствуют в ответе, в том числе с
нулевым значением:

- `capture.framesOffered/framesRead/framesFailed` — попытки чтения и их исход;
- `pipeline.framesReplacedLatest` — промежуточные кадры, заменённые reader-ом
  до получения аналитическим циклом;
- `motion.framesOffered/tasksAccepted/tasksCompleted/tasksSkippedBusy/tasksFailed`
  — вход, обработка и пропуски Motion;
- `tracking.framesOffered/updatesCompleted/framesSkippedUpstream` — кадры,
  дошедшие или не дошедшие до сопровождения через Motion;
- `yolo.full2304.*` и `yolo.tracking256.*` — предложенные, принятые,
  завершённые, заменённые и ошибочные задания распознавания;
- `*.resultsDiscardedStale` — готовые результаты старого поколения камеры или
  конфигурации, которые не были применены;
- `rtsp.openAttempts/openSuccesses/openFailures/readFailures/corruptFrames/
  reconnectScheduled/reconnectSuccesses` — подключения и переподключения RTSP.

Для RTSP непустой кадр дополнительно проверяется по разреженной BGR-выборке.
Массовое резкое изменение блоков считается повреждением декодирования: кадр не
попадает в Motion, Tracking, YOLO, JPEG и HLS, после чего VideoCapture
пересоздаётся. Такой случай увеличивает `capture.framesFailed`,
`rtsp.corruptFrames` и счётчики переподключения. После открытия первые два кадра
служат прогревом декодера, а третий согласованный кадр открывает публикацию.

Отключённый детектор не считается пропустившим кадр. Для занятого Motion
`framesSkippedUpstream` одновременно показывает потерю кадра для Tracking.
