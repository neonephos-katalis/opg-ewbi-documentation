# Deployment Guide

This guide describes how to build and deploy the Federation Manager.

The guide covers:

- Prerequisites
- Platform configuration for ARM64 and AMD64
- Registry authentication
- Building the Federation Manager images, including the debug image
- Preparing the Kubernetes environment
- Installing the Federation Manager with Helm
- Publishing Federation Manager images to GitHub Container Registry
- Installing separate Host and Guest instances for local testing
- Testing the federation flow using the sample resources
- Regenerating API code after OpenAPI specification changes

The guide also includes a reference section describing the project structure.

## Getting Started

### Prerequisites

- go version v1.25.0+
- docker version 17.03+.
- kubectl version v1.11.3+.
- Access to a Kubernetes v1.30+ cluster.

For local end-to-end testing using Kind, see [Manual Federation Testing with Kind](testing/manual-testing-kind.md) (full Guest ↔ Host flow) and [Manual Callback Testing with Kind](testing/manual-callbacks-testing-kind.md) (Host → Guest callback flow).

### Configuration: platform ARM64 or platform AMD64
This code is designed to run on both ARM64 and AMD64 platforms, but to enable this, some changes need to be made to the following files: values.yaml in (dist/chart) and the Makefile.

1. For the **value.yaml** in OPG-EWBI-OPERATOR (**dist/chart**), you need to change the repository parameter values (line 7 and line 49) as follows:

    **For Linux/ARM64**:
    - Line 7: ```ghcr.io/neonephos-katalis/opg-ewbi-operator```
    - Line 49: ```ghcr.io/neonephos-katalis/opg-ewbi-api```

    **For Linux/AMD64**:
    - Line 7: ```ghcr.io/neonephos-katalis/opg-ewbi-operator-amd64```
    - Line 49: ```ghcr.io/neonephos-katalis/opg-ewbi-api-amd64```

2. For the **Makefile** in OPG-EWBI-OPERATOR, you need to change lines 1-2 as follows:

    **For Linux/ARM64**:
    - Line 2: ```IMG ?= ghcr.io/neonephos-katalis/opg-ewbi-operator:neonephos```
    - Line 5: ```PLATFORM ?= linux/arm64```

    **For Linux/AMD64**:
    - Line 2: ```IMG ?= ghcr.io/neonephos-katalis/opg-ewbi-operator-amd:neonephos```
    - Line 5: ```PLATFORM ?= linux/amd64```

### Configuration: namespaced or cluster-wide

By default, the operator is namespaced: it watches only the namespace where it is installed, and the API creates all resources in that same namespace.

The operator can also run cluster-wide, watching all namespaces. The mode is chosen at installation time using the Helm chart's `clusterWide` value (`false` by default):

```bash
helm install federation dist/chart -n <namespace> --create-namespace --set clusterWide=true
```

With `clusterWide: true`, the chart installs a `ClusterRole` and `ClusterRoleBinding` instead of a `Role` and `RoleBinding`, and sets the `CONTROLLER_CLUSTER_WIDE` (operator) and `API_CLUSTER_WIDE` (API) environment variables to `true`.

The mode of an existing release can be switched with:

```bash
helm upgrade federation dist/chart -n <namespace> --set clusterWide=<true|false>
```

In cluster-wide mode:

- **Kubernetes technology:** Resources are created on the Host in the namespace specified by `spec.federationData.k8sOptions.namespace`. This no longer needs to be the namespace where the Host operator is installed.
- **REST technology:** When a federation is created, the API creates (or reuses) a namespace named `fedrest-<id>`. The `<id>` consists of 12 base36 characters derived from the federation ID and country code. All resources belonging to that federation are created in this namespace.

**Important:** A cluster-wide operator reconciles resources across every namespace, including those belonging to other releases, and writes cluster-scoped validation policies.

- Install at most one cluster-wide release per Kubernetes cluster.
- Do not combine cluster-wide and namespaced releases in the same cluster.
- To test Host and Guest in cluster-wide mode, use two separate clusters.
- To run multiple releases in one cluster, use namespaced mode.

## Deploy the federation manager
Install operator in host namespace, set API nodeport and set CRD to true to also install CRDs NodePorts are exposed in case testing outside of the cluster is needed.

