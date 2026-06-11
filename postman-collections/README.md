# MOSIP ID Repository — Postman Collections

One Postman collection per REST controller in this repository.

| Collection file | Controller | Service / Module | Default base URL |
|---|---|---|---|
| `IdRepoController.postman_collection.json` | IdRepoController | id-repository-identity-service | `http://localhost:8090/idrepository/v1/identity` |
| `IdRepoDraftController.postman_collection.json` | IdRepoDraftController | id-repository-identity-service | `http://localhost:8090/idrepository/v1/identity/draft` |
| `VidEventCallbackController.postman_collection.json` | VidEventCallbackController | id-repository-identity-service | `http://localhost:8090/idrepository/v1/identity` |
| `VidController.postman_collection.json` | VidController | id-repository-vid-service | `http://localhost:8091/idrepository/v1` |
| `CredentialRequestGeneratorController.postman_collection.json` | CredentialRequestGeneratorController | credential-request-generator | `http://localhost:8094/v1/credentialrequest` |
| `CredentialStoreController.postman_collection.json` | CredentialStoreController | credential-service | `http://localhost:8095/v1/credentialservice` |

Ports and servlet paths come from each module's `bootstrap.properties`.

## Import

1. Open Postman → **Import**.
2. Select one or more of the `.postman_collection.json` files (or drag the whole folder).

## Configure Variables

Each collection has its own variables (collection → **Variables** tab). Common ones:

| Variable | Description |
|---|---|
All variables ship with working sample values (format-wise) — a third party only has to replace identifiers with ones that exist in their environment, and set `authToken`.

| Variable | Sample value | What to put |
|---|---|---|
| `baseUrl` | `http://localhost:8090/idrepository/v1/identity` | Service base URL incl. servlet path. Deployed envs route all services through one gateway domain: `https://<env-domain>` + same path (e.g. `https://api-internal.dev1.mosip.net/idrepository/v1/identity`). |
| `authToken` | *(empty — required)* | JWT from authmanager (see below), sent as `Authorization` cookie. |
| `currentUtcTimestamp` | *(auto)* | Auto-set by a collection pre-request script; used by all `requesttime` fields. Do not set manually. |
| `uin` | `9758610623` | A 10-digit UIN that exists in your environment. |
| `vid` | `6048372815873185` | A 16-digit VID that exists in your environment. |
| `individualId` | `9758610623` | UIN or VID of the individual. |
| `rid` / `registrationId` | `27847321742954220250616011728` | 29-digit registration id. |
| `requestId` | `0bf035cdb55bdd31a5905f8eba0c542d` | Credential request id returned by `POST /requestgenerator`. |

Each variable also carries a description visible in Postman's Variables tab.

## Get an Auth Token

Almost all endpoints are role-protected. Fetch a token from authmanager:

```
POST https://<env-domain>/v1/authmanager/authenticate/clientidsecretkey
{
  "id": "string",
  "version": "string",
  "requesttime": "2026-06-11T10:00:00.000Z",
  "request": {
    "clientId": "mosip-idrepo-client",
    "secretKey": "<secret>",
    "appId": "idrepo"
  }
}
```

(For user-based auth use `/v1/authmanager/authenticate/useridPwd` instead; validate an existing token with `/v1/authmanager/authorize/admin/validateToken`.)

Copy the token from the `authorization` response header into the `authToken` variable of each collection. The client must have the roles configured under the `mosip.role.idrepo.*` properties (e.g. `REGISTRATION_PROCESSOR`, `RESIDENT`, `CREDENTIAL_REQUEST`, `ID_AUTHENTICATION`).

## Endpoint Summary

### IdRepoController (identity-service)
POST `/` addIdentity · PATCH `/` updateIdentity · GET `/idvid/{id}` retrieveIdentity (deprecated) · POST `/idvid/` retrieveIdentityById · POST `/idvid/v2` retrieveIdentityByIdV2 · GET `/authtypes/status/individualIdType/{IDType}/individualId/{ID}` (deprecated) · GET `/authtypes/status/{ID}` · POST `/authtypes/status` updateAuthtypeStatus · GET `/rid/{individualId}` (deprecated) · POST `/idvid-metadata/search` · GET `/{individualId}/update-counts`

