---
type: docs
title: "Dapr Placement control plane service overview"
linkTitle: "Placement"
description: "Overview of the Dapr Placement service"
---

{{% alert title="Deprecation notice" color="warning" %}}
Starting with Dapr v1.19, actor placement can be served by the [Scheduler service]({{% ref scheduler %}}), enabled by the opt-in flag `global.scheduler.placement.enabled=true` (`false` by default). The standalone Placement service is planned for deprecation in Dapr v1.21, when the Scheduler serves actor placement by default.
{{% /alert %}}

The Dapr Placement service is used to calculate and distribute distributed hash tables for the location of [Dapr actors]({{% ref actors %}}) running in [self-hosted mode]({{% ref self-hosted %}}) or on [Kubernetes]({{% ref kubernetes %}}). Grouped by namespace, the hash tables map actor types to pods or processes so a Dapr application can communicate with the actor. Anytime a Dapr application activates a Dapr actor, the Placement service updates the hash tables with the latest actor location.

## Self-hosted mode

The Placement service Docker container is started automatically as part of [`dapr init`]({{% ref self-hosted-with-docker %}}), unless you initialize with `dapr init --scheduler-placement`, which has the [Scheduler service serve actor placement](#serving-placement-from-the-scheduler-service) instead. It can also be run manually as a process if you are running in [slim-init mode]({{% ref self-hosted-no-docker %}}).

## Kubernetes mode

The Placement service is deployed as part of `dapr init -k`, or via the Dapr Helm charts. You can run Placement in high availability (HA) mode. [Learn more about setting HA mode in your Kubernetes service.]({{% ref "kubernetes-production#individual-service-ha-helm-configuration" %}})

Alternatively, actor placement can be served by the [Scheduler service]({{% ref scheduler %}}) instead of the standalone Placement service. See [Serving placement from the Scheduler service](#serving-placement-from-the-scheduler-service).

For more information on running Dapr on Kubernetes, visit the [Kubernetes hosting page]({{% ref kubernetes %}}).

## Serving placement from the Scheduler service

The Scheduler service can serve actor placement itself, so the standalone Placement service is not required to run.

In Kubernetes mode, set the Helm value:

```
global.scheduler.placement.enabled=true
```

With this setting, the Placement StatefulSet is not deployed and the Scheduler runs with `--placement-enabled=true`.

In self-hosted mode, initialize Dapr with `dapr init --scheduler-placement` (requires Dapr 1.19 or later). The Placement container is not started, and the CLI starts the Scheduler container with `--placement-enabled=true` for you. No further configuration is needed. The flag is not available with [slim-init mode]({{% ref self-hosted-no-docker %}}): `dapr init --slim` installs both the Placement and Scheduler binaries but does not run either of them, so you choose which one to run yourself, passing `--placement-enabled=true` if you run the Scheduler binary. Applications and sidecars need no configuration of their own: each sidecar takes actor placement from whichever service the control plane advertises, so the cluster always has exactly one placement authority. Toggling the setting in either direction requires no sidecar restarts.

The two services place actors with different algorithms: the Placement service uses a consistent hash ring, while the Scheduler uses rendezvous hashing. Both give the same guarantees (every sidecar deterministically agrees on a single host for each actor ID, and ownership spreads across hosts), but the resulting actor-to-host assignments differ. Switching the placement authority in either direction therefore reassigns actors once: affected actors are deactivated and reactivated on their new hosts with their state intact, as in any actor rebalance.

The Scheduler decides who the placement authority is by detecting whether a Placement service is deployed, so the switch is driven purely by deploying or removing the Placement service. In Kubernetes, the Scheduler watches for Placement pods and resolves the Placement service name; in every mode it also probes any Placement addresses the connected sidecars were configured with. Presence takes effect immediately, while absence must be confirmed by several consecutive checks: a Placement service that restarts or is briefly unreachable keeps the placement authority, and only removing it moves actor placement to the Scheduler.

{{% alert title="Important" color="warning" %}}
Complete your Dapr version rollout before enabling this setting. Sidecars running an older Dapr version can only use the Placement service: with it undeployed, their Actor and Workflow APIs stall until the pod is upgraded to a version that supports scheduler placement. No actor state is lost. While a Placement service is running, it remains the placement authority: the Scheduler does not serve placement while one is present, so the cluster keeps a single placement authority throughout the rollout. Once no Placement service remains, the Scheduler serves placement, and a connected older sidecar is logged as a warning, since nothing is left to serve it.
{{% /alert %}}

The [`/placement/state` API](#placement-tables) is only available from the standalone Placement service and is not served by the Scheduler.

### Prerequisites and behavior

- **Sidecar version.** Every sidecar must run Dapr 1.19 or later before the Placement service is removed. A sidecar signals that it supports Scheduler placement when it connects to the Scheduler, so you do not set anything on the sidecar.
- **Older sidecars.** An older sidecar can only use the Placement service. Once no Placement service remains, its actor and workflow APIs stall until the pod is upgraded. The older sidecar does not stop the Scheduler from serving the sidecars that are already upgraded: the Scheduler logs a warning and reports the sidecar in the `dapr_scheduler_placement_incapable_sidecars` metric.
- **No sidecar restarts.** Sidecars running Dapr 1.19 or later switch placement authority while running, in either direction. They keep their configured Placement address, so they can return to the Placement service if you turn the setting off.
- **Timing.** When the last Placement pod is gone, the Schedulers take over after a few seconds. On rollback, sidecars move to the Placement service as soon as its pods exist, and retry until they are ready, so expect a short pause in actor and workflow activity.
- **Actor reassignment.** Actors are deactivated and reactivated on their new hosts once per switch, in either direction. Actor state is kept.

### Verify which placement authority is active

**Sidecar logs.** Each sidecar logs, at `info` level, which service it connected to for actor placement:

| Log line | Meaning |
| -------- | ------- |
| `Connected to scheduler placement leader <address>` | The sidecar takes actor placement from the Scheduler placement leader at that address. |
| `Connected to placement service: <address>` | The sidecar takes actor placement from the Placement service. |

For example, on Kubernetes:

```bash
kubectl logs <pod-name> -c daprd | grep -E "Connected to (scheduler placement leader|placement service)"
```

A sidecar that cannot find a placement authority logs `Actor placement unavailable: the scheduler does not serve actor placement and no placement service is available to this sidecar`, and its actor and workflow APIs are unavailable until one is.

**Scheduler logs.** Only Schedulers that run with `--placement-enabled=true` log placement messages:

| Log line | Meaning |
| -------- | ------- |
| `Placement enabled on scheduler "<id>"` | Logged at startup. This Scheduler can serve placement. |
| `Placement leadership changed: leader=true` | This Scheduler became the placement leader and accepts placement streams from sidecars. |
| `Placement leadership changed: leader=false` | This Scheduler stopped being the placement leader. |

A Scheduler that detects a Placement service logs `A placement service is deployed, so actor placement stays with the placement service and this scheduler withholds its placement leader`. A Scheduler logs `A sidecar running an older Dapr version is connected while actor placement is served by the scheduler` when it serves placement while an older sidecar is connected. Upgrade that sidecar.

**Scheduler metrics.** The Scheduler exposes the following Prometheus metrics on its metrics endpoint (port `9090` by default), in addition to its other `dapr_scheduler_*` metrics:

| Metric | Type | Labels | Description |
| ------ | ---- | ------ | ----------- |
| `dapr_scheduler_placement_leader` | Gauge | | `1` on the Scheduler that is the placement leader, `0` on the others. Only emitted by Schedulers running with `--placement-enabled=true`. While the Placement service is the authority, it is `0` on every Scheduler. |
| `dapr_scheduler_placement_streams_connected` | Gauge | `namespace` | Number of sidecar placement streams connected to this Scheduler. Non-zero only on the leader. |
| `dapr_scheduler_placement_disseminations_total` | Counter | `namespace` | Number of placement dissemination rounds started. |
| `dapr_scheduler_placement_dissemination_latency` | Histogram (milliseconds) | `namespace` | Time from the start of a dissemination round to its completion. |
| `dapr_scheduler_placement_table_updates_total` | Counter | `namespace`, `actor_type` | Number of per actor type placement table rebuilds disseminated. |
| `dapr_scheduler_placement_incapable_sidecars` | Gauge | | Number of connected sidecars running a Dapr version that cannot use Scheduler placement. If it is non-zero after the switch, upgrade those sidecars. |

Exactly one Scheduler reports `dapr_scheduler_placement_leader` as `1` while the Scheduler is the placement authority.

To find sidecars that are older than 1.19 before you remove the Placement service, list the sidecar image of each pod:

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.spec.containers[?(@.name=="daprd")].image}{"\n"}{end}'
```

### High availability and the placement leader

The Scheduler replicas are peers when scheduling jobs: there is no leader for job scheduling. Actor placement is different. While the Scheduler serves placement, exactly one Scheduler replica is the placement leader:

- The leader is the replica with the lowest address among the Schedulers running with `--placement-enabled=true`. Every replica works this out from the same list of hosts, so they agree without extra coordination.
- Only the leader accepts placement streams from sidecars. The other replicas stay healthy and ready, but reject placement streams. Sidecars connect to the leader address that the Schedulers advertise.
- No replica is the leader while a Placement service is deployed, or while the Schedulers have not yet finished detecting whether one is deployed. A freshly started Scheduler cluster also waits for the first sidecar that supports Scheduler placement to connect.
- Placement tables are not persisted. The leader builds them from the live sidecar streams, and a new leader rebuilds them from scratch, so a change of leader causes a new dissemination round.
- All Scheduler replicas must run with the same `--placement-enabled` value. The Helm chart sets the same value on every replica.

### Kubernetes permissions

Every Scheduler in Dapr 1.19 detects whether a Placement service is deployed, whether or not `global.scheduler.placement.enabled` is set. In Kubernetes it watches the Placement pods (label `app=dapr-placement-server`) in the control plane namespace. The Scheduler `ClusterRole` (or `Role`, when `global.rbac.namespaced=true`) therefore needs `get`, `list` and `watch` on `pods`:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

The Dapr Helm chart grants this permission. If you manage the Scheduler RBAC resources yourself, add it on every Dapr 1.19 install, not only when you turn on Scheduler placement.

### Sidecar startup timeout

When a sidecar starts, it asks the Scheduler whether it serves actor placement and waits for the answer. The daprd flag `--actors-placement-startup-timeout` (annotation `dapr.io/actors-placement-startup-timeout`, default `30s`) sets how long the sidecar waits for the Scheduler to advertise a placement leader. The Scheduler only advertises after the sidecar's own jobs stream has registered, so set the value higher than your application [health probe]({{% ref "app-health.md" %}}) budget.

When the timeout expires and no Scheduler is reachable at all, the sidecar uses the Placement service. If a Scheduler is reachable but has not advertised a leader yet, the sidecar logs `Scheduler reachable but no placement leader advertised after <timeout>, still waiting` and keeps waiting. See the [arguments and annotations reference]({{% ref arguments-annotations-overview %}}).

## Placement tables

There is an [HTTP API `/placement/state` for Placement service]({{% ref placement_api %}}) that exposes placement table information. The API is exposed on the sidecar on the same port as the healthz. This is an unauthenticated endpoint, and is disabled by default. You need to set `DAPR_PLACEMENT_METADATA_ENABLED` environment or `metadata-enabled` command line args to true to enable it. If you are using helm you just need to set `dapr_placement.metadataEnabled` to true.

{{% alert title="Important" color="warning" %}}
When deploying actors into different namespaces ({{% ref namespaced-actors %}}), it is recommended to disable the `metadata-enabled` if you want to prevent retrieving actors from all namespaces. The metadata endpoint is scoped to all namespaces.
{{% /alert %}}

### Usecase:
The placement table API can be used to retrieve the current placement table, which contains all the actors registered across all namespaces. This is helpful for debugging and allowing tools to extract and present information about actors.

### HTTP Request

```
GET http://localhost:<healthzPort>/placement/state
```

### HTTP Response Codes

Code | Description
---- | -----------
200  | Placement tables information returned
500  | Placement could not return the placement tables information

### HTTP Response Body

**Placement tables API Response Object**

Name                   | Type                                                                  | Description
----                   | ----                                                                  | -----------
tableVersion           | int                                                                   | The placement table version
hostList               | [Actor Host Info](#actorhostinfo)[]                                   | A json array of registered actors host info.

<a id="actorhostinfo"></a>**Actor Host Info**

Name  | Type    | Description
----  | ----    | -----------
name  | string  | The host:port address of the actor.
appId | string  | app id.
actorTypes | json string array | List of actor types it hosts.
updatedAt | timestamp | Timestamp of the actor registered/updated.

### Examples

```shell
 curl localhost:8080/placement/state
```

```json
{
    "hostList": [{
            "name": "198.18.0.1:49347",
            "namespace": "ns1",
            "appId": "actor1",
            "actorTypes": ["testActorType1", "testActorType3"],
            "updatedAt": 1690274322325260000
        },
        {
            "name": "198.18.0.2:49347",
            "namespace": "ns2",
            "appId": "actor2",
            "actorTypes": ["testActorType2"],
            "updatedAt": 1690274322325260000
        },
        {
            "name": "198.18.0.3:49347",
            "namespace": "ns2",
            "appId": "actor2",
            "actorTypes": ["testActorType2"],
            "updatedAt": 1690274322325260000
        }
    ],
    "tableVersion": 1
}
```

## Disabling the Placement service


The Placement service can be disabled with the following setting:


```
global.actors.enabled=false
```

The Placement service is not deployed with this setting in Kubernetes mode. This not only disables actor deployment, but also disables workflows, given that workflows use actors. This setting only applies in Kubernetes mode, however initializing Dapr with `--slim` excludes the Placement service from being deployed in self-hosted mode. 

The Placement service is also not deployed when actor placement is served by the Scheduler service (`global.scheduler.placement.enabled=true`). Actors and workflows keep working in that case. See [Serving placement from the Scheduler service](#serving-placement-from-the-scheduler-service).


For more information on running Dapr on Kubernetes, visit the [Kubernetes hosting page](https://docs.dapr.io/operations/hosting/kubernetes/).

## Related links

[Learn more about the Placement API.]({{% ref placement_api %}})
