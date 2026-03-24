# 6. Asyncio: база, конкурентность, отмена, ограничения и production-практика

Этот раздел — практический конспект по `asyncio`: от базовых сущностей до конкурентного выполнения, отмены задач, ограничения нагрузки и более production-oriented подхода.

---

## 6.1. Этап 1. База

### `async def`

`async def` объявляет **корутинную функцию**.

```python
async def hello():
    print("hello")
```

Важно: вызов такой функции **не запускает её сразу**.  
Он создаёт **coroutine object**.

```python
coro = hello()
print(coro)  # <coroutine object hello at ...>
```

### Зачем это нужно

Так описывается асинхронная логика:

- сделать что-то;
- дойти до `await`;
- уступить управление;
- потом продолжить выполнение.

### Как это работает

`async def` создаёт функцию, которая при вызове возвращает объект-корутину.  
Реальное выполнение начинается только когда ты:

- делаешь `await coro`;
- передаёшь корутину в `asyncio.run()`;
- оборачиваешь её в `create_task()` или `TaskGroup.create_task()`.

---

### Coroutine function

Это сама функция, объявленная через `async def`.

```python
async def fetch_user():
    return {"id": 1}
```

Здесь `fetch_user` — **coroutine function**.

---

### Coroutine object

Это результат вызова coroutine function.

```python
async def fetch_user():
    return {"id": 1}

coro = fetch_user()
```

Здесь `coro` — **coroutine object**.

---

### Частая ошибка

```python
async def test():
    print("run")

async def main():
    test()  # корутина создана, но не запущена

asyncio.run(main())
```

Так появится warning про **`coroutine was never awaited`**.

> Если корутину создали, её нужно либо `await`-нуть, либо передать в `create_task()`.

---

### `await`

`await` — это точка, где текущая корутина:

- ждёт результат другой awaitable-сущности;
- уступает управление `event loop`.

```python
import asyncio

async def say():
    await asyncio.sleep(1)
    return "done"

async def main():
    result = await say()
    print(result)

asyncio.run(main())
```

### Что происходит

Когда выполнение доходит до `await asyncio.sleep(1)`, корутина **не блокирует поток**.  
Она говорит loop: _«я пока жду, можешь дать поработать другим задачам»_.

---

### Awaitable

**Awaitable** — это всё, что можно написать справа от `await`.

Основные виды:

- `coroutine object`
- `Task`
- `Future`

Пример:

```python
import asyncio

async def f():
    return 42

async def main():
    coro = f()                       # coroutine object
    task = asyncio.create_task(f())  # Task

    print(await coro)
    print(await task)

asyncio.run(main())
```

---

### `asyncio.run()`

Главная точка входа в async-программу.

```python
import asyncio

async def main():
    print("start")
    await asyncio.sleep(1)
    print("finish")

asyncio.run(main())
```

### Зачем нужен `asyncio.run()`

Он нужен, чтобы:

- создать `event loop`;
- выполнить верхнеуровневую корутину;
- корректно закрыть loop после завершения.

### Когда использовать

- в обычном Python-скрипте;
- один раз на верхнем уровне приложения.

### Когда не использовать

- внутри уже работающего `event loop`;
- внутри `async`-функции.

Нельзя:

```python
async def main():
    asyncio.run(other())  # ошибка
```

---

## 6.2. Этап 2. Основная конкурентность

Главная идея:

> **Конкурентность != параллелизм**

`asyncio` обычно даёт конкурентность в одном потоке: пока одна задача ждёт I/O, loop запускает другую.

---

### `create_task()`

Создаёт `Task` и планирует корутину к выполнению «скоро».

```python
import asyncio

async def work(name, delay):
    print(f"{name}: start")
    await asyncio.sleep(delay)
    print(f"{name}: done")
    return name

async def main():
    task1 = asyncio.create_task(work("A", 2))
    task2 = asyncio.create_task(work("B", 1))

    result1 = await task1
    result2 = await task2
    print(result1, result2)

asyncio.run(main())
```

### Зачем

Когда задачу надо запустить сейчас, а дождаться результата можно позже.

### Как это работает

`Task` — это специальная обёртка над корутиной.  
После `create_task()` корутина ставится в очередь loop и начинает исполняться конкурентно с текущей корутиной.

---

### `gather()`

Запускает несколько awaitable одновременно и ждёт все результаты.

```python
import asyncio

async def work(x):
    await asyncio.sleep(x)
    return x * 10

async def main():
    results = await asyncio.gather(
        work(1),
        work(2),
        work(3),
    )
    print(results)  # [10, 20, 30]

asyncio.run(main())
```

