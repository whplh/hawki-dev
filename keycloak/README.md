# Keycloak — local IdP for HAWKI development

Standalone [Keycloak](https://www.keycloak.org) ([github](https://github.com/keycloak/keycloak))
for testing HAWKI's OIDC login locally, wired into HAWKI's docker network. Not
production-hardened (dev mode, embedded H2 database).

## Getting started (admin)

Prerequisite: HAWKI is running via its docker compose (it creates the shared
`hawki_hawk_net` network this stack attaches to).

```bash
cd /home/slave/proj/keycloak
docker compose up -d          # first boot imports the "hawki" realm (~30-60s)
docker compose logs -f        # wait for "Import of realm 'hawki' requested"
```

Open the admin console at **http://localhost:8080** and sign in with
`admin` / `admin` (set on first login under *Manage account → Password*).

Pre-configured by the import (`realm-hawki.json`):

| Thing | Value |
|---|---|
| Realm | `hawki` (switch to it via the dropdown next to *Keycloak* in the top bar — the console opens on `master`) |
| Client | `hawki`, secret `hawki-local-dev-secret`, redirect URIs for HAWKI already registered |
| Test users | `alice` / `alice-dev` (`employeetype=staff`), `bob` / `bob-dev` (`employeetype=student`) |

Adding more users: *Realm hawki → Users → Create new user* — set an `employeetype`
attribute under *Credentials/Attributes* (it becomes the claim HAWKI reads as the
user's group). Changing the realm JSON file only affects **first import**; after
that, edit things in the console (or `docker compose down -v` to wipe and re-import).

## One-time: make `keycloak` resolvable

Both the browser and the HAWKI app must call the IdP by the **same URL**
(`http://keycloak:8080`) — the browser via your hosts file, the app via the docker
network alias. Add next to your existing `hawki.dev.local` entry in `/etc/hosts`:

```
127.0.0.1	keycloak
```

## What to change in HAWKI

`.env` of the HAWKI checkout (defaults cover everything except the first four):

```dotenv
AUTHENTICATION_METHOD=OIDC
OIDC_IDP="http://keycloak:8080/realms/hawki"
OIDC_CLIENT_ID=hawki
OIDC_CLIENT_SECRET=hawki-local-dev-secret

# Optional (these are the defaults):
# OIDC_SCOPES=profile,email
# OIDC_USERNAME_VAR=preferred_username
# OIDC_EMAIL_VAR=email
# OIDC_EMPLOYEETYPE_VAR=employeetype

# Optional: log out of Keycloak when logging out of HAWKI:
# OIDC_LOGOUT_URI=http://keycloak:8080/realms/hawki/protocol/openid-connect/logout
```

Then restart the app container (or refresh its config cache) and open
**https://hawki.dev.local** — the login button redirects to Keycloak; sign in as
`alice` / `alice-dev`. The first login registers the account in HAWKI (policy
consent + passkey), the second unlocks the existing keychain.

## How the wiring works

- `docker-compose.yml` attaches the service to HAWKI's external `hawki_hawk_net`
  network with the alias `keycloak`, so the app container resolves the IdP by
  plain docker DNS — same as it resolves its own services.
- Login port 8080 is published on `127.0.0.1` only (same convention as HAWKI).
- HAWKI's OIDC callback is always `/req/login` — the SPA starts logins at
  `/auth/redirect`, but the IdP returns to `/req/login`, where HAWKI hands the
  session back into the SPA. `/auth/redirect` is therefore *not* a redirect URI.
- Logging out redirects to the IdP with `post_logout_redirect_uri=/login`;
  that URL is pre-registered on the client via the `post.logout.redirect.uris`
  attribute (single URI only — Keycloak's realm import rejects lists there).
  Add your HAWKI host there if you use a different one.

## Troubleshooting

| Symptom | Fix |
|---|---|
| *Invalid redirect_uri* at Keycloak | Your HAWKI URL isn't in the client's redirect URIs — *Realm hawki → Clients → hawki → Valid redirect URIs* |
| Browser can't resolve `keycloak` | The `/etc/hosts` entry is missing |
| HAWKI log: connection refused to `keycloak:8080` | HAWKI's stack isn't running / the `hawki_hawk_net` network doesn't exist yet (`docker network ls`) |
| Realm import didn't run | Import happens on **first** boot only — `docker compose down -v && docker compose up -d` (wipes Keycloak data) |
| Check logs | `docker compose logs -f keycloak` |

Stop/start: `docker compose stop` / `docker compose up -d`. Reset everything
(including users created via the console): `docker compose down -v`.
