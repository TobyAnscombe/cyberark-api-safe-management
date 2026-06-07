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

- Ansible 2.9+
- `cyberark_token` present on the play — produced by [`tobyanscombe.cyberark_api_authentication`](https://github.com/TobyAnscombe/cyberark-api-management) or supplied by any other means
- A CyberArk Privilege Cloud tenant

## Install

```yaml
# requirements.yml
roles:
  - name: cyberark_api_authentication
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: main
  - name: cyberark_safe_management
    src: https://github.com/TobyAnscombe/cyberark-api-safe-management
    version: main
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
| `cyberark_safe_name` | yes | `""` | Safe name |
| `cyberark_safe_state` | no | `present` | `present` or `absent` |

### Safe properties

Applied on both create and update.

| Variable | Default | Description |
|---|---|---|
| `cyberark_safe_description` | `""` | Free-text description |
| `cyberark_safe_location` | `"\\"` | Vault folder location |
| `cyberark_safe_number_of_days_retention` | `7` | Days to retain object versions (mutually exclusive with versions retention) |
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
| `member_type` | no | `User` | `User`, `Group`, or `Role` |
| `permissions` | no | `cyberark_safe_member_default_permissions` | Permission dict (camelCase keys) |

### Predefined permission sets

Four variables ship with permissions matching CyberArk's built-in Privilege Cloud roles. Use them in a member's `permissions` key instead of spelling out every flag.

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
| `viewAuditLog` | ✓ | ✓ | ✓ | ✓ |
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

### Break glass access

A dedicated emergency-access member that is unconditionally added to every safe managed by this role, and is **never removed** by the purge logic.

| Variable | Default | Description |
|---|---|---|
| `cyberark_safe_break_glass_enabled` | `true` | Enable break glass member management |
| `cyberark_safe_break_glass_member` | `"CyberArk Break Glass Access"` | CyberArk group name to grant break glass access |
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

Break glass is enabled by default using the `CyberArk Break Glass Access` group. No configuration is needed unless you want to disable it or change the group name:

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

Two categories of member are always excluded from purge:
- **Built-in vault accounts** (`isPredefinedUser: true`, e.g. `Master`, `Batch`)
- **Break glass member** — excluded regardless of whether `cyberark_safe_break_glass_enabled` is true

> **Note:** The members endpoint is fetched as a single page. Safes with more than ~1000 members may return an incomplete list. This is not a common scenario in practice.

## License

MIT
