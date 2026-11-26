This guide covers testing callback propagation from the Host OP to the Guest OP within the REST-based federation communication model. It complements the manual testing guide, which demonstrates Guest-to-Host interaction through the EWBI REST API.

# Manual Testing on Kind (Full Guest ↔ Host Flow)

End-to-end guide for testing the complete federation flow on a single Kind cluster with two namespaces simulating a Guest OP and a Host OP.

## Architecture

```
kind-federation cluster
├── namespace: katalis-dev-host    ← simulates the Host OP
│   ├── operator pod               ← reconciles host-mode CRs (marks them Ready)
│   ├── federation-api pod         ← receives inbound EWBI HTTP calls
│   ├── Federation CR              ← created automatically by the Host API on POST /partner
│
└── namespace: katalis-dev-guest   ← simulates the Guest OP
    ├── operator pod               ← reconciles guest-mode CRs (makes outbound calls to host API)
    └── [federation-api pod]       ← optional: only needed to receive host→guest callbacks
```

**Flow initiated by the guest operator:**
1. Admin creates a `Federation` CR in `katalis-dev-guest` with `federation-relation=guest`
2. Guest operator calls `POST /partner` on the host API → host CR updated → returns `federationContextId` + 1 AZ
3. Guest selects the Availability Zones offered by the Host.
4. Admin creates `Image`, `Artefact`, `ApplicationOnboarding`, `ApplicationDeployment` CRs in `katalis-dev-guest`
5. Guest operator calls the corresponding host API endpoints for each CR
6. Host operator reconciles each host-mode CR and marks it `Ready`

## Prerequisites

- Go 1.24+, Docker, kind, kubectl, helm, kubebuilder
- A Kind cluster named `federation`:
  ```sh
  kind create cluster --name federation
  kubectl config use-context kind-federation
  ```

## 1. Build Everything

```sh
# Regenerate CRDs, deepcopy, and OpenAPI code from source
make manifests generate apigen

# Build the Helm chart (includes CRDs)
make base-chart

# Build both Docker images (operator + API)
# The Makefile defaults to --platform=linux/amd64.
# On Apple Silicon (M1/M2/M3) the Kind node runs linux/arm64 — override with PLATFORM:
#   make docker-build docker-build-api PLATFORM=linux/arm64 IMG=opg-ewbi-operator:dev IMG_API=opg-ewbi-api:dev
# On Intel Macs or Linux amd64 (default):
make docker-build docker-build-api IMG=opg-ewbi-operator:dev IMG_API=opg-ewbi-api:dev

# Load images into the kind cluster
kind load docker-image opg-ewbi-operator:dev --name federation
kind load docker-image opg-ewbi-api:dev --name federation
```

## 2. Deploy Host Namespace

```sh
helm upgrade --install federation-host dist/chart \
  --namespace katalis-dev-host --create-namespace \
  --set manager.container.image.repository=opg-ewbi-operator \
  --set manager.container.image.tag=dev \
  --set manager.container.image.pullPolicy=Never \
  --set federation.image.repository=opg-ewbi-api \
  --set federation.image.tag=dev \
  --set federation.image.pullPolicy=Never \
  --set crd.enable=true \
  --set manager.container.opgInsecureSkipVerify=true

kubectl -n katalis-dev-host get pods
```

> **Note:** `pullPolicy=Never` is required for Kind because images are loaded directly into the node, not pulled from a registry.

## 3. Deploy Guest Namespace

The guest only needs the operator (it makes outbound calls to the host API). Deploy without API, or with API if you want to test host→guest callbacks.

```sh
helm upgrade --install federation-guest dist/chart \
  --namespace katalis-dev-guest --create-namespace \
  --set manager.container.image.repository=opg-ewbi-operator \
  --set manager.container.image.tag=dev \
  --set manager.container.image.pullPolicy=Never \
  --set federation.image.repository=opg-ewbi-api \
  --set federation.image.tag=dev \
  --set federation.image.pullPolicy=Never \
  --set crd.enable=false \
  --set manager.container.opgInsecureSkipVerify=true

kubectl -n katalis-dev-guest get pods
```

## 4. Create the Guest Federation CR

This CR triggers the guest operator to call `POST /partner` on the host API.

Key fields in `config/samples/rest/federation.yaml`:
- `spec.federationData.restOptions.tokenUrl` — the **base URL** of the host API (no path suffix). The generated client appends route paths automatically. The sample file already sets `http://katalis-fm-api.katalis-dev-host.svc.cluster.local:8080`, which is correct for this Kind setup.
- `spec.federationData.restOptions.partnerStatusLink` — callback URL used by the Host to send partner status updates to the Guest.
- `spec.federationData.clientId` — sent as the `X-Client-ID` header to the Host API to identify the Guest OP.
- `metadata.labels["opg.ewbi.katalis.com/id"]` — the `origOPFederationId` business ID sent in the request body.

```sh
kubectl apply -f config/samples/rest/federation.yaml
```

### Verify Federation Establishment

```sh
# Guest operator calls the host API and updates the guest CR status
kubectl -n katalis-dev-guest get federation fed-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o yaml
# Look for: status.federationContextId, status.state: AVAILABLE, status.zoneDetails

# Host-side: the Federation CR should have been updated with initialDate, partner info and have opg.ewbi.katalis.com/federation-context-id label that matches the guest CR's status.federationContextId
kubectl -n katalis-dev-host get federation fed-e35f69d8-ae5a-456b-9f95-d950e4c03e8d -o yaml
```

