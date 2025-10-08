## API Endpoints

**File Overview**

This file defines a Flask Blueprint named student_bp that handles CRUD operations for Student records in the database using SQLAlchemy. It provides endpoints to create, read, update, and delete students.

#### Blueprint Initialization
```python
from flask import Blueprint, request, jsonify
from app.models import db, Student

student_bp = Blueprint('student', __name__)
```

- student_bp is a Flask Blueprint for student-related routes.
- Allows modular organization of routes for the student module.

---

**1. Add a Student**
```python
@student_bp.route('', methods=['POST'])
def add_student():
    data = request.get_json()
    if not data or not all(key in data for key in ('name', 'domain', 'gpa', 'email')):
        return jsonify({"error": "Missing data"}), 400
        # Validate required fields
    
    # Create a new student instance

    new_student = Student(
        name=data['name'],
        domain=data['domain'],
        gpa=data['gpa'],
        email=data['email']
    )
    
    db.session.add(new_student)
    db.session.commit()
    
    return jsonify({"message": "Student added successfully!"}), 201
```

- Endpoint: POST /students
- Receives JSON data with name, domain, gpa, and email.
- Validates that all required fields exist.
- Creates a new Student object and adds it to the database.
- Returns 201 if successful, or 400 if data is missing.

---

**2. Get All Students**
```python
@student_bp.route('', methods=['GET'])
def get_students():
    students = Student.query.all()
    student_list = [
        {
            "id": student.id,
            "name": student.name,
            "domain": student.domain,
            "gpa": student.gpa,
            "email": student.email
        } for student in students
    ]
    
    return jsonify(student_list), 200

```

- Endpoint: GET /students
- Fetches all student records from the database.
- Returns a JSON array of students with fields: id, name, domain, gpa, email.
- Returns 200 status code.

---

**3. Get Student by ID**
```python
@student_bp.route('<int:student_id>', methods=['GET'])    
def get_student(student_id):
    student = Student.query.get_or_404(student_id)
    
    return jsonify({
        "id": student.id,
        "name": student.name,
        "domain": student.domain,
        "gpa": student.gpa,
        "email": student.email
    }), 200
```

- Endpoint: GET /students/<student_id>
- Retrieves a single student by their ID using get_or_404.
- Returns student details as JSON.
- Returns 404 if student does not exist.

---

**4. Update Student**
```python
@student_bp.route('<int:student_id>', methods=['PUT'])
def update_student(student_id):
    student = Student.query.get_or_404(student_id)
    data = request.get_json()
    
    if not data or not any(key in data for key in ('name', 'domain', 'gpa', 'email')):
        return jsonify({"error": "No valid fields provided"}), 400

    if 'name' in data:
        student.name = data['name']
    if 'domain' in data:
        student.domain = data['domain']
    if 'gpa' in data:
        student.gpa = data['gpa']
    if 'email' in data:
        student.email = data['email']
    
    db.session.commit()
    
    return jsonify({"message": "Student updated successfully!"}), 200
```

- Endpoint: PUT /students/<student_id>
- Accepts JSON data to update any of name, domain, gpa, or email.
- Validates that at least one field is provided.
- Updates the database record and commits changes.
- Returns 200 if successful, 400 if no valid fields, 404 if student not found.

---

**5. Delete Student**
```python
@student_bp.route('<int:student_id>', methods=['DELETE'])
def delete_student(student_id):
    student = Student.query.get_or_404(student_id)
    
    db.session.delete(student)
    db.session.commit()
    
    return jsonify({"message": "Student deleted successfully!"}), 200   
```

- Endpoint: DELETE /students/<student_id>
- Deletes a student record from the database using get_or_404.
- Returns 200 if deletion is successful, 404 if student not found.

---
### Summary

- This file implements all CRUD operations for the Student model.
- It uses SQLAlchemy ORM for database interaction.

**Error handling:**

- 400 for invalid or missing data
- 404 for missing student records
- Modular structure allows easy integration into a larger Flask application via a Blueprint.
