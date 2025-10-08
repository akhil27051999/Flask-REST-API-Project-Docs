## Project Prerequisites

- **Create a project folder and a .venv folder within:**
  ```bash
  mkdir Flask-REST-API && cd Flask-REST-API
  python3 -m venv .venv
  ```
- **Activate the environment:**
  ```bash
  . .venv/bin/activate
  ```  
- **Within the activated environment, Install the required packages used in our project**

| **Package**          | **Purpose / Role**                                                 | **When You Need It**                                                               |
| -------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| **Flask**            | Lightweight web framework for building web applications in Python. | Core of your web app. Always needed for Flask projects.                            |
| **Flask-SQLAlchemy** | Flask extension that integrates SQLAlchemy ORM with Flask.         | Needed if your app uses a database. Simplifies database operations.                |
| **Flask-Migrate**    | Flask extension for database migrations, built on Alembic.         | Needed if you want to version your database schema and apply migrations safely.    |
| **psycopg2-binary**  | PostgreSQL database adapter for Python.                            | Needed if your app connects to a PostgreSQL database.                              |
| **gunicorn**         | Production WSGI HTTP server for serving Flask apps.                | Needed to run Flask in production (not required for local development).            |
| **python-dotenv**    | Loads environment variables from a `.env` file into Python.        | Needed to manage sensitive configuration (like DB credentials) without hardcoding. |
| **pytest**           | Python testing framework for writing and running tests.            | Needed if you want to test your app.                                               |
| **pytest-flask**     | Pytest plugin with helpers for testing Flask applications.         | Needed if you want easier testing of Flask routes, requests, and app context.      |
| **pytest-dotenv**    | Pytest plugin to load `.env` files during testing.                 | Needed if your tests require environment variables.                                |


```bash
pip install Flask Flask-SQLAlchemy Flask-Migrate psycopg2-binary python-dotenv pytest pytest-flask pytest-dotenv gunicorn
```
  
- **To verify and list the installations:**
```bash
python3 --version && pip list
```
#### Output:
```nginx
Python 3.10.12
```
```
Package             Version
------------------- --------------
attrs               23.2.0
Automat             22.10.0
Babel               2.10.3
bcc                 0.29.1
bcrypt              3.2.2
blinker             1.7.0
boto3               1.34.46
botocore            1.34.46
certifi             2023.11.17
chardet             5.2.0
click               8.1.6
cloud-init          25.1.4
colorama            0.4.6
command-not-found   0.3
configobj           5.0.8
constantly          23.10.4
cryptography        41.0.7
dbus-python         1.3.2
distro              1.9.0
distro-info         1.7+build1
ec2-hibinit-agent   1.0.0
hibagent            1.0.1
httplib2            0.20.4
hyperlink           21.0.0
idna                3.6
incremental         22.10.0
Jinja2              3.1.2
jmespath            1.0.1
jsonpatch           1.32
jsonpointer         2.0
jsonschema          4.10.3
launchpadlib        1.11.0
lazr.restfulclient  0.14.6
lazr.uri            1.0.6
markdown-it-py      3.0.0
MarkupSafe          2.1.5
mdurl               0.1.2
netaddr             0.8.0
netifaces           0.11.0
oauthlib            3.2.2
packaging           24.0
pexpect             4.9.0
pip                 24.0
ptyprocess          0.7.0
pyasn1              0.4.8
pyasn1-modules      0.2.8
Pygments            2.17.2
PyGObject           3.48.2
PyHamcrest          2.1.0
PyJWT               2.7.0
pyOpenSSL           23.2.0
pyparsing           3.1.1
pyrsistent          0.20.0
pyserial            3.5
python-apt          2.7.7+ubuntu5
python-dateutil     2.8.2
python-debian       0.1.49+ubuntu2
python-magic        0.4.27
pytz                2024.1
PyYAML              6.0.1
requests            2.31.0
rich                13.7.1
s3transfer          0.10.1
service-identity    24.1.0
setuptools          68.1.2
six                 1.16.0
sos                 4.8.2
ssh-import-id       5.11
systemd-python      235
Twisted             24.3.0
typing_extensions   4.10.0
ubuntu-pro-client   8001
ufw                 0.36.2
unattended-upgrades 0.1
urllib3             2.0.7
wadllib             1.3.6
wheel               0.42.0
zope.interface      6.1
```
  - copy the dependencies to requirements.txt :
  ```bash
  pip freeze > requirements.txt
  ```
  #### Output:
  **requirements.txt file looks like**
  
```
alembic==1.16.4
blinker==1.9.0
click==8.2.1
Flask==3.1.1
Flask-Migrate==4.1.0
Flask-SQLAlchemy==3.1.1
greenlet==3.2.4
iniconfig==2.1.0
itsdangerous==2.2.0
Jinja2==3.1.6
Mako==1.3.10
MarkupSafe==3.0.2
packaging==25.0
pluggy==1.6.0
psycopg2-binary==2.9.10
Pygments==2.19.2
pytest==8.4.1
pytest-dotenv==0.5.2
pytest-flask==1.3.0
python-dotenv==1.1.1
SQLAlchemy==2.0.43
typing_extensions==4.14.1
Werkzeug==3.1.3
gunicorn==23.0.0
```

  **Note:**
  - Going forward if we need any other necessary packages for our application we can install them in our virtual  environment (VENV) and update them in requirements.txt file using the above freeze command. 
  - So that whomever runs reqirements.txt file in their local can easily install the packages along with its  version in their local.
  ```bash
  pip install -r requirements.txt

## Push Files to GitHub

  **1. Initialize Git in our project folder**
  ```bash
  cd /path/to/your/project
  git init
  ```

  **2. Create a .gitignore file**
  
  This will exclude unnecessary files from being tracked (e.g., virtual environment, .env, __pycache__, etc.)
  
  Example `.gitignore`:
  ```bash
  venv/
  __pycache__/
  *.pyc
  .env
  *.db
  *.sqlite3
  .DS_Store
  ```
  - Save this as .gitignore in the root of our project folder.

  **3. Add and Commit Files**

  ```bash
  git add .
  git commit -m "Initial commit - Student CRUD API with Flask"
  ```

  **4. Create a New Repository on GitHub**

  **5. Connect the Local Repo to GitHub**
  ```bash
  git remote add origin https://github.com/your-username/student-crud-api.git
  ```

  **6. Push the Code**
  ```bash
  git branch -M main
  git push -u origin main
  ```
