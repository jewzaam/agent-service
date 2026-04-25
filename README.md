# agent-service

[![Test](https://github.com/jewzaam/agent-service/actions/workflows/test.yml/badge.svg)](https://github.com/jewzaam/agent-service/actions/workflows/test.yml) [![Quality](https://github.com/jewzaam/agent-service/actions/workflows/quality.yml/badge.svg)](https://github.com/jewzaam/agent-service/actions/workflows/quality.yml) [![Version Check](https://github.com/jewzaam/agent-service/actions/workflows/version-check.yml/badge.svg)](https://github.com/jewzaam/agent-service/actions/workflows/version-check.yml)
[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/) [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

Standalone FastAPI service hosting a Claude SDK agent. Decoupled from the
dashboard lifecycle — one agent per service instance with WebSocket streaming,
multi-client fan-out, mTLS + OIDC authentication, and Prometheus metrics.

## Overview

The agent service runs a single Claude SDK agent in an isolated working
directory. Multiple clients (dashboard, Discord bot) connect via WebSocket and
see the same event stream in real time. REST endpoints provide agent status and
management. All connections require mTLS + OIDC authentication.

## Installation

### Development

```bash
git clone https://github.com/jewzaam/agent-service.git
cd agent-service
make install-dev
```

### From Git

```bash
python3 -m pipx install git+https://github.com/jewzaam/agent-service.git
```

## Usage

```bash
# Copy and configure
cp config.example.yaml config.yaml
# Edit config.yaml with your settings

# Run the service
python -m agent_service config.yaml
```

## Development

```bash
make install-dev    # Install in editable mode with dev dependencies
make check          # Run the read-only quality gate (format-check, lint, typecheck, unit, coverage, reachability, version-check)
make test-unit      # Run unit tests only
make format         # Apply black formatting (mutating)
make help           # Show all available targets
```

## Documentation

- [Test Plan](TEST_PLAN.md) — testing philosophy, categories, coverage goals
- [Architecture Plan](Plan-remote-agent-service.md) — design and implementation plan
