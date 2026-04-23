# PaletteAI <-> JumpCloud SAML SSO Automation

Ansible playbook that fully automates the SAML SSO setup between PaletteAI (via Dex) and JumpCloud.

A **single** JumpCloud SAML application is shared by all PaletteAI tenants. Tenant membership is driven by JumpCloud user groups — typically the same `<tenant>-SpectroTeam` groups that `ansible/palette_saml_sso/` already creates for Palette, so one JumpCloud group grants access to both Palette and PaletteAI.

## Quickstart

```bash
cd ansible/paletteai_saml_sso

# 1. Point kubectl at the PaletteAI EKS cluster (the playbook patches Dex,
#    canvas, and applies Tenant CRs in-cluster).
aws eks update-kubeconfig --name spectroihmeae-ue2-paletteai \
    --region us-east-2 --profile spectro

# 2. Create your local env file from the template and fill in the 1 required secret.
cp sso-env.sh.example sso-env.sh
chmod 600 sso-env.sh
$EDITOR sso-env.sh
#   JUMPCLOUD_API_KEY          (jca_...)

# 3. (Optional) Edit the tenant list — it's a YAML list in group_vars, not an env var.
$EDITOR inventory/group_vars/all.yml   # paletteai_tenants: ...

# 4. Source the env and run the playbook.
source sso-env.sh
ansible-playbook playbook.yml

# Tear down (scoped, safe — only removes what this playbook created):
ansible-playbook teardown.yml
# To also delete the Tenant + Project CRs (not done by default):
PALETTEAI_TENANTS_TEARDOWN=true ansible-playbook teardown.yml
```

`sso-env.sh` is gitignored so real credentials won't get committed. The `sso-env.sh.example` template shows every env-var override the playbook consumes; the tenant list lives in `inventory/group_vars/all.yml`.

### Helm upgrade integration

The PaletteAI Helm chart's `dex.config.connectors` is hardcoded to `[]`, so every `ansible/paletteai/deploy.yml` run regenerates the Dex Secret and wipes the JumpCloud SAML connector installed here. `deploy.yml` has a `post_tasks` hook that auto-reconciles SSO after helm finishes — it invokes this playbook **only when `JUMPCLOUD_API_KEY` is set** (opt-in), so CI without SSO credentials won't fail.

Net result:

| Scenario | What happens |
|---|---|
| `JUMPCLOUD_API_KEY` set, run `deploy.yml` | Helm upgrade runs, then this playbook auto-runs and restores Dex connector |
| `JUMPCLOUD_API_KEY` unset, run `deploy.yml` | Helm upgrade runs; skip notice warns SSO is now broken until you run this playbook manually |
| Run this playbook directly | Reconciles SSO state; safe to run independently of helm |

## What it does

