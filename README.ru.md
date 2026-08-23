[English](README.md) | [Русский](README.ru.md)

# RasGate

RasGate добавляет небольшой HTTP API поверх утилиты `rac` платформы
1С:Предприятие. Клиент передаёт массив аргументов, RasGate запускает настроенный
исполняемый файл и возвращает код завершения, вывод и время работы команды.

```text
HTTP-клиент -> RasGate -> RAC -> RAS -> кластер 1С
```

Сервис намеренно остаётся простым: не вызывает shell, не разбирает вывод RAC и
не подменяет собой предметный API для кластеров, баз и сеансов.

Веб-интерфейса на `/` нет, поэтому браузер получает там обычный JSON-ответ
`404`. Для проверки самого сервиса используйте `/rasgate/status`.

## Требования

Для запуска нужны Windows или Linux, подходящая версия `rac` и сетевой доступ к
RAS. Готовые архивы включают .NET Runtime.

Для сборки из исходников нужен .NET SDK 10. Команды Makefile и release-скрипт
также требуют GNU Make и Bash. Для контейнерного запуска нужен Docker Compose.

## API-ключ

Метод `POST /rac/execute` защищён API-ключом. Без корректного
`RasGate:ApiKey` приложение не запустится. Допустимая длина ключа — от 32 до
512 символов; пробелы в начале и конце запрещены.

RasGate использует стандартную конфигурацию .NET, поэтому ключ можно хранить в
`appsettings.json` как при локальном, так и при промышленном запуске. Для
запуска из исходников редактируйте `src/RasGate.Web/appsettings.json`. У
опубликованного приложения измените `appsettings.json`, лежащий рядом с
исполняемым файлом RasGate. Добавьте `ApiKey` в существующую секцию `RasGate`:

```json
{
  "RasGate": {
    "InstanceName": "RasGate Application",
    "ApiKey": "<вставьте-сюда-случайный-ключ-длиной-не-менее-32-символов>"
  }
}
```

Замените значение в угловых скобках. Версия файла из репозитория отслеживается
Git и намеренно не содержит ключа. Не добавляйте настоящий ключ в коммит. Если
в production ключ хранится в этом файле, ограничьте доступ к каталогу
приложения, артефактам развёртывания и резервным копиям с конфигурацией.

Более безопасный вариант для локальной разработки — .NET User Secrets. Ключ
будет храниться за пределами рабочего каталога:

```bash
api_key="$(openssl rand -hex 32)"
dotnet user-secrets set \
  "RasGate:ApiKey" "$api_key" \
  --project src/RasGate.Web/RasGate.Web.csproj
```

PowerShell:

```powershell
$apiKey = [guid]::NewGuid().ToString("N") + [guid]::NewGuid().ToString("N")
dotnet user-secrets set `
  "RasGate:ApiKey" $apiKey `
  --project src/RasGate.Web/RasGate.Web.csproj
```

Переменная окружения — альтернативный способ для любого варианта запуска. Её
значение имеет приоритет над `appsettings.json`. В именах параметров .NET
двойное подчёркивание заменяет `:`:

```bash
export RasGate__ApiKey="$(openssl rand -hex 32)"
./RasGate.Web
```

Docker Compose читает ключ из `RASGATE_API_KEY` в локальном файле `.env`.
Независимо от способа настройки клиент должен передавать то же значение в
заголовке `X-Api-Key`.

## Локальный запуск

Укажите путь к RAC в `src/RasGate.Web/appsettings.json` или в переменной
`Rac__ExecutablePath`:

```json
{
  "RasGate": {
    "InstanceName": "RasGate Application"
  },
  "Rac": {
    "ExecutablePath": "/opt/1cv8/x86_64/rac"
  }
}
```

В Windows обратные слеши в JSON нужно экранировать:

```json
"ExecutablePath": "C:\\Program Files\\1cv8\\8.3.27.2214\\bin\\rac.exe"
```

Запустите проект и проверьте оба метода состояния:

```bash
dotnet run --project src/RasGate.Web/RasGate.Web.csproj

curl http://127.0.0.1:5050/rasgate/status
curl http://127.0.0.1:5050/rac/status
```

Пример безопасного вызова `rac --version`:

```bash
api_key='<тот же ключ, который указан в настройках RasGate>'
curl \
  --request POST \
  --header 'Content-Type: application/json' \
  --header "X-Api-Key: ${api_key}" \
  --data '{"arguments":["--version"]}' \
  http://127.0.0.1:5050/rac/execute
```

По умолчанию RasGate слушает только localhost. Для доступа по сети измените
`Urls`, ограничьте порт сетевым экраном и настройте TLS в RasGate или на
доверенном reverse-прокси.

### Проверка конфигурации

Проверить настройки без открытия HTTP-порта и запуска RAC:

```bash
./RasGate.Web --validate-config
```

Команда возвращает `0`, если настройки корректны, и ненулевой код при ошибке.
API-ключ в вывод не попадает.

## Запуск в качестве службы

В архивах для Windows и Linux есть скрипты установки службы. При необходимости
тот же исполняемый файл можно запускать вручную из консоли.

### Служба Windows

1. Распакуйте Windows-архив в постоянный каталог, например
   `C:\Program Files\RasGate`.
2. Настройте `RasGate:ApiKey` и `Rac:ExecutablePath` в
   `appsettings.json`.
3. Откройте Windows PowerShell от имени администратора в этом каталоге.
4. Запустите `.\install-service.ps1`.

