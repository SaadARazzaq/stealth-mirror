# Stealth Mirror

<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/080390f6-e5b5-4ad5-95c6-82f99022d17a" />

## 🕵️ An Event Driven Invisible, Anonymous Repository Mirror for Agencies

A fully automated, zero‑trace mirroring system that **isolates clients from developers**.  
Push any change to the **private organization’s private repository** (developer/team side -> added as collaborator) and it gets instantly copied to your **personal private repository** (client side -> client added as collaborator), with **all commit authors anonymised**. No polling, no visible configuration in the source repo, and the mirroring logic lives exclusively in the target repo.

Perfect for agencies who need to:
- Give clients access to a **sanitised, always‑up‑to‑date** codebase.
- Hide developer/team identities and internal commit history.
- Keep the mirroring completely invisible to the development team.

---

## 🏗 Architecture Overview

1. **Webhook (invisible to non‑admins)** in the organization repo sends push events to a **Cloudflare Worker**.  
2. The worker dispatches a `repository_dispatch` event to your personal private repo.  
3. A **GitHub Actions workflow** in your personal repo:  
   - Clones the organization repo.  
   - Rewrites **all commit authors** to “Anonymous”.  
   - Force‑pushes the anonymised history to the personal repo.  
   - Restores the workflow file (self‑healing).

The whole process takes seconds, uses only free services, and leaves **no trace in the organization repository** that developers can see.

> Note: Always make sure to add developers and team members inside the organization's repo with (Read/Write Acess Only)

---

## 📋 Prerequisites

- A **GitHub organization** with a private repository (source, dev‑side).  
- A **personal GitHub account** with a private repository (target, client‑side).  
- A free **Cloudflare** account (for the Worker).  
- Basic familiarity with copy‑pasting code and navigating GitHub settings.

---

## 🪜 Step‑by‑Step Setup

> **Boilerplate placeholders** – Throughout this guide, you’ll see placeholders like `YOUR_ORG_NAME/YOUR_SOURCE_REPO`.  
> **Replace them** with your actual organization name, source repo, target username, target repo, etc.

---

### 1. Set the Developer Permissions in the Organization Repo

- Navigate to the link https://github.com/YOUR_ORG_NAME/YOUR_SOURCE_REPO/settings/access
- Give **Write** access to the developer. `(This is recommended for contributors who actively contribute to your code.)`
<img width="1199" height="643" alt="image" src="https://github.com/user-attachments/assets/18a0b05e-c493-452d-b1b5-4f34ad2eba7d" />

- ⬇ If you are adding a member to the repo, choose `write` role here.

<img width="528" height="454" alt="image" src="https://github.com/user-attachments/assets/712528e7-4724-4b1d-bbe9-1859571f90dc" />

### 2. Create the Required Personal Access Tokens (PATs)

#### ⚙ Settings/Configurations to Set Before Creating Tokens:

##### 👉 Organizational Personal Access Token Srttings

<img width="826" height="761" alt="image" src="https://github.com/user-attachments/assets/044ebb90-ecdb-409b-a509-8b445acdc608" />

<img width="1122" height="705" alt="image" src="https://github.com/user-attachments/assets/841f346f-9e86-411a-abb6-8ada97f4aed1" />

You’ll need **two** Fine-grained PATs. 
1. SOURCE_REPO_PAT -> This Access token is of the `YOUR_ORG_NAME/YOUR_SOURCE_REPO`, the source repo (developer added repo) from which you want to mirror into your personal private repo (client added repo).
2. DISPATCH_PAT -> This Access token is of the `YOUR_USERNAME/YOUR_TARGET_REPO`, your personal target repo (client added repo) in which you want to mirror the source repo (developer added repo).

You’ll need **one** Personal access token (Classic).
1. WORKFLOW_RESTORE_PAT -> This is inside your main profile token.

Generate all these 3 tokens from your **personal GitHub account** (the one that has access to both repos).

Go to **Settings → Developer settings → Personal access tokens**. Or you can go to this link https://github.com/settings/personal-access-tokens