1. **JumpCloud app**: Finds or creates a Custom SAML 2.0 app labelled `PaletteAI` with:
   - IdP URL: `https://sso.jumpcloud.com/saml2/paletteai`
   - SP Entity ID and ACS URL: `https://{paletteai_domain}/dex/callback`
   - `idpInitUrl: https://{paletteai_domain}/ai` (forces SP-initiated flow on tile click, since Dex doesn't support IdP-initiated)
   - NameID = user email, `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`
   - Attributes: `email`, `firstName`, `lastName`, plus group attribute `memberOf`
   - **No SP certificate** (avoids the same JumpCloud signature-validation issue that Palette has)
   - Activates the app
2. **JumpCloud user groups**: For each tenant, finds the group named by `jumpcloudGroup`. Creates it if missing (tagged with `jumpcloud_group_marker`). Associates it with the SAML app. Idempotently adds each listed member.
3. **Dex ingress**: Ensures the `dex` ingress is bound to `{paletteai_domain}` so `https://{paletteai_domain}/dex/*` is reachable.
4. **Canvas impersonation proxy**: Patches `canvas-config` secret to set `impersonationProxy.groupsMode=passthrough` so Dex-returned groups are honored. **The Helm template `ansible/paletteai/roles/paletteai/templates/values.yaml.j2` also sets this value**, so a fresh Helm install ships the right config and this patch is a no-op.
5. **Dex SAML connector**: Parses JumpCloud's IdP metadata, builds a `jumpcloud` SAML connector in the Dex config secret, restarts Dex and Canvas.
6. **PaletteAI tenants**: Applies per-tenant `Namespace` + `Tenant` CR + `Project` CR, with `tenantRoleMapping.groups` pointing at the JumpCloud group. Waits for each project's RoleBinding to reflect the group.

## Prerequisites

- JumpCloud admin API key (`jca_...`)
- `kubectl` pointed at the PaletteAI EKS cluster:
  ```bash
  aws eks update-kubeconfig --name spectroihmeae-ue2-paletteai \
      --region us-east-2 --profile spectro
  ```

## Layout

```
ansible/paletteai_saml_sso/
  ansible.cfg                        # points at ./inventory
  playbook.yml                       # create/update SSO config (entrypoint)
  teardown.yml                       # tear down SSO config (scoped, safe)
  inventory/
    hosts.yml                        # localhost only
    group_vars/
      all.yml                        # every variable with env-var fallbacks
  tasks/
    jumpcloud_app.yml                # create/find SAML app, activate, fetch IdP metadata
    jumpcloud_groups.yml             # page JumpCloud groups, dispatch per-tenant
    jumpcloud_group_one.yml          # per-tenant group create/find + app assoc + member sync
    dex_ingress.yml                  # bind dex ingress to {paletteai_domain}
    canvas_impersonation.yml         # flip groupsMode to passthrough
    dex_connector.yml                # patch Dex secret, restart Dex + canvas
    tenants.yml                      # apply Tenant + Project CRs
  README.md
```

## Configuration

Every variable lives in `inventory/group_vars/all.yml`. Edit that file for persistent per-environment defaults, or override any value at runtime by setting the matching env var.

| Variable | Env var | Default | Purpose |
|---|---|---|---|
| `jumpcloud_api_key` | `JUMPCLOUD_API_KEY` | — | JumpCloud admin API key, **required** |
| `jumpcloud_app_label` | `JUMPCLOUD_APP_LABEL` | `PaletteAI` | Display label of the JumpCloud app |
| `jumpcloud_idp_url_slug` | `JUMPCLOUD_IDP_URL_SLUG` | `paletteai` | URL slug in the JumpCloud SSO URL |
| `jumpcloud_groups_attr_name` | `JUMPCLOUD_GROUPS_ATTR` | `memberOf` | SAML groups attribute name |
| `paletteai_domain` | `PALETTEAI_DOMAIN` | `paletteai.isc-spectro-dev.click` | Public hostname of the PaletteAI install |
| `paletteai_namespace` | `PALETTEAI_NAMESPACE` | `mural-system` | Kubernetes namespace Dex + Canvas live in |
| `paletteai_tenants` | — | *(see `group_vars/all.yml`)* | YAML list of `{name, displayName, jumpcloudGroup, members?}` |
| `paletteai_tenants_prune` | `PALETTEAI_TENANTS_PRUNE` | `false` | Delete Tenant CRs not in the list |
| `paletteai_tenants_teardown` | `PALETTEAI_TENANTS_TEARDOWN` | `false` | Delete Tenant + Project CRs during teardown |

### Tenants

Default value of `paletteai_tenants`:

```yaml
paletteai_tenants:
  - name: default
    displayName: "Default Tenant"
    jumpcloudGroup: "default-SpectroTeam"    # reused from palette_saml_sso
    members: [marty@insightsoftmax.net]

  - name: isc
    displayName: "ISC"
    jumpcloudGroup: "ISC-SpectroTeam"        # reused from palette_saml_sso
    members: [marty@insightsoftmax.net]
```

## Run

```bash
cd ansible/paletteai_saml_sso

# Point kubectl at the PaletteAI cluster once per shell session.
aws eks update-kubeconfig --name spectroihmeae-ue2-paletteai \
    --region us-east-2 --profile spectro

export JUMPCLOUD_API_KEY='jca_...'

ansible-playbook playbook.yml
```

## After running

1. Users in a tenant's `jumpcloudGroup` will see the `PaletteAI` tile in the JumpCloud portal.
2. Clicking it loads `https://{paletteai_domain}/ai`, which Dex redirects through JumpCloud (SP-initiated flow).
3. On success, Dex issues an OIDC token with the user's JumpCloud groups in the `groups` claim. Canvas's impersonation proxy (with `groupsMode=passthrough`) forwards those as `Impersonate-Group` headers, and the matching tenant's RoleBinding grants access.

Re-running the playbook is safe — the JumpCloud app, groups, Dex connector, and Tenant CRs are all reused if present.

## Scope guarantees

This playbook is **strictly scoped** to the PaletteAI integration.

**On the JumpCloud side:**
- Creates/finds the SAML application whose `ssoUrl` is exactly `https://sso.jumpcloud.com/saml2/{slug}` (default `.../saml2/paletteai`)
- For each listed tenant, finds the `jumpcloudGroup` (typically reused from `palette_saml_sso`) and only creates it if missing; new groups are tagged with the per-playbook marker
- Associates those groups with the SAML app
- On teardown: deletes only groups whose description begins with the `paletteai-jumpcloud-sso` marker. Groups created by `palette_saml_sso` (different marker) are preserved — only disassociated from the app.

**On the cluster side:**
- Patches the `dex` secret + ingress + `canvas-config` secret in `{paletteai_namespace}` only
- Applies per-tenant `Namespace`, `Tenant`, and `Project` CRs
- On teardown: removes the `jumpcloud` connector from the Dex config and (opt-in) deletes Tenant + Project CRs

It will **never**:
- Touch any JumpCloud app or group not belonging to this integration
- Create, modify, or delete any JumpCloud user
- Touch the Palette-side SSO (separate playbook in `ansible/palette_saml_sso/`)

## Full clean-room test

```bash
cd ansible/paletteai_saml_sso

aws eks update-kubeconfig --name spectroihmeae-ue2-paletteai \
    --region us-east-2 --profile spectro
export JUMPCLOUD_API_KEY='jca_...'

# Tear down (deletes the PaletteAI-marked app + groups, removes Dex connector)
ansible-playbook teardown.yml

# Re-create everything
ansible-playbook playbook.yml

# Re-run to confirm idempotency
ansible-playbook playbook.yml
```

Then verify in an incognito browser: log in to JumpCloud → click the `PaletteAI` tile → land in PaletteAI as a tenant member.