**Note:** The default build platform in the operator Makefile is `linux/amd64`. If you are building images for a different architecture, configure the `PLATFORM` variable accordingly, as described in the platform configuration section.

1. Create a `.netrc` file in your home directory (`~/.netrc` on macOS/Linux) with the following format:

    ```
    machine ghcr.io
    login your-username
    password your-token
    ```

    Make sure the file has appropriate permissions:
    ```bash
    chmod 600 ~/.netrc
    ```
2. ```git clone https://github.com/neonephos-katalis/opg-ewbi-operator```
3. After the download, open this folder via terminal and exec the following command:

  Build the **operator** image:
  ```make docker-build-controller```
      **or**
  ```docker build --no-cache -t ghcr.io/neonephos-katalis/katalis/fm-operator:neonephos --secret id=netrc,src=$HOME/.netrc .```

  **For debugging purposes**, you can build a debug image with bash and troubleshooting tools (curl, wget, tcpdump, dig, etc.):
  ```make docker-build-debug```
      **or**
  ```docker build --target debug --no-cache -t ghcr.io/neonephos-katalis/katalis/fm-operator:neonephos-debug --secret id=netrc,src=$HOME/.netrc .```

  To use the debug image, install the chart with `--set manager.image.tag=neonephos-debug` and exec into the pod with `kubectl exec -it <pod-name> -- bash`

4. Build the **API** image:

  ```make docker-build-api```
      **or**
  ```docker build -f Dockerfile.api --no-cache -t ghcr.io/neonephos-katalis/katalis/fm-api:neonephos --secret id=netrc,src=$HOME/.netrc .```

5. ```docker login ghcr.io```
6. In your cluster create a new namespace (e.g. ```kubectl create ns federation```) after this exec this command. (replace $username and $accessToken with your username and accessToken used for the docker login ghcr.io command)
  ```bash
      kubectl -n federation create secret docker-registry opg-registry-secret \
      --docker-server=ghcr.io \
      --docker-username=$username \
      --docker-password=$accessToken
  ```
7. In the end exec this command (in the **project root folder** via terminal)

    **NOTE:** Before executing this command, edit `dist/chart/templates/rbac/host_delete_protection_policy.yaml` at `spec.matchConditions.expression`, including the correct Guest service account name. Otherwise, you won't be able to delete resources.

    ```bash
    helm install federation-manager dist/chart -n federation \
    --set federation.services.federation.nodePort=30080 \
    --set crd.enabled=true
    ```

    After this command the OPG-EWBI-CONTROLLER and OPG-EWBI-API, if everything goes well, they will be available.
By running `kubectl get pods -A`, you should see two pods running in the federation namespace with names similar to:

- `katalis-fm-controller-manager-...`
- `katalis-fm-fm-api-...`

Use the following commands only if we you want to push the latest version of the controller and the api in github pacakge

**For Linux/ARM64:**
```bash
docker push ghcr.io/neonephos-katalis/opg-ewbi-operator:neonephos
docker push ghcr.io/neonephos-katalis/opg-ewbi-api:neonephos
```

**For Linux/AMD64:** **⚠️ In this moment the images are not push on GitHub please exec the following commands**
```bash
docker push ghcr.io/neonephos-katalis/opg-ewbi-operator-amd:neonephos
docker push ghcr.io/neonephos-katalis/opg-ewbi-api-amd:neonephos
```

**For Linux/ARM64:**
```bash
docker push ghcr.io/neonephos-katalis/opg-ewbi-operator-arm64:v1.0.0
docker push ghcr.io/neonephos-katalis/opg-ewbi-api-arm64:v1.0.0
```

**For Linux/AMD64:** **NOW IMAGES AVAILABLE**
```bash
docker push ghcr.io/neonephos-katalis/opg-ewbi-operator-amd64:v1.0.0
docker push ghcr.io/neonephos-katalis/opg-ewbi-api-amd64:v1.0.0
```

The code is written to work in both roles (HOST and GUEST).

If you want to test locally, you need two Helm installations: one for the Host and one for the Guest. Use the Helm command configurations below for the respective namespaces (`katalis-dev-host` and `katalis-dev-guest`), making sure to apply the required configuration for both namespaces.

```bash
helm install federationhost dist/chart -n katalis-dev-host \
  --set federation.services.federation.nodePort=30080 \
  --set crd.enabled=true
```

We set crd to false since we have already installed them

