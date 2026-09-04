# The demo project (layers 2 and 3)

The sandbox reads as three layers, each optional and building on the previous
one:

1. **The platform** (`clusters/sandbox/`): OKDP on kind, usable on its own.
2. **A project** (this directory): a namespace, its secrets, its PostgreSQL,
   its Connections and its data services, the same shape the Control Plane
   deploys from the service catalog.
3. **Example workloads** (`examples/`): the
   [okdp-examples](https://github.com/OKDP/okdp-examples) NYC-taxi medallion
   lakehouse, seeded by one Job.

Layer 2 requires the optional [storage component](../optional/README.md)
installed first.

## Layer 2: the project

```sh
kubectl apply -f clusters/sandbox/project-demo/00-namespace.yaml
kubectl apply -f clusters/sandbox/project-demo/05-project-context.yaml
kubectl apply -f clusters/sandbox/project-demo/10-secrets.yaml
kubectl -n demo wait --for=jsonpath='{.status.phase}'=READY release/demo-secrets --timeout=5m
kubectl apply -f clusters/sandbox/project-demo/20-storage-demo.yaml
kubectl -n kubocd-system wait --for=jsonpath='{.status.phase}'=READY release/storage --timeout=10m
kubectl apply -f clusters/sandbox/project-demo/30-postgres.yaml
kubectl -n demo wait --for=condition=Ready cluster/demo-pg --timeout=10m
kubectl -n demo wait --for=jsonpath='{.status.applied}'=true \
        database/demo-pg-hive database/demo-pg-superset database/demo-pg-superset-examples database/demo-pg-polaris database/demo-pg-airflow \
        --timeout=10m
kubectl apply -f clusters/sandbox/project-demo/40-connections.yaml
kubectl -n demo wait --for=jsonpath='{.status.phase}'=READY \
         connection/demo-db-hive connection/demo-db-superset connection/demo-db-superset-examples connection/demo-db-polaris connection/demo-db-airflow connection/demo-storage \
        --timeout=10m
kubectl apply -f clusters/sandbox/project-demo/50-services.yaml
kubectl -n demo wait --for=jsonpath='{.status.phase}'=READY release -l okdp.io/project=demo --timeout=30m
kubectl apply -f clusters/sandbox/project-demo/60-polaris-catalog-job.yaml
kubectl -n demo wait --for=condition=complete job/demo-polaris-catalog --timeout=10m
```

`20-storage-demo.yaml` extends the storage Release with the project buckets
and one grant per service. Re-apply `optional/storage/storage.yaml` to return
to the platform-only store. `20-storage-rustfs.yaml` is its RustFS counterpart,
to be applied over `optional/storage/rustfs.yaml`.

`05-project-context.yaml` is what makes the services of this namespace register
their own OIDC client rather than read one prepared by hand. It is picked up
through the Config's `defaultNamespaceContexts`, so it also covers the instances
the Control Plane creates.

The services follow the medallion layout of okdp-examples: `bronze` is the
hive catalog of Trino, `silver` and `gold` are Polaris warehouses that exist
once layer 3 has run. The `demo` Polaris catalog (job `demo-polaris-catalog`)
and the `iceberg` Trino catalog work without layer 3.

## Layer 3: okdp-examples

```sh
kubectl apply -f clusters/sandbox/project-demo/examples/okdp-examples.yaml
kubectl -n demo wait --for=jsonpath='{.status.phase}'=READY release/demo-okdp-examples --timeout=30m
```

One seed Job: bronze parquet data into S3, external tables on the `bronze`
catalog, `silver` and `gold` warehouses and their roles through polaris-admin.
Notebooks arrive through the JupyterHub welcome notebook, DAGs through the
Airflow git-sync, both pointed at the upstream okdp-examples repository.
