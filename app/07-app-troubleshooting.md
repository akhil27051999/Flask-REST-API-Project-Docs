## Postgres Connection Troubleshooting

### 1. Check .env values

- Local: POSTGRES_HOST=localhost
- Docker: POSTGRES_HOST=postgres
- Wrong host is the most common cause of could not translate host name "postgres" errors.

### 2. Verify container names / services

- Run docker ps and make sure the container name (or service name in docker-compose.yml) matches the host you’re using in .env.

### 3. Check if Postgres is running

- Local: sudo systemctl status postgresql
- Docker: docker logs <postgres_container>

### 4. Confirm DB is reachable

- Local: psql -U postgres -d studentdb -h localhost -p 5432
- Docker: docker exec -it <postgres_container> psql -U postgres -d studentdb

## Flask App Troubleshooting

### 1. Ensure Flask picks the right .env

- Add a debug log in your Config class to print SQLALCHEMY_DATABASE_URI.

- Example:
  ```sql
  print("DB URI ->", SQLALCHEMY_DATABASE_URI)
  ```

### 2. Check migrations & tables

- Run:
  ```sql
  flask db upgrade
  ```
- Then verify tables inside Postgres with \dt.

### 3. Blueprints & routes

- Make sure your blueprint uses @student_bp.route('/') and you register with url_prefix='/students'.

- Expected endpoints:
  - GET /students/
  - POST /students/
  - GET /students/<id>

### 4. Test with curl

- Health check: curl http://127.0.0.1:5000/health
- Students: curl http://127.0.0.1:5000/students/
