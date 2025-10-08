## PostgreSQL Setup for Flask (Ubuntu)

**1. Install PostgreSQL**
  ```bash
  sudo apt update
  sudo apt install postgresql postgresql-contrib
  ```

**2. Checking PostgreSQL Is Running**
  ```bash
  sudo systemctl status postgresql
  ```
  - If it's not running:
  ```bash
  sudo systemctl start postgresql
  ```
  - Enable on boot:
  ```bash
  sudo systemctl enable postgresql
  ```

**3. Access the PostgreSQL Shell**
  ```bash
  sudo -u postgres psql
  ```
  - Now you're in the interactive psql shell.

**4. Create a PostgreSQL User**
  ```sql
  CREATE USER postgres WITH PASSWORD 'postgres123';
  ```
  - Replace myuser and mypassword with your own values.

**5. Create a PostgreSQL Database**
  ```sql
  CREATE DATABASE studentdb;
  ```

**6. Grant User Access to the Database**
  ```sql
  GRANT ALL PRIVILEGES ON DATABASE studentdb TO postgres;
  ```

**7. List Users (Roles)**
  ```bash
  sudo -u postgres psql -c "\du"
  ```

**8. Test Login as the New User**
  ```bash
  psql -U postgres -d studentdb -h localhost -W
  ```

**9. List Databases**
  ```bash
  sudo -u postgres psql -c "\l"
  ```

**10. Connect to your database:**
```sql
\c <database_name>
```

**11. Connect directly to db**
```sql
sudo -u postgres psql -d studentdb
```

**12. Fetch details in tables**
```sql
SELECT * FROM students;
```

**12. Exit the `psql` Shell**
  ```sql
  \q
  ```

**13. Install PostgreSQL Driver in Your Flask Virtualenv**
  ```bash
  pip install psycopg2-binary
  ```

**14. Add DB Config in Flask**
  ```bash
  SQLALCHEMY_DATABASE_URI = "postgresql://postgres:postgres123@localhost:5432/studentdb"
  SQLALCHEMY_TRACK_MODIFICATIONS = False
  ```
