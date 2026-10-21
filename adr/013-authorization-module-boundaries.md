# ADR 013: Authorization Module Boundaries

| Status | Proposed   |
| ------ | ---------- |
| Date   | 2026-10-21 |

## Context and Problem Statement

As described in [RFC 008](../rfc/008-platform-mesh-modularization.md), Platform Mesh should support any authorization engine behind the webhook or deployments without a ReBAC authorization engine at all. Deployments without an authorization module would fall back to the Kubernetes RBAC. OpenFGA will remain a default but swappable module.

Currently, OpenFGA coupling can be found outside the authorization webhook. For example:
- Security operator manages account relationship tuples.
- IAM writes role assignments directly into OpenFGA.
- Search calls FGA to filter results or list accessible accounts.

Today's state would not allow to replace authorization just by replacing the authZ webhook.

The decision is where to establish the authorization boundary which allows core components to work with each supported backend.

To achieve this, the modularization will rely on the base concepts of [the RBAC chain in kcp](https://docs.kcp.io/kcp/main/concepts/authorization/authorizers/) like Kubernetes RBAC, workspace separation and `MaximalPermissionPolicy`. Additional engines are optional and can be integrated using the `SubjectAccessReview` webhook. The developed contract must define how these authorization components interact.

## Decision Drivers
- **Replaceability:** Core components should not be tied to a specific a authorization engine and be provider-agnostic.
- **Clear ownership:** Define who manages permissions for each backend
- **Rely on base concepts:** Kubernetes RBAC, kcp, `SubjectAccessReview` and KRM resources.
- **Supported Platform Mesh compositions**: OpenFGA stays a default engine, alternative authorization module can be plugged in. Support deployments without a authorization engine that rely on native RBAC.


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
