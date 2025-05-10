# Лабораторная работа №3
Задание: отследить потерю данных в кластере БД при отключении primary сервера.

Этапы:
* Имитировать равномерную нагрузку на БД
* Реалирзовать reverse proxy для автоматического `promote` standby сервера
* Построить графики нагруженности `primary` и `standby` серверов

## Настраиваем окружение
Генерировать нагрузку и отслеживать количество записей в БД буду при помощи [jupyter notebook](./lab3.ipynb)
```shell
cd lab3
python3 -m venv venv
source venv/bin/activate
```
```python
pip install psycopg2-binary
pip install matplotlib
pip install jupyter lab
jupyter lab
```

## Python скрипт
состоит из нескольких логических частей:
1) `Logger` - собирает информацию о количестве записей в БД и строит график
2) `Proxy` - записывает переданные данные в `primary`, а при потере соединения перенаправляет трафик на `standby`
3) Основной цикл - передаёт данные в `Proxy` для записи в БД и вызывает `Logger`

### Logger
```python
# Логгер количества записей
class Logger:
    def __init__(self):
        self.timestamps = []
        self.primary_record_count = []
        self.standby_record_count = []

    # получить количество записей в БД
    def get_db_record_count(self, db_config):
        try:
            with psycopg2.connect(**db_config) as conn:
                with conn.cursor() as cur:
                    cur.execute("SELECT COUNT(*) FROM data_table")
                    count = cur.fetchone()[0]
            return count
        except Exception as e:
            return -1
    
    # Сохранить информацию о количестве записей
    def log(self):
        self.timestamps.append(time.time())
        self.primary_record_count.append(self.get_db_record_count(PRIMARY_DB))
        self.standby_record_count.append(self.get_db_record_count(STANDBY_DB))

    # Построить график количества записей от времени
    def plot(self):
        times = [t - self.timestamps[0] for t in self.timestamps]
        print(f'количество записей в primary {max(self.primary_record_count)}/{len(times)}')
        print(f'количество записей в standby {max(self.standby_record_count)}/{len(times)}')
        plt.plot(times, self.primary_record_count, label='primary', marker='o')
        plt.plot(times, self.standby_record_count, label='standby')
        plt.legend()
        plt.xlabel("Время")
        plt.ylabel("Количество записей в БД") 
        plt.show()
```

### Proxy
```python
# reverse proxy для переключения с PRIMARY_DB на STANDBY_DB
class Proxy:
    def __init__(self):
        self.is_primary_alive = True
        self.primary_record_count = []
        self.standby_record_count = []

    # Выполнить promote в БД
    def promote(self, db_config):
        with psycopg2.connect(**db_config) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT pg_promote()")
                conn.commit()

    # Поместить данные в БД
    def raw_insert(self, db_config, value):
        try:
            with psycopg2.connect(**db_config) as conn:
                with conn.cursor() as cur:
                    cur.execute("INSERT INTO data_table (name) VALUES (%s)", (value,))
                    conn.commit()
            return True
        except Exception as e:
            print(f"Ошибка при вставке данных: {e}")  # Логируем ошибку
            return False
        
    # Поместить данные в "живую" БД
    def insert(self, value):
        if self.is_primary_alive:
            if not self.raw_insert(PRIMARY_DB, value):
                self.is_primary_alive = False
                self.promote(STANDBY_DB)       
        if not self.is_primary_alive:
            self.raw_insert(STANDBY_DB, value)
```

### Вспомогательные функции
```python
# Получение данных которые наобходимо записать в БД
def get_value() -> str:
    return "test string"

# Создать таблицу с тестовыми данными
def init_table(db_config):
    try:
        with psycopg2.connect(**db_config) as conn:
            with conn.cursor() as cur:
                cur.execute("CREATE TABLE data_table (id SERIAL, name TEXT);")
    except Exception as e:
        with psycopg2.connect(**db_config) as conn:
            with conn.cursor() as cur:
                cur.execute("DELETE FROM data_table;")
```

### Основной цикл
```python
proxy = Proxy()
logger = Logger()
init_table(PRIMARY_DB)

for _ in range(1000):
    proxy.insert(get_value())
    logger.log()
    
logger.plot()
```
## План
1) Поднимаем кластер БД
```
cd lab2
docker compose up
```
2) Запускаем [python скрипт](./lab3.ipynb)
3) Во время работы скрипта убиваем primary командой 
```shell
docker stop my_primary
```
4) Ожидаем завершения работы программы и построения графика
5) Если требуется повторить процесс, перезапускам контейнеры
```shell
docker start my_primary
docker restart my_standby
```

## Итоги
При автоматизированном `promote` сложно потерять данные т.к. новый запрос сразу попадает в `standby`. Однако это теоретически возможно т.к. `standby` может содержать не все данные из `primary`

![alt text](img/image.png)

Но предположим `promote` не автоматизирован. Тогда `standby` не позволит вносить новые данные из-за `read-only` политики. Соответственно пока админ не выполнит `promote` работа с БД будет ограничена только чтением. В данном случае это привело к потере 370 пишущих запросов 

![alt text](img/image2.png)