# keycloak

Helm chart that installs Keycloak in the cluster.

## Purpose

The chart wraps the codecentric `keycloakx` chart `7.4.0` (alias `keycloak`) as a dependency. We use it only through values and never fork its templates.
The dependency comes from `oci://ghcr.io/codecentric/helm-charts`. The image is the official `quay.io/keycloak/keycloak`.

The wrapper owns:

- An optional `InfisicalStaticSecret` that syncs the admin password into `keycloak-admin-credentials-secret` (`adminSecret.enabled`).
- Optional Traefik IngressRoutes, one per `ingressRoute.hosts` entry, named `keycloak-ingress-<name>`.

The chart is published to GHCR (`ghcr.io/raulstechtips/charts`).

## Before first install

These must already exist. The chart reads them but never creates them:

- `pg-keycloak-database-credentials-secret` in `keycloak`, with keys `username` and `password`.
- `raulstechtips-internal-ca-key-pair` in `keycloak`, used for `sslmode=verify-ca`.
- The wildcard TLS Secrets named in `ingressRoute.hosts`.
- `KEYCLOAK_ADMIN_PASSWORD` in Infisical at `/keycloak/admin`, and the `InfisicalAuth` named in `infisical.authRef`.

## Values reference

| Key | Default | Meaning |
| --- | --- | --- |
| `infisical.authRef.name` / `.namespace` | `""` | `InfisicalAuth` used by the admin password sync. Required when `adminSecret.enabled`. |
| `infisical.projectSlug` / `.environmentSlug` | `""` | Infisical project and environment. Required when `adminSecret.enabled`. |
| `adminSecret.enabled` | `false` | `true` syncs the admin password Secret. |
| `adminSecret.secretName` | `keycloak-admin-credentials-secret` | Must match the `KC_BOOTSTRAP_ADMIN_PASSWORD` Secret in `keycloak.extraEnv`. |
| `adminSecret.secretPath` | `/keycloak/admin` | Infisical path to read from. |
| `adminSecret.keys` | `KEYCLOAK_ADMIN_PASSWORD` | Map of Secret key to Infisical key. |
| `ingressRoute.enabled` | `false` | `true` renders the IngressRoutes. |
| `ingressRoute.hosts` | `[]` | List of `{name, host, tlsSecretName}`. At least one when enabled. |
| `ingressRoute.entryPoints` | `[websecure]` | Shared by every host. |
| `ingressRoute.ingressClass` | `traefik-external` | Value of the `kubernetes.io/ingress.class` annotation. |
| `ingressRoute.middlewares` | `[]` | List of `{name, namespace}`, shared by every host. |

Any `keycloak.*` key goes straight to the upstream chart. The database settings a deployment usually changes:

| Key | Default | Meaning |
| --- | --- | --- |
| `keycloak.database.hostname` | `pg-prod-rw.prod.svc.cluster.local` | Postgres host (upstream key, `KC_DB_URL_HOST`). |
| `keycloak.database.database` | `keycloak` | Database name (upstream key, `KC_DB_URL_DATABASE`). |
| `keycloak.database.existingSecret` | `pg-keycloak-database-credentials-secret` | Secret with key `password` (upstream key). The username always comes from key `username` of `pg-keycloak-database-credentials-secret`, set in `keycloak.extraEnv`; overriding this key does not change it. |
| `keycloak.customPostgresCA.customVerifySslMode` | `verify-ca` | Our own key. `sslmode` in `KC_DB_URL_PROPERTIES`. |
| `keycloak.customPostgresCA.customSslRootCert` | `/etc/custom-certs/postgres-ca.crt` | Our own key. `sslrootcert` in `KC_DB_URL_PROPERTIES`; must match the CA mount in `keycloak.extraVolumes`. |

## Upgrades

1. Bump the `keycloakx` dependency version in `charts/Chart.yaml` and run `helm dependency update charts`.
2. Set `keycloak.image.tag` in `charts/values.yaml`. A new Keycloak version changes the database schema at the first start, so back up the `keycloak` database first.
3. Merge, then sync the Application when you're ready.
