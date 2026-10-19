# ADR 013: Authorization Module Boundaries

### Summary

As described in [RFC 008](https://github.com/platform-mesh/architecture/blob/main/rfc/008-platform-mesh-modularization.md), Platform Mesh should support any authorization engine behind the webhook or deployments without a ReBAC authorization engine at all. Deployments without an authorization module would fall back to the Kubernetes RBAC. OpenFGA will remain a default but swappable module.

Core Platform Mesh components are provider-neutral and should not depend on a specific authorization engine.

To achieve this, the modularization will rely on the base concepts of [the RBAC chain in kcp](https://docs.kcp.io/kcp/main/concepts/authorization/authorizers/) like Kubernetes RBAC, workspace separation and `MaximalPermissionPolicy`. Additional engines are optional and can be integrated using the `SubjectAccessReview` webhook. The developed contract must define how these authorization components interact.

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
