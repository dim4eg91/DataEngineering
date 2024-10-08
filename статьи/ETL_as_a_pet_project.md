# ETL-проект для начинающих Data Engineers: От почтового сервера до Greenplum

![онлайн покупки](images/4.gif)

В этой статье я хочу поделиться своим пет-проектом, который посвящен созданию ETL-процесса — это один из ключевых компонентов в работе любого Data Engineer. В моем случае проект направлен на извлечение данных из электронной почты, их преобразование и последующую загрузку в базу данных Greenplum для дальнейшего анализа и визуализации.

Идея создания этого проекта возникла у меня из реальной необходимости: я хотел лучше контролировать свои расходы в крупных продуктовых сетях, особенно в таких магазинах, как "ВкусВилл". Основная задача заключалась в том, чтобы систематизировать свои покупки по категориям товаров, анализировать эти данные и видеть динамику расходов. Конечно, существуют приложения, которые уже умеют строить подобные графики, и даже сам "ВкусВилл" предоставляет такую возможность в своем личном кабинете. Но моя цель была гораздо глубже — я хотел создать собственную систему, где данные из разных магазинов могли бы агрегироваться в одном месте. Это позволило бы не только анализировать информацию о расходах, но и использовать её для более сложных вычислений и визуализаций.

Кроме того, я всегда хотел получить все эти данные в удобном и кастомизированном формате, который был бы легко настраиваемым под свои нужды. Готовые приложения часто не дают возможности гибко изменять категории товаров или детализировать данные до нужного уровня. Поэтому в рамках этого проекта я решил использовать такие инструменты, как Python для автоматизации процессов извлечения и обработки данных, и Greenplum в качестве базы данных для их хранения и последующего анализа.

В этой статье я расскажу, как, используя Python и базу данных Greenplum, мне удалось автоматизировать процесс извлечения данных из писем от магазина "ВкусВилл", структурировать их и загрузить в базу для дальнейшей обработки. На этапе трансформации данных я разобрал, как можно извлечь ключевую информацию из писем (например, названия товаров, их количество, цену и общую стоимость заказа), а затем преобразовать эти данные в формат, удобный для дальнейшей аналитики.

Проект охватывает весь жизненный цикл данных, начиная с их извлечения из внешних источников (электронная почта), обработки и преобразования в удобный формат, и заканчивая их загрузкой в мощную аналитическую базу данных, где можно запускать сложные запросы и визуализировать результаты.

Итак, начнем.

------

## Введение
### Используемые инструменты:

* Python для извлечения и обработки данных (IDEA PyCharm)

* Greenplum для хранения данных (Клиент dbeaver -> БД Greenplum)

* imaplib для подключения к почтовому серверу

* pandas для работы с данными

* psycopg2 для загрузки данных в Greenplum

### Подготовительные шаги:

* Настройка почтового сервера для извлечения писем (Extract).

* Написание парсера для обработки данных из писем (Transform).

* Преобразование полученных данных в формате CSV (Transform).

* Загрузка данных в временную таблицу базы данных Greenplum (Load).

-----

## EXTRACT
Здесь происходит извлечение данных из почтового ящика. Подключаемся к почтовому серверу, извлекаем непрочитанные письма и сохраняем их для дальнейшей обработки.

### 1. Подключение к почтовому серверу

```python
import imaplib
import email
from email.header import decode_header
import base64
import mail_config

# Загрузка данных для подключения из конфигурационного файла
mail_pass = mail_config.mail_pass  # Пароль почтового ящика
username = mail_config.username  # Имя пользователя (логин)
imap_server = "imap.mail.ru"  # Адрес почтового сервера

# Подключение к серверу через SSL (защищенное соединение)
imap = imaplib.IMAP4_SSL(imap_server)

# Логинимся на почтовый сервер с помощью логина и пароля
imap.login(username, mail_pass)

# Выбираем папку с входящими письмами для работы
imap.select("INBOX")
```
### 2. Извлечение непрочитанных писем

```python
# Поиск всех непрочитанных писем в почтовом ящике
unseen_mails = imap.search(None, 'UNSEEN')  

# Преобразуем результат поиска в строку для дальнейшей обработки
unseen_mails_str = str(unseen_mails[1])

# Выводим список ID непрочитанных писем
print('Непрочитанные письма: ', parsing_list_unseen_email(unseen_mails_str))
```
Пара комментариев по блоку выше:

