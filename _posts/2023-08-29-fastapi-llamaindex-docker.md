---
layout: post
title: Плейґраунд для експериментів з ШІ на базі llamaindex та fastapi
author: Микола Понич
lang: ua
date: 2023-08-29 03:09:25 +0300
tags: llamaindex fastapi docker python gpt swagger redoc rest api

--- 

Якось мені потрібно було пожмакати llm та тулзи для роботи з цим. Подивитись як правильно працювати та шукати
інфу в шалено великих контекстах документів через gpt3.5-turbo та gpt4 моделі (або їх аналоги).
Тема сама по собі вже не нова. Рішень в трендах на хабах повно.

Погугливши, почитавши документацію openapi, llama, тренди хабів, знайомства з llamahub та іншими моделями, я зрозумів що
є два шляхи:
**Fine-tuning** - це коли ми беремо модель і тренуємо її на своїх даних,
або **Embedding** - коли ми беремо модель і використовуємо її для векторизації документів.

**fine-tuning**

Звісно крута штука, дозволяє тонко налаштувати модель під свої потреби на основі своїх даних.
Але вона потребує багато ресурсів, часу та даних. Можливо вибрати базову модель для старту, але все одно мені здалось
складною.

**embedding**

Після пару статей та тредів з дискусіями, я зрозумів що для мене варіант з embedding більш підходящий.
Він не покриє всіх моїх потреб (поки що покриває), але більш зрозумілий і простий для реалізації.
Вчити нікого не потрібно, весь контекст ми генеруємо з даних, які ми вже маємо, дані достаємо з векторів.

В обох випадках я хотів мати змогу запустити все локально, можливо подивитись той чи інший результат,
поманіпулювати даними. Експериментувати, ламати, фіксити, вдосконалювати.
Тим більше що рішень багацько. Тому далі приклади якщо не з офіційної документації, то з особистих

## Вимоги до середовища:

- Робота з векторними базами данних;
- Сегментування документів, індексація, пошук, фрагментація;
- Тузли для роботи з моделями та даними, багато-багато тулзів та бібліотек;
- Зрозумілий та простий інтерфейс для взаємодії з моделями та даними;
- Шведке розгортання, запуск. Якщо можна в докері, то в докері, якщо можна локально, то локально;
- Щоб по простому, по домашньому. Я, наприклад не з пайтон-тусовки, і насправді дужке далекий від них,
  Але так чи інакше, всі вкусняшки ШІ там. Потрібно було знайти щось просте, зрозуміле

Сьогодні хотілось би поділитись рішеннями які в результаті я забрав собі, а також показати як це все можна
розвернути та запустити локально чи в докері.