Get the federation context ID for subsequent steps:
```sh
FCID=$(kubectl -n katalis-dev-guest get federation fed-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 \
  -o jsonpath='{.status.federationContextId}')
echo "FederationContextId: $FCID"
```

## 5. Upload an Image (Guest → Host)

The sample image already contains the `federation-context-id` `4d559f1b-f008-58c2-a2f8-0596892a0f7a`:

```sh
kubectl apply -f config/samples/rest/image.yaml

# Guest operator calls POST /{fcid}/images on host API
kubectl -n katalis-dev-guest get image image-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o yaml
# Verify: status.state = PENDING
kubectl -n katalis-dev-host get images  # Image CR should exist in host namespace
```

## 6. Upload an Artefact (Guest → Host)

The artefact sample references `image-2dae064c-28cc-456e-8b0a-dd67bab7d8f7` (the image from step 5):

```sh
kubectl apply -f config/samples/rest/artefactGuest.yaml

kubectl -n katalis-dev-guest get artefact artefact-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o yaml
# Verify: status.state = PENDING
```

## 7. Onboard an Application (Guest → Host)

The application onboarding sample references `artefact-2dae064c-28cc-456e-8b0a-dd67bab7d8f7` (the artefact from step 6):

```sh
kubectl apply -f config/samples/rest/appOnboard.yaml

kubectl -n katalis-dev-guest get applicationonboarding apponboard-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o yaml
# Verify: status.state = PENDING
```

## 8. Deploy an Application Deployment (Guest → Host)

The application deployment sample references `apponboard-2dae064c-28cc-456e-8b0a-dd67bab7d8f7` (the application onboarding from step 7) and zone `zone-es-madrid-001`:

```sh
kubectl apply -f config/samples/rest/appDeploy.yaml

kubectl -n katalis-dev-guest get applicationdeployment appdeploy-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o yaml
# Verify: status.state = PENDING
```

## 9. Verify the Full State

```sh
# Guest-side CRs (guest operator reconciled these, made outbound calls)
kubectl -n katalis-dev-guest get federations,images,artefacts,applicationonboardings,applicationdeployments

# Host-side CRs (created by host API upon receiving guest operator's requests)
kubectl -n katalis-dev-host get federations,images,artefacts,applicationonboardings,applicationdeployments

# Host-side Federation should be AVAILABLE; child CRs should be PENDING
kubectl -n katalis-dev-host get applicationonboardings -o yaml | grep state
```

## 10. Optional: Test the Host API Directly via curl

Port-forward to the host API to simulate raw API calls (bypassing the guest operator):

```sh
kubectl -n katalis-dev-host port-forward svc/katalis-fm-api 8080:8080 &
```

Then use the curl examples from the previous host-only flow. The API base URL is `http://localhost:8080`.

## 11. Cleanup

```sh
# Delete all guest CRs (triggers guest operator's finalizer + deletion calls to host API)
kubectl -n katalis-dev-guest delete applicationdeployments,applicationonboardings,artefacts,images,federations --all

# Wait for guest cleanup to complete, then uninstall
helm uninstall federation-guest -n katalis-dev-guest
helm uninstall federation-host -n katalis-dev-host

# Delete installed CRDs (not automatically removed by Helm because they are shared between guest and host)
kubectl delete crd federations.opg.ewbi.katalis.com images.opg.ewbi.katalis.com artefacts.opg.ewbi.katalis.com applicationonboardings.opg.ewbi.katalis.com applicationdeployments.opg.ewbi.katalis.com availabilityzones.opg.ewbi.katalis.com

# Delete kind cluster
kind delete cluster --name federation
```

## Next Steps

To test the Host → Guest callback flow (patching host-side CRs and verifying propagation to the guest), see [manual-callbacks-testing-kind.md](manual-callbacks-testing-kind.md).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Guest Federation stuck in `NOT_AVAILABLE`, no outbound call | `spec.federationData.restOptions.tokenUrl` wrong or missing | Must be the host API base URL **without** any path suffix (e.g. `http://katalis-fm-api.katalis-dev-host.svc.cluster.local:8080`) — the client appends route paths automatically |
| Host API returns 409 on `CreateFederation` | Host Federation already has `spec.initialDate` set (already established) | Delete and recreate the host Federation CR |
| `federation-context-id` label mismatch on child CRs | Used wrong/placeholder FCID | Set label from `Federation.Status.FederationContextId` |
| Child CRs return 500 `"application not found"` | Dependency not yet Ready on host | Check order: Image → Artefact → ApplicationOnboarding → ApplicationDeployment |
| Pods in `CrashLoopBackOff` | Missing env vars or RBAC | Check `kubectl -n <ns> logs <pod>` |
| Image pull errors in kind | Images not loaded | Re-run `kind load docker-image` and set `imagePullPolicy=Never` in helm values |
| `ErrImagePull` / `exec format error` on Kind | Image built for wrong CPU architecture | On Apple Silicon, build with `PLATFORM=linux/arm64` (Makefile defaults to amd64) |
| Host API returns 400 `"doesn't match schema"` | CR field values don't match swagger enum constraints | Check swagger.yaml for valid enum values (e.g., `multiUserClients`, `resourceConsumption`) |
| Stale CRDs from previous deployment | Old CRDs without Helm labels block `helm install` | Delete CRDs manually: `kubectl delete crd <name>`, patch stuck finalizers if needed |
| Guest operator caches wrong API URL | OPG client cached with old `tokenUrl` | Restart the guest operator pod to clear the OPG client cache |
