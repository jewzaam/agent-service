# Plan: step-ca Management (Deferred)

## Key questions

After reading, you should be able to answer:

- Why is automated CA tooling deferred for the initial deployment?
- What signal tells me it's time to pick this up?
- What does the manual cert posture look like in the meantime?
- What would step-ca actually deliver vs what we have now?
- What's the scope of work when we undefer this, and what decisions are still open?

## Status

**Deferred.** Initial deployment uses manually issued certs. Pick this up when
the operational cost of manual cert lifecycle exceeds the cost of running an
internal CA.

## Why deferred

For one user, one or two hosts, and 2–3 clients (dashboard, Discord bot), a
self-signed CA with long-TTL certs is cheaper to operate than running a CA
daemon. step-ca becomes worth the deploy when the manual chore stops scaling.

## Triggers to undefer

- More than ~3 clients, or new clients added more than once or twice a year
- Short-TTL certs become a requirement (defense-in-depth, compliance)
- Revocation matters — i.e., realistic risk of a client cert leak that needs
  to be revoked before its natural expiry
- A second service appears that wants to share the same trust domain

## Current manual posture (what step-ca replaces)

- Self-signed CA bootstrapped once via `openssl` or `mkcert`
- Long-TTL certs (1 year typical) — server cert + one client cert per client
- Rotation: regenerate, replace files in the paths listed in `config.yaml`,
  restart the agent service
- Trust bundle distribution: copy the CA cert to each client at provisioning
- Revocation: none. Rely on long-TTL + key hygiene; if a key leaks, the only
  practical mitigation is regenerating the CA and re-issuing everything

The agent service code is indifferent to who issued the certs — it only
requires that the server cert and client cert chain to the configured
`tls_ca`. No service changes are required to switch from manual to step-ca
beyond updating the cert files and (optionally) wiring renewal.

## What step-ca delivers

| Capability | Manual today | With step-ca |
|------------|--------------|--------------|
| Issuance | `openssl` by hand | `step ca certificate` |
| Renewal | Calendar reminder + manual replace | `step ca renew --daemon` |
| Trust distribution | Copy CA file by hand | `step ca bootstrap` |
| Revocation | None | Short-TTL by default; CRL optional |
| Service identity | CN matched against allowlist (manual) | Same, but issuance is automated |

## Scope of work when undeferred

1. **Topology decision.** Colocated with the agent service on the home lab VM,
   or stood up on its own host. Default expectation: colocated until a second
   consumer of the CA exists.
2. **Bootstrap step-ca.** Root + intermediate, root key kept offline if
   compliance-y posture is needed; otherwise online intermediate is fine for a
   personal trust domain.
3. **Choose a provisioner.** JWK for human-driven issuance, ACME for
   automated renewal, or both. ACME is the path of least resistance for
   long-running clients.
4. **Re-issue the server cert** and replace the manually issued one in the
   agent service config.
5. **Re-issue client certs** for each connected client (dashboard, Discord
   bot, future clients).
6. **Wire renewal.** `step ca renew --daemon` per host, or systemd timer +
   `step ca renew`. Decide whether the agent service tolerates SIGHUP-style
   reload or whether renewal triggers a service restart.
7. **Document the runbook** — issuance for a new client, renewal mechanics,
   what to do when a cert leaks.
8. **Decide revocation posture.** Short-TTL-only is the lean answer. Add CRL
   distribution only if the threat model demands it.

## Interaction with service auth

If the cert-as-identity path is taken for service clients (e.g., the Discord
bot authenticates by client cert subject instead of an OIDC bearer), step-ca
becomes the identity issuer for those services. The cert-as-identity design
works fine without step-ca — it just means the admin runs `openssl` per bot
instead of `step ca certificate`.

## Open questions

- **Topology** — colocated or separate host? Default: colocated.
- **TTL** — step-ca default is 24h. 7d may be a saner starting point for a
  small deployment to reduce renewal noise; revisit if defense-in-depth
  arguments push it back down.
- **Cert reload** — does the agent service handle reload without restart, or
  do we accept a restart per renewal cycle? Restart is simpler; reload is
  nicer for uptime-sensitive deployments.
- **ACME vs JWK provisioner** — ACME for automation, JWK for ad-hoc issuance.
  Likely both.

## Out of scope

- HSM integration for the root key
- Multi-tenant CA — one user, one trust domain
- Cross-organization trust federation
- PKI policy beyond what step-ca natively supports