* Я использую команду imap.search для поиска всех непрочитанных писем с ключом 'UNSEEN'.

* Результат представляет собой строку с ID писем, которую можно обрабатывать с помощью функции parsing_list_unseen_email.


### 3. Парсинг непрочитанных писем

```python
def parsing_list_unseen_email(unseen_emails: str):
    """
    Функция для получения списка ID непрочитанных писем.
    :param unseen_emails: Строка с ID непрочитанных писем.
    :return: Список ID писем.
    """
    lst_id_unseen_emails = []  # Пустой список для ID
    digit_char = ''  # Переменная для временного хранения цифр
    for char in unseen_emails:
        # Если текущий символ — цифра, добавляем его к временной строке
        if char.isdigit():
            digit_char += char
        else:
            # Если цифры закончились, сохраняем их как ID и обнуляем временную строку
            if digit_char:
                lst_id_unseen_emails.append(digit_char)
                digit_char = ''
    return lst_id_unseen_emails  # Возвращаем итоговый список ID
```

Снова комментарии:

* Функция обрабатывает строку с ID писем, разбивая ее на отдельные элементы (цифры) и возвращает список ID для дальнейшей работы.

* Проходим по каждому символу строки, извлекая только цифры, представляющие ID писем.

* Это нужно для того, чтобы знать, какие письма еще не были прочитаны

* Важно: если вы программно открыли письмо, оно пропадет из папки UNSEEN

### 4. Извлечение конкретного письма

```python
# Извлекаем одно из писем по его ID
res, msg = imap.fetch(b'3123', '(RFC822)')  # Здесь 3123 — это ID письма
msg = email.message_from_bytes(msg[0][1])  # Преобразуем байты в объект сообщения

# Читаем и выводим заголовок письма (например, тему)
print('\nЗаголовок письма:\n', decode_header(msg["Subject"])[0][0].decode())
```

* imap.fetch позволяет извлечь содержимое письма по его ID. То есть здесь можно явно указать, какое письмо прочитать.

* Используется библиотека email для декодирования байтов в читабельное сообщение, а затем извлекается заголовок (например, тема письма). Нам нужно найти заголовок, который будет говорить, что письмо пришло из ВкусВилл.

### 5. Извлечение тела письма

```python
def extract_multipart(msg):
    """
    Функция для извлечения содержимого письма с возможностью обработки вложенных сообщений.
    :param msg: Объект сообщения.
    """
    with open('D:/Mail_read_files/email_body.txt', 'w', encoding='utf-8') as f:
        # Если письмо многокомпонентное (содержит вложенные элементы)
        if msg.is_multipart():
            for part in msg.walk():  # Проходим по всем частям письма
                # Если часть письма — это текст (HTML-формат)
                if part.get_content_type() == 'text/html':
                    # Раскодируем содержимое и записываем его в файл
                    f.write(base64.b64decode(part.get_payload()).decode())
        else:
            # Если письмо не содержит вложенных частей, просто записываем его содержимое
            f.write(base64.b64decode(msg.get_payload()).decode())
    print('[INFO] Файл email_body.txt создан')
```

* is_multipart() проверяет, есть ли в письме вложенные элементы. Это важно для обработки писем с несколькими частями. Очень сложный момент для меня лично был, поскольку такие письма - как матрешка (требуется несколько раз углубиться в структуру, чтобы добраться до тела письма).

* Если письмо содержит HTML, то содержимое сохраняется в файл для дальнейшего анализа.

* Мы используем base64.b64decode, так как содержимое писем часто закодировано.

------

## TRANSFORM

После того как мы извлекли содержимое письма, переходим к его анализу. В проекте в этом примере я рассматриваю письма от магазина "Вкусвилл", содержащеи информацию о покупках.

### 1. Парсинг HTML-содержимого

