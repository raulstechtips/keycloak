# keycloak

Helm chart that installs Keycloak in the cluster.

## Purpose

The chart wraps the Bitnami `keycloak` chart `25.2.0` as a dependency. We use it only through values and never fork its templates.
The dependency comes from our Harbor mirror `oci://harbor.prod.raulstechtips.dev/raulstechtips/bitnami/charts`, a byte-identical copy of `registry-1.docker.io/bitnamicharts/keycloak:25.2.0`.

The wrapper owns:

- An optional `InfisicalStaticSecret` that syncs the admin password into `keycloak-admin-credentials-secret` (`adminSecret.enabled`).
- Optional Traefik IngressRoutes, one per `ingressRoute.hosts` entry, named `keycloak-ingress-<name>`.

Everything else (StatefulSet, Services, ConfigMap, PDB, NetworkPolicy) comes from the upstream chart.
`charts/values.yaml` lists only the upstream keys that differ from the 25.2.0 defaults, including the prod database wiring.

The chart is published to Harbor. Argo CD installs it with values from `cluster-config` (`apps/dev/keycloak.yaml`).

## Before first install

These must already exist. The chart reads them but never creates them:

- `pg-keycloak-database-credentials-secret` in `keycloak`, with keys `username` and `password` (CNPG database `keycloak` on `pg-prod`).
- `raulstechtips-internal-ca-key-pair` in `keycloak` (reflected), used for `sslmode=verify-ca`.
- The wildcard TLS Secrets named in `ingressRoute.hosts` (reflected from `global-certs`).
- `KEYCLOAK_ADMIN_PASSWORD` in Infisical at `/keycloak/admin`, and the `InfisicalAuth` named in `infisical.authRef`.

## Values reference

| Key | Default | Meaning |
| --- | --- | --- |
| `infisical.authRef.name` / `.namespace` | `""` | `InfisicalAuth` used by the admin password sync. Required when `adminSecret.enabled`. |
| `infisical.projectSlug` / `.environmentSlug` | `""` | Infisical project and environment. Required when `adminSecret.enabled`. |
| `adminSecret.enabled` | `false` | `true` syncs the admin password Secret. |
| `adminSecret.secretName` | `keycloak-admin-credentials-secret` | Must match `keycloak.auth.existingSecret`. |
| `adminSecret.secretPath` | `/keycloak/admin` | Infisical path to read from. |
| `adminSecret.keys` | `KEYCLOAK_ADMIN_PASSWORD` | Map of Secret key to Infisical key. |
| `ingressRoute.enabled` | `false` | `true` renders the IngressRoutes. |
| `ingressRoute.hosts` | `[]` | List of `{name, host, tlsSecretName}`. At least one when enabled. |
| `ingressRoute.entryPoints` | `[websecure]` | Shared by every host. |
| `ingressRoute.ingressClass` | `traefik-external` | Value of the `kubernetes.io/ingress.class` annotation. |
| `ingressRoute.middlewares` | `[]` | List of `{name, namespace}`, shared by every host. |

Any `keycloak.*` key goes straight to the upstream chart.

## Upgrades

1. Mirror the new upstream chart into Harbor:
   ```sh
   helm pull oci://registry-1.docker.io/bitnamicharts/keycloak --version <version>
   helm push keycloak-<version>.tgz oci://harbor.prod.raulstechtips.dev/raulstechtips/bitnami/charts
   ```
2. Bump the dependency version in `charts/Chart.yaml` and run `helm dependency update charts`.
3. Set `keycloak.image.tag` in `charts/values.yaml` (the image is mirrored to `harbor.prod.raulstechtips.dev/raulstechtips/keycloak`).
4. Merge, then sync the Application when you're ready.
