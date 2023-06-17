# extracting_data_from_sources
work on getting Greenery data form source systems (Direct Database Connection, APIs, SFTP)


workshop เราจะลองดึงข้อมูลผ่าน Protocol ต่างๆกัน

- APIs

- SFTP

- Direct Database Connection

ไปลุยกันเลย!!! เริ่มจากติดตั้ง Poetry
วิธีติดตั้ง Poetry

1. https://python-poetry.org/

2. curl -sSL https://install.python-poetry.org | python3 - ## เอาจาก website เป็นการเชื่อมเพื่อติดตั้ง poetry

3. pyproject.tomlที่ไฟล์นี้หลัก ๆ เราจะติดตั้ง 3 packages

    - requests - ใช้เชื่อมต่อกับ API

    - pysftp - ใช้เชื่อมต่อกับ SFTP

    - psycopg2 - ใช้เชื่อมต่อกับ Postgres database

```
[tool.poetry]
name = "bootcamp-project"
version = "0.1.0"
description = ""
authors = ["valen <valenlearningcurve@gmail.com>"]

[tool.poetry.dependencies]
python = "^3.9"
requests = "^2.29.0"
pysftp = "^0.2.9"
psycopg2 = "^2.9.6"

[tool.poetry.dev-dependencies]

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

ไฟล์ configuration pipeline.conf เราจะใช้เก็บพวก credentials ต่าง ๆ ซึ่งจริง ๆ ตาม practice แล้ว ไฟล์พวกนี้เราไม่ควรที่จะเอาเข้าไปเก็บใน version control หรือว่า Git ซึ่งเวลาที่เรา deploy งานเราขึ้น production แล้ว ไฟล์นี้อาจจะถูกใส่เข้ามาโดยทีม operation หรือ deployment ในช่วงนั้น

```
[sftp_config]
host = 34.87.139.82
port = 2222
username = greenery
password = hardpassword

[postgres_config]
host = 34.87.139.82
port = 5432
username = postgres
password = postgres
database = greenery

[api_config]
host = 34.87.139.82
port = 8000
```
ไฟล์ที่ดึงได้เราจะเก็บไว้ใน folder "data" ไฟล์ main-api.py ใช้ดึงข้อมูลจาก API

```
-- main-api.py
import configparser
import csv

import requests


parser = configparser.ConfigParser()
parser.read("pipeline.conf")
host = parser.get("api_config", "host")
port = parser.get("api_config", "port")

API_URL = f"http://{host}:{port}"
DATA_FOLDER = "data"

### Events
data = "events"
date = "2021-02-10"
response = requests.get(f"{API_URL}/{data}/?created_at={date}")
data = response.json()
with open(f"{DATA_FOLDER}/events.csv", "w") as f:
    writer = csv.writer(f)
    header = data[0].keys()
    writer.writerow(header)

    for each in data:
        writer.writerow(each.values())

### Users
data = "users"
date = "2020-10-23"
response = requests.get(f"{API_URL}/{data}/?created_at={date}")
data = response.json()
with open(f"{DATA_FOLDER}/users.csv", "w") as f:
    writer = csv.writer(f)
    header = data[0].keys()
    writer.writerow(header)

    for each in data:
        writer.writerow(each.values())

### Orders
data = "orders"
date = "2021-02-10"
response = requests.get(f"{API_URL}/{data}/?created_at={date}")
data = response.json()
with open(f"{DATA_FOLDER}/orders.csv", "w") as f:
    writer = csv.writer(f)
    header = data[0].keys()
    writer.writerow(header)

    for each in data:
        writer.writerow(each.values())
 ```
 
 ไฟล์  main-sftp.py ใช้ดึงข้อมูลจาก SFTP ใช้ดึงข้อมูล
 
- products

- promos

 ```
 import configparser

import pysftp

parser = configparser.ConfigParser()
parser.read("pipeline.conf")
username = parser.get("sftp_config", "username")
password = parser.get("sftp_config", "password")
host = parser.get("sftp_config", "host")
port = parser.getint("sftp_config", "port")

# Security risk! Don't do this on production
# You lose a protection against Man-in-the-middle attacks
cnopts = pysftp.CnOpts()
cnopts.hostkeys = None

DATA_FOLDER = "data"

files = [
    "products.csv",
    "promos.csv",
]
with pysftp.Connection(host, username=username, password=password, port=port, cnopts=cnopts) as sftp:
    for f in files:
        sftp.get(f, f"{DATA_FOLDER}/{f}")
        print(f"Finished downloading: {f}")
```

ไฟล์ main-postgres.py ดึงข้อมูลจาก Postgres (Direct Database Connection) ใช้ดึงข้อมูล

- addresses

- order_items

```
import csv
import configparser

import psycopg2


parser = configparser.ConfigParser()
parser.read("pipeline.conf")
dbname = parser.get("postgres_config", "database")
user = parser.get("postgres_config", "username")
password = parser.get("postgres_config", "password")
host = parser.get("postgres_config", "host")
port = parser.get("postgres_config", "port")

conn_str = f"dbname={dbname} user={user} password={password} host={host} port={port}"
conn = psycopg2.connect(conn_str)
cursor = conn.cursor()

DATA_FOLDER = "data"

table = "addresses"
header = ["address_id", "address", "zipcode", "state", "country"]
with open(f"{DATA_FOLDER}/addresses.csv", "w") as f:
    writer = csv.writer(f)
    writer.writerow(header)

    query = f"select * from {table}"
    cursor.execute(query)

    results = cursor.fetchall()
    for each in results:
        writer.writerow(each)


table = "order_items"
header = ["order_id", "product_id", "quantity"]
with open(f"{DATA_FOLDER}/{table}.csv", "w") as f:
    writer = csv.writer(f)
    writer.writerow(header)

    query = f"select * from {table}"
    cursor.execute(query)

    results = cursor.fetchall()
    for each in results:
        writer.writerow(each)
```