```python
import pandas as pd

# Инициализация пустых списков для хранения данных
st = {'product_name': [], 'count': [], 'price': [], 'total_price': [], 'order_date': [], 'shop_name': []}

def transform_data_vkusvill():
    """
    Парсинг письма от магазина "Вкусвилл" для извлечения данных о покупках.
    """
    with open(r'D:/Mail_read_files/email_body.txt', 'r', encoding='utf-8') as f:
        lines = f.readlines()  # Чтение всех строк файла
        for i, line in enumerate(lines):
            # Ищем строку с названием магазина
            if 'АО "Вкусвилл"' in line:
                shop_name = 'АО "Вкусвилл"'  # Сохраняем название магазина
                # Дата заказа находится на 14-й строке после найденного названия
                order_date = lines[i + 14].split('<')[0].strip()
            # Ищем строку с информацией о товаре
            if 'width="40%"' in line:
                st['order_date'].append(order_date)  # Сохраняем дату заказа
                st['shop_name'].append(shop_name)  # Сохраняем название магазина
                
                # Парсинг названия товара
                product_line = lines[i + 2]
                if ',кг' in product_line or ',шт' in product_line:
                    product_name = product_line.split(',')[0].strip()  # Название товара
                    st['product_name'].append(product_name)
                
                # Парсинг цены, количества и общей суммы
                st['price'].append(float(lines[i + 4].replace(',', '.')))
                st['count'].append(float(lines[i + 6].replace(',', '.')))
                total_price = float(lines[i + 10].split('<')[0].replace(',', '.'))
                st['total_price'].append(total_price)

transform_data_vkusvill()  # Вызов функции для парсинга
```

Пара слов про блок:

* Извлекаю данные о магазине, дате заказа, названии продукта, цене и количестве товара, используя специфические маркеры (например, 'АО "Вкусвилл"', 'width="40%"'). Думаю, что этих полей будет достаточно для дальнейшего анализа и построения визуализации.

* Эти данные сохраняются в словарь st для дальнейшей обработки.

### 2. Преобразование в DataFrame и сохранение в CSV

```python
# Преобразование данных в DataFrame
df = pd.DataFrame(st)

# Сохранение DataFrame в CSV для дальнейшей загрузки
df.to_csv('D:/Mail_read_files/email_body.csv', sep=';', encoding='utf-8', index=False)
```

---

## LOAD
На финальном этапе я загружаем данные из CSV - файла в временную таблицу базы данных Greenplum.

### 1. Загрузка данных в базу данных

```python
import psycopg2
import mail_config

def load_data_to_temp_table():
    """
    Загрузка данных из CSV в временную таблицу Greenplum.
    """
    answer = input('Точно загрузить новую пачку данных? Напишите "да" или "нет": ')
    
    # Проверка на подтверждение действия пользователя
    if answer.lower() == 'да':
        # Подключаемся к базе данных Greenplum
        with psycopg2.connect(
            database=mail_config.db_name,
            user=mail_config.user,
            password=mail_config.password,
            host=mail_config.host,
            port=mail_config.port
        ) as conn:
            # Открываем курсор для выполнения SQL-запросов
            with conn.cursor() as cur, open('D:/Mail_read_files/email_body.csv', 'r', encoding='utf-8') as file:
                # Загружаем данные из CSV в таблицу базы данных
                cur.copy_from(file, 'email_body_temp', sep=';')
                conn.commit()  # Подтверждаем транзакцию
                print("Данные успешно загружены")
    else:
        print("Загрузка отменена")

load_data_to_temp_table()  # Вызов функции для загрузки данных
```

---

Это был мой первый пет-проект, в котором мне пришлось сразу использовать несколько ключевых инструментов, необходимых для работы Data Engineer. Я понимаю, что статья может показаться сложной, но не стоит пугаться. Попробуйте разобрать код построчно и проанализировать каждый шаг — это отличный способ лучше понять, как все работает. Обычно после такого подхода многие моменты становятся более ясными и понятными.

Я постарался снабдить код подробными комментариями, чтобы помочь вам лучше понять каждый шаг процесса. Надеюсь, это поможет вам в освоении подобных задач.

Этот проект — только первый шаг, базовый этап извлечения данных из почты. В следующей статье я расскажу, как автоматизировать процесс, поставив его на расписание, а также покажу, как подключить BI-инструменты для создания визуализаций, чтобы вы могли видеть свои данные в удобной и красивой форме.