[llamaindex](https://gpt-index.readthedocs.io/en/latest/) - фреймворк для роботи з векторними базами данних, індексації,
фрагментації, пошуку і т.д.

[fastapi](https://fastapi.tiangolo.com/) - веб фреймворк для роботи з rest api, автодокументація, зручність розробки

Потужності та простота цих двох фреймворків мені сподобались, тому далі будемо працювати з ними.

Що ж, поїхали.

## Створення проекту git & venv оточення

Для початку створимо проект та перейдем в нього

```bash
git init --initial-branch=develop llm-srv && \
cd llm-srv # переходимо в root каталог проекту, далі будемо працювати з нього
```

Налаштуємо `.gitignore` файл

```bash
touch .gitignore
```

<details>
  <summary> `.gitignore` </summary>

```.gitignore
bin/
.DS_Store

# Byte-compiled / optimized / DLL files
__pycache__/
__pycache__
*.py[cod]
*$py.class

# C extensions
*.so

# Distribution / packaging
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# PyInstaller
#  Usually these files are written by a python script from a template
#  before PyInstaller builds the exe, so as to inject date/other infos into it.
*.manifest
*.spec

# Installer logs
pip-log.txt
pip-delete-this-directory.txt

# Unit test / coverage reports
htmlcov/
.tox/
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
.hypothesis/
.pytest_cache/

# Translations
*.mo
*.pot

# Django stuff:
*.log
local_settings.py
db.sqlite3

# Flask stuff:
instance/
.webassets-cache

# Scrapy stuff:
.scrapy

# Sphinx documentation
docs/_build/

# PyBuilder
target/

# Jupyter Notebook
.ipynb_checkpoints

# pyenv
.python-version

# celery beat schedule file
celerybeat-schedule

# SageMath parsed files
*.sage.py

# Environments
.env
.venv
env/
venv/
ENV/
env.bak/
venv.bak/

# Spyder project settings
.spyderproject
.spyproject

# Rope project settings
.ropeproject

# mkdocs documentation
/site

# mypy
.mypy_cache/

.idea/
.vscode/

# Project
postgres-data
```

</details>


Створимо віртуальне середовище для пайтона та активуємо його

```bash
virtualenv venv &&
  source venv/bin/activate
```

Зазвичай я використовую [virtualenv](https://virtualenv.pypa.io/en/latest/)
На `python3.7` та `python3.10` локально все запрацювало без проблем та додаткових налаштувань в середовищі.

todo:
Якщо у вас будуть проблеми, в низу я залишив список знайдених проблем та залишил інформацію як їх вирішити.

## llamaindex

Як мені здалося, llamaindex є хорошим інструментом для моїх потреб. Дуже багато рішень з коробки, в тому числі:

- індексувати ваші документи, та робити пошук по ним
- фрагментація, ранжування документів
- підтримує різні типи документів: текст, картинки, відео, аудіо, гео, графи, таблички, документи
- підтримка декількох видів баз даних в томі числі і потрібна мені - векторна
- мвіє: llama, gpt, gpt4 і інші моделі, в тому числі інтеграція
  з llamahub (про llamahub потрібно якось цілу статтю забахати)
- та багато іншого. перечислене мної вище - це тільки те що мені потрібно і на те на що я звернув увагу

Річ мені сподобалась, тому забираємо собі.

```bash
pip install llama-index
```

розгорнемо відразу всі залежності в `requirements.txt`

```bash
pip freeze > requirements.txt
```

Цього вже достатньо для того, щоб експериментувати.
Перевірмо чи все працює.
Створимо документ `./data/welcome.txt` в якому буде інформація для індексації та перевірки пошуку.

```bash
mkdir data && \
echo "The Bucha massacre was the mass murder of Ukrainian civilians and prisoners of war" \
  "by the Russian Armed Forces during the fight for and occupation of the city of Bucha as" \
  "part of the Russian invasion of Ukraine.\n" \
  "Photographic and video evidence of the massacre emerged on 1 April 2022" \
  "after Russian forces withdrew from the city." \
  "source: https://en.wikipedia.org/wiki/Bucha_massacre" > data/welcome.txt

```

Створимо `src/example.py`

```bash
mkdir src && \
touch src/example.py
```

<details>
  <summary> `src/example.py` </summary>

```python
from llama_index import VectorStoreIndex, SimpleDirectoryReader

query = "What happened in Bucha?"

documents = SimpleDirectoryReader('data').load_data()
index = VectorStoreIndex.from_documents(documents)

query_engine = index.as_query_engine()
query_response = query_engine.query(query)

print(query_response.response)
```

</details>

задамо зміну OPENAI_API_KEY для роботи з openai

> ключі можно генерувати в особистому кабінеті https://platform.openai.com/account/api-keys

```bash
export OPENAI_API_KEY=sk-5...kqe
```

Старт...

```bash
python src/example.py
```

**output:**

```bash
(venv) ➜  llm-srv git:(develop) ✗ python src/example.py
A mass murder of Ukrainian civilians and prisoners of war occurred in Bucha during the fight for and occupation 
 of the city by the Russian Armed Forces as part of the Russian invasion of Ukraine. 
Photographic and video evidence of the massacre emerged after Russian forces withdrew from the city.
```

todo:
Якщо будуть складності, в кінці є розділ "траблшутинги та рішення", куди виніс проблеми і рекомендації як з ними
боротись.

Структура проекту:

```
.
├── data
│   └── welcome.txt 
├── requirements.txt
└── src
    └── example.py
```

Не забуваємо комітитись

## fastapi

Далі по списку інтерфейс для роботи з ШІ.
Вже давно не роблю інтерфейсів там де не потрібно, натомість взяв собі за звичку викидувати cli або web api.
В нашому випадку cli вже налаштований, пройдемось по web api.

Погляд впав FastAPI. Мені здалось що це фреймворк який не тригирить запититувати: а як? а що? чого саме? Просто пишемо
апішку.
Легка документація, простий сетап, простий запуск, і ще я трошки типонутий (пайтон динамічний), тому також забераємо
собі.
В коробці:

- змога створювати rest api інтерфейси з усіма доступними методами
- автодокументація swagger або redoc
- можливість трошки затипизувати вхідні та вихідні данні

```bash
pip install fastapi
```

розвернемо залежності в requirements.txt

_не знаю чи потрібно це робити кожен раз чи ні, відпишіть пітоністи в коментарях_

```bash
pip freeze > requirements.txt
```

Цю всю вебку буде крутити uvicorn, такий собі вебсервер аля nginx, apache, iis і так далі
в ряд (кто ще не вміє себе хостити? в кожному язику в кожній лібі... всім по селфхосту)

Своримо нашу точку входу в апішку src/main.py

```bash
touch src/main.py
```

накинемо туди трохи коду в прикладів з документації fastapi та llamaindex

<details>
  <summary> `src/main.py` </summary>

```python
from fastapi import FastAPI
from llama_index import VectorStoreIndex, SimpleDirectoryReader

app = FastAPI()
documents = SimpleDirectoryReader('data').load_data()
index = VectorStoreIndex.from_documents(documents)

query_engine = index.as_query_engine()


@app.get("/")
def read_root(query: str):
    response = query_engine.query(query)
    return {"res": response.response}
```

</details>


Щоб запустити api частину, запустимо це за допомогою uvicorn з параметрами --reload (щоб не перезапускати кожен раз)

```bash
uvicorn src.main:app --reload
```

**output:**

```bash
(venv) ➜  llm-srv git:(develop) ✗ uvicorn src.main:app --reload
INFO:     Will watch for changes in these directories: ['./llm-srv']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [35544] using WatchFiles
INFO:     Started server process [35547]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

Після запуску, можемо відкрити http://127.0.0.1:8000/docs (для swagger)
або http://127.0.0.1:8000/redoc (для redoc)

Щоб зупинити сервер, натисніть Ctrl+C (CMD+C на Mac).

Структура проекту:

```
.
├── data
│   └── welcome.txt
├── requirements.txt
└── src
├── example.py
└── main.py
```

Комітимось і переходимо далі

## Пакуємо в докер

### `Dockerfile`

Далі не буде інструкціі як закрити все в докер (композ включно) правильно та по феншую.
Також не буде різних стейджів та ліній розробки.
Один-однесинький контейнер, який сам собі і збере і запустить.
Для проду забиріть коректний докерфайл в вашого девоОпса, дякую.

```
touch Dockerfile
```

<details>
  <summary> `Dockerfile` </summary>

```dockerfile
FROM python:3.10
# на випадок помилок при збірці на arm64v8 (буде кричати на платформу)
# FROM arm64v8/python:3.10
# метаінформація про контейнер, можна стерти, не впливає на роботу
LABEL maintainer="Mykola Ponych <n.ponich@gmail.com>"

# середовище очікує DOCKER_CONTAINER_PORT та OPENAI_API_KEY
# та вольюм /app/data

VOLUME /app/data

ARG DOCKER_CONTAINER_PORT=8000
ARG OPENAI_API_KEY

ENV DOCKER_CONTAINER_PORT=${DOCKER_CONTAINER_PORT}
ENV OPENAI_API_KEY=${OPENAI_API_KEY}
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

COPY . /app
WORKDIR /app

EXPOSE ${DOCKER_CONTAINER_PORT}

# fix: `WARNING: Running pip as the 'root' user can result in broken permissions`
#RUN addgroup --gid 1001 --system app && \
#    adduser --no-create-home --shell /bin/false --disabled-password --uid 1001 --system --group app && \
#    chown -R app:app /app

RUN chmod +x /app/entrypoint.sh
RUN pip install --no-cache-dir -r requirements.txt

#USER app

ENTRYPOINT ["/app/entrypoint.sh"]
```

Тут контейнер очикує двух параметрів DOCKER_CONTAINER_PORT та OPENAI_API_KEY які ми передамо при запуску контейнера.
Все збирається на базовому образі `python:3.10`.
Прокидуємо при зборці параметри оточення `OPENAI_API_KEY`,
`DOCKER_CONTAINER_PORT`,
`PYTHONDONTWRITEBYTECODE`,
`PYTHONUNBUFFERED`

</details>


Щоб не білдити імейдж кожного разу, запустим контейнер через entrypoint скрипт, де самі будемо визначати
що запускати та з якими параметрами.

```
touch entrypoint.sh
```

<details>
  <summary> `entrypoint.sh` </summary>

```bash
#!/bin/sh

echo "Starting the server"
pip install --no-cache-dir -r requirements.txt

uvicorn src.main:app --port ${DOCKER_CONTAINER_PORT} --host 0.0.0.0

```

Тут наступне:

- запускаємо інсталяцію оточення
- запускаємо uvicorn з параметрами які ми передали при запуску контейнера

</details>


Давайте забілдимо контейнер llm-srv

```bash
docker build -t llm-srv .
```

```bash
(venv) ➜  llm-srv git:(develop) ✗ docker build -t llm-srv .
[+] Building 6.2s (8/8) FINISHED                                                                      docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                                  0.0s
 => => transferring dockerfile: 792B                                                                                  0.0s
 => [internal] load .dockerignore                                                                                     0.0s
 => => transferring context: 245B                                                                                     0.0s
 => [internal] load metadata for docker.io/library/python:3.10                                                        1.8s
 => [internal] load build context                                                                                     0.7s
 => => transferring context: 1.47MB                                                                                   0.6s
 => CACHED [1/3] FROM docker.io/library/python:3.10@sha256:72e7f78d37f3e3f779eeab9bab7136155b35589321bb97c58c6415297  0.0s
 => [2/3] COPY . /app                                                                                                 2.6s
 => [3/3] WORKDIR /app                                                                                                0.0s
 => exporting to image                                                                                                1.0s
 => => exporting layers                                                                                               1.0s
 => => writing image sha256:f5e19f5352735ddd086fc6aaf7685b4ebe32601e5340108ca56719f6eafdac2a                          0.0s
 => => naming to docker.io/library/llm-srv
```

Запускаємо контейнер

```bash
docker run \
  -e "OPENAI_API_KEY=${OPENAI_API_KEY}" \
  -e "DOCKER_CONTAINER_PORT=8000" \
  -v ./data:/app/data \
  -p 8000:8000 \
  -it llm-srv

```

**output:**

```bash
(venv) ➜  llm-srv git:(develop) ✗ docker run \
  -e "OPENAI_API_KEY=${OPENAI_API_KEY}" \
  -e "DOCKER_CONTAINER_PORT=8000" \
  -v ./data:/app/data \
  -p 8000:8000 \
  -it llm-srv
Starting the server
#....
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. 
  It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv
[nltk_data] Downloading package punkt to /tmp/llama_index...
[nltk_data]   Unzipping tokenizers/punkt.zip.
INFO:     Started server process [56]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Так як каталог `data` копіюється коли ми робимо в докерфайлі `COPY . /app`,
в одночас ми маунтимо його в контейнер, то він буде перезаписуватись.
Але ми можемо вказати докеру щоб він ігнорував його при дії с контейнером

```bash
echo "data" > .dockerignore
```

Комітимось і переходимо далі

### `docker-compose`

Добавимо .env файл, щоб використовувати його в докер композі

```bash
touch .env
```

<details>
  <summary> `.env` </summary>

```dotenv
# App config
APP_NAME=llm-srv
APP_VERSION=0.0.1
APP_HOST=localhost
APP_HOST_PORT=80

# Docker config
DOCKER_CONTAINER_NAME=llm-srv-container
DOCKER_CONTAINER_PORT=8080
DOCKER_IMAGE_NAME=llm-srv-image:latest
OPENAI_API_KEY=${OPENAI_API_KEY}
```

</details>

Налаштуємо докер композ файл,

```bash
touch docker-compose.yml
```

<details>
  <summary> `docker-compose.yml` </summary>

```yaml
version: '3.9'
services:
  app:
    container_name: ${DOCKER_CONTAINER_NAME}
    build:
      context: .
      dockerfile: Dockerfile
    env_file:
      - .env
    ports:
      - "${APP_HOST_PORT}:${DOCKER_CONTAINER_PORT}"
    volumes:
      - .:/app
      - ./data:/app/data
```

Я не став доповнювати щось і обертати nginx чи тощо. Просто оприділив наш контейнер як сервіс і все
</details>


Структура:

```
.
├── .dockerignore
├── .gitignore
├── Dockerfile
├── data
│   └── welcome.txt
├── docker-compose.yml
├── entrypoint.sh
├── requirements.txt
└── src
    ├── example.py
    └── main.py
```

Ось і все, можемо комітитись

## Що в результаті

На виході отримуємо такий собі скелетон для розробки ШІ систем на базі
openai та інших моделей.:

- playground (або sandbox) для єкспериментів з llamaindex/openai та іншими моделями
- rest фульчик з автодокументациєю swagger або redoc
- llamaindex фреймворк для роботи з данними: індексація, фрагментація, пошук і т.д.
- можливість зпуска локально, з докера або докеркомпозу

В моємо випадку це - база і цю базу вже можливо якось розвивати під свої потреби.

## Можливі проблеми та рішення

venv менеджер `virtualenv`

```bash
pip install virtualenv
```

[Альтернативна установка `virtualenv`](https://virtualenv.pypa.io/en/latest/installation.html)

Щоб пофіксити напис при старті контейнера

`WARNING: Running pip as the 'root' user can result in broken permissions`, розкошуємо в `./Dockerfile` наступне:

```bash
RUN addgroup --gid 1001 --system app && \
    adduser --no-create-home --shell /bin/false --disabled-password --uid 1001 --system --group app && \
    chown -R app:app /app
    
USER app
```

Тут ми налаштовуємо користувача app з групою app, якій належить весь контент в каталозі `/app`
і просимо систему далі використовувати цього користувача.

**Хочу нагадати читачу**, що ми працюємо в `ROOT` каталозі проекту, тому всі команди виконуємо відповідно до цього.
В звязку с цим можут зістрічатися помилки, як:

```bash
Traceback (most recent call last):
  File "/.../llm-srv/src/example.py", line 5, in <module>
    documents = SimpleDirectoryReader('../data').load_data()
  File "/.../llm-srv/venv/lib/python3.10/site-packages/llama_index/readers/file/base.py", line 109, in __init__
    raise ValueError(f"Directory {input_dir} does not exist.")
ValueError: Directory ../data does not exist.
```

Щоб виправити подібне достатньо або запустити наш апп з кореневого каталогу проекту, або відповідно змінити
шляхи до файлів в коді (дивіться `example.py` та `main.py`)

### `nltk`

Після першого прикладу запуску `python` `src/example.py` я отриавав помилку

```
raceback (most recent call last):
File "src/example.py", line 6, in <module>
index = VectorStoreIndex.from_documents(documents)
entence_tokenizer
import nltk
ModuleNotFoundError: No module named 'nltk'
```

nltk (Natural Language Toolkit) - це залежність llamaindex якусь чомусь він за собою не потянув і викинув помилку.
Я перепровірив `requirements.txt`, там все було. Не знаю як це працює, але він не встановився.
Тим не менш, ставимо руцями

```bash
pip install nltk
```

### `uvicorn`

Не хотів запускатись з під докера, кастомним портом доки не запустив з параметром `--host`
Приклад:

```bash
uvicorn src.main:app --port 8080 --host 0.0.0.0
```

Також при запуску з докера виникла помилка:

```bash
Starting the server
/app/entrypoint.sh: 6: uvicorn: not found
(venv) ➜  llm-srv git:(develop) ✗ pip install uvicorn
Requirement already satisfied: uvicorn in ./venv/lib/python3.10/site-packages (0.23.2)
Requirement already satisfied: click>=7.0 in ./venv/lib/python3.10/site-packages (from uvicorn) (8.1.7)
Requirement already satisfied: h11>=0.8 in ./venv/lib/python3.10/site-packages (from uvicorn) (0.14.0)
Requirement already satisfied: typing-extensions>=4.0 in ./venv/lib/python3.10/site-packages (from uvicorn) (4.7.1)
```

Ситуація подібна `nltk`, лікуємо так же само:

```bash
pip install uvicorn
```

_скоріш за все я якось недопрацював з залежностями,_
_буду вдячний якщо хтось розкаже як це працює і як правильно_

### `openai`

```text
Could not load OpenAI model. Using default LlamaCPP=llama2-13b-chat. If you intended to use OpenAI, please check your OPENAI_API_KEY.
Original error:
No API key found for OpenAI.
Please set either the OPENAI_API_KEY environment variable or openai.api_key prior to initialization.
API keys can be found or created at https://platform.openai.com/account/api-keys
```

для коректної роботи за openai моделями, потрібно встановити змінну оточення `OPENAI_API_KEY`.

```bash
export OPENAI_API_KEY="sk-5...kqe" && uvicorn src.main:app
```

## debug

Далі приклад для pycahrm,
але ви можете використовувати будь який інструмент, там принцип такий же (як у всіх інтирпритоварних яп)

Ідемо в
`Pycharm` -> `Preferences` -> `Project` -> `Project Interpreter` -> `Add` -> `Existing environment`
та вибираємо наше віртуальне середовище.

![img.png](./****pycharm-venv.png)

Якщо працюємо локально, показуємо де знаходиться інтерпретатор python3 на вашій машині
Якщо docker, вказуємо контейнер, в якому знаходиться python3
Якщо docker-compose, вказуємо сервіс з якого ми хочемо використовувати python3

https://github.com/codelikes/simple-fastapi-llamaindex-env
