# ADR 013: Authorization Module Boundaries

| Status | Proposed   |
| ------ | ---------- |
| Date   | 2026-10-21 |

## Context and Problem Statement

As described in [RFC 008](../rfc/008-platform-mesh-modularization.md), Platform Mesh should support any authorization engine behind the webhook or deployments without a ReBAC authorization engine at all. Deployments without an authorization module would fall back to the Kubernetes RBAC. OpenFGA will remain a default but swappable module.

Currently, OpenFGA coupling can be found outside the authorization webhook. For example:

- Security operator manages account relationship tuples.
- IAM writes role assignments directly into OpenFGA, also from invitation workflows.
- Search calls FGA to filter results or list accessible accounts.

Today's state would not allow to replace authorization just by replacing the authZ webhook.

The decision is where to establish the authorization boundary which allows core components to work with each supported backend.

To achieve this, the modularization will rely on the base concepts of [the RBAC chain in kcp](https://docs.kcp.io/kcp/main/concepts/authorization/authorizers/) like Kubernetes RBAC, workspace separation and `MaximalPermissionPolicy`. Additional engines are optional and can be integrated using the `SubjectAccessReview` webhook. The developed contract must define how these authorization components interact.

## Decision Drivers

- **Replaceability:** Core components should not be tied to a specific a authorization engine and be provider-agnostic.
- **Clear ownership:** Define which manages permissions for each authorization implementation
- **Rely on base concepts:** Kubernetes RBAC, kcp, `SubjectAccessReview` and KRM resources.
- **Supported Platform Mesh compositions**: OpenFGA stays a default engine, alternative authorization module can be plugged in. Support deployments without a authorization engine that rely on native RBAC.

## Considered Options

1.  **Abstract away only the `SubjectAccessReview` webhook:** permission checks become independent of the engine, but it is not enough to decouple IAM and search and handle authorization state management.
2.  **Define a contract for authorization intent, permission checks and queries:** the contract can be implemented by existing components or a shared authorization service.
3.  **Define the contract that requires a separate operator for every authorization backend:** each authorization module would have own reconciliation component, but it makes deployment more complicated for a consumer and moves away ownership of a single reconciler form the IAM Service.

## Decision Outcome

Chosen option: **2. Define a contract for authorization intent, permission checks and queries.** As described [in RFC 008](../rfc/008-platform-mesh-modularization.md#authorization), the approach includes both the `SubjectAccessReview` webhook and KRM authorization intent projected into the engine by IAM Service.

The authorization contract covers permission checks, authorization intent and queries. Authorization intent is reconciled through an engine-neutral contract. IAM provides the default reconciler and backend adapters apply changes to the selected authorization implementation.

### kcp RBAC

To achieve this, the modularization will rely on the base concepts of [the RBAC chain in kcp](https://docs.kcp.io/kcp/main/concepts/authorization/authorizers/). Roles and bindings grant access within a workspace. `APIExport` and `APIBinding` make APIs available across workspaces. An API provider can additionally define a [`MaximalPermissionPolicy`](https://docs.kcp.io/kcp/main/concepts/apis/exporting-apis/#maximal-permission-policy) to limit access to exported resources.

Authorization modules are integrated through the `SubjectAccessReview` webhook. The integration must specify authorizer order and `Allow`, `Deny` and `NoOpinion` behavior. If a backend is temporarily unavailable it must not switch to native RBAC or allow failed checks.

kcp evaluates maximal permission policies (MPP) within its RBAC chain. The complication is that the RBAC chain and the external webhook are separate authorizers -- kcp API server has a built-in RBAC chain, including MPP checks, but it separately calls Platform Mesh’s authorization webhook over HTTPS. That means we cannot assume that every authorization path includes MPP checks. Authorization integration must preserve the applicable workspace and provider boundaries, including configured maximal permission policies, and demonstrate this in contract tests.

### Deployment scenarios

- Default: existing authorization with OpenFGA.
- No authorization module: Platform Mesh uses KCP/K8s authorization functionality.
- Alternative engine: can be plugged through the authorization contract without making changes to core components.

### Decisions / Open Questions

- IAM: Can we use native RBAC deployments without IAM-service? Where does baseline permissions and backend projection logic go if we want to extract it? (Core operator, separate AuthZ operator, security-operator or combination?)

### Current state

- IAM FGA functionality and K8S Native alternative:
  - List users and assignments (platform-mesh/services/iam-service/pkg/fga/fga.go:77) -> look up role bindings? 
  - List assignable roles (platform-mesh/services/iam-service/pkg/fga/fga.go:77) -> Roles/ClusterRoles (how do we handle role catalog/descriptions)
  - Assign and remove roles services/iam-service/pkg/fga/invite.go:178 -> CRUD for RoleBindings/ClusterRoleBindings 
  - Invite a user and assign roles (platform-mesh/services/iam-service/pkg/fga/invite.go:50) -> custom invitation workflow? 

- IAM writes assignments (platform-mesh/services/iam-service/pkg/fga/invite.go:178)
- Security-operator maintains account relationships (platform-mesh/operators/security-operator/internal/subroutine/account_tuples.go:99)
- OpenFGA model defines inheritance (helm-charts/charts/security-operator/values.yaml:171).
- In some places the IAM service has methods using both FGA and keycloak (e.g. platform-mesh/services/iam-service/pkg/resolver/pm/resolver_service.go:78)
