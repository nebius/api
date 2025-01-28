# Nebius AI Cloud API <br> [![CI][ci-img]][ci-url] [![License][license-img]][license-url]

This repository contains the `.proto` files defining the gRPC API for [Nebius AI Cloud](https://nebius.com) services.
These files describe the structure of requests and responses exchanged between your client application and Nebius services using Protocol Buffers.

## Tools

While you can interact directly with the Nebius AI Cloud API, we recommend leveraging the following tools to simplify your development and operations:

- [Nebius CLI](https://docs.nebius.com/cli)
- [Terraform Provider](https://docs.nebius.com/terraform-provider)
- [SDK for Go](https://github.com/nebius/gosdk)
- [Python SDK](https://github.com/nebius/pysdk)

Using these tools can save time and reduce the complexity of managing authentication, constructing requests, and handling responses.

## API Endpoints

Nebius AI Cloud gRPC services are accessed via endpoints formatted as `{service-name}.{base-address}`.
You can find a list of endpoints and services [here](endpoints.md). Below is an explanation of how these addresses are constructed:

1. **Base Address**:
   - The current base address is `api.eu.nebius.cloud:443`, though additional base addresses will be introduced soon.
1. **Service Name Derivation**:
   - Services with `option (api_service_name)`:
     - Some services explicitly define their name using this annotation (e.g., `option (api_service_name) = "foo.bar"`), this value becomes the `{service-name}` used in the address.
     - For example, requests to `TokenExchangeService` ([iam](nebius/iam/v1/token_exchange_service.proto)) would be sent to: `tokens.iam.api.eu.nebius.cloud:443`.
   - Services without `option (api_service_name)`:
     - For services lacking this annotation, the `{service-name}` is derived from the first level directory within the `.proto` file path.
     - For instance, requests to `DiskService` (declared in [nebius/**compute**/v1/disk_service.proto](nebius/compute/v1/disk_service.proto)) would be directed to `compute.api.eu.nebius.cloud:443`
1. **Special Case**: `OperationService`
   - `nebius.common.v1.OperationService` ([common](nebius/common/v1/operation_service.proto)) is an exception to the naming convention.
   - When fetching the status of an operation returned by another service, use the original service's address.
   - As an example, to fetch the status of an operation created by `DiskService` ([compute](nebius/compute/v1/disk_service.proto)), you would use: `compute.api.eu.nebius.cloud:443`.

## Authentication

Nebius AI Cloud uses bearer token authentication.
All requests must include an `Authorization: Bearer <IAM-access-token>` header.

### User Account Authentication

Prerequisites: Ensure you have installed and configured the `nebius` CLI as per the [documentation](https://docs.nebius.com/cli/).

Steps:

1. Run `nebius iam get-access-token` to retrieve your IAM access token.
1. Include this token in the `Authorization` header of your gRPC requests.

Example:

```bash
grpcurl -H "Authorization: Bearer $(nebius iam get-access-token)" \
  cpl.iam.api.eu.nebius.cloud:443 \
  nebius.iam.v1.ProfileService/Get
```

<details>
<summary>Sample Response</summary>

```json
{
  "userProfile": {
    "id": "useraccount-e00...",
    "federationInfo": {
      "federationUserAccountId": "...",
      "federationId": "federation-e00..."
    },
    "attributes": {
      ...
    }
  }
}
```
</details>

### Service Account Authentication

Prerequisites: You must have created a service account with the necessary credentials as outlined in the [documentation](https://docs.nebius.com/iam/service-accounts/manage/).

Service account credentials cannot be directly used for authentication.
Your service needs to obtain an IAM token using OAuth 2.0 with a compatible client library that implements RFC-8693 and JWT to generate a claim.

Steps:
1. Generate a JWT:
    - Use the RS256 signing algorithm.
    - Include the following claims:
        - `kid`: Public Key ID of your service account.
        - `iss`: Your service account ID.
        - `sub`: Your service account ID.
        - `exp`: Set a short expiration time (e.g., 5 minutes) as the token is only used for exchanging another token.
    - Sign the JWT with the service account’s private key.
1. Exchange JWT for an IAM Token:
    - Using gRPC: Call `nebius.iam.v1.TokenExchangeService/Exchange` on `tokens.iam.api.eu.nebius.cloud:443`.
    - Using HTTP: Send a POST request to `https://auth.eu.nebius.com:443/oauth2/token/exchange`.
1. Use the IAM Token from `access_token` from the response
1. Repeat the process before the IAM token expires (indicated by `expires_in` from the response).

<details>
<summary>Example using CLI tools and gRPC</summary>

```bash
SA_ID="serviceaccount-e00..."
KEY_ID="publickey-e00..."
PRIVATE_KEY_PATH="private_key.pem"

# https://github.com/mike-engel/jwt-cli
JWT=$(jwt encode \
  --alg RS256 \
  --kid $KEY_ID \
  --iss $SA_ID \
  --sub $SA_ID \
  --exp="$(date --date="+5minutes" +%s 2>/dev/null || date -v+5M +%s)" \
  --secret @${PRIVATE_KEY_PATH})

read -r -d '' REQUEST <<EOF
{
  "grantType": "urn:ietf:params:oauth:grant-type:token-exchange",
  "requestedTokenType": "urn:ietf:params:oauth:token-type:access_token",
  "subjectToken": "${JWT}",
  "subjectTokenType": "urn:ietf:params:oauth:token-type:jwt"
}
EOF

RESPONSE=$(grpcurl -d "$REQUEST" \
  tokens.iam.api.eu.nebius.cloud:443 \
  nebius.iam.v1.TokenExchangeService/Exchange)

TOKEN=$(jq -r '.accessToken' <<< $RESPONSE)

grpcurl -H "Authorization: Bearer $TOKEN" \
  cpl.iam.api.eu.nebius.cloud:443 \
  nebius.iam.v1.ProfileService/Get
```

```json
{
  "serviceAccountProfile": {
    "info": {
      "metadata": {
        "id": "serviceaccount-e00...",
        "parentId": "project-e00...",
        "name": "...",
        "createdAt": "..."
      },
      "spec": {},
      "status": {
        "active": true
      }
    }
  }
} 
```
</details>

<details>
<summary>Example using CLI tools and HTTP</summary>

```bash
SA_ID="serviceaccount-e00..."
KEY_ID="publickey-e00..."
PRIVATE_KEY_PATH="private_key.pem"

# https://github.com/mike-engel/jwt-cli
JWT=$(jwt encode \
  --alg RS256 \
  --kid $KEY_ID \
  --iss $SA_ID \
  --sub $SA_ID \
  --exp="$(date --date="+5minutes" +%s 2>/dev/null || date -v+5M +%s)" \
  --secret @${PRIVATE_KEY_PATH})

RESPONSE=$(curl https://auth.eu.nebius.com:443/oauth2/token/exchange \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
  -d "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
  -d "subject_token=${JWT}" \
  -d "subject_token_type=urn:ietf:params:oauth:token-type:jwt")

TOKEN=$(jq -r '.access_token' <<< $RESPONSE)

grpcurl -H "Authorization: Bearer $TOKEN" \
  cpl.iam.api.eu.nebius.cloud:443 \
  nebius.iam.v1.ProfileService/Get
```

```json
{
  "serviceAccountProfile": {
    "info": {
      "metadata": {
        "id": "serviceaccount-e00...",
        "parentId": "project-e00...",
        "name": "...",
        "createdAt": "..."
      },
      "spec": {},
      "status": {
        "active": true
      }
    }
  }
} 
```
</details>

## Method `Update` and `X-ResetMask` Header

In Nebius AI Cloud API, the `Update` method is designed to perform a "full-replace" of resource fields rather than a "patch" operation.
However, to maintain compatibility with clients of different versions, the server ensures that fields unknown to the client are not unintentionally modified.
To achieve this, Nebius employs a mechanism called the **Reset Mask**.

> Note: Nebius' Reset Mask is distinct from Google's [Field Mask](https://protobuf.dev/reference/protobuf/google.protobuf/#field-mask).

### Behavior

A field in a protobuf message is not transmitted over the wire if it has a default value (empty string, `0`, `false`, or NULL).
The server updates a resource field only if:
- The request contains a non-default value for that field, or
- The field is explicitly included in the mask provided in the `X-ResetMask` header.

For example, to detach (remove) secondary disks from a compute instance, you can use the following `X-ResetMask`:

```bash
grpcurl -H "Authorization: Bearer $TOKEN" \
  -H "X-ResetMask: spec.secondary_disks" \
  -d '{"metadata": {"id": "compute-instance-id"}}' \
  compute.api.eu.nebius.cloud:443 \
  nebius.compute.v1.InstanceService/Update
```

The Nebius SDK for all supported languages manages the `X-ResetMask` header automatically.

### Reset Mask Syntax

A reset mask is a comma-separated list of elements, where each element specifies one or more fields in a protobuf message.

**Example:**

```
a, b.c, d.e.12, f.(j.h,i.j).k, l.*.m
```

The fields matched by the mask above are:
- `a`
- `c` within the structure `b`
- The 12th element in the list `e` within the structure `d`
- The field `k` within maps `h` and `j` (found in objects `j` and `i`, respectively, under the common ancestor `f`)
- All `m` fields from any direct child of `l` (e.g., `l.q.m`)

#### Lists and Maps

When modifying lists or maps, all elements of the container are included in the operation since the client can fully read and understand these structures, regardless of version differences.

- **Lists**:
  - Elements within the range `0...min(incoming.length, previous.length)-1` are updated individually, with each treated as part of the field mask.
  - Elements beyond this range are either removed or added.
- **Maps**:
  - Elements present in the incoming message are updated.
  - Missing elements are removed, while new elements are added.

**Special Cases:**
- To clear a list or map, add its name as a key in the reset mask.
- To modify or fully replace a list/map, include it in the request without specifying it in the reset mask.
- To remove specific values from objects in a list or map, include the object's index in the reset mask.

#### Nested Structures

If a reset mask targets a structure (e.g., `a`), it does not clear the contents of its fields unless those fields are explicitly included in the mask.
However, if a field inside a structure is listed in the mask (e.g., `a.b`), the structure itself is reset if it is not present in the request payload.

**Example:**
- Resource: `{a: {b:1, c:2}}`
- Request: `{}` + `X-ResetMask: a.b`
- Result: `{a: null}` (the entire `a` structure is reset)

To preserve other fields in the structure, ensure the structure is explicitly included in the request:
- Resource: `{a: {b:1, c:2}}`
- Request: `{a: {}}` + `X-ResetMask: a.b`
- Result: `{a: {c: 2}}`

## Operations

Most methods that modify data return a `nebius.common.v1.Operation` ([operation.proto](nebius/common/v1/operation.proto)).
This message can represent both synchronous and asynchronous operations.
A synchronous operation is returned in a completed state, while an asynchronous one will update over time.

### Status

An operation is considered complete when its `status` field is set.
If the `status` field is `null`, the operation is still in progress.
Completed operations also include a `finished_at` timestamp.
If an error occurs, details will be provided in the `status.details` field, as explained in the [Error Details](#error-details) section.

### Parallel Operations

Nebius AI Cloud does not support concurrent operations on the same resource.
Attempting multiple operations simultaneously on the same resource may result in an error, or the initial operation could be aborted in favor of the newer request.

### Retention Policy

Ongoing operations are never deleted.
Completed operations, however, may be deleted according to the retention policy of the service.

## Error Details

Errors can occur either when an RPC method is called or when an asynchronous `Operation` completes unsuccessfully.
In both cases, the `google.rpc.Status` message is used to describe the error.
It may include additional details in the form of `nebius.common.v1.ServiceError` messages, providing more context about the issue.

A variety of `ServiceError` types can be returned (e.g. `BadRequest`, `QuotaFailure`, `TooManyRequests`).
Refer to the complete list of possible error types in [error.proto](nebius/common/v1/error.proto).

The `ServiceError` may also include a `retry_type` field, offering guidance on how to handle retries for the failed operation.

## Idempotency

To ensure the idempotency of modifying operations in the Nebius AI Cloud API, you can use the `X-Idempotency-Key` header.
Idempotency guarantees that the same operation will not be executed multiple times, even if the client retries the request due to network failures or timeouts.
For read-only methods such as `Get` and `List`, the `X-Idempotency-Key` header is ignored.

The value of `X-Idempotency-Key` must be a sufficiently long string of random alphanumeric characters (`[A-Za-z0-9-]`), preferably a random-based UUID (e.g., `7f95c54a-ee0e-4f8c-a64c-c9e0aac605a0`).

Example:
```bash
grpcurl -H "Authorization: Bearer $TOKEN" \
  -H "X-Idempotency-Key: 7f95c54a-ee0e-4f8c-a64c-c9e0aac605a0" \
  -d '{"metadata": {"id": "compute-instance-id"}, "spec": {"cpu": 4}}' \
  compute.api.eu.nebius.cloud:443 \
  nebius.compute.v1.InstanceService/Update
```

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

Copyright (c) 2024 Nebius B.V.



[ci-img]: https://github.com/nebius/api/actions/workflows/ci.yml/badge.svg
[ci-url]: https://github.com/nebius/api/actions/workflows/ci.yml
[license-img]: https://img.shields.io/github/license/nebius/api.svg
[license-url]: /LICENSE
