# Optional components

Nothing in this directory is applied by the sandbox install. Each subdirectory
holds a component the platform can run on but does not need, so applying one is
a deliberate act with consequences worth reading first.

## kubauth

The sandbox authenticates against **Keycloak**. Kubauth is the other identity
provider OKDP supports, and it is the one the Control Plane's user and group
screens talk to: it stores users, groups and OIDC clients as custom resources,
which is what makes them manageable through the API server.

Without it, `/api/v1/identity` answers `501` and the console hides the Identity
area entirely. That is a correct state, not a failure.

Installing it does not switch the platform over. Two things stay on Keycloak
until you change them yourself: the `oidc` block of the platform
Context, which every service reads to find its login endpoints, and the console
sign-in. Running both providers side by side means users exist in two places.

### Install

```sh
kubectl apply -f clusters/sandbox/optional/kubauth/kubauth.yaml
kubectl -n kubocd-system wait --for=condition=Ready release/kubauth --timeout=10m
```

Then create the first administrator, since kubauth ships with no bootstrap
account:

```sh
kubectl apply -f clusters/sandbox/optional/kubauth/admin.yaml
```

That gives you `useradmin` / `password`, in the `admins` group. Change the hash
in `admin.yaml` before doing anything you would regret.

### Point the Control Plane at it

Three keys, in two Contexts, and all three are needed. Each was checked on the
sandbox: with any one missing, the screens stay empty or unavailable.

**1. Open the routes.** The server reads the provider from its own Context, the
one it is handed through `contextName`, not from the platform one:

```sh
kubectl -n kubocd-system patch context okdp-control-plane --type=merge \
  -p '{"spec":{"context":{"identity":{"provider":"kubauth"}}}}'
```

**2. Say where the users live**, in the platform Context this time, because the
packages read the same key to place their OIDC clients:

```sh
kubectl -n kubocd-system patch context platform --type=merge \
  -p '{"spec":{"context":{"platform":{"identity":{"kubauthNamespace":"kubauth-users"}}}}}'
```

**3. Hand the OIDC clients over to kubauth.** This one is not optional, and it is
where the trap is:

```sh
kubectl -n kubocd-system patch context platform --type=merge \
  -p '{"spec":{"context":{"platform":{"identity":{"clientProvisioning":"kubauth"}}}}}'
```

`clientProvisioning` reads as "who creates the OIDC clients", and it does decide
that. But the server also refuses to resolve the users namespace unless it is set
to `kubauth`, and falls back to `okdp-system`, where no user lives. Skip this
step and `/api/v1/identity/users` answers `200` with an empty list, which looks
like a platform with no users rather than a misconfiguration.

**This third step switches the platform over.** Every package that was handed a
Secret name now generates its own client through kubauth, so the Releases
reconcile against a different provider. Do not run it on a sandbox you need
working in the next few minutes.

Check the whole chain answered:

```sh
kubectl -n okdp-system exec deploy/okdp-control-plane-server -- \
  wget -qO- http://localhost:8093/api/capabilities
```

`identity.userManagement` must be `true`. The provider is resolved per request,
so no restart is needed. The console reads the same endpoint at sign-in, so
reload it once for the Identity area to appear.

### Remove

```sh
kubectl delete -f clusters/sandbox/optional/kubauth/admin.yaml
kubectl delete -f clusters/sandbox/optional/kubauth/kubauth.yaml
kubectl -n kubocd-system patch context okdp-control-plane --type=json \
  -p '[{"op":"remove","path":"/spec/context/identity"}]'
kubectl -n kubocd-system patch context platform --type=merge \
  -p '{"spec":{"context":{"platform":{"identity":{"clientProvisioning":"existing"}}}}}'
kubectl -n kubocd-system patch context platform --type=json \
  -p '[{"op":"remove","path":"/spec/context/platform/identity/kubauthNamespace"}]'
```

## storage

SeaweedFS, the S3 object store of the sandbox, and the producer of the `s3`
Connection the data services consume. Nothing in the platform install depends
on it: only project services (hive-metastore, jupyterhub, polaris,
spark-history-server, and trino through them) require the `storage` role, and
without it their Releases wait in `WAIT_DEPENDENCIES` with an explicit message.

### Install

```sh
kubectl apply -f clusters/sandbox/optional/storage/storage.yaml
kubectl -n kubocd-system wait --for=condition=Ready release/storage --timeout=10m
```

### Bring your own S3 instead

An external S3-compatible store replaces SeaweedFS entirely:

1. Declare the `storage` role as satisfied in the KuboCD Config
   (`clusterRoles: [storage]` in the config content of
   `clusters/sandbox/flux/kubocd.yaml`), so the dependent Releases stop waiting.
2. Declare an `s3` Connection by hand in each project (apiUrl, internalUrl,
   region, pathStyle, secretRef), pointing at the store.
3. Provide the credentials Secret those Connections reference (the sandbox
   uses `creds-seaweedfs-s3`). Buckets and per-service identities are a
   project concern: each project layer creates what its services need.
4. Polaris uses STS: the store must support AssumeRole, or its S3 access needs
   adapting.

### Remove

```sh
kubectl delete release storage -n kubocd-system
```

## vault

The secret backend a Control Plane `SecretStore` can point at, in dev mode
(in-memory, sealed on restart). Nothing depends on it: the platform secrets
come from local-secrets-provider, and the Control Plane's secret-store screens
only need the External Secrets CRDs, which the platform install carries.

### Install

```sh
kubectl apply -f clusters/sandbox/optional/vault/vault.yaml
kubectl -n kubocd-system wait --for=condition=Ready release/vault --timeout=10m
```

### Remove

```sh
kubectl delete release vault -n kubocd-system
```