### Зачем

Когда нужно дождаться все результаты одним вызовом.

### Особенность

Результаты идут **в том порядке, в каком переданы аргументы**, а не в порядке завершения задач.

### Практический пример

```python
users, posts, comments = await asyncio.gather(
    fetch_users(),
    fetch_posts(),
    fetch_comments(),
)
```

---

### `TaskGroup`

Современный структурированный способ работы с группой задач.  
Добавлен в Python 3.11.

Если одна задача падает с исключением, остальные в группе отменяются.

```python
import asyncio

async def work(name, delay):
    await asyncio.sleep(delay)
    print(name)
    return name

async def main():
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(work("A", 2))
        t2 = tg.create_task(work("B", 1))

    print(t1.result(), t2.result())

asyncio.run(main())
```

### Зачем

`TaskGroup` полезен, когда нужно:

- не терять фоновые задачи;
- автоматически дождаться завершения всех задач;
- безопаснее обрабатывать ошибки.

### Когда `TaskGroup` лучше, чем `gather()`

Когда важна **structured concurrency**:

- задачи логически принадлежат одному блоку;
- при сбое одной задачи остальные тоже нужно остановить;
- не хочется вручную управлять жизненным циклом задач.

---

### `sleep()`

Асинхронная пауза:

```python
await asyncio.sleep(1)
```

### Зачем

- уступить управление loop;
- смоделировать I/O;
- сделать polling/retry;
- дать другим задачам шанс выполниться.

> `asyncio.sleep()` не блокирует поток, в отличие от `time.sleep()`.

---

### `wait()`

Ждёт набор задач по определённому условию:

- пока завершатся все;
- пока завершится первая;
- пока завершится первая с исключением.

```python
import asyncio

async def work(name, delay):
    await asyncio.sleep(delay)
    return name

async def main():
    tasks = {
        asyncio.create_task(work("A", 3)),
        asyncio.create_task(work("B", 1)),
        asyncio.create_task(work("C", 2)),
    }

    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED,
    )

    for t in done:
        print("done:", t.result())

    for t in pending:
        t.cancel()

asyncio.run(main())
```

### Зачем

Когда нужен более низкоуровневый контроль, чем у `gather()`.

---

### `as_completed()`

Позволяет обрабатывать результаты по мере готовности.

```python
import asyncio

async def work(name, delay):
    await asyncio.sleep(delay)
    return name

async def main():
    tasks = [
        asyncio.create_task(work("A", 3)),
        asyncio.create_task(work("B", 1)),
        asyncio.create_task(work("C", 2)),
    ]

    for future in asyncio.as_completed(tasks):
        result = await future
        print("got:", result)

asyncio.run(main())
```

### Зачем

Когда порядок результата важен не входной, а по принципу: **кто раньше закончил**.

### Типичный use case

- запросы в несколько внешних API;
- частичная выдача результатов как можно раньше.

---

## 6.3. Этап 3. Отмена и таймауты

Асинхронный код почти всегда должен уметь:

- отменяться;
- ограничиваться по времени;
- корректно чистить ресурсы.

---

### `cancel()`

Отправляет задаче запрос на отмену.

```python
import asyncio

async def worker():
    try:
        while True:
            print("working...")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("worker cancelled")
        raise

async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(2.5)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("main: task cancelled")

asyncio.run(main())
```

### Как это работает

`task.cancel()` не «убивает» задачу мгновенно.

Он помечает её как отменяемую, и на ближайшем `await` внутрь корутины будет брошен `CancelledError`.

---

### `CancelledError`

Это исключение, через которое проходит отмена.

```python
try:
    await task
except asyncio.CancelledError:
    print("задача была отменена")
```

### Важный момент

Если ты поймал `CancelledError` внутри задачи, обычно его нужно пробросить дальше:

```python
except asyncio.CancelledError:
    cleanup()
    raise
```

Иначе можно случайно «съесть» отмену и сломать shutdown-логику.

---

### `wait_for()`

Запускает awaitable с таймаутом.

```python
import asyncio

async def slow():
    await asyncio.sleep(5)
    return "ok"

async def main():
    try:
        result = await asyncio.wait_for(slow(), timeout=2)
        print(result)
    except asyncio.TimeoutError:
        print("timeout")

asyncio.run(main())
```

### Зачем

Когда нужен жёсткий лимит времени на одну операцию.

### Что важно понимать

