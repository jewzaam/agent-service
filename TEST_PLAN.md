# Test Plan

> This document describes the testing strategy for this project. It serves as the single source of truth for testing decisions and rationale.

## Overview

**Project:** agent-service
**Primary functionality:** Standalone FastAPI service hosting a Claude SDK agent with WebSocket streaming, multi-client fan-out, mTLS + OIDC auth, and Prometheus metrics.

## Testing Philosophy

This project follows the [Testing Standards](https://github.com/jewzaam/standards/blob/main/python/testing.md).

Key testing principles for this project:

- Unit tests with mocked dependencies (SDK, HTTP, subprocess) for all modules
- Integration tests for WebSocket flows and REST endpoints using FastAPI TestClient
- Each test creates a fresh FastAPI app instance via `create_app(config=...)` for isolation
- conftest.py safety guards block real subprocess, HTTP, urllib, and filesystem writes globally

## Test Categories

### Unit Tests

Tests for isolated functions with mocked dependencies.

| Module | Function | Test Coverage | Notes |
|--------|----------|---------------|-------|
| `config.py` | `AgentServiceConfig()` | Validation, required fields, extra field rejection, constraints | Pydantic model |
| `config.py` | `load_config()` | YAML loading, env var overrides (str, int, bool, list), missing file | |
| `models.py` | `ServerMessage`, `EventMessage`, etc. | Required fields, extra fields (open schema), serialization round-trip | |
| `models.py` | `ClientMessage`, `ReplayRequest` | Literal type enforcement, optional fields | |
| `metrics.py` | `create_metrics()` | All counters/gauges present, value assertions, registry isolation | |
| `ws_manager.py` | `broadcast()` | Fan-out to all clients, sequence numbering, timestamp assignment, dead connection cleanup | |
| `ws_manager.py` | `replay()` | Replay by sequence (content verified), buffer bounds | |
| `ws_manager.py` | `connect()`/`disconnect()` | Connection tracking, idempotent disconnect | |
| `auth.py` | `OIDCValidator.validate_token()` | Valid token, expired, wrong audience, wrong issuer, unauthorized subject | Sync tests for mutmut compat |
| `auth.py` | `_extract_bearer_token()` | Valid bearer, missing header, invalid format | |
| `auth.py` | `AuthContext` | Field construction, default claims | |
| `agent.py` | `AgentClient.send_message()` | Counter increment, sequential lock ordering | SDK stubbed |
| `agent.py` | `AgentClient.clear()` | Session reset | |
| `agent.py` | `AgentClient.status` | Field presence (agent_id, session_id, working_dir, timestamp) | |

### Integration Tests

Tests for multiple components working together.

| Workflow | Components | Test Coverage | Notes |
|----------|------------|---------------|-------|
| Health check | FastAPI + health router | GET /health returns 200, no auth required | |
| Metrics endpoint | FastAPI + metrics router + lifespan | GET /metrics returns 200 Prometheus format | Uses `with TestClient(app)` for lifespan |
| Agent status | FastAPI + agent router + auth | GET /agent returns status, 401 without auth | Auth via dependency override |
| Agent clear | FastAPI + agent router + auth | POST /agent/clear returns cleared | |
| WebSocket connect | FastAPI + WS router + ws_manager + agent | Connect, send message, disconnect | |
| WebSocket replay | FastAPI + WS router + ws_manager | Replay as first message accepted | |
| WS replay rejection | FastAPI + WS router | Replay after first message returns error | |
| WS with source | FastAPI + WS router + agent | Message with source field accepted | |

## Untested Areas

| Area | Reason Not Tested |
|------|-------------------|
| Claude SDK internals | Mocked — third-party library behavior |
| TLS handshake / mTLS | uvicorn/OS concern, not application logic |
| Prometheus scraping | Scraper concern, not service logic |
| `__main__.py` | Entry point with uvicorn.run — tested via smoke test, not unit test |
| Real OIDC provider flow | Requires network access; JWKS fetch and JWT validation tested with mocked keys |

## Bug Fix Testing Protocol

All bug fixes to existing functionality **must** follow TDD:

1. Write a failing test that exposes the bug
2. Verify the test fails before implementing the fix
3. Implement the fix
4. Verify the test passes
5. Verify reverting the fix causes the test to fail again
6. Commit test and fix together with issue reference

### Regression Tests

| Issue | Test | Description |
|-------|------|-------------|
| — | — | No bug fixes yet (greenfield project) |

## Coverage Goals

**Target:** 80%+ line coverage

**Philosophy:** Coverage measures completeness, not quality. A test that executes code without meaningful assertions provides no value. Focus on:

- Testing behavior, not implementation details
- Covering edge cases and error conditions
- Ensuring assertions verify expected outcomes

## Running Tests

```bash
# Run all tests
make test

# Run with coverage
make coverage

# Run specific test
pytest tests/test_module.py::TestClass::test_function
```

## Test Data

Test data is:
- Generated programmatically in fixtures where possible
- Stored in `tests/fixtures/` when static files are needed
- Documented in `tests/fixtures/README.md`

**No Git LFS** - all test data must be small (< 100KB) or generated.

## Maintenance

When modifying this project:

1. **Adding features**: Add tests for new functionality after implementation
2. **Fixing bugs**: Follow TDD protocol above (test first, then fix)
3. **Refactoring**: Existing tests should pass without modification (behavior unchanged)
4. **Removing features**: Remove associated tests

## Changelog

| Date | Change | Rationale |
|------|--------|-----------|
| 2026-04-05 | Initial test plan | Project creation |
