# Семафоры: подробное объяснение

## Что такое семафор?

**Семафор** - это механизм синхронизации, используемый в многопоточном и многопроцессном программировании для управления доступом к общим ресурсам.

### Простая аналогия
Представьте себе **столовую с ограниченным количеством мест**:
- **Столовая** - общий ресурс
- **Количество мест** - значение семафора
- **Посетители** - потоки/процессы
- **Занять место** - захватить семафор (`acquire`)
- **Освободить место** - освободить семафор (`release`)

## Основные концепции

### Счетчик семафора
Семафор содержит внутренний счетчик, который:
- Уменьшается при захвате (`acquire`)
- Увеличивается при освобождении (`release`)
- Не может быть отрицательным (для обычных семафоров)

### Типы семафоров

#### 1. Бинарный семафор (Mutex)
- Значение: 0 или 1
- Используется для взаимоисключений (mutual exclusion)
- Только один поток может захватить ресурс

#### 2. Счетный семафор
- Значение: любое неотрицательное целое число
- Ограничивает количество потоков, которые могут одновременно получить доступ к ресурсу

## Реализация семафоров в Python

### Использование `threading.Semaphore`

```python
import threading
import time
import random

# Создание семафора с начальным значением 3
semaphore = threading.Semaphore(3)

def worker(worker_id):
    """Функция рабочего потока"""
    print(f"Работник {worker_id} ждет доступа...")
    
    # Захват семафора
    semaphore.acquire()
    
    try:
        print(f"Работник {worker_id} получил доступ!")
        # Имитация работы
        time.sleep(random.uniform(1, 3))
        print(f"Работник {worker_id} завершил работу.")
    finally:
        # Освобождение семафора
        semaphore.release()
        print(f"Работник {worker_id} освободил доступ.")

# Создание и запуск потоков
threads = []
for i in range(10):
    t = threading.Thread(target=worker, args=(i,))
    threads.append(t)
    t.start()

# Ожидание завершения всех потоков
for t in threads:
    t.join()

print("Все работы завершены!")
```

**Вывод (пример):**
```
Работник 0 ждет доступа...
Работник 0 получил доступ!
Работник 1 ждет доступа...
Работник 1 получил доступ!
Работник 2 ждет доступа...
Работник 2 получил доступ!
Работник 3 ждет доступа...
Работник 4 ждет доступа...
Работник 5 ждет доступа...
Работник 0 завершил работу.
Работник 0 освободил доступ.
Работник 3 получил доступ!
...
```

## Бинарный семафор (Mutex)

```python
import threading

# Бинарный семафор (значение 1)
mutex = threading.Semaphore(1)

shared_counter = 0

def increment_counter(thread_id):
    global shared_counter
    
    for _ in range(5):
        # Захват мьютекса
        mutex.acquire()
        
        try:
            # Критическая секция
            current = shared_counter
            print(f"Поток {thread_id}: читает значение {current}")
            time.sleep(0.1)  # Имитация работы
            shared_counter = current + 1
            print(f"Поток {thread_id}: записывает значение {shared_counter}")
        finally:
            # Освобождение мьютекса
            mutex.release()
        
        time.sleep(0.01)

# Запуск потоков
threads = []
for i in range(3):
    t = threading.Thread(target=increment_counter, args=(i,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

print(f"Финальное значение счетчика: {shared_counter}")
```

## Семафоры с таймаутом

```python
import threading
import time

semaphore = threading.Semaphore(2)

def worker_with_timeout(worker_id):
    print(f"Работник {worker_id} пытается получить доступ...")
    
    # Попытка захвата семафора с таймаутом 2 секунды
    acquired = semaphore.acquire(timeout=2)
    
    if acquired:
        try:
            print(f"Работник {worker_id} получил доступ!")
            time.sleep(3)  # Долгая работа
            print(f"Работник {worker_id} завершил работу.")
        finally:
            semaphore.release()
    else:
        print(f"Работник {worker_id} не дождался доступа и ушел!")

# Запуск потоков
threads = []
for i in range(5):
    t = threading.Thread(target=worker_with_timeout, args=(i,))
    threads.append(t)
    t.start()
    time.sleep(0.5)  # Задержка между запусками

for t in threads:
    t.join()
```

## Практические примеры использования

### Пример 1: Ограничение подключений к базе данных

```python
import threading
import time
import random

class DatabaseConnectionPool:
    def __init__(self, max_connections=5):
        self.semaphore = threading.Semaphore(max_connections)
        self.active_connections = 0
        self.lock = threading.Lock()
    
    def get_connection(self, thread_id):
        print(f"Поток {thread_id} запрашивает подключение...")
        
        self.semaphore.acquire()
        
        with self.lock:
            self.active_connections += 1
            print(f"Поток {thread_id} получил подключение. Активных подключений: {self.active_connections}")
        
        # Имитация работы с БД
        time.sleep(random.uniform(1, 2))
        
        self.release_connection(thread_id)
    
    def release_connection(self, thread_id):
        with self.lock:
            self.active_connections -= 1
            print(f"Поток {thread_id} освободил подключение. Активных подключений: {self.active_connections}")
        
        self.semaphore.release()

# Использование
db_pool = DatabaseConnectionPool(3)

def database_user(thread_id):
    db_pool.get_connection(thread_id)

threads = []
for i in range(10):
    t = threading.Thread(target=database_user, args=(i,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

### Пример 2: Ограничение скорости запросов (Rate Limiting)

```python
import threading
import time