По истечении таймаута `wait_for()` отменяет внутреннюю задачу.  
Поэтому код внутри должен быть **cancellation-safe**.

---

### `timeout()`

Контекстный менеджер таймаута.

```python
import asyncio

async def main():
    try:
        async with asyncio.timeout(2):
            await asyncio.sleep(5)
    except TimeoutError:
        print("timed out")

asyncio.run(main())
```

### Зачем

Когда удобнее ограничить по времени **блок кода**, а не один вызов.

### Когда лучше, чем `wait_for()`

Когда внутри несколько `await`, и нужен **общий timeout** на весь участок.

---

### `shield()`

Защищает awaitable от внешней отмены.

```python
import asyncio

async def save_data():
    await asyncio.sleep(2)
    print("saved")

async def main():
    task = asyncio.create_task(save_data())

    try:
        await asyncio.wait_for(asyncio.shield(task), timeout=1)
    except asyncio.TimeoutError:
        print("main timeout, but save_data still running")

    await task

asyncio.run(main())
```

### Зачем

Когда снаружи происходит отмена или таймаут, но конкретную операцию нужно дать закончить:

- дозаписать лог;
- сохранить данные;
- аккуратно закрыть соединение.

> `shield()` не делает задачу бессмертной: если отменить саму внутреннюю задачу отдельно, она всё равно завершится отменой.

---

## 6.4. Этап 4. Ограничение нагрузки

Когда async-код слишком легко стартует тысячу задач сразу, можно:

- упереться в rate limit внешнего API;
- переполнить память;
- перегрузить БД;
- создать огромную очередь ожидания.

Поэтому нужны ограничители.

---

### `Semaphore`

Ограничивает число одновременно работающих секций.

```python
import asyncio

sem = asyncio.Semaphore(3)

async def fetch(i):
    async with sem:
        print(f"start {i}")
        await asyncio.sleep(2)
        print(f"done {i}")

async def main():
    await asyncio.gather(*(fetch(i) for i in range(10)))

asyncio.run(main())
```

### Что делает

Хотя задач 10, одновременно внутрь `async with sem` попадут только 3.

### Зачем

- ограничить число HTTP-запросов;
- ограничить число одновременных обращений к БД;
- не положить внешний сервис.

> `Semaphore` предназначен для async-кода и не является thread-safe.

---

### `Queue`

Асинхронная очередь для обмена между корутинами.

```python
import asyncio

async def producer(queue):
    for i in range(5):
        await queue.put(i)
        print("produced", i)
    await queue.put(None)  # сигнал завершения

async def consumer(queue):
    while True:
        item = await queue.get()
        try:
            if item is None:
                break
            print("consumed", item)
            await asyncio.sleep(1)
        finally:
            queue.task_done()

async def main():
    queue = asyncio.Queue()
    p = asyncio.create_task(producer(queue))
    c = asyncio.create_task(consumer(queue))

    await p
    await queue.join()
    await c

asyncio.run(main())
```

### Зачем

Когда один набор корутин производит работу, а другой — обрабатывает.

### Как работает

- `put()` кладёт элемент;
- `get()` забирает;
- `task_done()` сообщает: _«этот элемент обработан»_;
- `join()` ждёт, пока все добавленные элементы будут помечены обработанными.

---

### Producer / Consumer

Классический паттерн:

- `producer` создаёт задания;
- `consumer` выполняет их.

Пример с несколькими воркерами:

```python
import asyncio

async def producer(queue):
    for i in range(10):
        await queue.put(i)
    for _ in range(3):
        await queue.put(None)

async def consumer(name, queue):
    while True:
        item = await queue.get()
        try:
            if item is None:
                print(name, "stop")
                return
            await asyncio.sleep(1)
            print(name, "processed", item)
        finally:
            queue.task_done()

async def main():
    queue = asyncio.Queue()

    consumers = [
        asyncio.create_task(consumer(f"worker-{i}", queue))
        for i in range(3)
    ]

    await producer(queue)
    await queue.join()

    for c in consumers:
        await c

asyncio.run(main())
```

---

### Bounded queue

Это очередь с ограниченным размером:

```python
queue = asyncio.Queue(maxsize=100)
```

### Зачем

Чтобы producer не мог бесконечно быстро складывать задания в память.

Если очередь заполнена, `await queue.put(item)` начнёт ждать, пока consumer освободит место.

---

### Backpressure

**Backpressure** — это механизм, при котором медленный consumer естественно тормозит producer.

Самый простой способ сделать это в `asyncio` — использовать bounded queue.

