# matrix-x

Umbrella Helm chart that deploys a full, modern Matrix homeserver stack on Kubernetes.

## Components

| Subchart | Source | Purpose |
|---|---|---|
| `matrix` | [remram44/matrix-helm](https://github.com/remram44/matrix-helm) | Synapse homeserver + Element Web frontend |
| `mas` | local `charts/mas` | Matrix Authentication Service — OIDC provider replacing legacy password auth |
| `rtc` | local `charts/rtc` | LiveKit JWT service + Redis for MatrixRTC / Element Call |
| `static_wellknown` | local `charts/static_wellknown` | nginx serving `/.well-known/matrix/{server,client}` discovery endpoints |

## Architecture

```
Browser / clients
      │
      ▼
Traefik ingress
      │
      ├─► chat.example.org         → Synapse (matrix chart)
      ├─► chat.example.org/_matrix/client/*/login  (compat, priority 100)
      │                            → MAS (mas chart)
      ├─► auth.example.org         → MAS OIDC endpoints
      ├─► rtc.example.org          → LiveKit JWT service (rtc chart)
      └─► example.org/.well-known  → static_wellknown (nginx)

Internal
      Synapse ──► MAS (http://mas:8080)
      Synapse ──► PostgreSQL (bundled in matrix chart)
      MAS     ──► PostgreSQL (bundled in mas chart)
      rtc     ──► Redis (bundled in rtc chart)
```

## Prerequisites

- Kubernetes cluster with Traefik ingress controller
- cert-manager with a `letsencrypt-production` ClusterIssuer
- A running LiveKit SFU reachable from the cluster (external or self-hosted)
- A TURN server reachable from clients — **not included in this chart**; run [coturn](https://github.com/coturn/coturn) separately on a host with a public IP

## Installation

```bash
# 1. Fetch dependencies (downloads remram44/matrix chart into charts/)
helm dependency update

# 2. Install with your secrets — never put real secrets in values.yaml
helm install matrix-x . \
  --namespace matrix --create-namespace \
  -f values.yaml \
  --set matrix.homeserverConfig.turn_shared_secret="<TURN_SECRET>" \
  --set matrix.homeserverConfig.matrix_authentication_service.secret="<MAS_SHARED_SECRET>" \
  --set mas.secrets.sharedSecret="<MAS_SHARED_SECRET>" \
  --set mas.secrets.encryptionKey="$(openssl rand -hex 32)" \
  --set mas.secrets.signingKeyPem="$(openssl genrsa 4096)" \
  --set mas.postgresql.password="$(openssl rand -hex 16)" \
  --set rtc.livekit.key="<LIVEKIT_KEY>" \
  --set rtc.livekit.secret="<LIVEKIT_SECRET>"
```

> `matrix.homeserverConfig.matrix_authentication_service.secret` and `mas.secrets.sharedSecret` **must be identical**.

## Must-change values

Replace every occurrence of `example.org` with your actual domain before deploying.

### Domains

| Value path | Description |
|---|---|
| `matrix.homeserverConfig.server_name` | Matrix server name (bare domain, e.g. `chat` shows as `@user:example.org`) |
| `matrix.homeserverConfig.public_baseurl` | Public HTTPS URL of Synapse |
| `matrix.homeserverConfig.web_client_location` | URL where Element Web is served |
| `matrix.homeserverConfig.email.app_name` | Display name used in notification emails |
| `matrix.homeserverConfig.extra_well_known_client_content…livekit_service_url` | LiveKit JWT service public URL |
| `matrix.ingress.hosts` / `matrix.ingress.tls` | Synapse ingress host |
| `matrix.element.config.default_server_config.m.homeserver.*` | Element's homeserver config |
| `matrix.element.config.element_call.url` | Element Call / call.example.org URL |
| `matrix.element.ingress.host` | Element Web ingress host |
| `mas.mas.publicBaseUrl` | Public base URL of MAS |
| `mas.matrix.homeserver` | Your Matrix server name (same as `matrix.homeserverConfig.server_name`) |
| `mas.matrix.endpoint` | In-cluster URL of Synapse service |
| `mas.ingress.hosts` / `mas.compatIngress.host` | MAS ingress hosts |
| `rtc.livekit.url` | WebSocket URL of the LiveKit SFU |
| `rtc.livekit.fullAccessHomeservers` | Your Matrix server name |
| `rtc.ingress.host` | RTC JWT service ingress host |
| `static_wellknown.wellKnown.*` | All `.well-known` discovery URLs |
| `static_wellknown.ingress.host` | Well-known ingress host (bare domain) |

### Secrets (never commit — pass via `--set` or a sealed-secrets / external-secrets file)

| Value path | How to generate |
|---|---|
| `matrix.homeserverConfig.turn_shared_secret` | From your TURN server config |
| `matrix.homeserverConfig.matrix_authentication_service.secret` | `openssl rand -hex 32` — must equal `mas.secrets.sharedSecret` |
| `mas.secrets.sharedSecret` | `openssl rand -hex 32` — must equal the value above |
| `mas.secrets.encryptionKey` | `openssl rand -hex 32` |
| `mas.secrets.signingKeyPem` | `openssl genrsa 4096` |
| `mas.postgresql.password` | `openssl rand -hex 16` |
| `rtc.livekit.key` | From your LiveKit server config |
| `rtc.livekit.secret` | From your LiveKit server config |

### TURN server (external — not part of this chart)

TURN is **not deployed by this chart**. You need to run [coturn](https://github.com/coturn/coturn) yourself on a host with a public IP address. Once it is up, point Synapse at it:

```yaml
matrix:
  homeserverConfig:
    turn_uris:
      - "turn:turn.example.org:3478?transport=udp"
      - "turn:turn.example.org:3478?transport=tcp"
    turn_shared_secret: "<secret from coturn's static-auth-secret>"
    turn_user_lifetime: 86400000
    turn_allow_guests: true
```

A minimal coturn config to match:

```
use-auth-secret
static-auth-secret=<same secret>
realm=turn.example.org
listening-port=3478
tls-listening-port=5349
```

## MAS ↔ Synapse shared secret

Both `matrix.homeserverConfig.matrix_authentication_service.secret` and `mas.secrets.sharedSecret` must be set to the same value. This is the secret Synapse and MAS use to authenticate each other.

## Upgrading

```bash
helm dependency update
helm upgrade matrix-x . --namespace matrix -f values.yaml --reuse-values \
  --set matrix.homeserverConfig.turn_shared_secret="<TURN_SECRET>" \
  # ... other secrets
```
