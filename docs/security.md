# Security

This document describes the security mechanisms currently implemented by the EWBI Federation Platform.

Security-related behaviour is present at several points in the federation platform, including operator communication, API requests, callbacks and access to Kubernetes resources.

---

## Security Model

Federation communication is associated with a specific federation relationship.

The `Federation` resource stores partner communication information, including:

* Partner callback credentials
* Partner status link
* Guest partner credentials

The credential information includes a client ID and token URL.

The Federation Context ID is used to associate federation resources and operations with the corresponding federation relationship.

---

## Operator-to-Operator Trust

The current implementation uses federation-specific partner information when communicating with another operator.

For example, when establishing a federation, the Guest-side Federation controller uses the partner token URL and client ID configured in the federation resource when creating the client used to send the federation request.

The same partner information is used by Guest-side controllers when performing operations such as application onboarding.

The current implementation does not define a separate operator-registration mechanism.

---

## API Authentication

The API identifies the calling client using the `X-Client-ID` HTTP header.

The client ID is extracted from the incoming request and used when processing the request.

The client ID is also used when generating the Federation Context ID.

The current `ValidateAuthHeaders` handler returns `202 Accepted` without performing additional authentication validation.

As a result, the current implementation provides client identification through `X-Client-ID`, but does not perform additional authentication validation in this handler.

---

## Callback Security

Callbacks communicate status and lifecycle updates between federation partners. The callback destination depends on the resource and operation.

The Federation resource stores federation communication information in the following fields:

- `spec.federationData.clientId`
- `spec.federationData.clientSecret`
- `spec.federationData.restOptions.tokenUrl`
- `spec.federationData.restOptions.partnerStatusLink`

The callback destinations are not all taken from the Federation resource:

- **Image callbacks** use the `imageNotifLink` stored on the Image resource.
- **Artefact callbacks** use the `artefactNotifLink` stored on the Artefact resource.
- **Application onboarding callbacks** use `appInfo.appStatusCallbackLink` stored on the ApplicationOnboarding resource.
- **Application deployment callbacks** use `partnerStatusLink` from the Federation resource.

The Guest API provides callback endpoints that process received notifications and update the corresponding local resources.

The current implementation does not define additional callback authentication or authorisation checks.

---

## Kubernetes Security

The federation controllers use Kubernetes RBAC declarations for the resources they manage.

For example, the Federation controller declares permissions for:

* `federations`
* `federations/status`
* `federations/finalizers`

The ApplicationDeployment controller declares corresponding permissions for:

* `applicationdeployments`
* `applicationdeployments/status`
* `applicationdeployments/finalizers`

The controllers use the Kubernetes client to retrieve resources and update their status.

Federation resources use resource fields to associate resources with a federation context and identify whether a resource belongs to the Guest or Host side.

The operator sets the following Kubernetes label:

```text
opg.ewbi.katalis.com/id
```

The federation relationship is stored in `spec.relationType` for child resources and `spec.federationData.relationType` for a `Federation` resource. The federation context ID is stored in `spec.federationContextId` for child resources and `status.federationContextId` for a `Federation` resource.

The relationship is represented as `GUEST` or `HOST`. These fields are used when locating the Federation associated with a resource.

### Admission Policies

The platform defines admission policies that protect federation resource consistency and prevent certain operations.

A `ValidatingAdmissionPolicy` validates create, update and delete operations on `Image`, `Artefact`, `ApplicationOnboarding` and `ApplicationDeployment` resources. It checks that the resource's `federationContextId` belongs to a known `Federation` on the same side of the relationship (`HOST` or `GUEST`).

A separate admission policy prevents direct deletion of Host-side resources.

These policies help enforce federation resource consistency and protect Host-side resources against direct deletion. They complement the controller and RBAC mechanisms described above.

---

## Application and Workload Trust

The federation resources contain application and deployment information.

An `Artefact` can contain:

* Images
* Number of instances
* Restart policy
* Compute resource requirements
* Exposed interfaces

An `ApplicationDeployment` contains information including:

* Application ID
* Application version
* Zone information
* Resource consumption
* Resource pool
* Callback link

The current implementation uses this information as part of application onboarding and deployment.

The current implementation does not define a separate mechanism for verifying the trustworthiness of application images or artefacts.

The current implementation also does not define the security mechanisms used to isolate the resulting workload after deployment.

---

## Security Boundaries

The security-related boundaries in the current implementation can be represented as:

```mermaid
flowchart LR

    G["Guest"]
    API["EWBI API"]
    CR["Federation Resources"]
    CB["Callback Interface"]
    H["Host"]

    G -->|"X-Client-ID / federation request"| API
    API --> CR
    H -->|"Status callback"| CB
    CB --> CR
```

Federation-specific client and callback information is associated with the Federation resource and used during communication between operators.

---

## Security Gaps

The following areas are not defined by the current implementation.

### Authentication Validation

The API reads `X-Client-ID`, but the current `ValidateAuthHeaders` handler does not perform additional authentication validation.

### Transport Security

The EWBI API server listens over plain HTTP; HTTPS is not configured by the API server in the current implementation.

For outbound connections to partner URLs using HTTPS, the Operator verifies server certificates unless the `--opg-insecure-skip-verify` option is enabled.

Consequently, outbound HTTPS connections support certificate verification by default, but the API server does not itself provide HTTPS transport protection. The deployment environment must account for this when securing communication between operators.

### Credential Protection

The Federation resource contains client IDs and token URLs, but the current implementation does not define how these values are protected within the Kubernetes environment.

### Callback Authentication

The current implementation uses configured callback information when sending notifications, but does not define additional callback authentication or authorisation checks.

### Application and Artefact Verification

The current implementation represents application images and artefacts but does not define a separate mechanism for verifying their authenticity or integrity.

### Workload Security

The current implementation does not define the security controls used by the local platform to isolate or protect deployed workloads.

---

## Summary

The current implementation associates operator communication with federation-specific client and callback information, identifies API clients using `X-Client-ID`, and uses Kubernetes RBAC for access to federation resources.
The implementation does not currently define additional mechanisms for authentication validation, transport security, credential protection, callback authentication, application and artefact verification, or workload isolation.

## Related Documentation

* [Quick Start Guide](Quick-start-introduction.md) — Introduction to the platform and its key concepts.
* [Architecture](architecture.md) — Describes the high-level architecture of the EWBI Federation Platform
* [Core Components](components.md) — Detailed responsibilities and implementation of the platform components.
* [Federation Model and Workflows](federation.md) — Detailed federation behaviour and resource workflows.
* [Deployment](deployment.md) — Setup and deployment of the federation manager