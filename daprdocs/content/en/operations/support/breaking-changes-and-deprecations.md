---
type: docs
title: "Breaking changes and deprecations"
linkTitle: "Breaking changes and deprecations"
weight: 2500
description: "Handling of breaking changes and deprecations"
---

## Breaking changes 

Breaking changes are defined as a change to any of the following that cause compilation errors or undesirable runtime behavior to an existing 3rd party consumer application or script after upgrading to the next stable minor version of a Dapr artifact (SDK, CLI, runtime, etc):

- Code behavior
- Schema
- Default configuration value
- Command line argument
- Published metric
- Kubernetes resource template
- Publicly accessible API
- Publicly visible SDK interface, method, class, or attribute

Breaking changes can be applied right away to the following cases:

- Projects that have not reached version 1.0.0 yet
- Preview feature
- Alpha API
- Preview or Alpha interface, class, method or attribute in SDK
- Dapr Component in Alpha or Beta
- Interfaces for `github.com/dapr/components-contrib`
- URLs in Docs and Blog
- An **exceptional** case where it is **required** to fix a critical bug or security vulnerability.

### Process for applying breaking changes

There is a process for applying breaking changes:

1. A deprecation notice must be posted as part of a release. 
1. The breaking changes are applied two (2) releases after the release in which the deprecation was announced. 
   - For example, feature X is announced to be deprecated in the 1.0.0 release notes and will then be removed in 1.2.0.

## Deprecations

Deprecations can apply to:

1. APIs, including alpha APIs
1. Preview features
1. Components
1. CLI
1. Features that could result in security vulnerabilities

Deprecations appear in release notes under a section named “Deprecations”, which indicates:

- The point in the future the now-deprecated feature will no longer be supported. For example release x.y.z.  This is at least two (2) releases prior.
- Document any steps the user must take to modify their code, operations, etc if applicable in the release notes.

After announcing a future breaking change, the change will happen in 2 releases or 6 months, whichever is greater. Deprecated features should respond with warning but do nothing otherwise.


## Announced deprecations