### IdRepoDraftController (identity-service, `/draft`)
POST `/create/{registrationId}` · PATCH `/update/{registrationId}` · GET `/publish/{registrationId}` · DELETE `/discard/{registrationId}` · HEAD `/{registrationId}` hasDraft · GET `/{registrationId}` getDraft · PUT `/extractbiometrics/{registrationId}` · GET `/uin/{UIN}`

### VidEventCallbackController (identity-service)
POST `/callback/vid_credential_status_update` — WebSub callback, normally invoked by the hub (signature verified); manual calls may be rejected.

### VidController (vid-service)
POST `/vid` create · POST `/draft/vid` createDraft · GET `/vid/{VID}` retrieveUinByVid · GET `/vid/uin/{UIN}` retrieveVidsByUin · PATCH `/vid/{VID}` updateVidStatus · POST `/vid/{VID}/regenerate` · POST `/vid/deactivate` · POST `/vid/reactivate`

### CredentialRequestGeneratorController (credential-request-generator)
POST `/requestgenerator` · POST `/v2/requestgenerator/{rid}` · GET `/cancel/{requestId}` · GET `/get/{requestId}` · GET `/getRequestIds` · PUT `/retrigger/{requestId}` · POST `/callback/notifyStatus` (WebSub) · GET `/scheduleRetrySubscription` · GET `/scheduleWebsubSubscription`

### CredentialStoreController (credential-service)
POST `/issue` (plain `CredentialServiceRequestDto` body, not wrapped) · GET `/types`

## Request Body Conventions

Most write APIs use the standard MOSIP `RequestWrapper`:

```json
{
  "id": "<api-id, e.g. mosip.vid.create>",
  "version": "v1",
  "requesttime": "<UTC ISO-8601>",
  "request": { }
}
```

Exceptions: `POST /issue` (credential-service) and `POST /idvid/` (identity-service) take plain DTO bodies; `POST /authtypes/status` uses `requestTime` (camelCase) plus `individualId`/`consentObtained` at the top level.

The `id` value per API is validated against the service's `mosip.idrepo.*.id` / `mosip.vid.*.id` configuration — adjust the sample value if your environment uses different ids.

Notes:

- VID request bodies use the JSON key `UIN` (uppercase), e.g. `"request": {"UIN": "123...", "vidType": "PERPETUAL"}`.
- `requesttime` is auto-filled with `{{currentUtcTimestamp}}` by a collection-level pre-request script.
- Add/Update Identity bodies contain a full sample identity (fullName/gender/address as language-tagged eng/ara/fra arrays, dateOfBirth `yyyy/MM/dd`, `proofOfIdentity` and `individualBiometrics` referencing `fileReferenceID`). Adjust `IDSchemaVersion` (sample: 0.3) and attributes to match your environment's ID schema.
- The `documents[].value` for `individualBiometrics` must be a base64-encoded CBEFF XML — left as a placeholder since it is environment/biometric specific.
- Sample identifiers (UIN/VID/RID) are format-valid but won't exist in your database — replace them with real ones before running.

## Typical Flows

Identity: create draft → update draft → extract biometrics → publish → retrieve via `/idvid`.
VID: create VID for UIN → retrieve UIN by VID → update status / regenerate / deactivate / reactivate.
Credential: request via requestgenerator → check `/get/{requestId}` → retrigger or cancel if needed (credential-service `/issue` is normally invoked internally by the batch job).

## Troubleshooting

- **401 Unauthorized** — token missing/expired; refresh `authToken`.
- **403 Forbidden** — token lacks the required role for that endpoint.
- **400 IDR-xxx errors** — validation failure; check the wrapper `id`, `requesttime` format (UTC, not in future), and identity schema fields.
- **WebSub callbacks** — `/callback/*` endpoints expect hub signature headers; testing them directly usually returns an authentication error.
