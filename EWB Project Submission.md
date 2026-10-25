# EWB Internship Final Project: Student Management API

This document contains the complete project structure and all source code files required for your EWB internship submission today. You can copy each file into your local project directory.

---

## 🗂️ Project Directory Structure

Ensure your project folder is organized exactly like this:

```text
student_management/
│
├── .env
├── requirements.txt
├── README.md
├── main.py
└── app/
    ├── __init__.py
    ├── config.py
    ├── database.py
    ├── models.py
    ├── schemas.py
    └── routes/
        ├── __init__.py
        ├── create_student.py
        ├── get_student.py
        ├── update_student.py
        ├── delete_student.py
        └── ai_query.py
```

---

## 📄 File Contents

### 1. `requirements.txt`
```text
fastapi==0.110.0
uvicorn==0.28.0
sqlalchemy==2.0.28
pydantic==2.6.4
pydantic-settings==2.2.1
langchain-ollama==0.1.0
```

### 2. `.env`
```env
DATABASE_URL=sqlite:///./students.db
```

### 3. `app/__init__.py`
*(Leave this file completely empty)*

### 4. `app/config.py`
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "sqlite:///./students.db"

    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
        "extra": "ignore"
    }

settings = Settings()
```

### 5. `app/database.py`
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from app.config import settings

engine = create_engine(
    settings.database_url, connect_args={"check_same_thread": False}
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 6. `app/models.py`
```python
from sqlalchemy import Column, Integer, String
from app.database import Base

class Student(Base):
    __tablename__ = "students"

    id = Column(Integer, primary_key=True, index=True, autoincrement=True)
    student_name = Column(String, nullable=False)
    dob = Column(String, nullable=False)
    email = Column(String, unique=True, index=True, nullable=False)
    phone = Column(String, nullable=True)
    course = Column(String, nullable=True)
    address = Column(String, nullable=True)
    enrollment_date = Column(String, nullable=True)
```

### 7. `app/schemas.py`
```python
from pydantic import BaseModel, EmailStr
from typing import Optional

class StudentCreate(BaseModel):
    student_name: str
    dob: str
    email: EmailStr
    phone: Optional[str] = None
    course: Optional[str] = None
    address: Optional[str] = None
    enrollment_date: Optional[str] = None

class StudentUpdate(BaseModel):
    student_name: Optional[str] = None
    dob: Optional[str] = None
    email: Optional[EmailStr] = None
    phone: Optional[str] = None
    course: Optional[str] = None
    address: Optional[str] = None
    enrollment_date: Optional[str] = None

class StudentResponse(StudentCreate):
    id: int

    class Config:
        from_attributes = True
```

### 8. `app/routes/__init__.py`
*(Leave this file completely empty)*

### 9. `app/routes/create_student.py`
```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from app.database import get_db
from app.models import Student
from app.schemas import StudentCreate, StudentResponse

router = APIRouter(prefix="/students", tags=["Students"])

@router.post("/", response_model=StudentResponse, status_code=status.HTTP_201_CREATED)
def create_student(payload: StudentCreate, db: Session = Depends(get_db)):
    existing_student = db.query(Student).filter(Student.email == payload.email).first()
    if existing_student:
        raise HTTPException(status_code=400, detail="Student with this email already exists.")
    
    new_student = Student(**payload.model_dump())
    db.add(new_student)
    db.commit()
    db.refresh(new_student)
    return new_student
```

### 10. `app/routes/get_student.py`
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List, Optional
from app.database import get_db
from app.models import Student
from app.schemas import StudentResponse

router = APIRouter(prefix="/students", tags=["Students"])

@router.get("/", response_model=List[StudentResponse])
def list_students(course: Optional[str] = None, db: Session = Depends(get_db)):
    query = db.query(Student)
    if course:
        query = query.filter(Student.course.ilike(f"%{course}%"))
    return query.all()

@router.get("/{student_id}", response_model=StudentResponse)
def get_student(student_id: int, db: Session = Depends(get_db)):
    student = db.query(Student).filter(Student.id == student_id).first()
    if not student:
        raise HTTPException(status_code=404, detail="Student not found")
    return student
```