```python
import asyncio

async def producer(queue):
    for i in range(1000):
        await queue.put(i)
        print("put", i)

async def consumer(queue):
    while True:
        item = await queue.get()
        try:
            await asyncio.sleep(0.5)
        finally:
            queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=10)
    asyncio.create_task(consumer(queue))
    await producer(queue)

asyncio.run(main())
```

### Зачем

Чтобы не было ситуации:

- входной поток даёт 100k задач;
- обработчик не успевает;
- память растёт бесконечно.

---

## 6.5. Этап 5. Блокирующий код

`asyncio` хорош, пока код не блокирует loop.

Если внутри async-функции сделать CPU-тяжёлую работу или обычный блокирующий I/O, весь `event loop` «замрёт».

---

### `to_thread()`

Удобный способ вынести блокирующую функцию в отдельный поток.

```python
import asyncio
import time

def blocking_io():
    time.sleep(2)
    return "done"

async def main():
    result = await asyncio.to_thread(blocking_io)
    print(result)

asyncio.run(main())
```

### Зачем

Если есть обычная синхронная функция, которую нельзя быстро переписать на async:

- чтение файла старой библиотекой;
- `requests`;
- работа с SDK без async API.

---

### `run_in_executor()`

Более низкоуровневый способ отправить функцию в executor.

```python
import asyncio
import time

def blocking_io():
    time.sleep(2)
    return "done"

async def main():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, blocking_io)
    print(result)

asyncio.run(main())
```

### Зачем

Когда нужен больший контроль:

- свой `ThreadPoolExecutor`;
- свой `ProcessPoolExecutor`.

---

### I/O-bound vs CPU-bound

Это одно из ключевых различий в async-программировании.

#### I/O-bound

Задача в основном ждёт:

- сеть;
- диск;
- БД;
- внешний API.

Примеры:

- HTTP-запросы;
- чтение сокета;
- ожидание ответа PostgreSQL.

Для этого async подходит отлично.

#### CPU-bound

Задача в основном считает:

- сжатие;
- парсинг больших данных;
- ML-инференс;
- image/video processing.

Плохая идея:

```python
async def cpu_heavy():
    total = 0
    for i in range(10**8):
        total += i
    return total
```

Такой код заблокирует loop.

### Что делать

- I/O-bound sync-код → `to_thread()` или thread pool;
- CPU-bound → чаще `ProcessPoolExecutor` или отдельный воркер/сервис.

---

## 6.6. Этап 6. Low-level

Обычно это нужно не приложению, а библиотекам и фреймворкам. Но понимать полезно.

---

### `get_running_loop()`

Получить текущий работающий event loop.

```python
import asyncio

async def main():
    loop = asyncio.get_running_loop()
    print(loop)

asyncio.run(main())
```

### Зачем

Когда нужен доступ к low-level возможностям loop:

- `call_soon`;
- `call_later`;
- `run_in_executor`;
- `create_future`.

---

### Lifecycle loop

Жизненный цикл loop обычно такой:

1. создать loop;
2. запустить top-level coroutine;
3. выполнять задачи, callbacks и I/O;
4. отменить хвосты при shutdown;
5. закрыть loop.

В обычном прикладном коде всё это делает `asyncio.run()`.

Если вручную:

```python
import asyncio

async def main():
    await asyncio.sleep(1)

loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)
try:
    loop.run_until_complete(main())
finally:
    loop.close()
```

---

### `call_soon()`

Запланировать обычный callback как можно скорее.

```python
import asyncio

def callback():
    print("callback called")

async def main():
    loop = asyncio.get_running_loop()
    loop.call_soon(callback)
    await asyncio.sleep(0.1)

asyncio.run(main())
```

---

### `call_later()`

Запустить callback позже.

```python
import asyncio

def callback():
    print("later callback")

async def main():
    loop = asyncio.get_running_loop()
    loop.call_later(2, callback)
    await asyncio.sleep(3)

asyncio.run(main())
```

---

### `Future`

`Future` — low-level объект, представляющий результат, который появится позже.

`Task` — по сути специальный вид `Future`, который исполняет корутину.

Пример:

```python
import asyncio

async def main():
    loop = asyncio.get_running_loop()
    fut = loop.create_future()

    loop.call_later(1, fut.set_result, "ready")

    result = await fut
    print(result)

asyncio.run(main())
```

### Зачем понимать

Если работаешь с callback-based API и хочешь завернуть его в `async/await`.

---

### `loop.create_task()`

Почти то же, что `asyncio.create_task()`, но через конкретный loop.

