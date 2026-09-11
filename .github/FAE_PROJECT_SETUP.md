# FAE GitHub Project Setup

One-time setup so FAE can manage support issues by **Client UUID**.

## 1. Create the project

1. Open [BrookFirewall_Discuss](https://github.com/Brook-Design-LLC/BrookFirewall_Discuss) → **Projects** → **New project**
2. Choose **Table** layout
3. Name it **FAE Support** (or your preferred name)

## 2. Add the Client UUID field

1. In the project, click **+** next to existing fields
2. Add a **Text** field named exactly: `Client UUID`
3. Suggested table columns: **Title**, **Status**, **Client UUID**, **Assignee**
4. Sort or filter by **Client UUID** to group issues from the same user

## 3. Configure repository secrets

In the repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

| Secret | Value |
| --- | --- |
| `FAE_PROJECT_URL` | Full project URL, e.g. `https://github.com/orgs/Brook-Design-LLC/projects/1` |
| `GH_PAT` | Classic PAT with `repo` and **`project`** scopes (SSO authorized for the org if required) |

`GH_PAT` is also used by the log analysis workflow for private attachment downloads.

## 4. How automation works

1. [`analyze_log_issue.yml`](workflows/analyze_log_issue.yml) formats the issue from `.bkglog` / diagnostic `.zip` (includes Client UUID in the Info table), then dispatches `uuid_issue_history` via `GH_PAT` (GitHub does not forward `issues: edited` events from `GITHUB_TOKEN` actions).
2. [`update_uuid_issue_history.yml`](workflows/update_uuid_issue_history.yml) runs after that dispatch:
   - Waits ~2 minutes for GitHub search indexing
   - Appends **User Issue History** to the issue body
   - Adds the issue to the FAE project and fills the **Client UUID** field

If `FAE_PROJECT_URL` or `GH_PAT` is missing, history is still written; only the project step is skipped.

## 5. Verify

1. Open a test issue with a diagnostic archive that includes Client UUID
2. After formatting, wait ~2–5 minutes for **User Issue History** at the bottom of the issue
3. Confirm the issue appears in the FAE project with **Client UUID** populated
