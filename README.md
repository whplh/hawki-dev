# Dev tooling for hawki and testing

## Content

### Keycloak
A local keycloak instance that connects to the hawki dev stack and "only" requires a couple of env
adjustments ( See keycload/README.md for details or updated info )
#### TLDR
Add OIDC setup in your HAWKI dev `.env`: 
```
AUTHENTICATION_METHOD="OIDC"
OIDC_CLIENT_ID="hawki"
OIDC_CLIENT_SECRET="hawki-local-dev-secret"
OIDC_LOGOUT_URI="http://keycloak:8080/realms/hawki/protocol/openid-connect/logout"
OIDC_IDP="http://keycloak:8080/realms/hawki"
```

Run `docker compose up` in `./keycloak`