Проверка и перезапуск службы:

```powershell
Get-Service -Name RasGate
Restart-Service -Name RasGate
Invoke-RestMethod http://127.0.0.1:5050/rasgate/status
```

Удалить службу, не затрагивая настройки, логи и файлы приложения:

```powershell
.\uninstall-service.ps1
```

### Служба systemd

1. Распакуйте Linux-архив и настройте `appsettings.json`.
2. Запустите `sudo ./install-service.sh`.

Установщик копирует RasGate в `/opt/rasgate`, создаёт непривилегированного
пользователя `rasgate`, устанавливает `rasgate.service` и запускает службу.

```bash
systemctl status rasgate.service
sudo systemctl restart rasgate.service
journalctl -u rasgate.service -f
curl http://127.0.0.1:5050/rasgate/status
```

Удалить службу, не затрагивая `/opt/rasgate`, настройки и логи:

```bash
sudo /opt/rasgate/uninstall-service.sh
```

Скрипты не устанавливают RAC и не меняют настройки сетевого экрана или TLS.
RasGate использует адрес из параметра `Urls`.

## Конфигурация

| Параметр | Значение по умолчанию и ограничения |
|---|---|
| `Urls` | `http://127.0.0.1:5050` |
| `RasGate:InstanceName` | Имя, возвращаемое `/rasgate/status` |
| `RasGate:ApiKey` | Обязательный секрет длиной 32-512 символов |
| `Rac:ExecutablePath` | `rac`, абсолютный путь или команда из `PATH` |
| `Rac:TimeoutSeconds` | `30`, диапазон 1-3600 |
| `Rac:StatusCacheSeconds` | `30`, диапазон 1-300 |
| `Rac:MaxConcurrentProcesses` | `4`, диапазон 1-32 на один экземпляр RasGate |
| `Rac:MaxOutputBytes` | `4194304`, максимум `16777216` на каждый поток вывода |
| `Rac:MaxArgumentCount` | `128`, диапазон 1-128 |
| `Rac:MaxArgumentBytes` | `8192` байт UTF-8 на один аргумент |
| `Rac:MaxTotalArgumentBytes` | `24576` байт UTF-8 суммарно; не меньше `MaxArgumentBytes` |

Любой параметр можно переопределить переменной окружения, заменив `:` на `__`,
например `Rac__TimeoutSeconds=60`. Конфигурация читается при запуске, поэтому
после её изменения RasGate нужно перезапустить.

## Docker Compose

Скопируйте пример файла окружения и заполните ключ и каталог RAC:

```bash
cp .env.example .env
```

`RAC_HOST_PATH` должен указывать на каталог с Linux-версией `rac` и нужными ей
библиотеками. Каталог монтируется в `/opt/1c/rac` только для чтения.

```bash
docker compose up --build --detach
docker compose down
```

Контейнер работает без root-прав и с файловой системой только для чтения. Логи
хранятся в volume `logs`, а `/tmp` размещён в tmpfs. По умолчанию порт 5050
публикуется только на `127.0.0.1`.

## HTTP API

| Метод | Путь | Авторизация | Назначение |
|---|---|---|---|
| `GET` | `/rasgate/status` | нет | имя и версия RasGate |
| `GET` | `/rac/status` | нет | кэшированная доступность и версия RAC |
| `POST` | `/rac/execute` | `X-Api-Key` | выполнение команды RAC |

`/rac/status` всегда отвечает HTTP `200`. Доступность RAC находится в
`data.available`. Проверка кэшируется отдельно и не занимает слот выполнения
команд.

Запрос на выполнение содержит массив аргументов:

```json
{
  "arguments": ["cluster", "list", "localhost:1545"]
}
```

Ответ на завершившийся вызов:

```json
{
  "success": true,
  "data": {
    "outcome": "succeeded",
    "exitCode": 0,
    "standardOutput": "...",
    "standardError": "",
    "durationMilliseconds": 42,
    "timedOut": false
  }
}
```

Поле `success` относится к HTTP-вызову, а не к результату RAC. Результат нужно
определять по `outcome`, `exitCode` и `timedOut`:

- `succeeded` — RAC завершился с кодом 0;
- `failed` — RAC вернул ненулевой код;
- `unknown` — RasGate не может подтвердить внешний результат.

Нельзя автоматически повторять команду после `unknown`, разрыва соединения
после запуска процесса, превышения лимита вывода или ошибки очистки ресурсов.
Команда уже могла изменить кластер. RasGate не делает внутренние повторы и не
гарантирует однократное выполнение произвольной операции RAC.

Основные ошибки API: `400 bad_request`, `401 unauthorized`,
`429 rac_capacity_exceeded`, `502 rac_output_limit_exceeded`,
`502 rac_execution_outcome_unknown` и `503 rac_unavailable`.

В ответах есть `X-Trace-Id`, по которому удобно искать запись в серверном логе.
OpenAPI JSON доступен по `/openapi/v1.json` в окружении `Development`.

## Связанные проекты

RasGate входит в [Ras Ecosystem](https://github.com/RasEcosystem):

- [RasHub](https://github.com/RasEcosystem/ras-hub) — центральный сервис
  управления, который связывает RasStudio с RasGate и предоставляет единый API;
- [RasStudio Mono](https://github.com/RasEcosystem/ras-studio-mono) —
  экспериментальный монолитный веб-клиент для администрирования кластеров
  1С:Предприятия через RasHub и RasGate.

## Лицензия

См. [LICENSE](LICENSE).
