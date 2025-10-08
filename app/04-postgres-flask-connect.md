# Connecting Flask App Factory to PostgreSQL

## 1. Create .env file to store all the configs/secrets

**.env file**

```txt

# App Environment variables

FLASK_ENV=development           # env variable for development
DEBUG=True

# Postgres Database Credentials

POSTGRES_USER=postgres          # postgres username
POSTGRES_PASSWORD=postgres123   # postgres password
POSTGRES_DB=studentdb           # postgres database
POSTGRES_HOST=localhost         # postgres host for local setup
POSTGRES_PORT=5432              # postgres port

# Postgres Database URL

DATABASE_URL=postgresql://postgres:postgres123@localhost:5432/studentdb   
```

## 2. Create config.py Configure Environment Variable Loading from Local Sources

**config.py**
```python
import os
from dotenv import load_dotenv 

load_dotenv() # Load environment variables from a .env file

# This file is used to configure the application settings

class Config:
    DEBUG = os.getenv('DEBUG', 'False') == 'True'

    # postgresql settings
    POSTGRES_USER = os.getenv('POSTGRES_USER', 'user')
    POSTGRES_PASSWORD = os.getenv('POSTGRES_PASSWORD', 'password')
    POSTGRES_HOST = os.getenv('POSTGRES_HOST', 'localhost')
    POSTGRES_DB = os.getenv('POSTGRES_DB', 'dbname')
    POSTGRES_PORT = os.getenv('POSTGRES_PORT', '5432')

    SQLALCHEMY_DATABASE_URI = f"postgresql://{POSTGRES_USER}:{POSTGRES_PASSWORD}@{POSTGRES_HOST}:{POSTGRES_PORT}/{POSTGRES_DB}"
    SQLALCHEMY_TRACK_MODIFICATIONS = False  # always good to disable

```
  - We need to import os and dotenv to load environment variables from local sources.
  - `os.getenv` retrieves the environment variables and loads them into the configuration.

## 3. Create models.py to define database schemas as python classes

**models.py**

```python
from . import db  # import db from __init__.py

class Student(db.Model):
    __tablename__ = "students"
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), nullable=False)
    domain = db.Column(db.String(50), nullable=False)
    gpa = db.Column(db.Float, nullable=False)
```

## 4. Create a __init__.py file which makes a folder to behave like python package

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()

def create_app():
    app = Flask(__name__)
    app.config.from_object('app.config.Config') 

    db.init_app(app)
    migrate.init_app(app, db)

    with app.app_context():
        from . import models

    return app