<img width="1585" height="359" alt="image" src="https://github.com/user-attachments/assets/db45224b-7a8f-41d2-a7c7-19c3c49bfb11" />

<img width="1585" height="507" alt="image" src="https://github.com/user-attachments/assets/61defa81-00a0-4568-899a-b26ac7e7c1c2" />

#### 🔑 Token A – `SOURCE_REPO_PAT`  
- **Purpose:** Read access to the organization’s source repository.  
- **Scopes:** `repo` (full control of private repositories).  
- **Expiration:** No expiration (recommended).  
- **Where to store:** As a **secret** in the target repository (see Step 4).  
- **Special:** If your organization enforces SAML SSO, after creation click **Configure SSO** and authorize this token for the organization.
- **How to generate:**

<img width="1008" height="357" alt="image" src="https://github.com/user-attachments/assets/faa57b19-8bd8-42b1-b3fb-f0f7513d66ac" />

<img width="427" height="534" alt="image" src="https://github.com/user-attachments/assets/535d4c4f-38fc-4b3e-9536-47fab1a779df" />

<img width="576" height="357" alt="image" src="https://github.com/user-attachments/assets/22a337e6-9e7e-4378-a1cb-ed9870531275" />

<img width="786" height="170" alt="image" src="https://github.com/user-attachments/assets/e517d1ae-f840-4577-8920-93456795b420" />

**Copy and Save this Token Securely on your device**

#### 🔑 Token B – `DISPATCH_PAT` (aka `GITHUB_DISPATCH_TOKEN`)  
- **Purpose:** Allows the Cloudflare Worker to trigger the `repository_dispatch` event in the target repository.  
- **Scopes:** `repo` (needs write access to the target repo).  
- **Expiration:** No expiration.  
- **Where to store:** As an **environment variable** in the Cloudflare Worker (Step 2).
- **How to generate:**

<img width="1008" height="357" alt="image" src="https://github.com/user-attachments/assets/faa57b19-8bd8-42b1-b3fb-f0f7513d66ac" />

<img width="441" height="532" alt="image" src="https://github.com/user-attachments/assets/835b372d-4ffa-4688-a336-2e714df08c28" />

<img width="587" height="361" alt="image" src="https://github.com/user-attachments/assets/5587d1ea-5630-4c3f-b6f9-a2c62379ec50" />

<img width="790" height="180" alt="image" src="https://github.com/user-attachments/assets/1f71b900-a59a-4f95-861e-b9bce3087522" />

**Copy and Save this Token Securely on your device**

#### 🔑 Token C – `WORKFLOW_RESTORE_PAT`  
- **Purpose:** Needed to push the workflow file back into the target repository after it gets overwritten by the mirror.  
- **Scopes:** `repo` **and** `workflow` (Update GitHub Action workflows).  
- **Expiration:** No expiration.  
- **Where to store:** As a **secret** in the target repository (Step 4).
- **How to generate:**

<img width="1447" height="358" alt="image" src="https://github.com/user-attachments/assets/4d11d5a6-f40e-4a02-bcf9-7b069e7ed36c" />

<img width="828" height="269" alt="image" src="https://github.com/user-attachments/assets/fa23cc21-112f-43bd-a175-f4a09d5b3856" />

<img width="525" height="385" alt="image" src="https://github.com/user-attachments/assets/f4d4a6f7-e234-4a6f-9ea3-2ff407b6ccd1" />

<img width="637" height="73" alt="image" src="https://github.com/user-attachments/assets/79c0b7b9-95f0-4a7f-ae21-3f737189884a" />

**Copy and Save this Token Securely on your device**

> 🔒 All tokens should be kept secure. Never commit them to code. Always use GitHub Secrets or Cloudflare environment variables.

---

### 2. Set Up the Cloudflare Worker

The worker acts as an invisible bridge between the webhook and your target repo.