class RateLimiter:
    def __init__(self, requests_per_second):
        self.semaphore = threading.Semaphore(requests_per_second)
        self.timer = None
        self.requests_per_second = requests_per_second
    
    def make_request(self, request_id):
        # Захват семафора
        acquired = self.semaphore.acquire(blocking=False)
        
        if acquired:
            print(f"Запрос {request_id}: ВЫПОЛНЕН")
            # Освобождаем семафор через 1 секунду
            threading.Timer(1.0, self.semaphore.release).start()
        else:
            print(f"Запрос {request_id}: ОТКЛОНЕН (превышен лимит)")
        
        return acquired

# Использование: максимум 3 запроса в секунду
limiter = RateLimiter(3)

def make_requests():
    for i in range(10):
        limiter.make_request(i)
        time.sleep(0.2)  # 5 запросов в секунду

make_requests()
```

## Семафоры vs Мьютексы

### Сравнительная таблица:

| Характеристика | Семафор | Мьютекс |
|----------------|---------|---------|
| **Значение** | Любое неотрицательное число | 0 или 1 |
| **Владелец** | Нет понятия владельца | Есть владелец |
| **Освобождение** | Любой поток может освободить | Только владелец может освободить |
| **Использование** | Ограничение доступа к ресурсу | Взаимоисключение |
| **Пример** | Ограничение подключений к БД | Защита общей переменной |

## Продвинутые концепции

### Ограниченный семафор (BoundedSemaphore)

```python
import threading

# Ограниченный семафор - не может превысить начальное значение
bounded_sem = threading.BoundedSemaphore(3)

def worker_bounded(worker_id):
    bounded_sem.acquire()
    try:
        print(f"Работник {worker_id} работает...")
        time.sleep(1)
    finally:
        bounded_sem.release()
        # Если вызвать release() еще раз - будет ValueError

# Правильное использование
bounded_sem.acquire()
# ... работа ...
bounded_sem.release()

# НЕПРАВИЛЬНО (вызовет ValueError):
# bounded_sem.release()  # без предварительного acquire
```

### Семафоры в многопроцессном программировании

```python
import multiprocessing
import time

def process_worker(semaphore, process_id):
    """Рабочая функция для процесса"""
    print(f"Процесс {process_id} ждет...")
    
    with semaphore:  # Контекстный менеджер автоматически acquire/release
        print(f"Процесс {process_id} начал работу")
        time.sleep(2)
        print(f"Процесс {process_id} завершил работу")

if __name__ == "__main__":
    # Семафор для процессов (максимум 2 одновременно)
    semaphore = multiprocessing.Semaphore(2)
    
    processes = []
    for i in range(6):
        p = multiprocessing.Process(
            target=process_worker, 
            args=(semaphore, i)
        )
        processes.append(p)
        p.start()
    
    for p in processes:
        p.join()
```

## Распространенные ошибки и лучшие практики

### 1. Всегда используйте try-finally
```python
# ПРАВИЛЬНО:
semaphore.acquire()
try:
    # работа с защищенным ресурсом
    do_critical_work()
finally:
    semaphore.release()  # гарантированное освобождение

# РИСКОВАННО:
semaphore.acquire()
do_critical_work()
semaphore.release()  # если произойдет исключение - семафор не освободится
```

### 2. Используйте контекстный менеджер
```python
# Автоматическое управление семафором
with semaphore:
    do_critical_work()
# Семафор автоматически освобождается при выходе из блока
```

### 3. Избегайте взаимных блокировок (deadlock)
```python
# ОПАСНО - возможен deadlock
sem1 = threading.Semaphore(1)
sem2 = threading.Semaphore(1)

def thread1():
    sem1.acquire()
    sem2.acquire()  # может заблокироваться здесь
    # ...
    sem2.release()
    sem1.release()

def thread2():
    sem2.acquire()
    sem1.acquire()  # может заблокироваться здесь
    # ...
    sem1.release()
    sem2.release()
```

## Реальные сценарии использования

### 1. Веб-скрапинг с ограничением запросов
```python
import threading
import requests
import time

class Scraper:
    def __init__(self, max_concurrent=3, delay=1):
        self.semaphore = threading.Semaphore(max_concurrent)
        self.delay = delay
    
    def scrape_url(self, url, scraper_id):
        self.semaphore.acquire()
        
        try:
            print(f"Скрапер {scraper_id} обрабатывает {url}")
            response = requests.get(url, timeout=10)
            time.sleep(self.delay)  # Уважаем сервер
            print(f"Скрапер {scraper_id} завершил {url} - статус: {response.status_code}")
            return response.text
        finally:
            self.semaphore.release()

# Использование
scraper = Scraper(max_concurrent=2)
urls = ["https://httpbin.org/delay/1"] * 10

threads = []
for i, url in enumerate(urls):
    t = threading.Thread(target=scraper.scrape_url, args=(url, i))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

## Заключение

**Семафоры** - это мощный инструмент для:
- ✅ Ограничения одновременного доступа к ресурсам
- ✅ Реализации паттерна "пул ресурсов"
- ✅ Контроля скорости выполнения операций
- ✅ Синхронизации потоков/процессов

**Ключевые принципы:**
1. Семафоры управляют доступом, а не владением
2. Всегда освобождайте семафоры в блоке `finally`
3. Используйте контекстные менеджеры для безопасности
4. Выбирайте подходящий тип семафора для задачи

Семафоры являются фундаментальным building block для создания многопоточных и многопроцессных приложений!