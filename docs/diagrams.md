# Диаграммы

Все диаграммы записаны на языке Mermaid и отрисовываются GitHub автоматически —
исходный текст остаётся в репозитории и правится вместе с кодом.
Схема базы данных — в [database.md](database.md#1-схема).

---

## 1. Диаграмма компонентов

```mermaid
graph TB
    subgraph CL["Клиент"]
        CM["main.py<br/>цикл ввода, пароль, подтверждения"]
        CD["commands.py<br/>строка → запрос"]
        CR["render.py<br/>ответ → текст"]
        CC["connection.py<br/>ServerConnection"]
    end

    subgraph CO["common"]
        FR["framing.py<br/>FrameReader"]
    end

    subgraph SV["Сервер"]
        TS["tcp_server.py<br/>TCPServer"]
        CN["connection.py<br/>обмен с клиентом"]
        PR["protocol.py<br/>Request, ErrorCode"]
        DP["dispatcher.py<br/>доступ, аудит"]
        HD["handlers.py<br/>команды"]
        AP["app.py<br/>Services"]
    end

    subgraph FS["server/fs"]
        SB["sandbox.py"]
        MG["manager.py"]
        LK["locks.py"]
        FM["formats.py"]
        AR["archive.py"]
        DR["drives.py"]
    end

    subgraph DB["server/db"]
        RP["repository.py"]
        SC[("schema.sql<br/>SQLite")]
    end

    subgraph AU["server/auth"]
        PW["passwords.py<br/>scrypt"]
        AS["service.py"]
    end

    CM --> CD
    CM --> CR
    CM --> CC
    CC --> FR
    CC -. "TCP :9000" .-> CN

    TS --> CN
    TS --> AP
    CN --> FR
    CN --> PR
    CN --> DP
    DP --> HD
    DP --> RP
    HD --> MG
    HD --> FM
    HD --> AR
    HD --> DR
    HD --> RP
    HD --> AS
    MG --> SB
    MG --> LK
    AR --> MG
    AS --> PW
    AS --> RP
    RP --> SC
```

Пакеты `fs`, `db`, `auth` не имеют стрелок в сторону сети и протокола —
поэтому тестируются без сокетов.

---

## 2. Диаграмма классов: сервер

```mermaid
classDiagram
    class Dispatcher {
        -dict _commands
        -AuditSink _audit
        +register(name, handler, public, audited)
        +dispatch(request, session)
    }
    class Session {
        +str peer
        +AuthenticatedUser user
        +int failed_logins
        +bool should_close
        +is_authenticated() bool
    }
    class Services {
        +ServerConfig config
        +Database db
        +AuthService auth
        +FileManager files
        +FileRegistry registry
        +AuditLog audit
        +ZipLimits zip_limits
        +build(config)$ Services
    }
    class Handlers {
        +login() register() logout()
        +ls() info() read() write() delete()
        +mkdir() rmdir()
        +json_read() json_write() xml_read() xml_write()
        +zip() zip_list() unzip()
        +drives() log() ops()
    }
    class FileManager {
        +PathLocks locks
        +for_user(user_id) UserFiles
    }
    class UserFiles {
        +Sandbox sandbox
        +list_dir(path) info(path)
        +read_text(path) write_text(path, content, overwrite)
        +delete_file(path) make_dir(path) remove_dir(path)
    }
    class Sandbox {
        -Path _root
        +resolve(raw) Path
        +relative(path) str
    }
    class PathLocks {
        +hold(paths) contextmanager
    }
    class AuthService {
        +register(username, password)
        +login(username, password)
    }
    class Database {
        +initialize()
        +connect()
        +transaction()
    }
    class UserRepository
    class FileRegistry
    class AuditLog

    Dispatcher --> Handlers : вызывает
    Dispatcher --> AuditLog : пишет
    Dispatcher ..> Session : проверяет вход
    Handlers --> Services
    Services --> FileManager
    Services --> AuthService
    Services --> FileRegistry
    FileManager ..> UserFiles : создаёт
    FileManager --> PathLocks
    UserFiles --> Sandbox
    AuthService --> UserRepository
    UserRepository --> Database
    FileRegistry --> Database
    AuditLog --> Database
```

---

## 3. Последовательность: запись файла

```mermaid
sequenceDiagram
    actor U as Пользователь
    participant C as Клиент
    participant N as connection
    participant D as Dispatcher
    participant H as Handlers.write
    participant F as UserFiles
    participant DB as SQLite

    U->>C: write docs/a.txt + текст
    C->>N: {"cmd":"write","path":"docs/a.txt","content":"..."}
    N->>D: dispatch(request, session)
    D->>D: session.user есть?
    D->>H: write
    H->>F: write_text(path, content, overwrite)
    F->>F: sandbox.resolve → storage/user-7/docs/a.txt
    F->>F: блокировка пути
    F->>F: временный файл, fsync, os.replace
    F-->>H: WrittenFile(created=true)
    H->>DB: INSERT INTO files
    DB->>DB: триггер → INSERT INTO operations ('create')
    H-->>D: {"path","size","created"}
    D->>DB: INSERT INTO audit_log (успех)
    D-->>N: результат
    N-->>C: {"status":"ok",...}
    C-->>U: Создан /docs/a.txt
```

---

## 4. Последовательность: отражённая атака

Попытка выйти из каталога не доходит до файловой системы, но оставляет след.

```mermaid
sequenceDiagram
    actor A as Нарушитель
    participant N as connection
    participant D as Dispatcher
    participant H as Handlers.read
    participant S as Sandbox
    participant DB as SQLite

    A->>N: {"cmd":"read","path":"../../etc/passwd"}
    N->>D: dispatch
    D->>H: read
    H->>S: resolve("../../etc/passwd")
    S--xH: BadPathError
    H--xD: ProtocolError BAD_PATH
    D->>DB: INSERT INTO audit_log (success=0, details=BAD_PATH)
    D--xN: ProtocolError
    N-->>A: {"status":"error","code":"BAD_PATH",...}
    Note over N: соединение остаётся открытым
```

---

## 5. Состояния сессии

```mermaid
stateDiagram-v2
    state "Без входа" as anon
    state "Вошёл" as auth
    state "Закрыто" as closed

    [*] --> anon : подключение
    anon --> auth : login / register успешно
    anon --> anon : неудачный login (счётчик +1)
    anon --> closed : 5 неудачных login
    auth --> anon : logout
    auth --> auth : файловые команды
    anon --> closed : quit, разрыв, таймаут
    auth --> closed : quit, разрыв, таймаут
    closed --> [*]
```

В состоянии «Без входа» доступны только `register`, `login`, `ping`, `quit`;
остальное получает `AUTH_REQUIRED`.

---

## 6. Деятельность: распаковка архива

```mermaid
flowchart TD
    A["unzip архив каталог"] --> B{"Каталог<br/>уже есть?"}
    B -- да --> X1["ALREADY_EXISTS"]
    B -- нет --> C{"Записей больше<br/>предела?"}
    C -- да --> X2["UNSAFE_ARCHIVE"]
    C -- нет --> D["Для каждой записи"]
    D --> E{"Зашифрована<br/>или ссылка?"}
    E -- да --> X2
    E -- нет --> F{"Имя проходит<br/>правила пути?"}
    F -- нет --> X3["UNSAFE_ARCHIVE<br/>Zip Slip"]
    F -- да --> G{"Дубликат<br/>без учёта регистра?"}
    G -- да --> X2
    G -- нет --> H{"Сумма размеров и<br/>степень сжатия в пределах?"}
    H -- нет --> X4["UNSAFE_ARCHIVE<br/>zip-бомба"]
    H -- да --> D
    D -- "все проверены" --> I["Распаковка во временный каталог<br/>со счётом байт"]
    I --> J{"Байт больше<br/>объявленного?"}
    J -- да --> X5["UNSAFE_ARCHIVE<br/>временный каталог удалён"]
    J -- нет --> K["Переименование в целевой каталог"]
    K --> L["Реестр: все файлы одной транзакцией"]
```

Все проверки по заголовкам — до записи первого байта; счёт байт — на случай,
если заголовки лгут.

---

## 7. Потоки сервера

```mermaid
graph TB
    M["Главный поток<br/>accept()"]
    L{"Предел<br/>подключений?"}
    R["SERVER_BUSY"]
    T1["Поток клиента 1<br/>Session, FrameReader"]
    T2["Поток клиента 2<br/>Session, FrameReader"]
    S["Счётчик соединений<br/>Lock"]
    PL["PathLocks<br/>замок на путь"]
    DB[("SQLite<br/>BEGIN IMMEDIATE, WAL")]

    M --> L
    L -- да --> R
    L -- нет --> T1
    L -- нет --> T2
    M -.-> S
    T1 -.-> S
    T2 -.-> S
    T1 -.-> PL
    T2 -.-> PL
    T1 -.-> DB
    T2 -.-> DB
```

---

## 8. Развёртывание

```mermaid
graph LR
    subgraph H["Машина сервера"]
        SRV["python -m server.main<br/>:9000"]
        ST[("storage/<br/>user-1, user-2 …")]
        DBF[("data/filemanager.db")]
        SRV --> ST
        SRV --> DBF
    end
    CL1["python -m client.main"] -- TCP --> SRV
    CL2["python -m client.main"] -- TCP --> SRV
```

Штатно — всё на одной машине через `127.0.0.1`. Шифрования нет, поэтому по
сети — только в доверенной.