1. Log into [Cloudflare Dashboard](https://dash.cloudflare.com/) and go to **Workers & Pages**.
2. Create a new Worker, give it a name (e.g., `mirror-bridge`).
3. Replace the default code with the **worker.js** code below.
4. In the Worker’s **Settings → Variables**, add an **encrypted environment variable**:
   - **Name:** `GITHUB_DISPATCH_TOKEN`  
   - **Value:** The **DISPATCH_PAT** (Token B) you generated earlier.

<img width="1442" height="213" alt="image" src="https://github.com/user-attachments/assets/1adf493c-9fd9-4c28-aa9f-79939837e4ac" />

<img width="1440" height="925" alt="image" src="https://github.com/user-attachments/assets/511abbbe-a6b5-4235-867e-1981acb418f0" />

5. Deploy the Worker. Note the worker URL (e.g., `https://mirror-bridge.your-subdomain.workers.dev`).

#### 📄 `worker.js` (Cloudflare Worker Code)

```javascript
// Cloudflare Worker – Webhook Bridge
// Listens to push events from the source repo and dispatches to the target repo.

// 🔽 REPLACE with a random secret string – used to verify webhook integrity.
const WEBHOOK_SECRET = 'your-random-secret-string';

// 🔽 REPLACE with the target repository's dispatch URL.
// Pattern: https://api.github.com/repos/YOUR_USERNAME/YOUR_TARGET_REPO/dispatches
const REPO_B_DISPATCH_URL = 'https://api.github.com/repos/YOUR_USERNAME/YOUR_TARGET_REPO/dispatches';

async function verifySignature(request) {
  const signature = request.headers.get('X-Hub-Signature-256');
  if (!signature) return false;
  const body = await request.clone().text();
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw', encoder.encode(WEBHOOK_SECRET),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['verify']
  );
  const sigBytes = Uint8Array.from(
    signature.replace('sha256=', '').match(/.{1,2}/g).map(byte => parseInt(byte, 16))
  );
  const dataBytes = encoder.encode(body);
  return crypto.subtle.verify('HMAC', key, sigBytes, dataBytes);
}

export default {
  async fetch(request, env, ctx) {
    // The dispatch token comes from Cloudflare environment variables
    const GITHUB_DISPATCH_TOKEN = env.GITHUB_DISPATCH_TOKEN;

    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    const valid = await verifySignature(request);
    if (!valid) {
      return new Response('Invalid signature', { status: 403 });
    }

    const payload = await request.json();
    const eventType = request.headers.get('X-GitHub-Event');

    if (eventType === 'ping') {
      return new Response('Ping successful', { status: 200 });
    }
    if (eventType !== 'push') {
      return new Response('Ignored event', { status: 200 });
    }

    const dispatchBody = JSON.stringify({
      event_type: 'mirror-trigger',
      client_payload: {
        ref: payload.ref,
        repository: payload.repository.full_name,
        pusher: payload.pusher.name
      }
    });

    const response = await fetch(REPO_B_DISPATCH_URL, {
      method: 'POST',
      headers: {
        'Authorization': `token ${GITHUB_DISPATCH_TOKEN}`,
        'Content-Type': 'application/json',
        'User-Agent': 'cloudflare-worker-mirror'
      },
      body: dispatchBody
    });

    if (response.ok) {
      return new Response('Dispatch sent', { status: 200 });
    } else {
      const err = await response.text();
      return new Response(`Dispatch failed: ${err}`, { status: 500 });
    }
  }
};
```

---

### 3. Set Up the Webhook in the Organization Repo (Source)

Only **organization/repository admins** can see webhooks – developers won’t even know it exists.

1. Go to the **source repository** (the one developers push to) → **Settings → Webhooks → Add webhook**.
2. **Payload URL:** Paste your Cloudflare Worker URL (from Step 2).
3. **Content type:** `application/json`
4. **Secret:** Enter the **same random string** you used as `WEBHOOK_SECRET` in the worker.
5. **Events:** Select **“Just the push event”**.
6. Ensure **Active** is checked, then **Add webhook**.

After saving, GitHub will send a `ping` to your worker. If everything is correct, you’ll see a `200` response in the worker’s logs.

---

### 4. Set Up the Target Repository (Personal Private Repo)

This is the repository that will contain the mirrored, anonymised code.  
All the automation (workflow + secrets) lives here.

#### 4.1 Add the Required Secrets

In your target repository, go to **Settings → Secrets and variables → Actions** and add these two **repository secrets**:

| Secret Name            | Value                               |
|------------------------|-------------------------------------|
| `SOURCE_REPO_PAT`      | Token A (read access to source repo) |
| `DISPATCH_PAT`         | Token B (read and write access to target repo) |
| `WORKFLOW_RESTORE_PAT` | Token C (write + workflow scope)     |

#### 4.2 Add the GitHub Actions Workflow

Create a file in the target repository at `.github/workflows/mirror-on-dispatch.yml` with the content below.  
**Make sure to replace the placeholder URLs** (both the source and target repository URLs) as indicated in the comments.

```yaml
name: Mirror from org repo (anonymous, self-healing)

on:
  repository_dispatch:
    types: [mirror-trigger]

jobs:
  mirror:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      # Step 1: Save the current workflow file content
      - name: Checkout repo B to capture workflow file
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          path: original-repo-b

      - name: Save workflow file (base64)
        id: save
        run: |
          WORKFLOW_FILE="original-repo-b/.github/workflows/mirror-on-dispatch.yml"
          if [ -f "$WORKFLOW_FILE" ]; then
            echo "exists=true" >> $GITHUB_OUTPUT
            echo "content=$(base64 -w0 "$WORKFLOW_FILE")" >> $GITHUB_OUTPUT
          else
            echo "exists=false" >> $GITHUB_OUTPUT
          fi

      # Step 2: Clone org repo, anonymise entire history, force-push to target repo
      - name: Anonymise and mirror org repo
        run: |
          # 🔽 REPLACE "YOUR_ORG_NAME/YOUR_SOURCE_REPO.git" with your source repository (organization repo)
          git clone --bare https://x-access-token:${{ secrets.SOURCE_REPO_PAT }}@github.com/YOUR_ORG_NAME/YOUR_SOURCE_REPO.git source-bare
          cd source-bare

          # Anonymise all commits, tags, and branches
          git filter-branch --env-filter '
            export GIT_AUTHOR_NAME="Anonymous"
            export GIT_AUTHOR_EMAIL="anonymous@vyro.local"
            export GIT_COMMITTER_NAME="Anonymous"
            export GIT_COMMITTER_EMAIL="anonymous@vyro.local"
          ' -- --all

          # 🔽 REPLACE "YOUR_USERNAME/YOUR_TARGET_REPO.git" with your target repository (personal repo)
          git remote add target https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/YOUR_USERNAME/YOUR_TARGET_REPO.git
          git push target --mirror --force

      # Step 3: Restore the mirror workflow file using WORKFLOW_RESTORE_PAT
      - name: Restore mirror workflow via Git
        if: steps.save.outputs.exists == 'true'
        run: |
          # 🔽 REPLACE "YOUR_USERNAME/YOUR_TARGET_REPO.git" with your target repository
          git clone https://x-access-token:${{ secrets.WORKFLOW_RESTORE_PAT }}@github.com/YOUR_USERNAME/YOUR_TARGET_REPO.git repo-b-restore
          cd repo-b-restore
          mkdir -p .github/workflows
          echo "${{ steps.save.outputs.content }}" | base64 -d > .github/workflows/mirror-on-dispatch.yml
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .github/workflows/mirror-on-dispatch.yml
          git commit -m "🔄 Auto-restore mirror workflow"
          git push origin main
```

---

### 5. One‑Time Anonymisation (Only If the Source Repo Already Has Commits)

If your source repository already contains commit history, you must anonymise it **once** locally before the automated mirror starts.  
This prevents real author names from ever appearing in the target repository.

Run these commands in your terminal (replace the tokens and URLs):

```bash
# 1. Clone the source (organization) repo using SOURCE_REPO_PAT
# 🔽 Replace YOUR_SOURCE_REPO_PAT and YOUR_ORG_NAME/YOUR_SOURCE_REPO
git clone https://x-access-token:YOUR_SOURCE_REPO_PAT@github.com/YOUR_ORG_NAME/YOUR_SOURCE_REPO.git temp-source
cd temp-source

# 2. Anonymise all commits, tags, and branches
git filter-branch --env-filter '
export GIT_AUTHOR_NAME="Anonymous"
export GIT_AUTHOR_EMAIL="anonymous@vyro.local"
export GIT_COMMITTER_NAME="Anonymous"
export GIT_COMMITTER_EMAIL="anonymous@vyro.local"
' -- --all

# 3. Push the anonymised history to the target (personal) repo using WORKFLOW_RESTORE_PAT
# 🔽 Replace YOUR_WORKFLOW_RESTORE_PAT and YOUR_USERNAME/YOUR_TARGET_REPO
git remote add target https://x-access-token:YOUR_WORKFLOW_RESTORE_PAT@github.com/YOUR_USERNAME/YOUR_TARGET_REPO.git
git push target --mirror --force

# 4. Clean up (optional)
cd ..
rm -rf temp-source
```

After this, your target repository will have the full anonymised history. The workflow file will be deleted, but we’ll add it manually right afterwards (see Step 4.2).

---

### 6. Trigger and Verify

1. Ensure the workflow file (Step 4.2) is committed to the target repository.
2. Push **any change** (even a small edit) to the source repository.
3. Immediately check the **Actions** tab in your target repository – you should see a workflow run triggered by `repository_dispatch`.
4. After it completes, browse the target repository’s code and commit history.  
   All commits should show the author as **Anonymous**.
5. The workflow file will have been automatically restored (self‑healing).

From now on, every push to the source repository will be mirrored instantly and anonymously.

---

## ❗ Troubleshooting / Common Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Workflow not triggered after push | Webhook not set up, or worker environment variable missing | Verify webhook in source repo (admin only). Check worker logs. Ensure `GITHUB_DISPATCH_TOKEN` env var is set and valid. |
| `Repository not found` when cloning source | `SOURCE_REPO_PAT` lacks permissions, or SSO not authorized | Generate a classic PAT with `repo` scope, authorize it for the organization (Configure SSO). |
| `Workflows permission denied` during restore | The `WORKFLOW_RESTORE_PAT` is missing the `workflow` scope | Re‑create Token C with **both** `repo` and `workflow` scopes. |
| `fatal: options '--all' and '--tags' cannot be used together` | Git syntax error in manual terminal step | Use `git push target --mirror --force` instead of combining flags. |
| Merge‑base error during cherry‑pick | Histories have diverged; cherry‑pick requires a common ancestor | Use the full `git filter-branch` + `--mirror` approach (as in the final workflow) instead of incremental cherry‑picking. |
| Workflow disappears after mirror | Expected – the force‑push deletes it. The self‑healing step restores it automatically. | Ensure `WORKFLOW_RESTORE_PAT` is correctly stored as a secret and has the `workflow` scope. |

---

## 📚 Boilerplate Reference

Replace the following placeholders across all code and commands:

| Placeholder | Description |
|-------------|-------------|
| `YOUR_ORG_NAME/YOUR_SOURCE_REPO` | The full name of the organization’s source repository (e.g., `my‑org/internal‑project`) |
| `YOUR_USERNAME/YOUR_TARGET_REPO` | The full name of your personal target repository (e.g., `john‑doe/client‑portal`) |
| `YOUR_SOURCE_REPO_PAT` | Classic PAT with `repo` scope (read source) |
| `YOUR_WORKFLOW_RESTORE_PAT` | Classic PAT with `repo` + `workflow` scopes |
| `your-random-secret-string` | A strong random string for webhook verification |
| `your-worker-subdomain.workers.dev` | The Cloudflare Worker URL |

---

With this setup, you’ve built a **completely invisible, event‑driven, self‑healing, anonymous mirror**. The client never sees the developers, the developers never know about the mirror, and the agency maintains full control in the middle.

If you have any questions or want to extend this guide, feel free to open an issue or contribute.

**Happy mirroring! 🚀**
