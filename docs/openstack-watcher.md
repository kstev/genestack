# Deploy Watcher

OpenStack Watcher is the resource optimization service for OpenStack. It audits
the cloud against a goal (for example server consolidation, workload balancing,
or host maintenance), produces an action plan of live migrations and other
operations, and can apply that plan. This document outlines the deployment of
OpenStack Watcher using Genestack.

Watcher runs three components:

| Component | Kind | Purpose |
|---|---|---|
| `watcher-api` | Deployment | REST API (`infra-optim`, port 9322) |
| `watcher-decision-engine` | StatefulSet | Runs audits and strategies, builds action plans |
| `watcher-applier` | StatefulSet | Executes action plans |

## Prerequisites

- Nova and Placement are deployed.
- Strategies that need utilization data require a datasource. Watcher 2026.1
  supports Gnocchi and Aetos; the Prometheus datasource is deprecated. Without
  a datasource, strategies that only use Nova data (for example
  `host_maintenance`, `zone_migration` or `dummy`) still work.

## Create secrets

!!! note "Information about the secrets used"

    Service secrets are managed idempotently by this service's install script. The installer creates any missing Kubernetes secrets and reuses existing values.

## Run the package deployment

!!! example "Run the Watcher deployment Script `/opt/genestack/bin/install-watcher.sh`"

    ``` shell
    --8<-- "bin/install-watcher.sh"
    ```

!!! tip

    Enable Watcher in `/etc/genestack/openstack-components.yaml` (`watcher: true`)
    to include it when running `/opt/genestack/bin/setup-openstack.sh`.

## Configure a datasource

Datasources are set with `conf.watcher.watcher_datasources.datasources` in
`/etc/genestack/helm-configs/watcher/watcher-helm-overrides.yaml`.

``` yaml
conf:
  watcher:
    watcher_datasources:
      datasources: gnocchi
```

## Validate functionality

``` shell
kubectl --namespace openstack exec -ti openstack-admin-client -- openstack optimize service list
kubectl --namespace openstack exec -ti openstack-admin-client -- openstack optimize goal list
kubectl --namespace openstack exec -ti openstack-admin-client -- openstack optimize strategy list
```

Run a dummy audit to exercise the decision engine and applier end to end:

``` shell
kubectl --namespace openstack exec -ti openstack-admin-client -- \
  openstack optimize audittemplate create dummy-template dummy --strategy dummy
kubectl --namespace openstack exec -ti openstack-admin-client -- \
  openstack optimize audit create -a dummy-template
kubectl --namespace openstack exec -ti openstack-admin-client -- openstack optimize actionplan list
```
