# RequestStats — деплой, backup/restore, retention логов

Это гайд по тестовому заданию: поднять стенд на Windows Server, развернуть Java-приложение
`request-stats.war` в Tomcat 9, подключить его к SQL Server Express, настроить бэкапы и
retention логов, и один раз реально проверить restore базы.

Ниже — весь путь от чистой машины до рабочего стенда, шаг за шагом, так, чтобы любой другой
человек мог повторить это с нуля, глядя только на этот файл.

Схема того, что получилось в итоге:

```
Браузер
  ↓  http://localhost:8080/request-stats/
Apache Tomcat 9 (Windows Service)
  ↓
request-stats.war (Java 11)
  ↓  JDBC (mssql-jdbc 12.6.1)
SQL Server Express — localhost:1433
  ↓
БД RequestStats → таблица dbo.request_history
```

---

## 1. Что нужно перед стартом

- Windows Server (у меня — актуальная сборка Windows Server, VM)
- JDK 11 (WAR собран под Java 11, драйвер `mssql-jdbc-12.6.1` тоже под Java 11)
- Apache Tomcat **9.0.x** (не 10, не 11 — в ТЗ явно указана 9-я ветка, у неё другое
  пространство имён `javax.servlet` вместо `jakarta.servlet`, и WAR под 10/11 просто не встанет)
- SQL Server Express
- Файлы `request-stats.war` и `init.sql` из репозитория задания

---

## 2. Windows Server VM

Стандартная установка Windows Server, статический локальный админ-аккаунт, RDP включён для
удобства работы.

---

## 3. Java 11

Ставится обычным установщиком OpenJDK/Temurin 11 x64.

Проверка:

```
java -version
```

Должно быть что-то вроде `openjdk version "11.0.32"`. Это важно проверить в первую очередь —
если Java не той версии или не видна в PATH, ни Tomcat, ни WAR не поднимутся.

---

## 4. SQL Server Express

### 4.1. Установка

Ставил стандартным установщиком SQL Server Express (Basic/Custom — не принципиально, лишь бы
был Database Engine).

### 4.2. Включаем SQL Server Authentication

По умолчанию SQL Server стоит в режиме Windows Authentication. Нашему WAR нужен логин `sa` с
паролем (он так прописан внутри `db.properties`), поэтому:

1. SQL Server Management Studio → правой кнопкой на сервере → **Properties → Security**
2. Переключить на **SQL Server and Windows Authentication mode**
3. Перезапустить службу SQL Server, чтобы режим применился

### 4.3. Включаем sa и задаём пароль

```
Security → Logins → sa → Properties
```

- задать пароль
- на вкладке Status убедиться, что Login включён (Enabled)

> Пароль реального стенда я не привожу в README и не коммичу в репозиторий — держу его
> отдельно (переменная окружения / закрытый конфиг). В примерах команд ниже он подставлен как
> `<SA_PASSWORD>` — при воспроизведении стенда подставляете свой.

### 4.4. Включаем TCP/IP и проверяем порт 1433

По умолчанию сетевой протокол TCP/IP в SQL Server Express выключен.

1. **SQL Server Configuration Manager → SQL Server Network Configuration → Protocols for
   SQLEXPRESS**
2. Включить **TCP/IP**
3. Открыть свойства TCP/IP → вкладка **IP Addresses** → в самом низу, в секции **IPAll**,
   прописать `TCP Port = 1433` (по умолчанию там часто динамический порт, а нам нужен
   фиксированный, потому что именно `1433` зашит в `db.properties` внутри WAR)
4. Перезапустить службу `SQL Server (SQLEXPRESS)`

Проверка, что порт реально слушается:

```
netstat -ano | findstr :1433
```

### 4.5. Firewall

Если стенд проверяют не локально, а по сети — открыть входящее правило на TCP 1433. У меня
приложение и SQL Server на одной машине, WAR ходит на `localhost:1433`, дополнительно ничего
не открывал.

---

## 5. Инициализация базы (init.sql)

`init.sql` создаёт базу `RequestStats` и таблицу `dbo.request_history`, если их ещё нет (там
стоят проверки `IF DB_ID(...) IS NULL` / `IF OBJECT_ID(...) IS NULL`, так что скрипт безопасно
перезапускать).

Структура таблицы:

| Поле | Тип | Комментарий |
|---|---|---|
| id | BIGINT IDENTITY(1,1) | первичный ключ |
| method | NVARCHAR(16) | HTTP-метод запроса |
| path | NVARCHAR(512) | путь запроса |
| client_ip | NVARCHAR(64) | IP клиента, может быть NULL |
| user_agent | NVARCHAR(512) | User-Agent, может быть NULL |
| created_at | DATETIME2(0) | момент записи, по умолчанию SYSUTCDATETIME() |

Плюс индексы по `method` и по `created_at DESC`.

Запуск:

```
sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -i init.sql
```

Проверка, что таблица создалась и пустая:

```
sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -d RequestStats -Q "SELECT COUNT(*) FROM dbo.request_history"
```

