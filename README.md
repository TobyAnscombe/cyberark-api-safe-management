# cyberark_safe_management

Ansible role that idempotently creates, updates, or deletes a CyberArk Privilege Cloud safe and reconciles its member permissions via the REST API.

Designed to run after the [`tobyanscombe.cyberark_auth`](https://github.com/TobyAnscombe/cyberark-api-management) role, which produces the `cyberark_token` bearer token this role consumes.

## How it works

1. Validates required variables are set
2. **present** — checks if the safe exists; creates it (POST) or updates it (PUT) to match the declared properties
3. **absent** — checks if the safe exists and deletes it (DELETE) if so
4. When `cyberark_safe_members` is non-empty (or `cyberark_safe_members_purge` is true), reconciles membership:
   - Adds members not already on the safe
   - Updates permissions for members already on the safe
   - Optionally removes non-predefined members absent from the desired list

All API calls run `delegate_to: localhost` / `run_once: true` — safe management is a control-plane operation, not per-host work.

## Requirements

- Ansible 2.9+
- `cyberark_token` set on the play (use [`tobyanscombe.cyberark_auth`](https://github.com/TobyAnscombe/cyberark-api-management) or supply it yourself)
- A CyberArk Privilege Cloud tenant

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

### Safe properties (present only)

| Variable | Default | Description |
|---|---|---|
| `cyberark_safe_description` | `""` | Free-text description |
| `cyberark_safe_location` | `"\\"` | Vault location path |
| `cyberark_safe_number_of_days_retention` | `7` | Days to retain object versions (mutually exclusive with versions retention) |
| `cyberark_safe_number_of_versions_retention` | `null` | Number of versions to retain |
| `cyberark_safe_auto_purge_enabled` | `false` | Auto-purge expired accounts |
| `cyberark_safe_olac_enabled` | `false` | Object-level access control |

### Member management (present only)

| Variable | Default | Description |
|---|---|---|
| `cyberark_safe_members` | `[]` | List of member objects to reconcile (see below) |
| `cyberark_safe_members_purge` | `false` | Remove non-predefined members not in `cyberark_safe_members` |
| `cyberark_safe_member_default_permissions` | all-false dict | Permissions applied when a member omits the `permissions` key |

#### Member object structure

```yaml
cyberark_safe_members:
  - name: svc_myapp          # required — vault user, group, or role name
    member_type: User         # User | Group | Role  (default: User)
    search_in: Vault          # Vault | Directory    (default: Vault)
    permissions:              # omit to use cyberark_safe_member_default_permissions
      useAccounts: true
      retrieveAccounts: true
      listAccounts: true
      addAccounts: false
      updateAccountContent: false
      updateAccountProperties: false
      initiateCPMAccountManagementOperations: false
      specifyNextAccountContent: false
      renameAccounts: false
      deleteAccounts: false
      unlockAccounts: false
      manageSafe: false
      manageSafeMembers: false
      backupSafe: false
      viewAuditLog: false
      viewSafeMembers: true
      accessWithoutConfirmation: false
      createFolders: false
      deleteFolders: false
      moveAccountsAndFolders: false
      requestsAuthorizationLevel1: false
      requestsAuthorizationLevel2: false
```

## Output

| Variable | Description |
|---|---|
| `cyberark_safe_detail` | Safe object returned by CyberArk after create or update |

## Usage

### Install

```yaml
# requirements.yml
roles:
  - name: cyberark_auth
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: main
  - name: cyberark_safe_management
    src: https://github.com/TobyAnscombe/cyberark-api-safe-management
    version: main
```

```bash
ansible-galaxy install -r requirements.yml
```

### Minimal playbook

```yaml
- name: Manage CyberArk safe
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"
    cyberark_safe_name: "my-application-safe"

  roles:
    - cyberark_auth
    - cyberark_safe_management
```

### Create a safe with members

```yaml
- name: Manage CyberArk safe
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"
    cyberark_safe_name: "my-application-safe"
    cyberark_safe_description: "Secrets for my-application"
    cyberark_safe_number_of_days_retention: 30

    cyberark_safe_members:
      - name: svc_myapp
        member_type: User
        search_in: Vault
        permissions:
          useAccounts: true
          retrieveAccounts: true
          listAccounts: true
          viewSafeMembers: true

  roles:
    - cyberark_auth
    - cyberark_safe_management
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

By default (`cyberark_safe_members_purge: false`) the role is additive — members already on the safe that are absent from `cyberark_safe_members` are left untouched. Set `cyberark_safe_members_purge: true` to enforce the list as the complete desired state. Built-in vault accounts (`isPredefinedUser: true`, e.g. `Master`, `Batch`) are always excluded from purge.

> **Note:** The members API is fetched without pagination. Safes with a very large number of members (>1000) may return an incomplete list via the first page. This is not a common scenario for typical usage.

## License

MIT
