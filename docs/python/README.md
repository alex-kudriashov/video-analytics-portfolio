# PythonVideoAnalyticsServer

Независимая от Qt реализация REST-контракта `QtVideoAnalyticsServer` для
`JsVideoAnalyticsViewer` и `QtVideoAnalyticsViewer`.

Проект **не использует Qt, PyQt, PySide, Qt DLL или исполняемый файл
QtVideoAnalyticsServer**. OpenCV подключается через headless Python-модуль
`cv2`.

Текущая версия сервера указана в `pyproject.toml`. История изменений сервера
ведётся в [Release notes](ReleaseNotes.txt).

## Состояние реализации

Первый этап предоставляет совместимый REST-каркас:

- PBKDF2-SHA256 авторизация и Bearer-сессии;
- чтение существующего `auth.json` Qt-сервера;
- камеры и полная конфигурация Viewer без публикации RTSP-секретов;
- RTSP-захват в отдельном процессе, автоматическое открытие камеры и
  переподключение после обрыва;
- выбор серверного видео из папки `files` и передача его в аналитику как камеры;
- RTSP по TCP, отбрасывание повреждённых FFmpeg-пакетов и защита Motion,
  YOLO/HLS/JPEG от тяжело разрушенных HEVC-кадров;
- плавный JPEG/HLS-показ 15 FPS в отдельном потоке с ограниченным буфером;
- детектор движения с нормализованными областями и отрисовкой рамок;
- сопровождение движущихся объектов с постоянными ID, сглаживанием и
  траекториями;
- YOLO-инференс через ONNX Runtime CPU, NMS по классам, COCO-метки,
  маскирование и области распознавания;
- фактический JPEG-preview входа нейросети `/detection/objects/input`;
- ONVIF PTZ основной камеры через совместимые команды `move/stop`;
- сохранение конфигурации Motion/Tracking/YOLO между перезапусками;
- управление состоянием stream, HLS, Motion и YOLO;
- серверная запись исходных кадров общего RTSP-захвата в H.264/MP4;
- бинарная загрузка ONNX с сохранением по SHA-256;
- PostgreSQL-история Motion Tracking и YOLO с фотографиями объектов;
- фильтрация истории по области, одной дате и часу, серверная сортировка по
  области, дате, времени и длительности;
- preview и profiling-маршруты;
- JSON-поля и основные HTTP-коды исходного сервера.

HLS/FFmpeg формирует выбранный пользователем поток 15 или 30 FPS из кадров с
наложенной аналитикой. При 30 FPS недостающие кадры повторяются только на выходе
HLS и не запускают повторно Motion/YOLO.

## Установка

```powershell
Set-Location C:\work\Python\PythonVideoAnalyticsServer
py -3.10 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e ".[dev]"
Copy-Item settings.example.json settings.json
```

## Первый запуск

```powershell
.\.venv\Scripts\python.exe -m python_video_analytics_server `
  --host 0.0.0.0 --port 8080 `
  --settings .\settings.json --auth .\auth.json `
  --login admin --password ChangeThisPassword
```

На Windows те же параметры, общие настройки, модель, FFmpeg и PostgreSQL
подготавливает HTTP-скрипт:

```powershell
.\scripts\run-windows.ps1
```

Без `--password` сервер создаст случайный пароль и один раз напечатает его в
консоли. После создания `auth.json` параметры логина и пароля игнорируются.

`auth.json` сохраняет совместимую верхнеуровневую учётную запись администратора
и может содержать массив дополнительных пользователей. Поддерживаются роли:

- `admin` — полный доступ ко всем REST-командам;
- `viewer` — чтение камер, конфигурации, кадров, статуса и истории, а также
  запрос обновлённого предпросмотра входа YOLO.

Пароли хранятся только как PBKDF2-SHA256-хеши. Ответ входа и маршрут
`GET /api/v1/auth/session` возвращают роль текущей Bearer-сессии. Попытка
пользователя `viewer` изменить состояние сервера завершается HTTP `403`.

Администратор также может изменять список камер через Viewer, видеть число
подключённых клиентов и открывать внешний журнал сервера. В Docker журнал
сохраняется в `C:\work\settings\logs\python-server`, а камеры — в общем
`C:\work\settings\settings.json`.

Для HTTPS передайте PEM-сертификат и закрытый ключ:

```powershell
.\.venv\Scripts\python.exe -m python_video_analytics_server `
  --port 8443 `
  --ssl-certfile .\server-cert.pem --ssl-keyfile .\server-key.pem
