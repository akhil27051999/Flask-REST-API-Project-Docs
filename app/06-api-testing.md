## API Endpoint Testing Notes

### 1. Base URL

  Assuming the blueprint is registered with prefix /students:
  ```
  http://localhost:5000/students
  ```

### 2. Endpoints

**a) Add a Student**

  - `Method`: POST
  - `URL`: `/students`
  - `Headers`: Content-Type: application/json

  *Body Example:*
  ```json
  {
    "name": "Alice",
    "domain": "Computer Science",
    "gpa": 3.8,
    "email": "alice@example.com"
  }
  ```

  *Expected Response:*
  ```json
  {"message": "Student added successfully!"}
  ```
  - `Status Code`: 201 Created
  - `Error Cases`: Missing any field → 400 Bad Request

**b) Get All Students**
  - `Method`: GET
  - `URL`: /students
  *Response Example:*
  ```json
  [
    {
        "id": 1,
        "name": "Alice",
        "domain": "Computer Science",
        "gpa": 3.8,
        "email": "alice@example.com"
    }
  ]
  ```
  - `Status Code`: 200 OK

**c) Get Student by ID**
  - `Method`: GET
  - `URL`: /students/<student_id>
  *Response Example:*
  ```json
  {
    "id": 1,
    "name": "Alice",
    "domain": "Computer Science",
    "gpa": 3.8,
    "email": "alice@example.com"
  }
  ```
  - `Status Codes`: 
     - 200 OK – Student found
     - 404 Not Found – Student ID does not exist

**d) Update Student**
  - `Method`: PUT
  - `URL`: /students/<student_id>
  - `Headers`: Content-Type: application/json

  *Body Example:* (update one or more fields)
  ```json
  {
    "gpa": 3.9,
    "domain": "Data Science"
  }
  ```
  *Expected Response:*
  ```json
  {"message": "Student updated successfully!"}
  ```
  - `Status Codes`:
    - 200 OK – Update successful
    - 400 Bad Request – No valid fields provided
    - 404 Not Found – Student ID does not exist

**e) Delete Student**
  - `Method`: DELETE
  - `URL`: /students/<student_id>
  *Expected Response:*
  ```json
  {"message": "Student deleted successfully!"}
  ```
  - `Status Codes`:
    - 200 OK – Deletion successful
    - 404 Not Found – Student ID does not exist

### 3. Testing Tools

  - Postman – GUI for sending requests and viewing responses
  - cURL – Command-line testing

*Example cURL for adding a student:*
  ```bash
  curl -X POST http://localhost:5000/students \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","domain":"CS","gpa":3.8,"email":"alice@example.com"}'
  ```