| Feature                                                                                                                                                                                                                                                                                                     | Deprecation announcement | Removal        |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------|----------------|
| GET /v1.0/shutdown API (Users should use [POST API]({{% ref kubernetes-job.md %}}) instead)                                                                                                                                                                                                                 | 1.2.0                    | 1.4.0          |
| Java domain builder classes deprecated (Users should use [setters](https://github.com/dapr/java-sdk/issues/587) instead)                                                                                                                                                                                    | Java SDK 1.3.0           | Java SDK 1.5.0 |
| Service invocation will no longer provide a default content type header of `application/json` when no content-type is specified. You must explicitly [set a content-type header]({{% ref "service_invocation_api.md#request-contents" %}}) for service invocation if your invoked apps rely on this header. | 1.7.0                    | 1.9.0          |
| gRPC service invocation using `invoke` method is deprecated. Use proxy mode service invocation instead. See [How-To: Invoke services using gRPC ]({{% ref howto-invoke-services-grpc.md %}}) to use the proxy mode.                                                                                         | 1.9.0                    | 1.10.0         |
| The CLI flag `--app-ssl` (in both the Dapr CLI and daprd) has been deprecated in favor of using `--app-protocol` with values `https` or `grpcs`. [daprd:6158](https://github.com/dapr/dapr/issues/6158) [cli:1267](https://github.com/dapr/cli/issues/1267)                                                 | 1.11.0                   | 1.13.0         |
| Hazelcast PubSub Component                                                                                                                                                                                                                                                                                  | 1.9.0                    | 1.11.0         |
| Twitter Binding Component                                                                                                                                                                                                                                                                                   | 1.10.0                   | 1.11.0         |
| NATS Streaming PubSub Component                                                                                                                                                                                                                                                                             | 1.11.0                   | 1.13.0         |
| Workflows API Alpha1 `/v1.0-alpha1/workflows` being deprecated in favor of Workflow Client                                                                                                                                                                                                                  | 1.15.0                   | 1.17.0         |
| Migration of `http-max-request-size` flags/annotations to `max-body-size`. See [How-To: Handle larger body requests]({{% ref increase-request-size.md %}})                                                                                                                                                  | 1.14.0                   | 1.17.0         |
| Removal of the workflow instance ID reuse policy. The `IGNORE` and `TERMINATE` actions on ID conflicts are no longer supported: creating a workflow whose instance ID belongs to a running workflow always fails with a conflict. SDK options that still expose the policy (such as `reuse_id_policy` in the Python SDK) are deprecated and silently ignored by 1.18+ runtimes.                              | 1.18.0                   | 1.18.0         |
| Workflow instance ID reuse requires the whole workflow tree to be terminal. Recreating a completed, failed, or terminated workflow's instance ID is rejected while any child workflow created by its previous execution (checked recursively) is still running. Use unique instance IDs, or purge the workflow to free its ID unconditionally.                                                                | 1.18.2                   | 1.18.2         |
| The standalone Placement service is planned for deprecation in 1.21, when the Scheduler serves actor placement by default. From 1.19 you can opt in early with the Helm value `global.scheduler.placement.enabled=true` (`false` by default). See [Placement service]({{% ref placement.md %}}). | 1.19.0                   | Not yet scheduled |
| Python SDK: `dapr.ext.grpc` topic handlers whose event parameter is not annotated with `SubscriptionMessage` receive the deprecated `cloudevents.sdk.event.v1.Event` type, and `subscribe()` emits a `DeprecationWarning`. Annotate handlers with `dapr.ext.grpc.SubscriptionMessage`. From Python SDK 1.20, unannotated handlers receive `SubscriptionMessage`. [python-sdk:1118](https://github.com/dapr/python-sdk/pull/1118) [release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md) | Python SDK 1.19.0        | Python SDK 1.22.0 |
| Python SDK: a gRPC endpoint with an `http://` or `https://` prefix, for example `https://grpc.example.com:443` in `DAPR_GRPC_ENDPOINT`, emits a `DeprecationWarning` that names the equivalent form. Use `host:port?tls=true` or `host:port?tls=false` instead. See [Dapr Sidecar Endpoints]({{% ref "python-client.md#dapr-sidecar-endpoints" %}}). Endpoint parsing and TLS behavior are unchanged. [python-sdk:1276](https://github.com/dapr/python-sdk/pull/1276) [release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md) | Python SDK 1.19.0        | Not yet scheduled |

## Breaking changes in 1.19

The changes below require action, or break existing code or configuration, when you upgrade from 1.18 to 1.19. The Dapr runtime API changes in 1.19 are additive. The [1.19 release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md) list every change.

### Dapr runtime and Kubernetes

- **Apply the 1.19 CRDs before `helm upgrade`.** `helm upgrade` does not update CRDs. Until you apply the 1.19 `WorkflowAccessPolicy` CRD, the API server drops the new `requires` field, and a rule that uses `requires` applies without that gate. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [Upgrade Dapr on a Kubernetes cluster]({{% ref kubernetes-upgrade.md %}}))
- **The Scheduler RBAC needs read access to pods.** The Scheduler ClusterRole (or Role, in namespaced RBAC mode) now also requests `get`, `list` and `watch` on `pods`, in addition to `namespaces`. The Helm chart applies this on upgrade. If you manage RBAC outside the chart, add the permission. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [Scheduler service]({{% ref scheduler.md %}}))
- **Dapr Kubernetes Services set `appProtocol`.** Every port of every Service that Dapr creates now declares `appProtocol`. If you use a service mesh that selects proxy behavior from this field, check how it treats these ports. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10232](https://github.com/dapr/dapr/pull/10232), [Dapr and service meshes]({{% ref service-mesh.md %}}))
- **Service invocation: caller identity headers are set only by the sidecar.** With mTLS on, the target app gets `dapr-caller-app-id` and `dapr-caller-namespace` only from its own sidecar. The sidecar sets them from the caller's mTLS identity and removes any value that the calling app sets. This change is also in 1.18.6. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10683](https://github.com/dapr/dapr/pull/10683), [Service invocation API]({{% ref service_invocation_api.md %}}))
- **Service invocation: methods with encoded separators are rejected.** A method that contains `%2F`, `%5C`, `%2E` (in any letter case) or a literal `\` now fails before the access control check. This covers the HTTP and gRPC APIs, calls between sidecars, proxied gRPC calls and actor method invocation. Change callers that rely on such method names. This change is also in 1.18.6. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10684](https://github.com/dapr/dapr/pull/10684), [Service invocation API]({{% ref service_invocation_api.md %}}), [Service invocation access control]({{% ref invoke-allowlist.md %}}))
- **Actor invoke rejects `dapr.internal.*` actor types.** The gRPC `InvokeActor` call and the HTTP `/v1.0/actors/{type}/{id}/method/{method}` API now fail for every `dapr.internal.*` actor type with `ERR_ACTOR_TYPE_RESERVED` (gRPC `PermissionDenied`, HTTP 403). This change is also in 1.18.6. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10685](https://github.com/dapr/dapr/pull/10685), [Actors API]({{% ref actors_api.md %}}), [Error codes]({{% ref error-codes-reference.md %}}))
- **Workflows: `AlreadyExists` for a duplicate active instance ID.** Scheduling a workflow with the ID of a running instance now fails with the gRPC `AlreadyExists` status code. Before 1.19 it failed with an unclassified error. Code that branches on the status code must handle `AlreadyExists`. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10153](https://github.com/dapr/dapr/pull/10153), [Workflow API]({{% ref workflow_api.md %}}))
- **Workflows: a failed workflow has no `dapr.workflow.output`.** The workflow Get API no longer copies the failure message into `dapr.workflow.output` for a workflow in the `FAILED` state. Read `dapr.workflow.failure.error_message` and `dapr.workflow.failure.error_type` instead. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10138](https://github.com/dapr/dapr/pull/10138), [Workflow API]({{% ref workflow_api.md %}}))
- **Actors: timer operations fail on a host that does not own the actor.** A timer operation on a sidecar that does not own the actor now fails with `ERR_ACTOR_TIMER_NOT_OWNED` (HTTP 403, gRPC `PermissionDenied`). In 1.18 the same call succeeded and could leak a timer. This can surface during actor rebalancing. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10380](https://github.com/dapr/dapr/pull/10380), [Actor timers and reminders]({{% ref actors-timers-reminders.md %}}))
- **Actors: reminder and timer span names are bounded.** The reminder and timer HTTP endpoints now emit span names in the form `<operation>Actor<Reminder|Timer>/<actorType>`, for example `RegisterActorReminder/OrderActor`. The span name no longer contains the actor ID or the reminder or timer name. Update dashboards and alerts that match the old span names. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10141](https://github.com/dapr/dapr/pull/10141), [Distributed tracing]({{% ref tracing-overview.md %}}))
- **The metrics endpoint answers only GET and HEAD.** The metrics server now answers POST, PUT, PATCH and DELETE with 405 and `Allow: GET, HEAD`. Prometheus scraping is unchanged. Update scripts or probes that call the endpoint with another method. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10669](https://github.com/dapr/dapr/pull/10669), [Metrics overview]({{% ref metrics-overview.md %}}))
- **`--healthz-listen-address` is honored.** Placement, Scheduler, Sentry, the sidecar injector and the operator now bind their healthz server to the address in this flag. Before 1.19 the flag had no effect and the server bound all interfaces. If you set the flag, make sure your health probes can reach that address. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10394](https://github.com/dapr/dapr/pull/10394), [Scheduler service]({{% ref scheduler.md %}}))
- **Jobs and reminders: a timezone prefix with `@every` is rejected.** A schedule such as `CRON_TZ=Europe/Rome @every 1h` now fails at parse time with `timezone is not supported for @every schedules`. Before, the timezone was silently ignored. Remove the prefix from `@every` schedules. Cron expressions with a timezone keep working. This change is also in 1.18.3. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [kit:161](https://github.com/dapr/kit/pull/161), [Jobs API]({{% ref jobs_api.md %}}))
- **The rollback floor is 1.17.7.** Dapr 1.19 writes an Ed25519-keyed CA into the `dapr-trust-bundle` secret. Sentry versions before 1.17.7 cannot parse it and crash on startup. When you downgrade from 1.19, go to 1.18.x, or go through 1.17.7 (or a later 1.17.x) before any further rollback. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:9598](https://github.com/dapr/dapr/pull/9598), [dapr:9904](https://github.com/dapr/dapr/pull/9904), [Security concepts]({{% ref security-concept.md %}}))

### Components

- **Azure Event Hubs: checkpoints move only past successfully handled events.** Before, the checkpoint could move past a failed batch. Now the component retries a failed batch with the `backOff*` settings until the handler succeeds, and `backOffMaxRetries` no longer stops the retries, so an event that always fails holds the checkpoint of its partition. A consumer group with no stored checkpoint now starts from the earliest retained event, not the latest. In the concurrent mode, the new `maxConcurrentHandlers` option (default `100`) limits the batches per partition that wait for a checkpoint. This change applies to the pub/sub component and the input binding, and is also in 1.18.5. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [components-contrib:4598](https://github.com/dapr/components-contrib/pull/4598), [components-contrib:4606](https://github.com/dapr/components-contrib/pull/4606), [Azure Event Hubs pub/sub]({{% ref setup-azure-eventhubs.md %}}), [Azure Event Hubs binding]({{% ref eventhubs.md %}}))

### Python SDK

- **Extension packages are no longer published separately.** Remove `dapr-ext-fastapi`, `dapr-ext-grpc`, `dapr-ext-langgraph`, `dapr-ext-strands`, `dapr-ext-workflow` and `flask-dapr`. Then install the matching extras of the core package, for example `pip install "dapr[fastapi,workflow]"`. Uninstall the old packages before you reinstall `dapr`. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [python-sdk:1054](https://github.com/dapr/python-sdk/pull/1054), [Python SDK]({{% ref python %}}))
- **Higher minimum `protobuf` and `grpcio` versions.** The SDK needs `protobuf` 6.33.5 or later, and `grpcio` and `grpcio-status` 1.81.1 or later. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [python-sdk:1126](https://github.com/dapr/python-sdk/pull/1126), [Python SDK]({{% ref python %}}))
- **`DAPR_API_TIMEOUT_SECONDS` no longer limits workflow waits.** `wait_for_workflow_start` and `wait_for_workflow_completion` ignore this setting. Without their own `timeout_in_seconds`, they wait with no time limit. If you relied on the global setting, pass `timeout_in_seconds`. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [python-sdk:1244](https://github.com/dapr/python-sdk/pull/1244), [Python SDK client]({{% ref python-client.md %}}))
- **Conversation tool calls reject `None` for required parameters.** When the SDK runs a tool function for a Conversation API tool call, `None` for a required (non-Optional) parameter now raises `ToolArgumentError`. Before, it became the string `'None'`, or failed later with an unclear error. Handle this error where you run tool calls. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [python-sdk:1225](https://github.com/dapr/python-sdk/pull/1225), [Python SDK client]({{% ref python-client.md %}}))

### Java SDK

- **The Spring integration requires Spring Boot 4.0.** Move to Spring Boot 4.0 before you take SDK 1.19, or stay on SDK 1.18.x with Spring Boot 3.5. The two combinations are not binary compatible and fail at runtime. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [java-sdk:1772](https://github.com/dapr/java-sdk/pull/1772), [Spring Boot integration]({{% ref spring-boot %}}))
- **The separate Spring Boot 4 artifacts are removed.** Replace `dapr-spring-boot-4-starter`, `dapr-spring-boot-4-starter-test` and `dapr-spring-boot-4-autoconfigure` with the artifacts without `-4` in the name. Replace `dapr-spring-6-data` with `dapr-spring-data`, and change imports from `io.dapr.spring6.data` to `io.dapr.spring.data`. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [java-sdk:1772](https://github.com/dapr/java-sdk/pull/1772), [Spring Boot integration]({{% ref spring-boot %}}))
- **`scheduleNewWorkflow` throws `WorkflowInstanceAlreadyExistsException`.** For a duplicate active instance ID, `scheduleNewWorkflow` now throws this exception instead of a raw `StatusRuntimeException`. Update code that catches the old exception. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [java-sdk:1782](https://github.com/dapr/java-sdk/pull/1782), [Java SDK]({{% ref java %}}))
- **Netty 4.2.** The SDK now pulls Netty 4.2.x. A project that pins Netty 4.1.x directly can get a dependency conflict. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [java-sdk:1814](https://github.com/dapr/java-sdk/pull/1814), [Java SDK]({{% ref java %}}))

### .NET SDK

- **`DaprWorkflowClient.GetWorkflowStateAsync` rethrows errors other than NotFound.** Before SDK 1.18.5, any gRPC error from the runtime was turned into a `WorkflowState` with `Exists` set to `false`. Now only a `NotFound` status gives `Exists == false`, and every other gRPC failure reaches your code as an `RpcException`. Handle `RpcException` where you poll workflow state. This change is also in SDK 1.18.5. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dotnet-sdk:1877](https://github.com/dapr/dotnet-sdk/pull/1877), [Workflow management methods]({{% ref dotnet-workflow-management-methods.md %}}))

### JavaScript SDK

- **New workflow engine, and `grpcOptions` on `DaprWorkflowClient`.** The SDK replaced its workflow engine and wire-protocol handling. `WorkflowContext`, `DaprWorkflowClient` and `WorkflowState` keep their public method names, but re-test your workflow code after you upgrade. On `DaprWorkflowClient`, only the message-size options in `grpcOptions` still take effect. Keepalive, credential, compression and other grpc-js settings are ignored without an error. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [js-sdk:857](https://github.com/dapr/js-sdk/pull/857), [JavaScript SDK workflow]({{% ref js-workflow %}}))

### Dapr CLI

- **Errors and warnings go to stderr.** The `init`, `list`, `uninstall`, `version` and workflow subcommands now write error and warning messages to stderr, so the `-o json` output stays parseable. Exit codes are unchanged. Update scripts that read these messages from stdout. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [cli:1673](https://github.com/dapr/cli/pull/1673), [Dapr CLI overview]({{% ref cli-overview.md %}}))
- **Workflow activity job names end in `::0`.** On a 1.19 runtime, `dapr scheduler list`, `get` and `delete` show workflow activity jobs as `activity/<app-id>/<instance-id>::<task-id>::0`. Earlier runtimes used a per-execution counter in the last component, for example `::1`. Update scripts that build these names. ([Release notes](https://github.com/dapr/dapr/blob/master/docs/release_notes/v1.19.0.md), [dapr:10283](https://github.com/dapr/dapr/pull/10283), [dapr:10504](https://github.com/dapr/dapr/pull/10504), [dapr scheduler]({{% ref dapr-scheduler.md %}}))

## Related links

- Read the [Versioning Policy]({{% ref support-versioning.md %}})
- Read the [Supported Releases]({{% ref support-release-policy.md %}})
