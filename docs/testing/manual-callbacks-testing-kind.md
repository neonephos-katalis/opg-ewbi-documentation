# Manual Testing on Kind (Callbacks: Host → Guest)

This guide covers testing callback propagation from the Host OP to the Guest OP within the REST-based federation communication model. It complements the manual testing guide, which demonstrates Guest-to-Host interaction through the EWBI REST API.

This flow assumes the base federation has already been created and that the guest-side resources already exist in `PENDING` state.

## Architecture
```
kind-federation cluster
├── namespace: katalis-dev-host    ← simulates the Host OP
│   ├── operator pod               ← sends callbacks to the guest API
│   ├── federation-api pod         ← provides the Host API
│   └── host-side CRs              ← patched manually to trigger callbacks
│
└── namespace: katalis-dev-guest   ← simulates the Guest OP
    ├── operator pod               ← reconciles guest-mode CRs
    ├── federation-api pod         ← receives host callbacks
    └── guest-side CRs             ← should reflect callback state changes
```

**Callback flow initiated by the host side:**
1. Admin patches a host-side CR status
2. Host operator sends the callback to the guest API
3. Guest-side CR status is updated to match the callback payload

## Prerequisites

- The Kind environment from `manual-testing-kind.md` is already running
- Federation is established on both sides
- Guest and host resources already exist
- Initial state is:
  - `Federation.status.state=AVAILABLE`
  - `Image.status.state=PENDING`
  - `Artefact.status.state=PENDING`
  - `ApplicationOnboarding.status.state=PENDING`
  - `ApplicationDeployment.status.appInstanceInfo.appInstanceState=PENDING`

## 1. Verify Initial State Before Callback Tests

```sh
kubectl -n katalis-dev-guest get federation,image,artefact,applicationonboarding,applicationdeployment
kubectl -n katalis-dev-host get federation,image,artefact,applicationonboarding,applicationdeployment
```

Expected:
- Federation is AVAILABLE on both sides
- Child resources are PENDING on both sides

You can inspect the exact status fields with:

```sh
kubectl -n katalis-dev-guest get image image-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
kubectl -n katalis-dev-guest get artefact artefact-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
kubectl -n katalis-dev-guest get applicationonboarding apponboard-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
kubectl -n katalis-dev-guest get applicationdeployment appdeploy-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.appInstanceInfo.appInstanceState}{"\n"}'
```

Expected:
- All four: `PENDING`

## 2. Image Callback Cases

### Image → READY

```sh
kubectl patch images image-5b959240-ab77-5da3-b3ff-8be24bb86b25 \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"state":"READY"}}'

kubectl -n katalis-dev-guest get image image-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
```

Expected: `READY`

### Image → ERROR

```sh
kubectl patch images image-5b959240-ab77-5da3-b3ff-8be24bb86b25 \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"state":"ERROR"}}'

kubectl -n katalis-dev-guest get image image-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
```

Expected: `ERROR`

## 3. Artefact Callback Cases

### Artefact → READY

```sh
kubectl patch artefacts artefact-f1f1db9a-a828-55d4-8687-a6b40c61c6f7 \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"state":"READY"}}'

kubectl -n katalis-dev-guest get artefact artefact-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
```

Expected: `READY`

### Artefact → ERROR

```sh
kubectl patch artefacts artefact-f1f1db9a-a828-55d4-8687-a6b40c61c6f7 \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"state":"ERROR"}}'

kubectl -n katalis-dev-guest get artefact artefact-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
```

Expected: `ERROR`

## 4. ApplicationOnboarding Callback Cases

### ApplicationOnboarding → ONBOARDED

```sh
kubectl patch applicationonboardings apponboard-441af12b-346d-57d8-b96a-2df354328c9c \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"state":"ONBOARDED"}}'

kubectl -n katalis-dev-guest get applicationonboarding apponboard-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
```

Expected: `ONBOARDED`

### ApplicationOnboarding → FAILED

```sh
kubectl patch applicationonboardings apponboard-441af12b-346d-57d8-b96a-2df354328c9c \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"state":"FAILED"}}'

kubectl -n katalis-dev-guest get applicationonboarding apponboard-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.state}{"\n"}'
```

Expected: `FAILED`

## 5. ApplicationDeployment Callback Cases

### ApplicationDeployment → READY

```sh
kubectl patch applicationdeployments appdeploy-3a468fe4-38d8-5ad8-9da6-b1e9b81a300d \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"appInstanceInfo":{"appInstanceState":"READY"}}}'

kubectl -n katalis-dev-guest get applicationdeployment appdeploy-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.appInstanceInfo.appInstanceState}{"\n"}'
```

Expected: `READY`

### ApplicationDeployment → FAILED

```sh
kubectl patch applicationdeployments appdeploy-3a468fe4-38d8-5ad8-9da6-b1e9b81a300d \
  -n katalis-dev-host --type=merge --subresource=status \
  -p '{"status":{"appInstanceInfo":{"appInstanceState":"FAILED"}}}'

kubectl -n katalis-dev-guest get applicationdeployment appdeploy-2dae064c-28cc-456e-8b0a-dd67bab7d8f7 -o jsonpath='{.status.appInstanceInfo.appInstanceState}{"\n"}'
```

Expected: `FAILED`

## 6. Verify Full Callback Propagation

```sh
kubectl -n katalis-dev-guest get images,artefacts,applicationonboardings,applicationdeployments
kubectl -n katalis-dev-host get images,artefacts,applicationonboardings,applicationdeployments
```

Expected:
- Host resources reflect the patched states
- Matching guest resources also updated after callback delivery


## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Host resource patched but guest state does not change | Guest API not deployed or not reachable | Verify guest federation API pod/service is running |
| Host state changes but callback is never delivered | `spec.federationData.restOptions.partnerStatusLink` missing or incorrect in Federation spec | Verify the guest Federation's `spec.federationData.restOptions.partnerStatusLink` |
| Callback returns error | Guest-side object not found | Verify guest resource exists and labels match the expected federation context |
| Guest status remains PENDING | Callback ID or federation context mismatch | Check `opg.ewbi.katalis.com/federation-callback-id` and `opg.ewbi.katalis.com/federation-context-id` labels |
| Patch succeeds but wrong resource updated | Using generated host object name from a previous run | Re-list host resources and patch the current object name |
| No callback logs visible | Wrong namespace or pod selected | Check logs in both `katalis-dev-host` and `katalis-dev-guest` |

## 8. Useful Commands

Watch guest operator logs:

```sh
kubectl -n katalis-dev-guest logs -l app.kubernetes.io/name=opg-ewbi-operator -f
```

Watch host operator logs:

```sh
kubectl -n katalis-dev-host logs -l app.kubernetes.io/name=opg-ewbi-operator -f
```

List all current resources:

```sh
kubectl -n katalis-dev-guest get federations,images,artefacts,applicationonboardings,applicationdeployments
kubectl -n katalis-dev-host get federations,images,artefacts,applicationonboardings,applicationdeployments
```