---

## 6. Tomcat 9

### 6.1. Установка

Скачал с официального сайта Apache **Tomcat 9.0.x, 64-bit Windows ZIP**. Распаковал в
`C:\Tomcat9`, так чтобы сразу внутри лежали `bin`, `conf`, `lib`, `logs`, `temp`, `webapps`,
`work` — без вложенной папки `apache-tomcat-9.0.xx` внутри.

### 6.2. Первый запуск (без Service, просто проверить, что работает)

```
cd C:\Tomcat9\bin
startup.bat
```

Подождать 5–10 секунд, проверить порт:

```
netstat -ano | findstr :8080
```

Ключевые строчки в логе `C:\Tomcat9\logs\catalina.<дата>.log`:

```
Java: 11.0.32.1
CATALINA_BASE: C:\Tomcat9
Starting ProtocolHandler ["http-nio-8080"]
Server startup in [...] milliseconds
```

Проверка в браузере на самой VM: `http://localhost:8080` — должна открыться стандартная
стартовая страница Apache Tomcat.

### 6.3. Устанавливаем Tomcat как Windows Service

Только после того, как обычный `startup.bat` отработал без ошибок:

```
shutdown.bat
cd C:\Tomcat9\bin
service.bat install Tomcat9
```

Проверка:

```
sc query Tomcat9
```

Ожидаемо: `STATE : 4 RUNNING`.

### 6.4. Деплой WAR

```
copy request-stats.war C:\Tomcat9\webapps\request-stats.war
```

Tomcat при работающей службе сам распакует WAR в `C:\Tomcat9\webapps\request-stats\` в течение
нескольких секунд.

### 6.5. Настройка подключения к БД

Внутри WAR, в `WEB-INF/classes/db.properties`, зашито:

```
db.url=jdbc:sqlserver://localhost:1433;databaseName=RequestStats;encrypt=false;trustServerCertificate=true
db.user=sa
db.password=Your_strong_Password123
```

Пароль-заглушку меняем на реальный пароль `sa`. Если WAR уже распакован, править нужно прямо в
`C:\Tomcat9\webapps\request-stats\WEB-INF\classes\db.properties`, и после правки перезапустить
службу Tomcat:

```
net stop Tomcat9
net start Tomcat9
```

### 6.6. Проверка приложения

```
http://localhost:8080/request-stats/
```

Должна открыться JSP-страница со счётчиками, и при обновлении — новые строки должны появляться
в `dbo.request_history`. Проверка со стороны базы:

```
sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -d RequestStats -Q "SELECT TOP 5 * FROM dbo.request_history ORDER BY id DESC"
```

---

## 7. Backup

### 7.1. Стратегия

Полные (FULL) бэкапы в `.bak`, складываются в `C:\SQLBackups\`, имя файла с таймстампом:

```
RequestStats_YYYYMMDD_HHMMSS.bak
```

### 7.2. Скрипт

`scripts/backup-db.ps1` делает `BACKUP DATABASE` через `sqlcmd`, сам формирует имя файла и
пишет отметку об успехе/ошибке в `C:\SQLBackups\backup-history.log`:

```powershell
$server      = "localhost,1433"
$database    = "RequestStats"
$backupDir   = "C:\SQLBackups"
$logFile     = "$backupDir\backup-history.log"
$sa_password = $env:REQUESTSTATS_SA_PASSWORD

if (!(Test-Path $backupDir)) {
    New-Item -ItemType Directory -Path $backupDir | Out-Null
}

$timestamp  = Get-Date -Format "yyyyMMdd_HHmmss"
$backupFile = "$backupDir\${database}_$timestamp.bak"

$query = "BACKUP DATABASE [$database] TO DISK = N'$backupFile' WITH INIT, STATS = 10;"

sqlcmd -S $server -U sa -P $sa_password -Q $query

