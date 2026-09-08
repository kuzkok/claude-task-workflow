# Сигналы краша и насыщения по рантаймам

Справочник к скиллу `investigating-crash-logs`. Открывай раздел того рантайма, чьи логи разбираешь. Если рантайм неизвестен — определи его по формату стектрейса и именам потоков в хвосте лога.

## Терминальные сигналы

| Рантайм | Исчерпание памяти | Необработанное исключение дошло до верха | Смерть потока / воркера |
|---------|-------------------|------------------------------------------|--------------------------|
| JVM (Java, Kotlin, Scala) | `java.lang.OutOfMemoryError: Java heap space`, `GC overhead limit exceeded`, `unable to create new native thread` | `Exception in thread "main"`, срабатывание `UncaughtExceptionHandler` | `Thread ... terminated`, `Exception ... dispatching signal SIGTERM to handler` (OOM настолько тяжёлый, что shutdown-хендлер не смог выделить память) |
| .NET (C#) | `System.OutOfMemoryException`, `Insufficient memory to continue` | `Unhandled exception. System.*` с последующим завершением, `Stack overflow.` | `ThreadPool starvation`, зависшие `System.Threading.Tasks` |
| Python | `MemoryError`, `Killed` от ядра без трейса | `Traceback (most recent call last)` в самом конце файла | gunicorn `[ERROR] Worker (pid:N) was sent SIGKILL! Perhaps out of memory?`, `WORKER TIMEOUT`, `celery ... WorkerLostError` |
| Node.js | `FATAL ERROR: ... JavaScript heap out of memory`, `Allocation failed - JavaScript heap out of memory` | `Uncaught Error`, `UnhandledPromiseRejection` с завершением процесса | смерть воркера `cluster`, `worker exited with code` |
| Go | `fatal error: out of memory`, `runtime: out of memory` | `panic:` со следующим за ним `goroutine N [running]` | `fatal error: all goroutines are asleep - deadlock!` |
| Нативный код (C/C++, расширения) | `std::bad_alloc` | `terminate called after throwing an instance of` | `Segmentation fault`, `core dumped`, `double free or corruption` |

Общие для всех рантаймов, приходят от ядра или оркестратора: `SIGKILL`, `Exit code: 137` (OOMKilled), `Exit code: 143` (SIGTERM), `OOMKilled` в описании пода, `Evicted`.

## Имена потоков и воркеров как свидетельство насыщения

Принцип один для всех рантаймов: если терминальная ошибка приходит в момент, когда десятки воркеров одновременно держат глубокие стеки, проблема в отсутствии admission control, а не в размере памяти. Ограничивай конкурентность, а не масштабируй реплики.

### JVM

- `http-nio-<port>-exec-<N>` — воркер Tomcat. `N` близко к `server.tomcat.threads.max` = пул насыщен.
- `ForkJoinPool.commonPool-worker-<N>`, `ForkJoinPool-<id>-worker-<N>` — общий асинхронный пул. Блокирующий вызов внутри commonPool — антипаттерн.
- `Catalina-utility-*` — обслуживание Tomcat; смерть = катастрофическое состояние.
- `pool-<N>-thread-<M>` — кастомный `Executors.newFixedThreadPool`; грепни проект на определение пула.
- Счётчик ожидания: `LimitLatch`; хвост `ClientAbortException` = клиенты отваливаются раньше, чем сервер отвечает.
- Дамп потоков: `jstack <pid>`; принудительный OOM для проверки: `-Xmx128m` плюс `-XX:+HeapDumpOnOutOfMemoryError`.

### .NET

- `.NET ThreadPool Worker` — воркер пула. Много таких в стеке одновременно = ThreadPool starvation, классика при `.Result` / `.Wait()` на async-коде.
- `.NET Long Running Task`, `.NET TP Gate` — служебные; проверь `ThreadPool.GetAvailableThreads`.
- Kestrel: очередь запросов растёт, в логах `Heartbeat took longer than` = поток heartbeat не получает процессор.
- Дамп потоков: `dotnet-dump collect` плюс `dotnet-dump analyze` с командой `clrthreads`; лимит памяти для проверки: переменная `DOTNET_GCHeapHardLimit`.

### Python

- `ThreadPoolExecutor-<N>_<M>` — воркер пула потоков.
- gunicorn: `[CRITICAL] WORKER TIMEOUT (pid:N)` = воркер не отдал ответ за `--timeout`; синхронные воркеры под нагрузкой съедаются медленным IO.
- uvicorn/asyncio: событийный цикл один, `Task was destroyed but it is pending!` и растущая задержка = цикл блокируют синхронным вызовом.
- celery: `WorkerLostError`, `Received SIGKILL`, префетч задач держит память.
- Дамп: `faulthandler.dump_traceback_later`, `py-spy dump --pid <pid>`; лимит памяти для проверки: `resource.setrlimit(RLIMIT_AS, ...)` или лимит контейнера.

### Node.js

- Событийный цикл один, поэтому насыщение видно не по именам потоков, а по задержке цикла и росту очереди.
- `Allocation failed` вместе с растущим RSS = утечка или неограниченный буфер, а не конкурентность.
- Пул libuv (`UV_THREADPOOL_SIZE`, по умолчанию 4) — узкое место для fs и crypto.
- Дамп: `node --heapsnapshot-signal=SIGUSR2`; лимит памяти для проверки: `--max-old-space-size=128`.

### Go

- `goroutine N [running]` в панике; полный дамп горутин — по `SIGQUIT`.
- Тысячи горутин в состоянии `[chan send]` или `[select]` = нет ограничения на порождение горутин.
- Дамп: `SIGQUIT` или `runtime/pprof`; лимит памяти для проверки: `GOMEMLIMIT`.
