# integrations-api-examples

Examples of how to use Arkions integrations API.

## Prerequisites

- Node.js 18+
- npm

## Install

```bash
npm install
```

## Build

```bash
npm run build
```

## Configure .env

Edit `.env` in the repo root and set your values.

Required values in `.env`:

- `INTEGRATIONS_API_KEY` - Available to rotate and copy in Arkion UI
- `TENANT_ID` - Available in Arkion UI
- `PRIVATE_KEY` - Used to create assertion token used for access token exchange
- `PUBLIC_KEY` - Needs to be set in Arkion UI so the integrations-api can verify and scope requests in generated access token
- `ARKION_PUBLIC_KEY` - Available to copy from Arkion UI. Used by webhook receiver to verify received JWT

Optional:

- `INTEGRATIONS_BASE_URL` (defaults to `https://integrations-gateway.app.arkion.co`)
- `INTEGRATIONS_ORIGIN` (only required when your Arkion setup enforces Origin validation)

The app generates an assertion JWT from `PUBLIC_KEY` + `PRIVATE_KEY`, then sends it to `POST /tenant/{tenant_id}/auth/token`.

## Example Scenarios

All scenarios run through the same helper:

```bash
npm run scenario -- <scenario-name> [args...]
```

Available scenarios:

- `get-assertion-token`
- `get-project <project_id>`
- `get-projects [status_name]`
- `get-images <project_id>`
- `get-image-objects <project_id> <image_id>`
- `get-image-object-types <project_id> <image_id>`

Primary flow used by most scenarios:

1. Exchange an assertion token for an access token:
	- `POST /tenant/{tenant_id}/auth/token`
2. Call the target tenant endpoint with bearer auth.

## Expected output

The script prints:

- A short status line for token creation
- A short status line for project fetch
- Pretty JSON response of the project

## Equivalent curl calls

Token exchange:

```bash
curl -X POST "https://integrations-gateway.app.arkion.co/tenant/<tenant_id>/auth/token" \
	-H "x-api-key: $INTEGRATIONS_API_KEY" \
	-H "Content-Type: application/json" \
	-d '{"token":"<generated_assertion_token>"}'
```

If your Arkion setup requires Origin validation, also send:

```bash
-H "Origin: $INTEGRATIONS_ORIGIN"
```

Project call with bearer token:

```bash
curl -X GET "https://integrations-gateway.app.arkion.co/tenant/<tenant_id>/projects/<project_id>" \
	-H "x-api-key: $INTEGRATIONS_API_KEY" \
	-H "Authorization: Bearer <access_token>"
```

## Notes

- `INTEGRATIONS_ORIGIN` is only required when your Arkion setup enforces Origin validation.
- Assertion token is generated at runtime from `.env` keys (`PUBLIC_KEY`, `PRIVATE_KEY`).
- If you receive `token_expired` or `invalid_token`, request a new token and retry.
- If you receive `project_access_denied`, use a tenant token that has access to the project.

## AWS API Gateway Usage Plan Limits

When the integrations endpoint is behind AWS API Gateway usage plans:

- Throttling (RPS or burst limits) returns HTTP `429` (`THROTTLED`).
- Quota exceeded (for example monthly quota) also returns HTTP `429` (`QUOTA_EXCEEDED`).
- API Gateway may include `Retry-After`, and throttling/quota are applied on a best-effort basis.

This example client handles those cases by:

- Classifying HTTP `429` responses using API Gateway header/body hints (`x-amzn-errortype`, response message) as `THROTTLED`, `QUOTA_EXCEEDED`, or `BURST` when available.
- Retrying retryable `429` responses with exponential backoff + jitter.
- Skipping retries for `QUOTA_EXCEEDED` (not transient until quota window resets).
- Retrying HTTP `504` responses with the same exponential backoff + jitter strategy (treating gateway integration timeout/network issues as transient).
- Respecting `Retry-After` when present.
- Surfacing a clear error hint when retries are exhausted.

## Uploading images

### Docs
- See the flow diagram in [docs/image-upload-flow-overview.md](docs/image-upload-flow-overview.md)
- See the data model diagram in [docs/data-model-diagram.md](docs/data-model-diagram.md)

