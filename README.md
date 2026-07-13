# cyberark_safe_management

Ansible role that idempotently creates, updates, or deletes a CyberArk Privilege Cloud safe and reconciles its member permissions via the REST API.

Designed to run after the [`tobyanscombe.cyberark_api_authentication`](https://github.com/TobyAnscombe/cyberark-api-management) role, which produces the `cyberark_token` bearer token this role consumes.

## How it works

1. Validates that `cyberark_token`, `cyberark_subdomain`, and `cyberark_safe_name` are set
2. **present** — checks if the safe exists; creates it (POST) or updates it (PUT) to match the declared properties
3. **absent** — checks if the safe exists and deletes it (DELETE) if so
4. When members are declared (or purge is enabled), reconciles safe membership:
   - Adds members not already on the safe
   - Updates permissions for members already on the safe
   - Optionally removes non-predefined members absent from the desired list
5. If break glass is enabled, ensures the break glass member is always present with full permissions

All API calls run `delegate_to: localhost` / `run_once: true` — safe management is a control-plane operation, not per-host work.

## Requirements

- Ansible 2.14+
- `cyberark_token` present on the play — produced by [`tobyanscombe.cyberark_api_authentication`](https://github.com/TobyAnscombe/cyberark-api-management) or supplied by any other means
- A CyberArk Privilege Cloud tenant

## Install

```yaml
# requirements.yml
roles:
  - name: cyberark_api_authentication
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: v1.1.0
  - name: cyberark_safe_management
    src: https://github.com/TobyAnscombe/cyberark-api-safe-management
    version: v1.3.2
```

```bash
ansible-galaxy install -r requirements.yml
```

## Role variables

### Connection

| Variable | Required | Default | Description |
|---|---|---|---|
| `cyberark_subdomain` | yes | `""` | Privilege Cloud subdomain — forms `<subdomain>.privilegecloud.cyberark.cloud` |
| `cyberark_validate_certs` | no | `true` | Validate TLS certificates |

### Safe identity

| Variable | Required | Default | Description |
|---|---|---|---|
| `cyberark_safe_name` | yes | `""` | Safe name — max 28 characters (Privilege Cloud limit; validated up front, fails fast rather than a generic API 400) |
| `cyberark_safe_state` | no | `present` | `present` or `absent` |

### Safe properties

Applied on both create and update.

| Variable | Default | Description |
|---|---|---|
| `cyberark_safe_description` | `""` | Free-text description |
| `cyberark_safe_location` | `"\\"` | Vault folder location |
| `cyberark_safe_number_of_days_retention` | `null` | Days to retain object versions (mutually exclusive with versions retention) |
| `cyberark_safe_number_of_versions_retention` | `null` | Number of versions to retain |
| `cyberark_safe_auto_purge_enabled` | `false` | Auto-purge expired accounts |
| `cyberark_safe_olac_enabled` | `false` | Object-level access control |

### Member management

The role separates members into three tiers, applied in order and with different purge behaviour:

| Tier | Variable | Purge behaviour |
|---|---|---|
| Standard | `cyberark_safe_standard_members` | Never purged |
| Custom | `cyberark_safe_members` | Purged when `cyberark_safe_members_purge: true` |
| Break glass | `cyberark_safe_break_glass_*` | Never purged |

| Variable | Default | Description |
|---|---|---|
| `cyberark_safe_standard_members` | four built-in groups (see below) | Always applied to every safe; never removed by purge |
| `cyberark_safe_members` | `[]` | Per-safe custom members (e.g. service accounts) |
| `cyberark_safe_members_purge` | `false` | Remove members not in `cyberark_safe_members` (standard and break glass members always excluded) |
| `cyberark_safe_members_reconcile_permissions` | `false` | When `false`, only add missing members; when `true`, also push declared permissions onto existing members |
| `cyberark_safe_purge_system_members` | `["PSMAppUsers"]` | CyberArk system-managed members always excluded from purge — extend, don't override |
| `cyberark_safe_purge_exclude_members` | `[]` | Environment-specific members excluded from purge (e.g. API service account, CPM account name) |
| `cyberark_safe_member_default_permissions` | all-false | Fallback permissions when a `cyberark_safe_members` entry omits the `permissions` key |

`cyberark_safe_standard_members` defaults to the four Privilege Cloud groups. Add your Entra groups to the corresponding CyberArk group; permissions flow through automatically without per-safe configuration:

```yaml
cyberark_safe_standard_members:
  - name: "Privilege Cloud Administrators"
    member_type: Group
    permissions: "{{ cyberark_safe_permissions_administrator }}"
  - name: "Privilege Cloud Auditors"
    member_type: Group
    permissions: "{{ cyberark_safe_permissions_auditor }}"
  - name: "Privilege Cloud Safe Managers"
    member_type: Group
    permissions: "{{ cyberark_safe_permissions_safe_manager }}"
  - name: "Privilege Cloud Access Approvers"
    member_type: Group
    permissions: "{{ cyberark_safe_permissions_approver }}"
```

Each item in `cyberark_safe_members` (custom additions):

| Key | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | CyberArk group, user, or role name |
| `member_type` | no | `User` | `User`, `Group`, or `Role` — see note below |
| `permissions` | no | `cyberark_safe_member_default_permissions` | Permission dict (camelCase keys) |

> **Group → Role fallback:** when `member_type: Group` fails with SFWS0010 (member not found as a vault group), the role automatically retries with `memberType: Role`. This covers SCIM-synced Entra groups that exist only as CyberArk Identity roles and have no vault group equivalent — no caller change required. Successful retries are noted as `(as Role)` in the provision summary (`standard member (as Role)` for a standard-tier retry, `(as Role)` alone for a custom-tier retry). Explicitly set `member_type: Role` to skip the initial Group attempt.

> **Provision summary comments:** an `add` row for a standard-tier member (the always-on defaults in `cyberark_safe_standard_members`) is noted as `standard member` in the summary, matching how the always-on break glass member is noted as `break glass` — both are global, non-optional members, unlike custom-tier members which get no comment.

### Predefined permission sets

Four variables ship with permissions matching CyberArk's built-in Privilege Cloud roles. Use them in a member's `permissions` key instead of spelling out every flag.

These vars, plus `cyberark_safe_break_glass_permissions` and the additional presets below, all live in their own file, `defaults/main/permission_presets.yml`, rather than the role's main `defaults/main/main.yml` — external consumers can `include_vars` that file directly as a single source of truth instead of hand-copying the values (as `ansible-cyberark` now does). `defaults/main.yml` is a directory as of v1.2.1; if any tooling expects it to be a single file, resolve role defaults through Ansible's normal precedence rather than reading that path directly.

| Variable | Maps to |
|---|---|
| `cyberark_safe_permissions_approver` | Access Approver |
| `cyberark_safe_permissions_auditor` | Privilege Cloud Auditor |
| `cyberark_safe_permissions_safe_manager` | Privilege Cloud Safe Manager |
| `cyberark_safe_permissions_administrator` | Privilege Cloud Administrator |

#### Permission matrix

| Permission | Approver | Auditor | Safe Manager | Administrator |
|---|:---:|:---:|:---:|:---:|
| `useAccounts` | | | | ✓ |
| `retrieveAccounts` | | | | ✓ |
| `listAccounts` | ✓ | ✓ | ✓ | ✓ |
| `addAccounts` | | | ✓ | ✓ |
| `updateAccountContent` | | | ✓ | ✓ |
| `updateAccountProperties` | | | ✓ | ✓ |
| `initiateCPMAccountManagementOperations` | | | | ✓ |
| `specifyNextAccountContent` | | | | ✓ |
| `renameAccounts` | | | ✓ | ✓ |
| `deleteAccounts` | | | ✓ | ✓ |
| `unlockAccounts` | | | ✓ | ✓ |
| `manageSafe` | | | ✓ | ✓ |
| `manageSafeMembers` | | | ✓ | ✓ |
| `backupSafe` | | | | ✓ |
| `viewAuditLog` | | ✓ | ✓ | ✓ |
| `viewSafeMembers` | ✓ | ✓ | ✓ | ✓ |
| `accessWithoutConfirmation` | | | | ✓ |
| `createFolders` | | | ✓ | ✓ |
| `deleteFolders` | | | ✓ | ✓ |
| `moveAccountsAndFolders` | | | ✓ | ✓ |
| `requestsAuthorizationLevel1` | ✓ | | | |
| `requestsAuthorizationLevel2` | | | | |

Override individual keys with Ansible's `combine` filter:

```yaml
permissions: "{{ cyberark_safe_permissions_auditor | combine({'retrieveAccounts': true}) }}"
```

### Additional permission presets

A few more presets ship for common access patterns beyond CyberArk's four built-in roles — not mapped to a specific built-in role, but common enough across deployments to ship rather than every consumer redefining them.

| Variable | Description |
|---|---|
| `cyberark_safe_permissions_no_access` | All-false. Useful as a base for `combine()`-ing a narrow, explicit grant on top of. |
| `cyberark_safe_permissions_ppa_owner` | Personal privileged account owner, CPM-managed (`automatic_management: true`) — user triggers CPM rotation rather than setting the password directly. |
| `cyberark_safe_permissions_personal_owner` | Personal account owner, no CPM (`automatic_management: false`) — user manages their own password directly. Differs from `ppa_owner` in exactly `updateAccountContent` and `initiateCPMAccountManagementOperations`; defined as a `combine()` on top of it. |
| `cyberark_safe_permissions_connect_only` | PSM session access only, no password retrieval — e.g. third-party vendor access. |
| `cyberark_safe_permissions_retrieve_only` | Password/secret retrieval only, no PSM session — inverse of `connect_only`; e.g. automation or scripts needing the credential value. |

### Break glass access

A dedicated emergency-access member that is unconditionally added to every safe managed by this role, and is **never removed** by the purge logic.

| Variable | Default | Description |
|---|---|---|
| `cyberark_safe_break_glass_enabled` | `true` | Enable break glass member management |
| `cyberark_safe_break_glass_member` | `"Privilege Cloud Break Glass Access"` | CyberArk Identity role name to grant break glass access — must be an Identity API role, not a vault-native group |
| `cyberark_safe_break_glass_member_type` | `Group` | `User`, `Group`, or `Role` |
| `cyberark_safe_break_glass_permissions` | all-true (see defaults) | Override to restrict the break glass permission set |

The default permission set is identical to `cyberark_safe_permissions_administrator` with one key difference: `accessWithoutConfirmation: true` is always set so the account can bypass dual-control workflows in an emergency. `requestsAuthorizationLevel1` and `requestsAuthorizationLevel2` are `false` — setting those would impose confirmation requirements on the break glass account itself.

**Break glass vs Administrator** — both receive full permissions. The distinction is intent and lifecycle: the administrator role is for day-to-day admin users declared in `cyberark_safe_members`; break glass is a single protected account that is guaranteed to be present on every safe and immune to purge.

## Output

| Variable | Description |
|---|---|
| `cyberark_safe_detail` | Safe object returned by CyberArk after create or update |

## Usage

### Minimal playbook

```yaml
- name: Manage CyberArk safe
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"
    cyberark_safe_name: "my-application-safe"
    # cyberark_client_id and cyberark_client_secret from vault

  roles:
    - cyberark_api_authentication
    - cyberark_safe_management
```

### Add a per-safe custom member

```yaml
  vars:
    cyberark_safe_name: "my-application-safe"
    cyberark_safe_members:
      - name: svc_myapp
        member_type: User
        permissions:
          useAccounts: true
          retrieveAccounts: true
          listAccounts: true
          viewSafeMembers: true
```

### With break glass

Break glass is enabled by default using the `Privilege Cloud Break Glass Access` Identity role. **Important:** the break glass member must be a role created via the CyberArk Identity API (`/Roles/StoreRole`) — vault-native groups created directly in Privilege Cloud cannot be added to safes via the REST API. No configuration is needed unless you want to disable it or change the role name:

```yaml
  vars:
    cyberark_safe_name: "my-application-safe"
    cyberark_safe_break_glass_enabled: false   # disable if not required
```

### Delete a safe

```yaml
  vars:
    cyberark_safe_name: "my-application-safe"
    cyberark_safe_state: absent
```

### Vault-encrypt credentials

```bash
ansible-vault encrypt_string 'my-client-id'     --name cyberark_client_id
ansible-vault encrypt_string 'my-client-secret' --name cyberark_client_secret
```

```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

## Member purge behaviour

By default the role is additive — existing members not listed in `cyberark_safe_members` are left untouched. Set `cyberark_safe_members_purge: true` to enforce the declared list as the complete desired state.

Six categories of member are always excluded from purge:
- **Built-in vault accounts** (`isPredefinedUser: true`, e.g. `Master`, `Batch`)
- **Read-only memberships** (`isReadOnly: true`) — safe creator or Identity-managed direct assignments that cannot be deleted or updated via the Members API
- **Standard members** — the four built-in Privilege Cloud groups in `cyberark_safe_standard_members`
- **Break glass member** — excluded regardless of whether `cyberark_safe_break_glass_enabled` is true
- **System members** — `cyberark_safe_purge_system_members` (defaults to `PSMAppUsers`, which Privilege Cloud adds automatically for PSM sessions)
- **Explicit exclusions** — `cyberark_safe_purge_exclude_members` for environment-specific accounts such as the API service account or CPM account name

> **Note:** The members endpoint is fetched as a single page. Safes with more than ~1000 members may return an incomplete list. This is not a common scenario in practice.

## License

MIT