```

Готовый HTTPS-профиль запускается отдельным скриптом:

```powershell
.\scripts\run-windows-https.ps1
```

HTTP-профиль слушает `http://127.0.0.1:8080`, HTTPS-профиль —
`https://127.0.0.1:8443`. Они используют общий runtime и рассчитаны на
поочерёдную работу. OpenAPI доступен по `/docs`.

## Docker на Ubuntu 24.04

Контейнер собирается на Ubuntu 24.04 с системным Python 3.12 в отдельном
`/opt/venv`. OpenCV 4.14 Headless, ONNX Runtime и остальные Python-зависимости
устанавливаются готовыми wheel-пакетами зафиксированных версий. FFmpeg не
собирается из исходников: Server запускает внешний пакет Ubuntu
`/usr/bin/ffmpeg`.

Сначала должен работать общий PostgreSQL-контейнер Windows:

```powershell
docker compose -f C:\work\Docker\PostgreSQL\docker-compose.yml up --detach --wait
```

HTTP-профиль с первой сборкой образа запускается так:

```powershell
.\scripts\run-docker.ps1 -AdminPassword 'change-me' -Build
```

Ключ `-Viewer` дополнительно запускает JS Viewer:

```powershell
.\scripts\run-docker.ps1 -AdminPassword 'change-me' -Viewer
```

HTTPS-профиль использует локальные `server-cert.pem` и `server-key.pem` либо
пути, явно переданные параметрами `-CertificatePath` и `-PrivateKeyPath`:

```powershell
.\scripts\run-docker-https.ps1 -AdminPassword 'change-me'
```

HTTP REST доступен на `http://127.0.0.1:18080`, HTTPS REST — на
`https://127.0.0.1:18443`, HLS — на `http://127.0.0.1:18888/live.m3u8`, Viewer —
на `http://127.0.0.1:15173`. В health-ответе и заголовке Viewer контейнер
отображается как `PythonVideoAnalyticsServer Docker`; нативная Windows-версия
сохраняет имя `PythonVideoAnalyticsServer`.

Оба Docker-профиля подключаются к `qt-video-analytics-postgresql` через
`host.docker.internal:5432` и не создают второй PostgreSQL. Настройки
Motion/Tracking/YOLO монтируются из общего `runtime\shared\data`; служебные HLS,
модели и авторизация контейнера хранятся отдельно в `runtime\docker\python-*`.
HTTP и HTTPS являются
сменяемыми профилями одного контейнера и рассчитаны на поочерёдный запуск.
Python и Clang Docker Server используют единый локальный контур портов, поэтому
перед запуском одного следует остановить другой либо передать свободные
`-ServerPort`, `-HlsPort` и `-ViewerPort`. Контейнерный Viewer обращается к
`python-server:8080` и `python-server:8888` внутри Compose-сети; HTTPS-профиль
автоматически переключает REST upstream на TLS. Для публичного HTTPS-сайта HLS
по умолчанию публикуется через единый путь `/hls/`, который можно изменить
параметром `-PublicHlsUrl`.

Готовые сценарии `C:\work\settings\run-python*.ps1` и Docker используют тот же
`runtime\shared\data`, что Qt и Clang Server. Поэтому Motion/Tracking, области,
YOLO 2304, пороги, разрешённые классы и выбранные области одинаковы во всех
реализациях и сохраняются в одном месте.

Кнопка «Запись» во Viewer передаёт исходный кадр единственного `VideoCapture`
до Motion/YOLO в одноэлементный слот recorder. FFmpeg не подключается к RTSP
повторно, кодирует H.264/MP4 без звука и сохраняет файлы в
`C:\work\settings\files`. Доступны интервалы 10, 15, 30 и 60 минут; остановка
асинхронно финализирует обычный MP4, не останавливая просмотр.