```

## 5. Flask Migrate Commands

**1. flask db init**

  - Apply only once per project
  - Creates a new migrations/ folder in our project.
  - Inside it, we'll see files like:
    - env.py → Alembic environment configuration (connects to your Flask app & models).
    - script.py.mako → template for new migration scripts.
    - versions/ → where each migration script will live.
  - We run this only once to set up migrations.
  - Think of this as “setting up Git for your database”.


**2. flask db migrate -m "message"**

  - Scans our models (db.Model classes) and compares them to the current database schema.
  - If it finds differences (new models, new fields, modified fields), it generates a migration script in migrations/versions/.
  - That script contains the SQL CREATE TABLE, ALTER TABLE, etc. needed to sync the database with our models.
  - The -m "message" is just a human-readable commit message.
  - Think of this as git add + git commit but for your database schema.

**3. flask db upgrade**

  - Applies the latest migration(s) to our actual database (runs the SQL inside the migration file).
  - For example:
    - If it’s the first migration, it will create your tables (students etc.).
    - If we later add a new column, this will ALTER TABLE to include it.
    - Tracks applied migrations in a special table called alembic_version.
  - Think of this as git push — it actually changes the database.

**4. flask db downgrade → rolls back to the previous migration (undo).**

**5. flask db history → shows all migrations with IDs.**

**6. flask db current → shows the migration your DB is currently at.**

## 6. Verify tables in Postgres

  - Commands to connect to our DB:
  ```bash
  psql -h localhost -U postgres -d studentdb
  \dt
  \d students
  ```

**Output**

```bash
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ psql -h localhost -U postgres -d studentdb
Password for user postgres: 
psql (16.9 (Ubuntu 16.9-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

studentdb=# \dt
              List of relations
 Schema |      Name       | Type  |  Owner   
--------+-----------------+-------+----------
 public | alembic_version | table | postgres
 public | students        | table | postgres
(2 rows)


studentdb=# \d students
                                   Table "public.students"
 Column |         Type          | Collation | Nullable |               Default                
--------+-----------------------+-----------+----------+--------------------------------------
 id     | integer               |           | not null | nextval('students_id_seq'::regclass)
 name   | character varying(50) |           | not null | 
 domain | character varying(50) |           | not null | 
 gpa    | double precision      |           | not null | 
Indexes:
    "students_pkey" PRIMARY KEY, btree (id)
```

## 7. Insert student details into studentdb using Flask Shell

**Add new student using flask shell**
```sql
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ flask shell
Python 3.12.3 (main, Jun 18 2025, 17:59:45) [GCC 13.3.0] on linux
App: app
Instance: /home/ubuntu/Flask-REST-API/instance
>>> from app import db; Student
<class 'app.models.Student'>
>>> new_student = Student(name='Akhil Thydai', domain='Electronics and Communication Engineering', gpa ='7.01')
>>> db.session.add(new_student)
>>> db.session.commit()
>>> exit
```
**Query the newly added student details**

```sql
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ psql -U postgres -d studentdb -h localhost -W
Password: 
psql (16.9 (Ubuntu 16.9-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

studentdb=# \d
               List of relations
 Schema |      Name       |   Type   |  Owner   
--------+-----------------+----------+----------
 public | alembic_version | table    | postgres
 public | students        | table    | postgres
 public | students_id_seq | sequence | postgres
(3 rows)

studentdb=# \d students
                                   Table "public.students"
 Column |         Type          | Collation | Nullable |               Default                
--------+-----------------------+-----------+----------+--------------------------------------
 id     | integer               |           | not null | nextval('students_id_seq'::regclass)
 name   | character varying(50) |           | not null | 
 domain | character varying(50) |           | not null | 
 gpa    | double precision      |           | not null | 
Indexes:
    "students_pkey" PRIMARY KEY, btree (id)

studentdb=# SELECT * FROM students;
 id |     name     |                  domain                   | gpa  
----+--------------+-------------------------------------------+------
  1 | Akhil Thydai | Electronics and Communication Engineering | 7.01
(1 row)

studentdb=# SELECT * FROM students WHERE id = 1;
 id |     name     |                  domain                   | gpa  
----+--------------+-------------------------------------------+------
  1 | Akhil Thydai | Electronics and Communication Engineering | 7.01
(1 row)

studentdb=# \q
```

## 8. Added new column 'email' into the student table

```python
from . import db   # Import db from the app package

class Student(db.Model):
    __tablename__ = "students"
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), nullable=False)
    domain = db.Column(db.String(50), nullable=False)
    gpa = db.Column(db.Float, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
```

**once the models.py file is updated with email, apply the changes to the database we have two options to**

- **Option 1: Allow NULL emails (quick fix, for testing)**

  **In our model:**
  ```
  email = db.Column(db.String(120), unique=True, nullable=True)
  ```
  **Then run:**
    - `flask db migrate -m "make email nullable"`
    - `flask db upgrade`

- **Option 2: Add a default value for existing rows**
  - If we want email to be required (nullable=False), we need to tell Alembic what to put in existing rows.
  - In our migration file (migrations/versions/..._added_new_column_email.py),
   
    update:
    ```python
    with op.batch_alter_table('students', schema=None) as batch_op:
    batch_op.add_column(sa.Column('email', sa.String(length=120), nullable=False, server_default='test@example.com'))
    ```
  - This way, existing rows will get "test@example.com" as email.
  - Then you can later drop the server_default if not needed.

  
- **Run:** `flask db upgrade`

```bash
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ flask db upgrade
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 7aee6c929e5c -> 8f8748164629, added new column email
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ 
```

- **Verify**
```sql
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ psql -U postgres -d studentdb -h localhost -W
Password: 
psql (16.9 (Ubuntu 16.9-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

studentdb=# \d students
                                    Table "public.students"
 Column |          Type          | Collation | Nullable |                Default                
--------+------------------------+-----------+----------+---------------------------------------
 id     | integer                |           | not null | nextval('students_id_seq'::regclass)
 name   | character varying(50)  |           | not null | 
 domain | character varying(50)  |           | not null | 
 gpa    | double precision       |           | not null | 
 email  | character varying(120) |           | not null | 'test@example.com'::character varying
Indexes:
    "students_pkey" PRIMARY KEY, btree (id)
    "students_email_key" UNIQUE CONSTRAINT, btree (email)

studentdb=# SELECT * FROM students;
 id |     name     |                  domain                   | gpa  |      email       
----+--------------+-------------------------------------------+------+------------------
  1 | Akhil Thydai | Electronics and Communication Engineering | 7.01 | test@example.com
(1 row)

studentdb=# 
```
## 9. Update and verify email for an existing student details

- **Update the email using flask shell**

```sql
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ flask shell
Python 3.12.3 (main, Jun 18 2025, 17:59:45) [GCC 13.3.0] on linux
App: app
Instance: /home/ubuntu/Flask-REST-API/instance
>>> from app import db
>>> from app.models import Student
>>> student = Student.query.get(1)
>>> student.email = "160101130028@cutm.ac.in"
>>> db.session.commit()
>>> student = Student.query.get(1)
>>> print(student.email)
160101130028@cutm.ac.in
>>> 
now exiting InteractiveConsole...
```

- **Verify the updated email in the table**
```sql
(venv) ubuntu@ip-10-0-6-246:~/Flask-REST-API$ psql -U postgres -d studentdb -h localhost -W
Password: 
psql (16.9 (Ubuntu 16.9-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

studentdb=# SELECT * FROM students;
 id |     name     |                  domain                   | gpa  |          email          
----+--------------+-------------------------------------------+------+-------------------------
  1 | Akhil Thydai | Electronics and Communication Engineering | 7.01 | 160101130028@cutm.ac.in
(1 row)

studentdb=# 
```