```bash
helm install federationguest dist/chart -n katalis-dev-guest \
--set federation.services.federation.nodePort=30081 \
--set crd.enabled=false
```
**Choose the names you prefer for namespaces and helm; those listed here are provided as examples.**

### Test:

**Note:** This section describes testing using the REST-based sample resources in `config/samples/rest/`. The Kubernetes-mode samples in `config/samples/k8s/` will be documented in a future issue. 

**Note:** You can launch the end-to-end federation test from GitHub Actions locally using [act](https://github.com/nektos/act). Make sure to stop any local Kind cluster before starting the test.

#### Establish the federation

The Guest initiates federation with the Host by creating a `Federation` resource. The Host Federation is automatically created by the Host API when it receives the `POST /partner` request.

Apply the Federation resource in the Guest namespace:

```bash
kubectl apply -f config/samples/rest/federation.yaml
```

In federation.yaml, you will see that we define the URL of the Host's EWBI API, as well as the callback URL for the Guest API.

Example:

Host API (tokenUrl): http://katalis-fm-api.host.svc.cluster.local:8080
Guest callback URL (partnerStatusLink): http://katalis-fm-api.guest.svc.cluster.local:8080
The Host uses the callback URL to send partner status updates back to the Guest.

You should see the `FederationContextId` in the Federation status. Copy it and use it for the subsequent CRs.

#### Deploy resources through the federation

Once the federation has been established, apply the following sample resources in order:

```bash
kubectl apply -f config/samples/rest/zone.yaml
kubectl apply -f config/samples/rest/image.yaml
kubectl apply -f config/samples/rest/artefact.yaml
kubectl apply -f config/samples/rest/appOnboard.yaml
kubectl apply -f config/samples/rest/appDeploy.yaml
```

These resources are created in the Guest and propagated to the Host through the EWBI API.

#### Delete the sample resources

To remove the resources, delete them in reverse order:

```bash
kubectl delete -f config/samples/rest/appDeploy.yaml
kubectl delete -f config/samples/rest/appOnboard.yaml
kubectl delete -f config/samples/rest/artefact.yaml
kubectl delete -f config/samples/rest/image.yaml
kubectl delete -f config/samples/rest/zone.yaml
kubectl delete -f config/samples/rest/federation.yaml
```


## Regenerate API Code

To regenerate Go client/server code after OpenAPI specification changes:

```bash
make apigen
```

This generates code from `api/ewbi/swagger.yaml` into:
- `api/ewbi/client/` — Generated client code
- `api/ewbi/server/` — Generated server code
- `api/ewbi/models/` — Generated model definitions

## Project Structure

```
.
├── api/
│   ├── ewbi/                          # EWBI Federation API specifications and generated code
│   │   ├── Federation_API_v1.4.0.yaml # EWBI 1.4.0 OpenAPI specification
│   │   ├── swagger.yaml               # Modified EWBI 1.4.0 specification used for code generation
│   │   ├── apigen.sh                  # Code generation script
│   │   ├── client/                    # Generated client code
│   │   ├── server/                    # Generated server code
│   │   └── models/                    # Generated model definitions
│   └── operator/
│       └── v1beta1/                   # CRD type definitions
├── cmd/
│   ├── main.go                        # Operator entry point
│   └── api/
│       └── main.go                    # API server entry point
├── internal/                          # Operator internals
│   ├── config/                        # API server configuration
│   ├── controller/                    # Reconcilers
│   ├── indexer/                       # Field indexers
│   ├── k8s/                           # Kubernetes resource implementations
│   ├── rest/                          # REST API resource implementations
│   ├── multipart/                     # Multipart form helpers
│   ├── opg/                           # OPG client cache
│   └── options/                       # Namespace config
├── pkg/                               # API server packages
│   ├── handler/                       # HTTP handlers
│   ├── metastore/                     # K8s-backed persistence
│   ├── deployment/                    # Install/uninstall logic
│   └── ...
├── test/
│   ├── e2e/                           # End-to-end tests for federation resources
│   └── mock/                          # Mock implementations used in testing
├── config/                            # Kustomize manifests
├── dist/chart/                        # Helm chart
├── Dockerfile                         # Operator image
├── Dockerfile.api                     # API server image
├── Dockerfile.apigen                  # Code generator image
└── Makefile                           # Build targets
```
