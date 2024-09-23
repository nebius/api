# Nebius API

This repository contains the `.proto` files defining the gRPC API for [nebius.ai](https://nebius.ai). These files describe the structure of requests and responses exchanged between your client application and Nebius services using Protocol Buffers.

## API Endpoints

Nebius gRPC services are accessed via endpoints formatted as `{service-name}.{base-address}`. Below is an explanation of how these addresses are constructed:

1. **Base Address**:
   - The current base address is `api.eu-north1.nebius.cloud:443`, though additional base addresses will be introduced soon.
1. **Service Name Derivation**:
   - Services with `option (api_service_name)`:
     - Some services explicitly define their name using this annotation (e.g., `option (api_service_name) = "foo.bar"`), this value becomes the `{service-name}` used in the address.
     - For example, requests to `TokenExchangeService` ([iam](nebius/iam/v1/token_exchange_service.proto)) would be sent to: `tokens.iam.api.eu-north1.nebius.cloud:443`.
   - Services without `option (api_service_name)`:
     - For services lacking this annotation, the `{service-name}` is derived from the first level directory within the `.proto` file path.
     - For instance, requests to `DiskService` (declared in [nebius/**compute**/v1/disk_service.proto](nebius/compute/v1/disk_service.proto)) would be directed to `compute.api.eu-north1.nebius.cloud:443`
1. **Special Case**: `OperationService`
   - `nebius.common.v1.OperationService` ([common](nebius/common/v1/operation_service.proto)) is an exception to the naming convention.
   - When fetching the status of an operation returned by another service, use the original service's address.
   - As an example, to fetch the status of an operation created by `DiskService` ([compute](nebius/compute/v1/disk_service.proto)), you would use: `compute.api.eu-north1.nebius.cloud:443`.

## Authentication

Nebius uses bearer token authentication. All requests must include an `Authorization: Bearer <IAM-access-token>` header.

### User Account Authentication

Prerequisites: Ensure you have installed and configured the `nebius` CLI as per the [documentation](https://docs.nebius.ai/cli/).

Steps:

1. Run `nebius iam get-access-token` to retrieve your IAM access token.
1. Include this token in the `Authorization` header of your gRPC requests.

Example:

```bash
grpcurl -H "Authorization: Bearer $(nebius iam get-access-token)" \
  cpl.iam.api.eu-north1.nebius.cloud:443 \
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

Prerequisites: You must have created a service account with the necessary credentials as outlined in the [documentation](https://docs.nebius.ai/iam/service-accounts/manage/).

Service account credentials cannot be directly used for authentication. Your service needs to obtain an IAM token using OAuth 2.0 with a compatible client library that implements RFC-8693 and JWT to generate a claim.

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
    - Using gRPC: Call `nebius.iam.v1.TokenExchangeService/Exchange` on `tokens.iam.api.eu-north1.nebius.cloud:443`.
    - Using HTTP: Send a POST request to `https://auth.eu-north1.nebius.ai:443/oauth2/token/exchange`.
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
  tokens.iam.api.eu-north1.nebius.cloud:443 \
  nebius.iam.v1.TokenExchangeService/Exchange)

TOKEN=$(jq -r '.accessToken' <<< $RESPONSE)

grpcurl -H "Authorization: Bearer $TOKEN" \
  cpl.iam.api.eu-north1.nebius.cloud:443 \
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

RESPONSE=$(curl https://auth.eu-north1.nebius.ai:443/oauth2/token/exchange \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
  -d "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
  -d "subject_token=${JWT}" \
  -d "subject_token_type=urn:ietf:params:oauth:token-type:jwt")

TOKEN=$(jq -r '.access_token' <<< $RESPONSE)

grpcurl -H "Authorization: Bearer $TOKEN" \
  cpl.iam.api.eu-north1.nebius.cloud:443 \
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


## Operations

Most methods that modify data return a `nebius.common.v1.Operation` ([operation.proto](nebius/common/v1/operation.proto)). This message can represent both synchronous and asynchronous operations. A synchronous operation is returned in a completed state, while an asynchronous one will update over time.

### Status

Operations are considered complete when the `status` field is set. If `status` is null, the operation is still in progress. Additionally, a completed operation will include a `finished_at` timestamp.

In some cases, the `status.details` field provides additional information via a `nebius.common.v1.ServiceError` ([error.proto](nebius/common/v1/error.proto)).

### Parallel Operations

Nebius does not support performing multiple operations on the same resource simultaneously. Attempting to do so may result in an error, or the first operation might be aborted in favor of the new one.

### Retention Policy

Ongoing operations are never deleted. However, completed operations may be subject to deletion based on the service's retention policy.



[//]: # (TODO: grpcurl examples)
[//]: # (TODO: Errors)
[//]: # (TODO: X-Idempotency-Key)
[//]: # (TODO: X-ResetMask)
[//]: # (TODO: X-SelectMask)
[//]: # (TODO: resource_version)
