# FastAPI Project - Development Guidelines (Enhanced)

## 📑 Quick Navigation
- [Critical Constraints](#critical-constraints)
- [Project Architecture](#project-architecture)
- [Development Workflow](#development-workflow)
- [Code Quality Standards](#code-quality-standards)
- [Performance Guidelines](#performance-guidelines)
- [Security Checklist](#security-checklist)
- [Testing Strategy](#testing-strategy)
- [Common Patterns Library](#common-patterns-library)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## Critical Constraints

### Database
**NEVER** create new database connections. **ALWAYS** use `database_manager.py` connection pool.

```python
# ✅ CORRECT - Use centralized database manager
from database_manager import get_db, DatabaseName, db

# Option 1: Direct accessor (preferred)
students = await db.school.school_students.find_one({"_id": student_id})

# Option 2: Enum-based getter
school_db = get_db(DatabaseName.SCHOOL)
students = await school_db.school_students.find_one({"_id": student_id})

# ❌ WRONG - Never create new connections
from motor.motor_asyncio import AsyncIOMotorClient
client = AsyncIOMotorClient(...)  # FORBIDDEN!
```

**Adding New Databases:**
1. Add to `.env`: `DATABASE_NAME_MYDB=mydb_name`
2. Add to `DatabaseName` enum in `database_manager.py`
3. Add property to `DatabaseAccessor` class
4. Use: `db.mydb.collection_name`

### Authentication
**ALWAYS** use JWT auth from `app.middleware.auth`.

```python
# ✅ CORRECT - Required auth for protected endpoints
from app.middleware.auth import jwt_bearer_required, require_authenticated_user

@router.post("/protected")
async def protected_endpoint(
    request: Request,
    token_data: Dict = Depends(jwt_bearer_required),
    user: Dict = Depends(require_authenticated_user)
):
    user_id = user["userId"]  # Access from request.state.user
    user_type = user["userType"]
    # ... business logic

# ✅ CORRECT - Optional auth for public endpoints
from app.middleware.auth import jwt_bearer_optional, get_optional_user

@router.get("/public")
async def public_endpoint(
    token_data: Dict = Depends(jwt_bearer_optional),
    user: Optional[Dict] = Depends(get_optional_user)
):
    if user:
        # Personalized response for authenticated users
        pass
    else:
        # Generic response for guests
        pass

# ❌ WRONG - Manual token validation
authorization = request.headers.get("Authorization")  # FORBIDDEN!
```

### Configuration Management
**NEVER** hardcode: database names, API keys, URLs, secrets, credentials.

```python
# ✅ CORRECT - Environment variables
import os
from functools import lru_cache

@lru_cache()
def get_settings():
    return {
        "api_key": os.getenv("OPENAI_API_KEY"),
        "s3_bucket": os.getenv("AWS_S3_BUCKET"),
        "redis_url": os.getenv("REDIS_URL"),
        "environment": os.getenv("ENVIRONMENT", "development"),
    }

# ❌ WRONG - Hardcoded values
API_KEY = "sk-xxx"  # FORBIDDEN!
DATABASE = "schoolsIndia"  # FORBIDDEN!
```

---

## Project Architecture

### Directory Structure
```
FastAPI/
├── app/
│   ├── api/              # HTTP route handlers (thin layer)
│   │   ├── school/       # School-related endpoints
│   │   ├── teacher/      # Teacher-related endpoints
│   │   └── student/      # Student-related endpoints
│   ├── services/         # Business logic + DB operations (fat layer)
│   │   ├── school_services/
│   │   ├── teacher_services/
│   │   └── student_services/
│   ├── schemas/          # Pydantic models (validation)
│   │   ├── school_schemas/
│   │   ├── teacher/
│   │   └── student/
│   ├── utils/            # Reusable utilities (CHECK HERE FIRST!)
│   │   ├── phone_utils.py
│   │   ├── storage_quota_mongo.py
│   │   ├── structured_logger.py
│   │   ├── s3_utils.py
│   │   └── ...
│   ├── middleware/       # Request/response processing
│   │   └── auth.py
│   ├── core/             # Core functionality
│   │   ├── security.py
│   │   └── encryption.py
│   └── constants/        # Application constants
├── database_manager.py   # Centralized DB connection pool
├── tests/                # Test suite
├── migrations/           # Database migrations
└── docs/                 # Documentation

```

### Layer Responsibilities

#### 1. **API Layer** (`app/api/`)
**Purpose:** HTTP handling ONLY. Keep it thin (< 50 lines per endpoint).

```python
# ✅ CORRECT - Thin API layer
@router.post("/students", response_model=StudentResponse)
async def create_student(
    request: CreateStudentRequest,
    token_data: Dict = Depends(jwt_bearer_required),
    user: Dict = Depends(require_authenticated_user)
):
    """Create a new student record"""
    try:
        result = await student_service.create_student(
            data=request.dict(),
            created_by=user["userId"]
        )
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        logger.error("Student creation failed", error=str(e))
        raise HTTPException(status_code=500, detail="Internal server error")

# ❌ WRONG - Fat API layer with business logic
@router.post("/students")
async def create_student(request: CreateStudentRequest):
    # Validating phone number (should be in service)
    if not phone_utils.validate_phone(request.phone):
        raise HTTPException(400, "Invalid phone")

    # Database operations (should be in service)
    existing = await db.school.students.find_one({"phone": request.phone})
    if existing:
        raise HTTPException(409, "Student exists")

    # Complex business logic (should be in service)
    student_id = generate_student_id(request.class_name)
    await db.school.students.insert_one({...})
    # ... 100+ lines of logic (WRONG!)
```

#### 2. **Service Layer** (`app/services/`)
**Purpose:** Business logic, database operations, external API calls.

```python
# ✅ CORRECT - Service with business logic
from app.utils.phone_utils import validate_and_normalize_phone
from app.utils.structured_logger import StructuredLogger
from database_manager import db

logger = StructuredLogger("student_service", "create_student")

async def create_student(data: Dict, created_by: str) -> Dict:
    """
    Create a new student with validation and business rules

    Args:
        data: Student data dictionary
        created_by: User ID of creator

    Returns:
        Created student document

    Raises:
        ValueError: If validation fails
        DuplicateStudentError: If student already exists
    """
    # Validation
    phone = validate_and_normalize_phone(data["phone"])

    # Check duplicates
    existing = await db.school.school_students.find_one(
        {"phone": phone},
        {"_id": 1}
    )
    if existing:
        logger.warning("Duplicate student", extra={"phone": phone})
        raise DuplicateStudentError(f"Student with phone {phone} exists")

    # Generate student ID
    student_id = await generate_student_id(data["class_id"])

    # Prepare document
    student_doc = {
        "_id": student_id,
        "phone": phone,
        "name": data["name"],
        "class_id": data["class_id"],
        "created_by": created_by,
        "created_at": datetime.utcnow(),
        "updated_at": datetime.utcnow(),
        "is_active": True
    }

    # Insert
    result = await db.school.school_students.insert_one(student_doc)

    logger.info("Student created", extra={
        "student_id": student_id,
        "class_id": data["class_id"]
    })

    return student_doc
```

#### 3. **Schema Layer** (`app/schemas/`)
**Purpose:** Request/response validation with Pydantic.

```python
# ✅ CORRECT - Comprehensive schemas with validation
from pydantic import BaseModel, Field, validator
from typing import Optional
from datetime import datetime

class CreateStudentRequest(BaseModel):
    """Request schema for creating a student"""
    name: str = Field(..., min_length=2, max_length=100)
    phone: str = Field(..., regex=r"^\+?[1-9]\d{9,14}$")
    class_id: str = Field(..., min_length=1)
    email: Optional[str] = Field(None, regex=r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
    date_of_birth: Optional[datetime] = None

    @validator("name")
    def validate_name(cls, v):
        if not v.strip():
            raise ValueError("Name cannot be empty")
        return v.strip().title()

    class Config:
        schema_extra = {
            "example": {
                "name": "Rahul Kumar",
                "phone": "+919876543210",
                "class_id": "10-A",
                "email": "rahul@example.com"
            }
        }

class StudentResponse(BaseModel):
    """Response schema for student data"""
    id: str = Field(..., alias="_id")
    name: str
    phone: str
    class_id: str
    is_active: bool
    created_at: datetime

    class Config:
        allow_population_by_field_name = True
```

#### 4. **Utils Layer** (`app/utils/`)
**Purpose:** Reusable, domain-agnostic utilities.

**ALWAYS CHECK UTILS FIRST** before writing new code!

Current utilities:
- `phone_utils.py` - Phone validation/normalization
- `structured_logger.py` - Production logging
- `storage_quota_mongo.py` - Storage management
- `s3_utils.py` - AWS S3 operations
- `background_upload_manager.py` - Background tasks
- `rate_limiter.py` - Rate limiting
- `time_utils.py` - Date/time utilities
- `academic_year_helper.py` - Academic year logic

```python
# ✅ CORRECT - Reusable utility function
# File: app/utils/id_generator.py
from typing import Optional
import random
import string

async def generate_unique_id(
    collection,
    prefix: str,
    length: int = 10,
    max_retries: int = 5
) -> str:
    """
    Generate unique ID with collision detection

    Args:
        collection: MongoDB collection to check
        prefix: ID prefix (e.g., "STU", "TCH")
        length: ID length (default: 10)
        max_retries: Maximum collision retries

    Returns:
        Unique ID string
    """
    for _ in range(max_retries):
        id_suffix = ''.join(random.choices(string.ascii_uppercase + string.digits, k=length))
        unique_id = f"{prefix}{id_suffix}"

        existing = await collection.find_one({"_id": unique_id}, {"_id": 1})
        if not existing:
            return unique_id

    raise RuntimeError(f"Failed to generate unique ID after {max_retries} attempts")
```

---

## Code Quality Standards

### 1. Type Hints (Required)
```python
# ✅ CORRECT - Full type hints
from typing import Dict, List, Optional, Union
from datetime import datetime

async def get_students(
    class_id: str,
    is_active: bool = True,
    limit: int = 20,
    offset: int = 0
) -> Dict[str, Union[List[Dict], int]]:
    """Get students with pagination"""
    pass

# ❌ WRONG - No type hints
async def get_students(class_id, is_active=True, limit=20):
    pass
```

### 2. Docstrings (Required for Services)
```python
# ✅ CORRECT - Comprehensive docstring
async def create_assignment(
    title: str,
    content_items: List[Dict],
    student_ids: List[str],
    teacher_id: str
) -> Dict:
    """
    Create and assign homework to students

    This function:
    1. Validates content items exist
    2. Creates assignment document
    3. Creates individual student assignments
    4. Sends notifications

    Args:
        title: Assignment title (max 200 chars)
        content_items: List of questions/flashcards
        student_ids: Target student IDs
        teacher_id: Creating teacher's ID

    Returns:
        {
            "assignment_id": str,
            "assigned_count": int,
            "failed_count": int
        }

    Raises:
        ValueError: If content_items is empty
        ValidationError: If content items don't exist

    Example:
        >>> result = await create_assignment(
        ...     title="Math Chapter 5",
        ...     content_items=[{"content_id": "8MA501q01", "marks": 2}],
        ...     student_ids=["STU123", "STU456"],
        ...     teacher_id="TCH789"
        ... )
    """
    pass
```

### 3. Error Handling (Required)
```python
# ✅ CORRECT - Specific exception handling
from pymongo.errors import DuplicateKeyError, PyMongoError

async def create_student(data: Dict) -> Dict:
    try:
        result = await db.school.school_students.insert_one(data)
        return {"success": True, "student_id": data["_id"]}

    except DuplicateKeyError:
        logger.warning("Duplicate student", extra={"student_id": data["_id"]})
        raise ValueError(f"Student {data['_id']} already exists")

    except PyMongoError as e:
        logger.error("Database error creating student", error=str(e))
        raise DatabaseError("Failed to create student") from e

    except Exception as e:
        logger.error("Unexpected error", error=str(e), exc_info=True)
        raise

# ❌ WRONG - Bare except or too broad
try:
    result = await db.school.students.insert_one(data)
except:  # FORBIDDEN!
    pass

try:
    result = await db.school.students.insert_one(data)
except Exception:  # Too broad
    return {"error": "Something went wrong"}
```

### 4. Input Validation (Required)
```python
# ✅ CORRECT - Validation with clear errors
def validate_class_id(class_id: str) -> str:
    """Validate and normalize class ID"""
    if not class_id or not class_id.strip():
        raise ValueError("class_id cannot be empty")

    class_id = class_id.strip().upper()

    if not re.match(r'^[1-9][0-2]?-[A-Z]$', class_id):
        raise ValueError(
            f"Invalid class_id format: {class_id}. "
            "Expected format: '10-A', '12-B', etc."
        )

    return class_id

# ❌ WRONG - No validation
def create_student(class_id):
    # Direct database insert without validation
    await db.school.students.insert_one({"class_id": class_id})
```

### 5. Logging Standards
```python
from app.utils.structured_logger import StructuredLogger

logger = StructuredLogger("attendance", "mark_attendance")

# ✅ CORRECT - Structured logging with context
logger.info("Attendance marked", extra={
    "students_count": 30,
    "class_id": "10-A",
    "teacher_id": teacher_id,
    "date": date.isoformat(),
    "present": 28,
    "absent": 2
})

logger.error("Failed to mark attendance",
    error=str(e),
    extra={
        "class_id": "10-A",
        "teacher_id": teacher_id
    }
)

# ❌ WRONG - Unstructured logging
print(f"Attendance marked for class 10-A")  # FORBIDDEN!
logger.info("Attendance marked")  # Missing context
logger.debug("Student list: " + str(students))  # Don't log sensitive data
```

**Logging Levels:**
- **ERROR**: Production issues requiring immediate attention
- **WARNING**: Potential issues (rate limits, retries, deprecated usage)
- **INFO**: Key business events (user actions, important state changes)
- **DEBUG**: Remove before production (development/troubleshooting only)

---

## Performance Guidelines

### 1. Database Queries

#### Use Projections (Required)
```python
# ✅ CORRECT - Projection to limit fields
students = await db.school.school_students.find(
    {"class_id": "10-A"},
    {"_id": 1, "name": 1, "roll_number": 1}  # Only needed fields
).to_list(length=None)

# ❌ WRONG - Fetching all fields
students = await db.school.school_students.find({"class_id": "10-A"}).to_list(length=None)
```

#### Pagination (Required for Lists)
```python
# ✅ CORRECT - Pagination with defaults
async def get_students(
    class_id: str,
    limit: int = 20,
    offset: int = 0
) -> Dict:
    """Get students with pagination"""
    # Enforce max limit
    limit = min(limit, 100)

    cursor = db.school.school_students.find({"class_id": class_id})

    # Get total count (cached if possible)
    total = await cursor.count()

    # Apply pagination
    students = await cursor.skip(offset).limit(limit).to_list(length=limit)

    return {
        "students": students,
        "total": total,
        "limit": limit,
        "offset": offset,
        "has_more": (offset + limit) < total
    }

# ❌ WRONG - No pagination
students = await db.school.school_students.find({"class_id": "10-A"}).to_list(length=None)
```

#### Use Aggregation Pipelines
```python
# ✅ CORRECT - Aggregation for complex queries
pipeline = [
    {"$match": {"class_id": "10-A", "is_active": True}},
    {"$lookup": {
        "from": "school_classes",
        "localField": "class_id",
        "foreignField": "_id",
        "as": "class_info"
    }},
    {"$unwind": "$class_info"},
    {"$project": {
        "_id": 1,
        "name": 1,
        "class_name": "$class_info.name",
        "section": "$class_info.section"
    }},
    {"$limit": 20}
]

students = await db.school.school_students.aggregate(pipeline).to_list(length=None)
```

#### Database Indexes (Required)
```python
# Create indexes in migration scripts
await db.school.school_students.create_index([("class_id", 1), ("is_active", 1)])
await db.school.school_students.create_index([("phone", 1)], unique=True)
await db.school.school_students.create_index([("created_at", -1)])
```

### 2. Async/Await (Required)
```python
# ✅ CORRECT - Proper async/await
async def get_student_data(student_id: str) -> Dict:
    student = await db.school.school_students.find_one({"_id": student_id})
    if not student:
        raise ValueError("Student not found")

    # Parallel fetch
    assignments, attendance = await asyncio.gather(
        get_assignments(student_id),
        get_attendance(student_id)
    )

    return {
        "student": student,
        "assignments": assignments,
        "attendance": attendance
    }

# ❌ WRONG - Sequential awaits (slower)
async def get_student_data(student_id: str):
    student = await db.school.students.find_one({"_id": student_id})
    assignments = await get_assignments(student_id)  # Waits unnecessarily
    attendance = await get_attendance(student_id)    # Waits unnecessarily
    return {student, assignments, attendance}
```

### 3. Caching Strategy
```python
from functools import lru_cache
import redis.asyncio as redis

# In-memory cache for static data
@lru_cache(maxsize=128)
def get_class_list() -> List[str]:
    return ["1-A", "1-B", ..., "12-A", "12-B"]

# Redis cache for dynamic data
async def get_student_cached(student_id: str) -> Optional[Dict]:
    cache_key = f"student:{student_id}"

    # Try cache first
    cached = await redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # Fetch from DB
    student = await db.school.school_students.find_one({"_id": student_id})
    if student:
        # Cache for 5 minutes
        await redis_client.setex(cache_key, 300, json.dumps(student, default=str))

    return student
```

---

## Security Checklist

### 1. Input Sanitization (Required)
```python
# ✅ CORRECT - Sanitize user inputs
import re

def sanitize_search_query(query: str) -> str:
    """Sanitize search query to prevent injection"""
    # Remove MongoDB operators
    query = re.sub(r'[\$\{\}]', '', query)
    # Limit length
    return query[:100].strip()

# Use in queries
search = sanitize_search_query(user_input)
results = await db.school.students.find({"name": {"$regex": search, "$options": "i"}})
```

### 2. SQL/NoSQL Injection Prevention
```python
# ✅ CORRECT - Parameterized queries
user_input = request.query_params.get("student_id")
student = await db.school.students.find_one({"_id": user_input})

# ❌ WRONG - String concatenation (vulnerable)
query = f'{{"_id": "{user_input}"}}'  # DANGEROUS!
student = await db.school.students.find_one(eval(query))
```

### 3. Sensitive Data Handling
```python
# ✅ CORRECT - Never log sensitive data
logger.info("User login", extra={
    "user_id": user["_id"],
    "login_method": "otp"
    # ❌ DON'T: "phone": user["phone"], "otp": otp
})

# ✅ CORRECT - Hash sensitive data
import hashlib

def hash_phone(phone: str) -> str:
    return hashlib.sha256(phone.encode()).hexdigest()

# ❌ WRONG - Logging sensitive data
logger.info(f"OTP sent: {otp} to {phone}")  # FORBIDDEN!
```

### 4. Rate Limiting
```python
from app.utils.rate_limiter import RateLimiter

rate_limiter = RateLimiter(max_requests=100, window_seconds=60)

@router.post("/send-otp")
async def send_otp(phone: str):
    # Check rate limit
    if not await rate_limiter.check(f"otp:{phone}"):
        raise HTTPException(429, "Too many requests. Try again later.")

    # Send OTP
    await send_otp_service(phone)
```

---

## Testing Strategy

### Test Coverage Requirements
- **API Endpoints**: 10+ test cases per endpoint
- **Services**: Unit tests for business logic
- **Utils**: 100% coverage for reusable utilities
- **Integration**: End-to-end tests for critical flows

### Test Case Categories (Minimum 10 per API)
1. ✅ **Happy path** - Valid inputs, successful response
2. ✅ **Missing required fields** - 400 error
3. ✅ **Invalid field types** - 422 validation error
4. ✅ **Invalid field values** - 400 error with message
5. ✅ **Boundary conditions** - Min/max values, empty lists
6. ✅ **Authentication failures** - No token, invalid token, expired token
7. ✅ **Authorization failures** - Wrong user type, insufficient permissions
8. ✅ **Database errors** - Connection failures, constraint violations
9. ✅ **Duplicate entries** - 409 conflict
10. ✅ **Large payloads** - Stress testing
11. ✅ **Concurrent requests** - Race conditions
12. ✅ **Injection attempts** - SQL/NoSQL injection, XSS

### Test Example
```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

class TestCreateStudent:
    """Test suite for POST /students"""

    def test_create_student_success(self):
        """Test successful student creation"""
        response = client.post("/students", json={
            "name": "Rahul Kumar",
            "phone": "+919876543210",
            "class_id": "10-A"
        }, headers={"Authorization": f"Bearer {valid_token}"})

        assert response.status_code == 201
        assert response.json()["name"] == "Rahul Kumar"

    def test_create_student_missing_name(self):
        """Test missing required field"""
        response = client.post("/students", json={
            "phone": "+919876543210",
            "class_id": "10-A"
        }, headers={"Authorization": f"Bearer {valid_token}"})

        assert response.status_code == 422

    def test_create_student_invalid_phone(self):
        """Test invalid phone format"""
        response = client.post("/students", json={
            "name": "Test",
            "phone": "invalid",
            "class_id": "10-A"
        }, headers={"Authorization": f"Bearer {valid_token}"})

        assert response.status_code == 400
        assert "phone" in response.json()["detail"].lower()

    def test_create_student_duplicate(self):
        """Test duplicate student creation"""
        # Create first student
        client.post("/students", json={...})

        # Try to create again
        response = client.post("/students", json={...})

        assert response.status_code == 409

    def test_create_student_no_auth(self):
        """Test without authentication"""
        response = client.post("/students", json={...})

        assert response.status_code == 401

    # ... 5+ more test cases
```

---

## Common Patterns Library

### Pattern 1: CRUD Service Template
```python
# File: app/services/base_crud_service.py
from typing import Dict, List, Optional, TypeVar, Generic
from datetime import datetime
from motor.motor_asyncio import AsyncIOMotorCollection

T = TypeVar('T')

class BaseCRUDService(Generic[T]):
    """Base CRUD service with common operations"""

    def __init__(self, collection: AsyncIOMotorCollection, id_prefix: str):
        self.collection = collection
        self.id_prefix = id_prefix

    async def create(self, data: Dict, created_by: str) -> Dict:
        """Create new document"""
        doc = {
            **data,
            "created_by": created_by,
            "created_at": datetime.utcnow(),
            "updated_at": datetime.utcnow(),
            "is_active": True
        }

        await self.collection.insert_one(doc)
        return doc

    async def get_by_id(self, id: str, projection: Optional[Dict] = None) -> Optional[Dict]:
        """Get document by ID"""
        return await self.collection.find_one({"_id": id}, projection)

    async def update(self, id: str, data: Dict, updated_by: str) -> bool:
        """Update document"""
        result = await self.collection.update_one(
            {"_id": id},
            {
                "$set": {
                    **data,
                    "updated_by": updated_by,
                    "updated_at": datetime.utcnow()
                }
            }
        )
        return result.modified_count > 0

    async def delete(self, id: str, deleted_by: str) -> bool:
        """Soft delete document"""
        result = await self.collection.update_one(
            {"_id": id},
            {
                "$set": {
                    "is_active": False,
                    "deleted_by": deleted_by,
                    "deleted_at": datetime.utcnow()
                }
            }
        )
        return result.modified_count > 0

    async def list_paginated(
        self,
        filter: Dict = {},
        limit: int = 20,
        offset: int = 0,
        sort: List[tuple] = [("created_at", -1)]
    ) -> Dict:
        """List documents with pagination"""
        limit = min(limit, 100)

        cursor = self.collection.find(filter).sort(sort)
        total = await self.collection.count_documents(filter)

        items = await cursor.skip(offset).limit(limit).to_list(length=limit)

        return {
            "items": items,
            "total": total,
            "limit": limit,
            "offset": offset,
            "has_more": (offset + limit) < total
        }
```

### Pattern 2: Response Wrapper
```python
# File: app/utils/response_utils.py
from typing import Any, Optional
from fastapi.responses import JSONResponse

def success_response(
    data: Any,
    message: str = "Success",
    status_code: int = 200
) -> JSONResponse:
    """Standard success response"""
    return JSONResponse(
        status_code=status_code,
        content={
            "success": True,
            "message": message,
            "data": data
        }
    )

def error_response(
    message: str,
    error_code: str,
    status_code: int = 400,
    details: Optional[Dict] = None
) -> JSONResponse:
    """Standard error response"""
    return JSONResponse(
        status_code=status_code,
        content={
            "success": False,
            "error_code": error_code,
            "message": message,
            "details": details or {}
        }
    )
```

### Pattern 3: Dependency Injection
```python
# File: app/dependencies/common.py
from fastapi import Depends, Query
from typing import Dict

async def get_pagination_params(
    limit: int = Query(20, ge=1, le=100),
    offset: int = Query(0, ge=0)
) -> Dict[str, int]:
    """Reusable pagination parameters"""
    return {"limit": limit, "offset": offset}

# Usage in API
@router.get("/students")
async def list_students(
    pagination: Dict = Depends(get_pagination_params),
    user: Dict = Depends(require_authenticated_user)
):
    return await student_service.list_students(**pagination)
```

---

## Development Workflow

### Pre-Development Checklist
- [ ] Read relevant documentation
- [ ] Check `app/utils/` for existing utilities
- [ ] Understand database schema
- [ ] Review similar existing endpoints
- [ ] Plan schema → service → API structure

### Development Checklist (New API)
- [ ] Create Pydantic schemas in `app/schemas/`
- [ ] Implement service logic in `app/services/`
- [ ] Create API route in `app/api/`
- [ ] Add JWT authentication (required/optional)
- [ ] Use `database_manager` for DB access
- [ ] Add comprehensive error handling
- [ ] Add type hints to all functions
- [ ] Add docstrings to service functions
- [ ] Use structured logger from `app/utils/`
- [ ] Implement pagination for list endpoints
- [ ] Add database indexes if needed
- [ ] Write 10+ test cases
- [ ] Remove debug logs before commit
- [ ] Test edge cases manually

### Code Review Checklist
- [ ] No hardcoded values (DB names, API keys, URLs)
- [ ] Authentication implemented correctly
- [ ] Database accessed via `database_manager` only
- [ ] Proper error handling (specific exceptions)
- [ ] Input validation (Pydantic + manual)
- [ ] Type hints on all functions
- [ ] Docstrings on service functions
- [ ] Structured logging (no print statements)
- [ ] Pagination on list endpoints
- [ ] No sensitive data in logs
- [ ] Async/await used properly
- [ ] No SQL/NoSQL injection vulnerabilities
- [ ] Tests covering edge cases
- [ ] Performance optimized (indexes, projections)

---

## Troubleshooting Guide

### Common Issues

#### 1. Database Connection Errors
```python
# Problem: Multiple database connections
# Solution: Use database_manager

# ❌ WRONG
from motor.motor_asyncio import AsyncIOMotorClient
client = AsyncIOMotorClient(...)

# ✅ CORRECT
from database_manager import db
students = await db.school.school_students.find_one({...})
```

#### 2. Authentication Not Working
```python
# Problem: Wrong auth dependency
# Solution: Use correct auth based on endpoint type

# For protected endpoints
from app.middleware.auth import jwt_bearer_required, require_authenticated_user

@router.post("/protected")
async def endpoint(
    token_data: Dict = Depends(jwt_bearer_required),
    user: Dict = Depends(require_authenticated_user)
):
    pass

# For public endpoints with optional auth
from app.middleware.auth import jwt_bearer_optional, get_optional_user

@router.get("/public")
async def endpoint(
    user: Optional[Dict] = Depends(get_optional_user)
):
    pass
```

#### 3. Pagination Not Working
```python
# Problem: Not implementing pagination
# Solution: Always paginate list endpoints

async def get_students(limit: int = 20, offset: int = 0):
    limit = min(limit, 100)  # Enforce max
    cursor = db.school.students.find({})
    return await cursor.skip(offset).limit(limit).to_list(length=limit)
```

#### 4. Performance Issues
```python
# Problem: Fetching all fields
# Solution: Use projections

# ❌ SLOW
students = await db.school.students.find({}).to_list(length=None)

# ✅ FAST
students = await db.school.students.find(
    {},
    {"_id": 1, "name": 1, "class_id": 1}  # Only needed fields
).limit(20).to_list(length=20)
```

---

## Quick Reference

### Available Databases
`SCHOOL`, `JNVST`, `CURRICULUM`, `CONTENT`, `TEACHER`, `PARENT`, `MEMORIES`, `AUTH`, `NOTIFICATIONS`, `AI_USAGE`, `REFERRALS`, `FEEDBACK`, `SALES`, `PAYMENTS`

### Auth Dependencies
- `jwt_bearer_required` - Requires valid JWT token
- `require_authenticated_user` - Requires authenticated (non-guest) user
- `jwt_bearer_optional` - Optional JWT (public endpoints)
- `get_optional_user` - Returns user dict or None

### Common Utils (CHECK HERE FIRST!)
- Phone: `app/utils/phone_utils.py`
- Logging: `app/utils/structured_logger.py`
- Storage: `app/utils/storage_quota_mongo.py`
- S3: `app/utils/s3_utils.py`
- Time: `app/utils/time_utils.py`
- Rate Limiting: `app/utils/rate_limiter.py`

### Pagination Defaults
- Default limit: 20
- Maximum limit: 100
- Default offset: 0

---

## Core Principles

🎯 **Production-Ready Code** - Error handling, validation, logging
🏗️ **Modular Architecture** - Separation of concerns (API/Service/Schema)
🔒 **Security First** - Auth, sanitization, no sensitive data in logs
⚡ **Performance Optimized** - Pagination, indexes, projections, caching
🧪 **Test Thoroughly** - 10+ test cases per endpoint
📝 **Type Safety** - Type hints everywhere
🚫 **No Hardcoding** - Use environment variables
♻️ **Code Reusability** - Check utils before writing new code
📊 **Structured Logging** - Use StructuredLogger with context
🔍 **Code Quality 8.5+** - Clean, maintainable, documented code

---

**Last Updated:** 2025-11-13
**Version:** 2.0 (Enhanced)
