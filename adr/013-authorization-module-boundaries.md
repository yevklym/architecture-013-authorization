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

To achieve this, the modularization will rely on the base concepts of [the RBAC chain in kcp](https://docs.kcp.io/kcp/main/concepts/authorization/authorizers/) like Kubernetes RBAC, workspace separation and `MaximalPermissionPolicy`. Additional engines are optional and can be integrated using the `SubjectAccessReview` (SAR) webhook. The developed contract must define how these authorization components interact.

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

### Authorization Contract

| Part                  | Responsibility                                                                                                                                                                                   |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Permission checks     | Evaluate an authenticated subject, groups, account/workspace scope and requested operation. Use `SubjectAccessReview` for the kcp webhook and an engine-neutral interface for service checks.    |
| Authorization intent  | Describe policies and assignments as KRM resources. Define their scope, revocation rules, ownership, creation and update.                                                                        |
| Authorization queries | Provide the role catalog and assignments needed by IAM. Provide accessible accounts and resource checks needed by search. Define the expected results and access restrictions for these queries. |

`SubjectAccessReview` can only answer a permission request, it cannot store assignments or list accessible resources. Direct calls to a webhook check only that authorizer., so service checks must preserve the same applicable restrictions as API access. The native implementation can use SAR against the target kcp workspace.

Webhook, IAM, search or other service interpret the same identity, groups and account restrictions in the same way. A user's access should not become broader by simply using a different service. Temporary grants must have documented expiry behavior and possible cache must not extend access violating that expiry rule.

### Managed state and ownership

As described in RFC 008, IAM owns reconciliation of authorization intent into the authorization engine. This means one owner for each projected permission, with backend-specific implementations behind the contract. The security-operator produces authorization intent instead of writing directly into OpenFGA. The flow for engine-specific setup looks approximately like this: _KRM authorization intent -> IAM reconciler -> backend implementation -> engine-specific policies and relationships_. With native RBAC, roles and bindings represent the authorization intent themselves. They are managed directly by kcp. No additional projection step is needed.

| Component                          | Responsibility                                                                                                                                                                         |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IAM                                | Expose role and assignment management, validate changes, and persist authorization intent. Provide the default reconciler where backend projection is required.                        |
| security-operator                  | Produce baseline authorization intent for organization and account lifecycle, including initial access and account relationships.                                                      |
| Backend implementation used by IAM | Implement the authorization contract using native RBAC resources or engine-specific policies and relationships. Keep implementation-specific identifiers and APIs behind the contract. |
| kcp and optional webhook           | Evaluate access requests using the configured authorization chain.                                                                                                                     |

Native resources such as `Role`, `ClusterRole`, `RoleBinding` and `ClusterRoleBinding` require no additional projection. Additional CRDs should be be used only if existing resources cannot describe a required authorization intent.

### IAM in native RBAC deployments

All three possible Platform Mesh compositions have IAM as a common API for role catalogs/assignments and their lifecycle. The access management functionality must work independently of an identity provider's integration.
Although kcp can enforce RBAC permissions without IAM, keeping it allows permission-management workflows and Portal integration.

### Provisioning and access grants

Permission assignment and user provisioning have independent of each other lifecycles. Creating an `Invite` does not grant permissions. A workflow may require both, but assignments use the same authorization contract for invited and externally provisioned users. Access management must work when Platform Mesh user provisioning is disabled. The contract defines how an invitation email resolves to an authenticated subject in the relevant organization and issuer scope.

Alternative IdP integrations may project explicit grants into provider roles or claim mappings, but KRM remains the source of authorization intent. Owner, scope, expiry and revocation rules are required for each projection. IAM provides a default reconciliation process into authorization backend.

### Deployment scenarios

| Scenario             | Behavior                                                                                                                        |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Default with OpenFGA | OpenFGA and its authorization webhook are deployed. Existing role assignment and account inheritance behavior remain supported. |
| Native RBAC          | No external engine or ReBAC webhook is deployed or registered. IAM implements permission management using native RBAC.          |
| Alternative engine   | An adapter implements permission checks, projection and query contracts. No changes in core components are required.            |

The `PlatformMesh` resource allows to configure an authorization module, and `platform-mesh-operator` propagates its configuration to the relevant components. There are no OpenFGA-specific attributes in `AccountInfo`. Core operators, search, shared APIs and libraries, Portal, and IAM service have no dependency on OpenFGA.

---

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