```python
import asyncio

async def work():
    await asyncio.sleep(1)
    return 123

async def main():
    loop = asyncio.get_running_loop()
    task = loop.create_task(work())
    print(await task)

asyncio.run(main())
```

Обычно достаточно `asyncio.create_task()`.

---

### Task factory

Task factory позволяет кастомизировать создание задач loop’ом.

Например:

- логировать создание каждой задачи;
- подменять класс task;
- добавлять tracing/context.

```python
import asyncio

def factory(loop, coro, **kwargs):
    print("creating task for", coro)
    return asyncio.Task(coro, loop=loop, **kwargs)

async def work():
    await asyncio.sleep(1)

async def main():
    loop = asyncio.get_running_loop()
    loop.set_task_factory(factory)
    task = asyncio.create_task(work())
    await task

asyncio.run(main())
```

Обычно это встречается в библиотеках, observability-инструментах и framework internals.

---

### Transports / Protocols

Это старый и более низкоуровневый стиль asyncio-сетевого программирования:

- `transport` отвечает за передачу данных;
- `protocol` — объект с callbacks вроде `connection_made`, `data_received`, `connection_lost`.

```python
import asyncio

class EchoClientProtocol(asyncio.Protocol):
    def connection_made(self, transport):
        self.transport = transport
        transport.write(b"hello")

    def data_received(self, data):
        print("received:", data.decode())
        self.transport.close()

    def connection_lost(self, exc):
        print("connection closed")

async def main():
    loop = asyncio.get_running_loop()
    await loop.create_connection(
        lambda: EchoClientProtocol(),
        "127.0.0.1",
        8888,
    )
    await asyncio.sleep(1)

asyncio.run(main())
```

### Зачем знать

Чтобы понимать internals.

Но в большинстве приложений сейчас чаще используют high-level API или сторонние библиотеки.

---

## 6.7. Этап 7. Debug и production thinking

Вот здесь async-код начинает отличаться от «игрушечных» примеров.

---

### Debug mode

`asyncio` умеет запускаться в debug mode. Это помогает видеть:

- где была создана незапущенная корутина;
- где создана задача с потерянным исключением;
- дополнительные диагностические tracebacks.

```python
import asyncio

async def main():
    await asyncio.sleep(0.1)

asyncio.run(main(), debug=True)
```

### Когда включать

- локальная отладка;
- тесты проблемных async-сценариев;
- расследование подвисающих задач.

---

### Never awaited coroutine

Ошибка: корутину создали, но не `await`-нули и не запланировали.

```python
import asyncio

async def test():
    print("run")

async def main():
    test()  # плохо

asyncio.run(main())
```

Что будет:

- warning;
- код внутри `test()` не выполнится.

Правильно:

```python
async def main():
    await test()
```

или

```python
async def main():
    task = asyncio.create_task(test())
    await task
```

---

### Never retrieved exception

Ошибка: задача упала, но никто не забрал её исключение.

```python
import asyncio

async def bug():
    raise RuntimeError("boom")

async def main():
    asyncio.create_task(bug())
    await asyncio.sleep(0.1)

asyncio.run(main())
```

В итоге будет сообщение вроде:

```text
Task exception was never retrieved
```

### Почему так происходит

Ты создал фоновую задачу, но:

- не сделал `await task`;
- не вызвал `task.result()`;
- не обработал исключение.

Правильно:

```python
async def main():
    task = asyncio.create_task(bug())
    try:
        await task
    except RuntimeError as e:
        print("handled:", e)
```

---

### Graceful shutdown

Корректное завершение async-приложения обычно выглядит так:

1. перестать принимать новую работу;
2. отменить фоновые задачи;
3. дождаться их завершения;
4. закрыть ресурсы: сокеты, БД, producer’ы, consumers.

Пример:

```python
import asyncio

async def worker():
    try:
        while True:
            print("tick")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("worker cleanup")
        await asyncio.sleep(0.2)
        raise

async def main():
    task = asyncio.create_task(worker())

    try:
        await asyncio.sleep(3)
    finally:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            print("shutdown complete")

asyncio.run(main())
```

### Production idea

Shutdown — это не «просто убить всё».

Нужно дать задачам шанс:

- завершить текущую транзакцию;
- освободить lock/semaphore;
- отправить финальные данные;
- закрыть соединения.

---

### Cancellation-safe cleanup

Это умение писать `finally` и `except CancelledError` так, чтобы cleanup не ломался при отмене.

Базовый шаблон:

