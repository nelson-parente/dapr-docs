---
type: docs
title: "Service invocation API reference"
linkTitle: "Service invocation API"
description: "Detailed documentation on the service invocation API"
weight: 1400
---

Dapr provides users with the ability to call other applications that are using Dapr with a unique named identifier (appId), or HTTP endpoints that are not using Dapr.
This allows applications to interact with one another via named identifiers and puts the burden of service discovery on the Dapr runtime.

## Invoke a method on a remote Dapr app

This endpoint lets you invoke a method in another Dapr enabled app.

### HTTP Request

```
PATCH/POST/GET/PUT/DELETE http://localhost:<daprPort>/v1.0/invoke/<appID>/method/<method-name>
```

## Invoke a method on a non-Dapr endpoint

This endpoint lets you invoke a method on a non-Dapr endpoint using an `HTTPEndpoint` resource name, or a Fully Qualified Domain Name (FQDN) URL.

### HTTP Request

```
PATCH/POST/GET/PUT/DELETE http://localhost:<daprPort>/v1.0/invoke/<HTTPEndpoint name>/method/<method-name>

PATCH/POST/GET/PUT/DELETE http://localhost:<daprPort>/v1.0/invoke/<FQDN URL>/method/<method-name>
```

### HTTP Response codes

When a service invokes another service with Dapr, the status code of the called service will be returned to the caller.
If there's a network error or other transient error, Dapr will return a `500` error with the detailed error message.

In case a user invokes Dapr over HTTP to talk to a gRPC enabled service, an error from the called gRPC service will return as `500` and a successful response will return as `200OK`.

Code | Description
---- | -----------
XXX  | Upstream status returned
400  | Method name not given
403  | Invocation forbidden by access control
500  | Request failed

### Method names that are rejected

Starting with Dapr v1.19 (and v1.18.6), Dapr rejects a method name that contains any of the following. The request fails with an `invalid method` error before the [access control]({{% ref invoke-allowlist.md %}}) check, and it never reaches the called app:

- `%2F`, `%5C` or `%2E` in any letter case (percent-encoded `/`, `\` and `.`).
- A literal `\` character.

The check applies to service invocation over the HTTP and gRPC APIs, calls received from other sidecars, proxied gRPC calls, and actor method invocation. Dapr already rejected `#`, `?`, null bytes and control characters.

The HTTP API decodes the URL once before the check. A method that is double-encoded by the caller, such as `a%252Fb`, or a backslash that is single-encoded, such as `a%5Cb`, is rejected. A single-encoded `%2F` or `%2E` sent to the invoke API is decoded to `/` or `.` first, and still works. Other percent-encoded characters, such as `%20`, are accepted and passed on unchanged.

For [actor method invocation]({{% ref "actors_api#invoke-actor-method" %}}), the HTTP API does not decode the method, so a single-encoded `%2F`, `%5C` or `%2E` is rejected with HTTP `400`, and the gRPC API returns `InvalidArgument`.

### URL Parameters

Parameter | Description
--------- | -----------
daprPort | the Dapr port
appID | the App ID associated with the remote app
HTTPEndpoint name | the HTTPEndpoint resource associated with the external endpoint
FQDN URL | Fully Qualified Domain Name URL to invoke on the external endpoint
method-name | the name of the method or url to invoke on the remote app

> Note, all URL parameters are case-sensitive.

### Request Contents

In the request you can pass along headers:

```json
{
  "Content-Type": "application/json"
}
```

Within the body of the request place the data you want to send to the service:

```json
{
  "arg1": 10,
  "arg2": 23,
  "operator": "+"
}
```

### Request received by invoked service

Once your service code invokes a method in another Dapr enabled app or non-Dapr endpoint, Dapr sends the request, along with the headers and body, on the `<method-name>` endpoint.

The Dapr app or non-Dapr endpoint being invoked will need to be listening for and responding to requests on that endpoint.

### Cross namespace invocation

On hosting platforms that support namespaces, Dapr app IDs conform to a valid FQDN format that includes the target namespace.
For example, the following string contains the app ID (`myApp`) in addition to the namespace the app runs in (`production`).

```
myApp.production
```

#### Namespace supported platforms

- Kubernetes

### Examples

You can invoke the `add` method on the `mathService` service by sending the following:

```shell
curl http://localhost:3500/v1.0/invoke/mathService/method/add \
  -H "Content-Type: application/json"
  -d '{ "arg1": 10, "arg2": 23}'
```

The `mathService` service will need to be listening on the `/add` endpoint to receive and process the request.

For a Node app this would look like:

```js
app.post('/add', (req, res) => {
  let args = req.body;
  const [operandOne, operandTwo] = [Number(args['arg1']), Number(args['arg2'])];

  let result = operandOne + operandTwo;
  res.send(result.toString());
});

app.listen(port, () => console.log(`Listening on port ${port}!`));
```

> The response from the remote endpoint will be returned in the response body.

In case when your service listens on a more nested path (e.g. `/api/v1/add`), Dapr implements a full reverse proxy so you can append all the necessary path fragments to your request URL like this:

`http://localhost:3500/v1.0/invoke/mathService/method/api/v1/add`

In case you are invoking `mathService` on a different namespace, you can use the following URL:

`http://localhost:3500/v1.0/invoke/mathService.testing/method/api/v1/add`

In this URL, `testing` is the namespace that `mathService` is running in.

### Headers in Service Invocation Requests

When Dapr invokes a service, it automatically adds the following headers to the request:

| Header | Description | Example |
|--------|-------------|---------|
| `dapr-caller-app-id` | The ID of the calling application | `myapp` |
| `dapr-caller-namespace` | The namespace of the calling application | `production` |
| `dapr-callee-app-id` | The ID of the called application | `mathService` |

These headers are available in both HTTP and gRPC service invocation requests.

The called app can rely on these headers only because its own sidecar sets them. Starting with Dapr v1.19 (and v1.18.6), the sidecar of the called app removes any `dapr-caller-app-id`, `dapr-caller-namespace` and `dapr-callee-app-id` values that the calling app sent, in any header casing, and sets the headers itself:

- When [mTLS]({{% ref mtls.md %}}) is enabled, `dapr-caller-app-id` and `dapr-caller-namespace` come from the identity (SPIFFE ID) in the certificate of the calling sidecar. `dapr-callee-app-id` is the app ID of the called app.
- When mTLS is disabled, there is no verified identity. The sidecar keeps the single value that the calling sidecar set and drops all others.
- On gRPC proxy calls, the calling sidecar replaces the identity metadata instead of appending to it, and also sets `dapr-caller-namespace`. Before this change, `dapr-caller-namespace` was not set on proxied calls.

Dapr's own [access control]({{% ref invoke-allowlist.md %}}) policies do not use these headers. They use the mTLS identity of the calling sidecar.

#### Non-Dapr Endpoint Example

If the `mathService` service was a non-Dapr application, then it could be invoked using service invocation via an `HTTPEndpoint`, as well as a Fully Qualified Domain Name (FQDN) URL.

```shell
curl http://localhost:3500/v1.0/invoke/mathHTTPEndpoint/method/add \
  -H "Content-Type: application/json"
  -d '{ "arg1": 10, "arg2": 23}'

curl http://localhost:3500/v1.0/invoke/http://mathServiceURL.com/method/add \
  -H "Content-Type: application/json"
  -d '{ "arg1": 10, "arg2": 23}'
```

## Next Steps
- [How-To: Invoke and discover services]({{% ref howto-invoke-discover-services.md %}})
