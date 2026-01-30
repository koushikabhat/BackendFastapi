# FastAPI Project - Development Guidelines

## Critical Constraints

### Database
- **NEVER** create new database connections
- **ALWAYS** use `database_manager.py` connection pool
- **NEVER** hardcode database names
- Add new databases: `.env` → `database_manager.py` enum → accessor
```python
from database_manager import get_db, DatabaseName, db
# Use: db.school.collection_name or get_db(DatabaseName.SCHOOL)
```

### Authentication
- **ALWAYS** use JWT auth from `app.middleware.auth`
- Required: `jwt_bearer_required` + `require_authenticated_user`
- Optional: `jwt_bearer_optional` or `get_optional_user`
- Access user: `request.state.user` or via dependency

### Configuration
- **NEVER** hardcode: database names, API keys, URLs, credentials
- **ALWAYS** use `.env` variables with `os.getenv()`

## Project Structure

```
app/
├── api/        # HTTP handlers only - call services
├── schemas/    # Pydantic models (request/response)
├── services/   # Business logic + DB operations
├── utils/      # Reusable utilities (CHECK HERE FIRST!)
├── middleware/ # Auth, etc.
└── core/       # Security, config
```

**Pattern:**
- APIs: Handle HTTP, validate, call services
- Services: Business logic, database operations
- Schemas: Input/output validation
- Utils: Common code (date, string, validation, response)

## Code Quality Requirements

1. **Production-grade:** Error handling, type hints, docstrings, input validation
2. **Performance:** Async/await, pagination (limit=20, max=100), projections, indexes, aggregation pipelines
3. **Security:** JWT auth, input sanitization, no injection vulnerabilities, no sensitive data in logs
4. **Clean Code:** Remove unused imports/code/comments, consolidate duplicates
5. **Logging:** Use comprehensive logging from `StructuredLogger`, remove debug/info logs after development, keep only ERROR/WARNING/critical INFO
6. **Testing:** Test EVERY new API with 10+ edge cases (valid, missing fields, invalid types, boundaries, auth failures, DB errors, duplicates, large payloads, concurrent requests, injection attempts)
7. **Code Reusability**: Use or create shared utilities as much as possble
8. **Maintainability**: Each module/file < 600 lines (recommended best practice)
9. **API Parameters:** NEVER use URL query parameters. ALWAYS use POST with JSON body and Pydantic schemas
10. **Index Management:** NEVER create indexes directly in services. ALWAYS define in `app/utils/startup_indexes.py` using IndexManager with version tracking

## Development Checklist (New API)

- [ ] Schema in `app/schemas/`, service in `app/services/`, route in `app/api/`
- [ ] JWT auth applied appropriately
- [ ] Database from `database_manager.py` only
- [ ] Error handling, input validation, type hints
- [ ] Reuse `app/utils/` functions
- [ ] Remove debug logs, unused code
- [ ] 10+ test cases covering edge cases
- [ ] Performance optimized (queries, pagination)

## Quick Reference

**Auth Dependencies:**
- `jwt_bearer_required` - requires valid token
- `require_authenticated_user` - requires authenticated user (not guest)
- `jwt_bearer_optional` - optional auth
- `get_optional_user` - returns user or None


**Common Utils Location:** `app/utils/` - Always check before writing new common code

---

**Core Principles:** Production-ready code. Modular architecture. No hardcoding. Test thoroughly. Optimize performance. Security first.