```python
import asyncio

async def worker():
    resource = "opened"
    try:
        while True:
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("got cancel")
        raise
    finally:
        print("cleanup resource:", resource)
```

Если cleanup сам по себе содержит `await`, он тоже может быть прерван отменой. Тогда иногда нужен `shield()`:

```python
import asyncio

async def flush():
    await asyncio.sleep(1)
    print("flushed")

async def worker():
    try:
        while True:
            await asyncio.sleep(1)
    finally:
        await asyncio.shield(flush())
```

### Зачем это нужно

Чтобы при отмене не потерять:

- финальную запись на диск;
- `ack` в очередь;
- `commit/rollback`;
- закрытие сетевого соединения.

---

## Как всё это складывается в общую модель

Упрощённо `asyncio` работает так:

1. Ты запускаешь `asyncio.run(main())`.
2. Создаётся `event loop`.
3. Loop начинает исполнять `main()`.
4. Когда встречается `await`, корутина может приостановиться.
5. Пока она ждёт, loop даёт поработать другим задачам.
6. `Task` — это запланированная корутина.
7. `Future` — обещание будущего результата.
8. `Semaphore` и `Queue` помогают не перегрузить систему.
9. Отмена проходит через `CancelledError`.
10. Блокирующий код надо уносить в thread/process executor.

---

## Практическая шпаргалка: что использовать и когда

| Задача | Что использовать |
|---|---|
| Просто запустить async-код | `asyncio.run(main())` |
| Параллельно ждать несколько операций | `await asyncio.gather(a(), b(), c())` |
| Запустить задачу сейчас, дождаться позже | `task = asyncio.create_task(work())` |
| Структурированно запустить группу задач | `async with asyncio.TaskGroup() as tg:` |
| Ограничить одновременную нагрузку | `asyncio.Semaphore(10)` |
| Сделать producer/consumer очередь | `asyncio.Queue(maxsize=100)` |
| Ограничить время выполнения | `asyncio.wait_for(...)` или `asyncio.timeout(...)` |
| Не блокировать event loop синхронной функцией | `await asyncio.to_thread(blocking_func)` |
| Получить low-level loop | `asyncio.get_running_loop()` |

---

## Мини-пример: почти production

Здесь вместе используются:

- очередь;
- producer/consumer;
- ограничение нагрузки;
- timeout;
- graceful shutdown.

```python
import asyncio
import random

async def fetch(item, sem):
    async with sem:
        await asyncio.sleep(random.uniform(0.2, 1.0))
        return f"processed-{item}"

async def producer(queue):
    for i in range(20):
        await queue.put(i)
    for _ in range(3):
        await queue.put(None)

async def consumer(name, queue, sem):
    try:
        while True:
            item = await queue.get()
            try:
                if item is None:
                    return

                try:
                    result = await asyncio.wait_for(fetch(item, sem), timeout=2)
                    print(name, result)
                except asyncio.TimeoutError:
                    print(name, "timeout on", item)
            finally:
                queue.task_done()
    except asyncio.CancelledError:
        print(name, "cancelled")
        raise

async def main():
    queue = asyncio.Queue(maxsize=5)
    sem = asyncio.Semaphore(3)

    consumers = [
        asyncio.create_task(consumer(f"worker-{i}", queue, sem))
        for i in range(3)
    ]

    try:
        await producer(queue)
        await queue.join()
    finally:
        for c in consumers:
            c.cancel()

        await asyncio.gather(*consumers, return_exceptions=True)

asyncio.run(main())
```

### Что здесь происходит

- `producer` кладёт задачи в bounded queue;
- `consumer` забирают задачи;
- `Semaphore(3)` ограничивает число одновременных `fetch`;
- `wait_for(..., timeout=2)` защищает от зависания операции;
- в `finally` идёт отмена consumers и аккуратный shutdown.

---

## Главное, что стоит запомнить

- `async def` создаёт coroutine function, а её вызов — coroutine object;
- без `await` или `create_task()` корутина не выполнится;
- `await` не блокирует поток, а уступает управление loop;
- `create_task()` нужен для конкурентного запуска;
- `gather()` — чтобы дождаться все результаты;
- `TaskGroup` — более безопасный способ запуска группы задач;
- отмена — это нормальный механизм управления, а не ошибка;
- любой async-код должен быть готов к `CancelledError`;
- `Semaphore` и bounded `Queue` защищают от перегрузки;
- блокирующий sync-код нельзя просто так вызывать внутри async — его нужно выносить в `to_thread()` или executor.