### 11. `app/routes/update_student.py`
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from app.database import get_db
from app.models import Student
from app.schemas import StudentUpdate, StudentResponse

router = APIRouter(prefix="/students", tags=["Students"])

@router.put("/{student_id}", response_model=StudentResponse)
def update_student(student_id: int, payload: StudentUpdate, db: Session = Depends(get_db)):
    student = db.query(Student).filter(Student.id == student_id).first()
    if not student:
        raise HTTPException(status_code=404, detail="Student not found")

    update_data = payload.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(student, key, value)

    db.commit()
    db.refresh(student)
    return student
```

### 12. `app/routes/delete_student.py`
```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from app.database import get_db
from app.models import Student

router = APIRouter(prefix="/students", tags=["Students"])

@router.delete("/{student_id}", status_code=status.HTTP_200_OK)
def delete_student(student_id: int, db: Session = Depends(get_db)):
    student = db.query(Student).filter(Student.id == student_id).first()
    if not student:
        raise HTTPException(status_code=404, detail="Student not found")

    db.delete(student)
    db.commit()
    return {"status": "success", "message": f"Student with ID {student_id} deleted successfully."}
```

### 13. `app/routes/ai_query.py`
```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from langchain_ollama import ChatOllama
from app.database import get_db
from app.models import Student

router = APIRouter(prefix="/students", tags=["AI Extension"])

llm = ChatOllama(model="llama3.2")

SYSTEM_PROMPT = """
You are a helpful assistant for a student management database system. 
Use the provided student database records to answer the user's question accurately, professionally, and concisely.
"""

@router.post("/query-ai")
def query_local_ai(user_question: str, db: Session = Depends(get_db)):
    students = db.query(Student).all()
    db_records = "\n".join([
        f"ID: {s.id}, Name: {s.student_name}, DOB: {s.dob}, Email: {s.email}, Course: {s.course}, Enrolled: {s.enrollment_date}"
        for s in students
    ])
    
    prompt = f"{SYSTEM_PROMPT}\n\nDatabase Records:\n{db_records}\n\nUser Question: {user_question}"
    response = llm.invoke(prompt)
    
    return {
        "status": "success",
        "question": user_question,
        "response": response.content
    }
```

### 14. `main.py`
```python
from fastapi import FastAPI
from app.database import engine, Base
from app.routes import create_student, get_student, update_student, delete_student, ai_query

Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="EWB Student Management & AI API",
    description="Production-grade FastAPI backend with SQLite persistence and optional Ollama AI integration.",
    version="1.0.0"
)

app.include_router(create_student.router)
app.include_router(get_student.router)
app.include_router(update_student.router)
app.include_router(delete_student.router)
app.include_router(ai_query.router)

@app.get("/")
def read_root():
    return {
        "message": "Welcome to the EWB Student Management API!",
        "documentation": "Visit /docs for interactive Swagger UI."
    }
```

### 15. `README.md`
```markdown
# EWB Student Management Backend API

## Overview
A production-ready RESTful API built with **FastAPI**, **SQLAlchemy**, **Pydantic**, and **SQLite** featuring complete CRUD operations and an optional local AI query extension powered by **Ollama (Llama 3.2)**.

## Prerequisites
- Python 3.10+
- Ollama (optional, for local AI extension)

## Installation & Setup
1. Clone the repository or extract the project folder.
2. Create and activate a virtual environment:
   ```bash
   python -m venv venv
   .\venv\Scripts\Activate.ps1
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

## Running the Application
Start the development server using Uvicorn:
```bash
python -m uvicorn main:app --reload
```

## API Documentation
Interactive API docs are available via Swagger UI at:
👉 `http://127.0.0.1:8000/docs`