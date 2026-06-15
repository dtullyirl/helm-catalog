# github-auth

Configures GitHub as an OpenShift identity provider and syncs GitHub org teams to OpenShift Groups using the [group-sync-operator](https://github.com/redhat-cop/group-sync-operator).

---

## Prerequisites

Before enabling this chart you need to complete three things on the GitHub side:

1. Create a GitHub OAuth App (one per cluster)
2. Create a GitHub Personal Access Token for team sync
3. Create the necessary secrets on the spoke cluster

---

## 1. GitHub Organisation setup

### Recommended team structure

Map your GitHub teams directly to OpenShift RBAC roles. A suggested starting structure:

| GitHub Team | Purpose | OpenShift role |
|---|---|---|
| `platform-admins` | Platform engineers with full cluster access | `cluster-admin` |
| `platform-readonly` | Auditors / on-call visibility | `view` |
| `digital-dev` | Digital value stream developers | `edit` on their namespaces |
| `payments-dev` | Payments value stream developers | `edit` on their namespaces |
| `mortgages-dev` | Mortgages value stream developers | `edit` on their namespaces |

### Creating teams in GitHub

1. Go to your GitHub organisation: `https://github.com/orgs/<YOUR_ORG>/teams`
2. Click **New team**
3. Set **Team name** (e.g. `platform-admins`)
4. Set **Description**
5. Set visibility to **Visible** (required for GroupSync to enumerate members)
6. Click **Create team**

Repeat for each team in the table above.

### Adding members to teams

```
GitHub Org → Teams → <team-name> → Members → Add a member
```

Or via GitHub CLI:

```bash
# Add a user to a team
gh api orgs/<YOUR_ORG>/teams/<TEAM_SLUG>/memberships/<USERNAME> \
  --method PUT \
  --field role=member

# List members of a team
gh api orgs/<YOUR_ORG>/teams/<TEAM_SLUG>/members
```

> **Note:** Users must be members of the GitHub organisation before they can be added to a team.
> Invite them at `https://github.com/orgs/<YOUR_ORG>/people`.

---

## 2. Create a GitHub OAuth App (per cluster)

Each spoke cluster gets its own OAuth App so that the callback URL is cluster-specific and credentials can be rotated independently.

### Steps

1. Go to your GitHub organisation settings:
   `https://github.com/organizations/<YOUR_ORG>/settings/applications`

2. Click **OAuth Apps** → **New OAuth App**

3. Fill in the fields:

   | Field | Value |
   |---|---|
   | Application name | `openshift-<cluster-name>` (e.g. `openshift-digital-dev`) |
   | Homepage URL | `https://console-openshift-console.apps.<cluster-domain>` |
   | Authorization callback URL | `https://oauth-openshift.apps.<cluster-domain>/oauth2callback/GitHub` |

   > The callback URL path `/oauth2callback/GitHub` must match the `idpName` value in your `clusterdef.yaml`. If you change `idpName`, update the callback URL too.

4. Click **Register application**

5. On the next screen:
   - Note the **Client ID** — this goes into `clusterdef.yaml` as `githubAuth.clientID`
   - Click **Generate a new client secret**
   - Copy the secret immediately — GitHub only shows it once

6. Repeat for each spoke cluster.

### Finding your cluster domain

```bash
oc get ingresses.config cluster -o jsonpath='{.spec.domain}'
```

---

## 3. Create a GitHub Personal Access Token (for group sync)

The group-sync-operator needs a token to call the GitHub API and enumerate org teams and members.

### Steps

1. Go to `https://github.com/settings/tokens` (use a service/bot account, not a personal account)

2. Click **Generate new token (classic)**

3. Set:
   - **Note:** `openshift-group-sync-<cluster-name>`
   - **Expiration:** 90 days (or set a reminder to rotate)
   - **Scopes:** tick `read:org` only — no other scopes are needed

4. Click **Generate token** and copy the value immediately

> **Recommendation:** Use a GitHub bot/machine account for this token rather than a personal account, so the token does not expire when a team member leaves.

---

## 4. Create secrets on the spoke cluster

These two secrets must exist before the Argo CD Application is synced. Run these commands against the spoke cluster (`oc login` to the spoke first).

```bash
# OAuth App client secret — used by the OpenShift auth operator
oc create secret generic github-oauth-client-secret \
  --from-literal=clientSecret=<GITHUB_CLIENT_SECRET> \
  -n openshift-config

# GitHub PAT — used by group-sync-operator to enumerate teams
# The group-sync-operator namespace must exist first (created by the operator install)
oc create secret generic github-groupsync-token \
  --from-literal=token=<GITHUB_PAT> \
  -n group-sync-operator
```

> These secrets are intentionally not managed by this chart. They contain credentials
> that must never appear in Git. Future improvement: replace with External Secrets
> Operator pulling from AWS Secrets Manager or HashiCorp Vault.

---

## 5. Enable the chart in platform-config

In the spoke cluster's `clusterdef.yaml`, add:

```yaml
# ── GitHub OAuth ──────────────────────────────────────────────────────────────
# Requires two pre-created secrets on the cluster — see helm-catalog/github-auth/README.md.
# clientID is unique per cluster (from the GitHub OAuth App created for this cluster).
githubAuth:
  clientID: "Ov23ctXXXXXXXXXXXX"    # from GitHub OAuth App settings
  organizations:
    - your-github-org                  # only members of this org can log in
  groupSync:
    organization: your-github-org

platform:
  argocdApps:
    githubAuth:
      enabled: true
```

---

## 6. How team names map to OpenShift Groups

After the first GroupSync run, OpenShift Groups are created matching the GitHub team slugs:

| GitHub team slug | OpenShift Group name |
|---|---|
| `platform-admins` | `platform-admins` |
| `platform-readonly` | `platform-readonly` |
| `digital-dev` | `digital-dev` |

You can verify with:

```bash
oc get groups
oc describe group platform-admins
```

These Group names can then be used in:

- **Argo CD RBAC** (`argocdInstance.rbac.platformGroup` in `default.yaml`)
- **RoleBindings** for namespace access
- **ClusterRoleBindings** for cluster-wide access

---

## 7. Trigger a manual group sync

GroupSync runs on the configured schedule. To trigger an immediate sync:

```bash
# Annotate the GroupSync CR to force an immediate reconcile
oc annotate groupsync github-group-sync \
  redhatcop.redhat.io/forcereconciliation=true \
  -n group-sync-operator --overwrite

# Watch sync status
oc get groupsync github-group-sync -n group-sync-operator -o yaml | grep -A 10 status
```

---

## 8. Troubleshooting

| Symptom | Check |
|---|---|
| Login page does not show GitHub option | Verify `oauth.config.openshift.io/cluster` has the identityProvider entry: `oc get oauth cluster -o yaml` |
| `invalid_client` error on GitHub redirect | Client ID or secret mismatch — re-check the secret and `clientID` in clusterdef.yaml |
| `redirect_uri_mismatch` from GitHub | Callback URL in the GitHub OAuth App does not match `https://oauth-openshift.apps.<domain>/oauth2callback/<idpName>` |
| Groups not appearing after login | Check GroupSync status: `oc get groupsync -n group-sync-operator` — ensure PAT has `read:org` scope |
| User can log in but has no access | Add the user to a GitHub team and wait for next GroupSync, or trigger manually (see above) |