Viewer получает доступные файлы через `GET /api/v1/recordings/files`. В ответе
есть только имя и размер, полный серверный путь не раскрывается. Команда
`POST /api/v1/stream/file` принимает `{"name":"запись.mp4"}`; сервер повторно
проверяет расширение и запрещает выход из папки `files`. Файл читает тот же
`VideoCapture` в дочернем процессе и подаёт кадры в Motion/Tracking/YOLO с
частотой исходного файла.

Журнал каждого HTTP-запроса по умолчанию отключён, чтобы запросы JPEG-preview
не заполняли консоль 15 строками в секунду. Для временной диагностики запустите
сервер с параметром `--access-log`; ошибки и важные служебные сообщения
выводятся независимо от этого параметра.

При запуске сервер автоматически переводит `stdout`, `stderr` и кодовую
страницу консоли Windows в UTF-8. Эта настройка наследуется дочерними Python-
процессами и предотвращает искажение русских сообщений в PowerShell и PyCharm.

## PostgreSQL-история

Сервер использует таблицы `tracking_history` и `yolo_history` исходного
QtVideoAnalyticsServer. Существующая история сразу доступна обоим Viewer, а
новые события записываются в тот же JSON-формат.

Оба маршрута чтения истории принимают параметры `region`, `date` в формате
`YYYY-MM-DD`, `time` в формате целого часа `HH:00`, `timezoneOffset`, `sort`
(`region`, `date`, `time`, `duration`) и `order` (`asc`, `desc`). Значения
передаются в параметризованный SQL; динамически выбираются только выражения из
фиксированного списка сортировки.

Параметры по умолчанию совпадают с Qt-сервером: адрес `127.0.0.1`, порт `5432`,
база и пользователь `qtvideoanalytics`. Пароль безопаснее задавать переменной
окружения:

```powershell
$env:QTVIDEOANALYTICS_DB_PASSWORD = "пароль PostgreSQL"
```

Поддерживаются также `QTVIDEOANALYTICS_DB_HOST`,
`QTVIDEOANALYTICS_DB_PORT`, `QTVIDEOANALYTICS_DB_NAME` и
`QTVIDEOANALYTICS_DB_USER`, либо параметры `--db-host`, `--db-port`,
`--db-name`, `--db-user`, `--db-password`. При временной недоступности базы
видео и остальные REST-маршруты продолжают работать; следующий запрос или
событие повторит подключение.

## HLS

При запуске сервер автоматически поднимает локальную HLS-страницу
`http://127.0.0.1:8888/`. Плейлист `/live.m3u8` используется собственным
hls.js в JsVideoAnalyticsViewer через локальный прокси и остаётся совместимым
со встроенным web-view QtVideoAnalyticsViewer. Номер сегмента берётся из Unix
epoch в микросекундах, поэтому после перезапуска FFmpeg браузер не смешивает в
кэше сегменты разных разрешений.

По умолчанию используется `C:\work\tools\ffmpeg.exe`. Путь и порт можно
изменить параметрами:

```powershell
--ffmpeg C:\work\tools\ffmpeg.exe --hls-port 8888
```

HLS намеренно доступен только через `127.0.0.1`: существующий контракт Viewer
не передаёт Bearer-токен при запросах плейлиста и сегментов, поэтому открывать
этот порт для всей локальной сети небезопасно. Разрешение 1280p, 720p, 480p или
исходное и фиксированная частота 15/30 FPS переключаются обычным REST-маршрутом
`/api/v1/hls/config`; автоматического выбора FPS нет.

## Профилирование

Метрики накапливаются непосредственно в отдельном `video-capture-worker`, где
работают OpenCV и ONNX Runtime, и раз в секунду передаются основному процессу.
Маршрут `GET /api/v1/diagnostics/profiling` возвращает память video-процесса,
счётчики и `mean/p50/p95/p99/max` последних 2048 измерений. `DELETE` очищает
накопитель перед новым прогоном.

Основные длительности: `capture.read`, `motion.detect`, `tracking.update`,
`yolo.full2304.total` и `yolo.tracking256.taskTotal`. Счётчики показывают
прохождение кадров по тому же контракту, что и Windows Clang:

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

Все эти поля присутствуют даже при нулевых значениях. Отключённый детектор не
считается пропустившим кадр.

## Проверка

```powershell
.\.venv\Scripts\python.exe -m pytest
```