### Prerequisites

- Lookup `tenant_id` in `app.arkion.co/tenant-admin` UI or ask arkion. The `app` part in the domain can differ depending on region and if a custom domain/environment has been paid for.
- Get a list of customers within your tenant with GET `/tenant/{tenant_id}/customers`
- Each customer in the response will contain an `id` that you will use in other requests as `customer_id`. The customer response will also contain your customer `name`. If the name contains `sandbox` then it is meant to be used for testing/qa purposes. Otherwise it is for production use. Production customers are typically contract based.

### Upload

- Optionally create a project with `POST /tenant/{tenant_id}/projects` or use an existing project
- Create a flight with `POST /tenant/{tenant_id}/projects/{project_id}/flights`
- For each image call `GET /tenant/{tenant_id}/projects/{project_id}/upload/presigned_upload_url` which will return a `signed_url` in the response which is used to upload the image directly our AWS S3 bucket
- When all images for the flight is uploaded call `POST /tenant/{tenant_id}/projects/{project_id}/upload/start_import` to start our inference job
- To check if inference is done call `GET /tenant/{tenant_id}/projects/{project_id}/upload/inference_status` (this is inference only and does not mean that manual analysis has been done!)


## Webhook Receiver Example

The repo also includes an example local server that can receive webhook POST calls from the integrations API.

Webhook authentication:

- Each webhook request includes `Authorization: Bearer <token>`.
- The server verifies that token with `ARKION_PUBLIC_KEY`.
- Arkion webhook headers are accepted and logged with payload processing.
- `X-Arkion-Webhook-Event`
- `X-Arkion-Tenant-Id`

Start the server:

```bash
npm run server
```

Watch mode for local testing (auto-restarts on file changes):

```bash
npm run dev
```

Optional port override:

```bash
WEBHOOK_PORT=9999 npm run server
```

Expose local webhook server with cloudflared (macOS CLI):

```bash
brew install cloudflared
npm run server
cloudflared tunnel --url http://localhost:8787
```

cloudflared prints a public `https://<random>.trycloudflare.com` URL you can use as the webhook endpoint target.

Notes:

- Keep the generated tunnel URL private.
- Stop and restart the tunnel when you want a new URL.

Available webhook endpoints:

- `POST /ping` - Test that integration works
- `POST /project-report-available` - Get information about all defects in a project. Rotates access when about to expire.
- `POST /project-archived` - Get image, image objects, and image object types for all project images
- `POST /urgent-deficiency` - Get information about a defect

Background task behavior:

- Webhook endpoints create task events that run in the background so the endpoint can return `204` immediately.

Utility endpoints:

- `GET /health`

Example curl calls:

```bash
curl -X POST "http://localhost:8787/ping" \
	-H "Content-Type: application/json" \
	-H "Authorization: Bearer <webhook_token>" \
	-H "X-Arkion-Webhook-Event: ping" \
	-H "X-Arkion-Tenant-Id: <tenant_id>" \
	-d '{"event_id":"evt_1","message":"ping"}'

curl -X POST "http://localhost:8787/project-report-available" \
	-H "Content-Type: application/json" \
	-H "Authorization: Bearer <webhook_token>" \
	-H "X-Arkion-Webhook-Event: project-report-available" \
	-H "X-Arkion-Tenant-Id: <tenant_id>" \
	-d '{"event_id":"evt_2","project_id":42,"report_id":"rpt_100"}'

curl -X POST "http://localhost:8787/project-archived" \
	-H "Content-Type: application/json" \
	-H "Authorization: Bearer <webhook_token>" \
	-H "X-Arkion-Webhook-Event: project-archived" \
	-H "X-Arkion-Tenant-Id: <tenant_id>" \
	-d '{"event_id":"evt_3","project_id":42,"archived":true}'

curl -X POST "http://localhost:8787/urgent-deficiency" \
	-H "Content-Type: application/json" \
	-H "Authorization: Bearer <webhook_token>" \
	-H "X-Arkion-Webhook-Event: urgent-deficiency" \
	-H "X-Arkion-Tenant-Id: <tenant_id>" \
	-d '{"event_id":"evt_4","project_id":42,"severity":"critical"}'
```
