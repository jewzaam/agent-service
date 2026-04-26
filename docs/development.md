# Development Guide

Setting up a local environment to develop a client (or the service
itself) without doing the full production setup.

## Key questions

After reading, you should be able to answer:

- What's the cheapest path to a running service for client development?
- How do I generate dev certs without standing up a real CA?
- How do I get an OIDC token without registering a real OAuth App?
- How do I smoke-test the service before writing a client?
- How do I iterate on a client against a live agent?

## The problem

The production setup requires a real OIDC provider, a real CA, real
client certs, and a real config file. That's appropriate for production.
It's painful when all you want to do is poke at the WebSocket and see
what comes back.

## Self-signed dev certs

Same `openssl` recipe as in the setup guide, but with short-lived,
disposable keys. Drop this in `scripts/dev-certs.sh` (or run by hand):

```bash
mkdir -p .dev-certs && cd .dev-certs

# CA
openssl genrsa -out ca-key.pem 2048
openssl req -new -x509 -days 30 -key ca-key.pem \
    -subj "/CN=dev-ca" -out ca.pem

# Server cert (CN=localhost so the client trusts the hostname)
openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem \
    -subj "/CN=localhost" -out server.csr
openssl x509 -req -days 30 -in server.csr \
    -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
    -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1") \
    -out server.pem

# Client cert
openssl genrsa -out client-key.pem 2048
openssl req -new -key client-key.pem \
    -subj "/CN=dev-client" -out client.csr
openssl x509 -req -days 30 -in client.csr \
    -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
    -out client.pem
```

Add `.dev-certs/` to `.gitignore`. These keys are only safe for local
development.

## OIDC for development

The service requires a valid OIDC ID token signed by an issuer it can
fetch JWKS from. There's no built-in dev bypass. Two practical paths:

### Path A — real GitHub OAuth App pointing at localhost

Register an OAuth App with the callback set to
`http://localhost:<port>/callback` for whatever local script handles the
code exchange. Request `read:user`. This gives you a real ID token from
a real issuer and exercises the same code paths as production.

Tradeoff: you need internet, and your dev machine becomes "authorized"
in the App's allowlist alongside production users. Use a separate App
for dev.

### Path B — local OIDC stub

Run a small OIDC mock server (e.g., `oidc-server-mock`,
`mock-oidc-user-server`) on `http://localhost:<oidc-port>`. Configure
the service with that issuer URL and a synthetic audience. The mock
issues real-format JWTs you can mint with arbitrary claims.

Tradeoff: requires running another container/process. Worth it if your
laptop is offline often.

## Dev config

Drop `config.dev.yaml` somewhere outside the repo (or git-ignored):

```yaml
agent_working_dir: ./.dev-agent-workspace
tls_cert: ./.dev-certs/server.pem
tls_key: ./.dev-certs/server-key.pem
tls_ca: ./.dev-certs/ca.pem
oidc_issuer: https://token.actions.githubusercontent.com   # or your local stub
oidc_audience: dev-agent-service
authorized_subjects:
  - "your-github-numeric-id"
replay_buffer_size: 100
session_resume_on_startup: false
listen_address: "127.0.0.1:8443"
```

Run:

```bash
python -m agent_service config.dev.yaml --debug
```

`--debug` is verbose; you'll want it while iterating.

## Smoke test before writing a client

```bash
# Health (no auth)
curl --cacert .dev-certs/ca.pem https://localhost:8443/health

# Authenticated REST
curl --cacert .dev-certs/ca.pem \
    --cert .dev-certs/client.pem --key .dev-certs/client-key.pem \
    -H "Authorization: Bearer $OIDC_TOKEN" \
    https://localhost:8443/agent

# WebSocket with websocat
websocat --no-close \
    --ca-file .dev-certs/ca.pem \
    --cert .dev-certs/client.pem --key .dev-certs/client-key.pem \
    -H "Authorization: Bearer $OIDC_TOKEN" \
    wss://localhost:8443/agent/ws
```

Once you can paste `{"type":"message","content":"hi"}` into the websocat
session and see events stream back, the loop is closed.

## Iterating on a client

The service holds agent state across client disconnects, so:

- Restart your client freely without losing the agent's context
- Use replay to verify your reconnect path works (`{"type":"replay","since":<n>}`
  as the first frame)
- Watch the service's `--debug` output to see what your client is sending

For the minimal Python client skeleton, see
[client-guide.md](client-guide.md#reference-minimal-python-client).

## Testing the service itself

```bash
make test-unit          # fast feedback while editing
make check              # full read-only quality gate before committing
```

Tests use FastAPI's `TestClient` and stub the SDK — no real Claude API
calls, no real network. `tests/conftest.py` blocks subprocess, HTTP, and
filesystem writes globally to keep tests hermetic.

## Open questions

- **Dev-mode flag.** Should the service support a `--dev` flag that
  bypasses OIDC and accepts unsigned tokens? Lowers the barrier
  drastically but adds an exploitable code path that must never ship
  enabled. Status: not implemented; opinions split.
- **Reference clients.** Beyond the minimal Python skeleton in the
  client guide, there's no published reference for a Tkinter client or
  a Discord bot. The actual implementations live in their own repos —
  link them here once they stabilize.
- **OIDC stub recommendation.** Several mock OIDC servers exist; none
  is endorsed yet. Pin one once the dev workflow has been used in
  anger.
- **Test fixtures for cert handling.** Cert generation is currently a
  shell script; could be promoted to a `make dev-certs` target if
  contributors find it friction.
