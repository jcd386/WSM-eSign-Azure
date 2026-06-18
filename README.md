# WSM eSign — Azure Deployment

One-click deployment of the **WSM eSign PDF service** to your own Azure subscription as an [Azure Container App](https://learn.microsoft.com/azure/container-apps/overview). The service overlays signatures onto PDFs, merges documents, fills forms, and generates documents from templates — the backend that Salesforce calls during electronic signing.

This template pulls a **prebuilt public container image** and exposes a secure HTTPS endpoint. You don't build or compile anything.

## Deploy

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fjcd386%2FWSM-eSign-Azure%2Fmain%2Fazuredeploy.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fjcd386%2FWSM-eSign-Azure%2Fmain%2FcreateUiDefinition.json)
[![Visualize](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/visualizebutton.svg)](https://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2Fjcd386%2FWSM-eSign-Azure%2Fmain%2Fazuredeploy.json)

> **Deploying into a specific customer tenant?** If your account is a guest in multiple Azure directories, the button opens whichever directory you used last. Pin the deploy to the right tenant by prefixing the portal URL with the customer's verified domain (or tenant ID):
>
> `https://portal.azure.com/<customer-domain.com>#create/Microsoft.Template/uri/...`
>
> Or switch first: portal avatar (top right) → **Switch directory** → pick the customer → click Deploy again.

### 1. Click **Deploy to Azure**
Sign in, pick a **Subscription** and create (or pick) a **Resource group**. Choose a **Region** near your Salesforce instance. Optionally set a size and an API key (leave the key blank to auto-generate a strong one). Click **Review + create** → **Create**. Provisioning takes ~3–5 minutes.

### 2. Copy the outputs
When the deployment finishes, open **Outputs**. You'll see:
- **endpointUrl** — your service URL (e.g. `https://wsm-esign-api.<region>.azurecontainerapps.io`)
- **apiKey** — the key the service expects in the `X-API-Key` header

Sanity check (optional): open `endpointUrl` + `/health` in a browser — it returns `{"status":"ok"}`.

### 3. Point Salesforce at it
In your Salesforce org, edit the **`eSign_API`** External/Named Credential:
- Set the **URL** to the **endpointUrl** from step 2.
- Set the **`X-API-Key`** header value to the **apiKey** from step 2.

No Apex or flow changes are needed — Salesforce calls the service through the Named Credential. Run a test signing to confirm the completed PDF is generated.

## What gets created

| Resource | Purpose |
|----------|---------|
| Container App (`<name>`) | Runs the eSign service, HTTPS ingress, autoscaling |
| Container Apps Environment (`<name>-env`) | Hosting environment (Consumption) |
| Log Analytics workspace (`<name>-logs`) | Container logs/metrics for troubleshooting |

The API key is stored as a Container Apps **secret** and injected as the `ESIGN_API_KEY` environment variable — it is never baked into the image.

## Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| Service name | `wsm-esign-api` | Names the app + resources |
| Region | resource group's region | Pick one near your Salesforce instance |
| Size (vCPU / memory) | 1.0 vCPU / 2 GiB | Memory pairs at 2 GiB per vCPU. Use 2 vCPU / 4 GiB if you do heavy `/generate-documents` (LibreOffice) work |
| Availability | Always warm | Warm = no cold start (recommended). Scale to zero = cheapest, ~10–30s first-call latency after idle |
| Max replicas | 5 | Scale-out ceiling under load |
| API key | auto-generated | Leave blank to generate; or set your own to match an existing config |

## Cost & robustness

- **Always warm (default):** one small replica runs continuously — predictable, no cold start. Best for a service Salesforce calls synchronously during signing.
- **Scale to zero:** effectively free at low volume (within Azure's monthly Container Apps free grant), at the cost of a cold start on the first request after idle.
- The signing-path call (`/compose-pdf`) completes in seconds and stays well under the Consumption ingress request timeout (~240s).

### Advanced: very long document jobs
Large `/generate-documents` (LibreOffice) or huge merges that could exceed ~4 minutes need the Container Apps **Premium Ingress** feature on a Workload Profiles environment (a small always-on cost). This is **not** part of the one-click template. If you need it, deploy normally, then enable Premium Ingress on the environment per the [Azure docs](https://learn.microsoft.com/azure/container-apps/ingress-overview). The standard signing flow does not require it.

## Word-perfect PDF conversion (optional)

By default DOCX→PDF conversion uses the bundled **LibreOffice** — no setup, very close fidelity. For Word-identical output you can route conversion through **Microsoft Graph** (Word Online) using the customer's *own* Microsoft 365 tenant. The container reads the Graph config per request and falls back to LibreOffice whenever it isn't supplied, so this is purely additive.

Two ways to supply the config:

**A. From Salesforce (principal, no custom headers).** Store the credentials on the eSign **External Credential** principal; the package's Apex merges them into the request **body** via `{!$Credential...}` (the same secure mechanism as `sf_client_secret`) — never as custom headers, and Apex never reads the secret.

1. Register an app in the customer's Entra tenant: Microsoft Graph **application** permission `Files.ReadWrite.All` (grant admin consent), a client secret, and a licensed user with a provisioned OneDrive as scratch space.
2. On the External Credential `eSign_API` principal, add four auth parameters: `MS_TENANT_ID`, `MS_CLIENT_ID`, `MS_CLIENT_SECRET`, `MS_USER_ID`. **These names must match exactly** — a `{!$Credential...}` reference to a missing param hard-fails the callout.
3. In **eSign Settings**, turn on **Graph Creds From Principal** (default off). Only enable it *after* the four params exist; with it off, the package omits the fields entirely, so orgs that haven't configured Graph are unaffected.

**B. From Azure env vars.** Set `MS_TENANT_ID`, `MS_CLIENT_ID`, `MS_CLIENT_SECRET` (secret), `MS_USER_ID` on the Container App. Use this when the customer prefers the Graph secret stay entirely inside their Azure boundary (no SF involvement). The container reads body values first, then falls back to these env vars.

Verify: a generation log line `Graph API conversion: … bytes PDF` means Graph is active; `falling back to LibreOffice` means the config was absent/incomplete (signing still works).

## Security

- Authentication is an `X-API-Key` (or `Esign-Api-Key`) header, compared in constant time. Set a strong key (auto-generated by default).
- Optional Microsoft Graph credentials arrive in the request body (merged by Salesforce from the External Credential principal) or from env vars — never as custom headers; the secret is never read by Apex and never logged.
- HTTPS/TLS is automatic on the `*.azurecontainerapps.io` endpoint.
- The container runs as a non-root user. The image contains no secrets.
- Rotate the key any time: update the `esign-api-key` secret on the Container App and the `X-API-Key` value in Salesforce.

## Building the image (maintainers only)

The deployed image is published publicly at `ghcr.io/jcd386/wsm-esign-api`. It is built from the WSM eSign service source (FastAPI + PyMuPDF + LibreOffice). To rebuild and publish a new tag (must be built for **linux/amd64**):

```bash
# from the WSM source repo (where src/esign_app.py + docker/esign-api/ live)
docker buildx build --platform linux/amd64 \
  -f docker/esign-api/Dockerfile \
  -t ghcr.io/jcd386/wsm-esign-api:v1 -t ghcr.io/jcd386/wsm-esign-api:latest \
  --push .
```

Then bump the `containerImage` default in `azuredeploy.json` if you cut a new major tag.

---

Maintained by [We Summit Mountains](https://wesummitmountains.com). Issues and PRs welcome.