if ($LASTEXITCODE -eq 0) {
    Add-Content -Path $logFile -Value "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - SUCCESS - $backupFile"
} else {
    Add-Content -Path $logFile -Value "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - FAILED"
}
```

Запуск вручную:

```
powershell -ExecutionPolicy Bypass -File scripts\backup-db.ps1
```

### 7.3. Автоматический запуск по расписанию

Зарегистрирован в Task Scheduler на ежедневный запуск в 03:00:

```
schtasks /create /tn "RequestStatsBackup" /tr "powershell.exe -ExecutionPolicy Bypass -File C:\RequestStats\scripts\backup-db.ps1" /sc daily /st 03:00 /ru SYSTEM
```

Запуск по расписанию проверен: файл `RequestStats_20260905_182734.bak` был создан именно так,
автоматически, и в `backup-history.log` появилась строка `SUCCESS` с этим именем и временем.
Это тот же самый бэкап, который используется для restore в разделе 8.

---

## 8. Restore — реальная проверка

Это самая важная часть с точки зрения проверки задания: не просто "бэкап есть", а
доказательство, что из него реально восстанавливаются рабочие данные. Восстанавливал бэкап
**под новым именем БД** — `RequestStats_restore` — чтобы не трогать боевую базу.

### 8.1. Смотрим логическую структуру бэкапа

Перед restore под другим именем базы нужно знать логические имена файлов внутри бэкапа
(`LogicalName`) — иначе непонятно, что подставлять в `MOVE`:

```
sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -Q "RESTORE FILELISTONLY FROM DISK = N'C:\SQLBackups\RequestStats_20260905_182734.bak'"
```

Вывод показал `LogicalName` = `RequestStats` и `RequestStats_log` — эти имена и используются в
`MOVE` ниже.

### 8.2–8.4. Restore, проверка статуса, сверка данных

Выполняется одним заходом, все пять команд подряд в одном окне, без очистки экрана между ними:

```
sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -Q "RESTORE DATABASE [RequestStats_restore] FROM DISK = N'C:\SQLBackups\RequestStats_20260905_182734.bak' WITH MOVE N'RequestStats' TO N'C:\Program Files\Microsoft SQL Server\MSSQL16.SQLEXPRESS\MSSQL\DATA\RequestStats_restore.mdf', MOVE N'RequestStats_log' TO N'C:\Program Files\Microsoft SQL Server\MSSQL16.SQLEXPRESS\MSSQL\DATA\RequestStats_restore_log.ldf', REPLACE, STATS = 10;"

sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -Q "SELECT name, state_desc FROM sys.databases WHERE name = 'RequestStats_restore'"

sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -d RequestStats_restore -Q "SELECT COUNT(*) AS RowCount_Restore FROM dbo.request_history"

sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -d RequestStats -Q "SELECT COUNT(*) AS RowCount_Original FROM dbo.request_history"

sqlcmd -S localhost,1433 -U sa -P "<SA_PASSWORD>" -d RequestStats_restore -Q "SELECT TOP 5 * FROM dbo.request_history ORDER BY id DESC"
```

Результат: `RESTORE DATABASE` успешно обработал 433 страницы, база `RequestStats_restore` в
статусе `ONLINE`, `RowCount_Restore = 4` совпало с `RowCount_Original = 4`, и все 4 строки в
`_restore` (те же `id`, `method`, `path`, `created_at`) идентичны оригиналу.


Готовые запросы под это лежат в `sql/restore.sql` и `sql/verify_restore.sql`.

---
<img width="1024" height="768" alt="resrtore" src="https://github.com/user-attachments/assets/59cd7174-00a9-4a2e-8854-5017c2f798b6" />


## 9. Retention логов Tomcat

Логи Tomcat без ротации со временем копятся и занимают место. Настроил автоматическую очистку.

### 9.1. Скрипт

`scripts/cleanup-tomcat-logs.ps1` удаляет файлы логов старше 14 дней и пишет отметку о каждом
запуске в `cleanup-history.log`:

```powershell
$logsPath = "C:\Tomcat9\logs"
$daysToKeep = 14

Get-ChildItem -Path $logsPath -File |
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-$daysToKeep) } |
    Remove-Item -Force

Add-Content -Path "$logsPath\cleanup-history.log" `
    -Value "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - очистка выполнена, хранение $daysToKeep дней"
```

### 9.2. Task Scheduler

```
schtasks /create /tn "TomcatLogCleanup" /tr "powershell.exe -ExecutionPolicy Bypass -File C:\RequestStats\scripts\cleanup-tomcat-logs.ps1" /sc daily /st 02:00 /ru SYSTEM
```

Проверка вручную, не дожидаясь ночи:

```
schtasks /run /tn "TomcatLogCleanup"
type C:\Tomcat9\logs\cleanup-history.log
```

---

## 10. Структура репозитория

```
RequestStats/
├── README.md                      — этот файл
├── sql/
│   ├── init.sql                   — создание БД и таблицы (из задания)
│   ├── backup.sql                 — пример прямого BACKUP DATABASE без обвязки на PowerShell
│   ├── restore.sql                — restore в *_restore с MOVE
│   └── verify_restore.sql         — сверка количества строк и содержимого
├── scripts/
│   ├── backup-db.ps1              — регулярный backup с таймстампом в имени + лог в backup-history.log
│   └── cleanup-tomcat-logs.ps1    — retention логов Tomcat
├── docs/
│   └── deployment.md              — этот же процесс в сжатом виде, для быстрой сверки
└── screenshots/
    └── 01-restore-success.png     — единственный скриншот: полный вывод restore + проверки
```

---

## 11. Чек-лист для проверяющего

| # | Пункт | Статус |
|---|---|---|
| 1 | Стенд поднят: Windows Server, SQL Server и Tomcat работают как службы, WAR развёрнут, UI открывается и пишет в БД | ✅ |
| 2 | Гайд по повторению с нуля | ✅ (этот файл) |
| 3 | Backup настроен, restore реально выполнен и сверен со скриншотом | ✅ |
| 4 | Retention логов Tomcat настроен и проверен | ✅ |
| 5 | Короткое видео работающего приложения | план ниже |

---

