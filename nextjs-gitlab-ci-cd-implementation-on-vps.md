# GitLab CI/CD Full Documentation for Next.js VPS Deployment

This document explains the full implementation from zero setup to production deployment, including why each step exists.

## 1. Goal

Automate deployment of a Next.js project to a VPS with GitLab CI/CD so that:

1. Every merge or push to main is verified.
2. Deployment runs over SSH without manual server login.
3. Environment variables are injected safely for build/runtime.
4. PM2 restarts the app automatically.
5. Common CI/VPS failures are handled predictably.

---

## 2. Final Pipeline Design

The implementation uses two stages:

1. verify
2. deploy

### verify stage

Runs in GitLab runner and checks project health:

1. pnpm install --frozen-lockfile
2. pnpm run lint
3. pnpm run build

If verify fails, deploy does not run.

### deploy stage

Runs only on main and does the following:

1. Loads SSH key from GitLab file variable.
2. SSH into VPS.
3. Marks deploy path as safe Git directory.
4. Ensures VPS trusts gitlab.com host key.
5. Optionally rewrites origin to SSH URL.
6. Writes .env.production from GitLab variables.
7. Performs deterministic code sync:
   1. git fetch
   2. git checkout -f
   3. git reset --hard
   4. git clean -fd
8. Installs dependencies and builds app.
9. Restarts PM2 process.

---

## 3. Implemented .gitlab-ci.yml Example

```yaml
stages:
  - verify
  - deploy

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH

default:
  image: node:20-alpine
  before_script:
    - corepack enable

verify:
  stage: verify
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
  script:
    - pnpm install --frozen-lockfile
    - pnpm run lint
    - pnpm run build

deploy:
  stage: deploy
  image: alpine:3.20
  needs:
    - verify
  variables:
    GIT_STRATEGY: none
    DEPLOY_PATH: "/www/wwwroot/your-app-domain.com"
    APP_NAME: "your-app-name"
    SSH_PORT: "22"
    GIT_REPO_SSH_URL: "git@gitlab.com:your-group/your-repo.git"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  environment:
    name: production
  before_script:
    - apk add --no-cache openssh-client
    - eval "$(ssh-agent -s)"
    - tr -d '\r' < "$SSH_PRIVATE_KEY" > /tmp/ci_ssh_key
    - echo >> /tmp/ci_ssh_key
    - chmod 600 /tmp/ci_ssh_key
    - ssh-keygen -y -f /tmp/ci_ssh_key >/dev/null || (echo "Invalid SSH private key in SSH_PRIVATE_KEY file variable" && exit 1)
    - ssh-add /tmp/ci_ssh_key
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan -p "$SSH_PORT" "$SSH_HOST" >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
  script:
    - |
      ssh -p "$SSH_PORT" "$SSH_USERNAME@$SSH_HOST" "
        set -euo pipefail
        git config --global --add safe.directory '$DEPLOY_PATH'
        mkdir -p ~/.ssh
        chmod 700 ~/.ssh
        ssh-keyscan -H gitlab.com >> ~/.ssh/known_hosts
        chmod 644 ~/.ssh/known_hosts

        cd '$DEPLOY_PATH'

        if [ -n "$GIT_REPO_SSH_URL" ]; then
          git remote set-url origin "$GIT_REPO_SSH_URL"
        fi

        umask 077
        {
          printf '%s\n' \
            "NEXT_PUBLIC_API_BASE_URL_LIVE=$NEXT_PUBLIC_API_BASE_URL_LIVE" \
            "NEXT_PUBLIC_API_BASE_URL_LIVE_V2=$NEXT_PUBLIC_API_BASE_URL_LIVE_V2" \
            "NEXT_PUBLIC_API_IMG_URL=$NEXT_PUBLIC_API_IMG_URL" \
            "NEXT_PUBLIC_IMG_HOST_URL=$NEXT_PUBLIC_IMG_HOST_URL" \
            "NEXT_PUBLIC_BASE_URL=$NEXT_PUBLIC_BASE_URL" \
            "NEXT_PUBLIC_SITE_NAME=$NEXT_PUBLIC_SITE_NAME" \
            "NEXT_PUBLIC_SITE_TITLE=$NEXT_PUBLIC_SITE_TITLE" \
            "NEXT_PUBLIC_PIXEL_ANALYTICS=$NEXT_PUBLIC_PIXEL_ANALYTICS" \
            "NEXT_PUBLIC_GOOGLE_TAG_MANAGER=$NEXT_PUBLIC_GOOGLE_TAG_MANAGER" \
            "NEXT_PUBLIC_GOOGLE_ANALYTICS=$NEXT_PUBLIC_GOOGLE_ANALYTICS"
        } > .env.production

        echo '==> Syncing latest code...'
        git fetch origin main --prune
        git checkout -f main || git checkout -f -B main origin/main
        git reset --hard origin/main
        git clean -fd

        echo '==> Installing dependencies...'
        corepack enable
        pnpm install --frozen-lockfile

        echo '==> Building app...'
        pnpm build

        echo '==> Restarting app with PM2...'
        (pm2 restart '$APP_NAME' --update-env || pm2 start pnpm --name '$APP_NAME' -- start)
        pm2 save

        echo '==> Deployment complete!'
      "
```

---

## 4. Required GitLab Variables

Set these in GitLab: Settings -> CI/CD -> Variables.

### SSH and server variables

1. SSH_PRIVATE_KEY
   1. Type: File
   2. Visibility: Visible
   3. Protected: Enabled for production branches
2. SSH_HOST
3. SSH_USERNAME
4. SSH_PORT
5. GIT_REPO_SSH_URL

### App environment variables

1. NEXT_PUBLIC_API_BASE_URL_LIVE
2. NEXT_PUBLIC_API_BASE_URL_LIVE_V2
3. NEXT_PUBLIC_API_IMG_URL
4. NEXT_PUBLIC_IMG_HOST_URL
5. NEXT_PUBLIC_BASE_URL
6. NEXT_PUBLIC_SITE_NAME
7. NEXT_PUBLIC_SITE_TITLE
8. NEXT_PUBLIC_PIXEL_ANALYTICS
9. NEXT_PUBLIC_GOOGLE_TAG_MANAGER
10. NEXT_PUBLIC_GOOGLE_ANALYTICS
11. NEXT_PUBLIC_GOOGLE_CLIENT_ID
12. NEXT_PUBLIC_GOOGLE_CLIENT_SECRET

Important:

1. Do not mask variables with spaces.
2. Any variable prefixed with NEXT_PUBLIC_ is exposed to browser-side code.
3. Sensitive secrets should not use NEXT_PUBLIC_ prefix.

---

## 5. One-Time VPS Setup

### A. Install required tools

```bash
sudo apt update
sudo apt install -y git curl
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo corepack enable
sudo npm i -g pm2
```

### B. Prepare app directory

```bash
mkdir -p /www/wwwroot/your-app-domain.com
cd /www/wwwroot/your-app-domain.com
```

### C. Create VPS key for GitLab repository access

This key is for VPS -> GitLab fetch.

```bash
ssh-keygen -t ed25519 -C "vps-gitlab-deploy" -f ~/.ssh/id_ed25519_gitlab -N ""
```

Configure SSH identity:

```bash
cat >> ~/.ssh/config <<'EOF'
Host gitlab.com
  HostName gitlab.com
  User git
  IdentityFile ~/.ssh/id_ed25519_gitlab
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

Add this public key in GitLab Project -> Settings -> Repository -> Deploy Keys:

```bash
cat ~/.ssh/id_ed25519_gitlab.pub
```

Verify access:

```bash
ssh -T git@gitlab.com
```

---

## 6. CI Runner Key Setup (GitLab -> VPS SSH)

This key is for GitLab runner -> VPS.

Generate key locally:

```bash
ssh-keygen -t ed25519 -C "gitlab-ci-vps-deploy" -f ~/.ssh/gitlab_ci_vps -N ""
```

Install public key on VPS deploy user:

```bash
ssh-copy-id -i ~/.ssh/gitlab_ci_vps.pub -p 22 your_user@your_vps_ip
```

Set private key content in GitLab variable SSH_PRIVATE_KEY (File type).

---

## 7. Why These Specific Fixes Were Added

These were implemented from real deployment errors:

1. Key permission or parse failure:
   1. Fixed with normalized temp key file, chmod 600, ssh-keygen validation.
2. Dubious ownership in Git:
   1. Fixed with safe.directory config.
3. Host key verification failure to GitLab from VPS:
   1. Fixed with ssh-keyscan for gitlab.com on VPS session.
4. HTTPS auth prompt during git pull:
   1. Fixed by using SSH remote URL and deploy key.
5. Local file modifications blocking checkout:
   1. Fixed with forced checkout/reset/clean strategy.

---

## 8. Deployment Flow End-to-End

1. Developer pushes code to main.
2. GitLab verify job runs lint + build.
3. If verify passes, deploy job starts.
4. Deploy job loads SSH key and connects to VPS.
5. VPS syncs code to exact origin/main state.
6. VPS regenerates .env.production.
7. VPS installs dependencies and builds Next.js app.
8. PM2 restarts app process.
9. Deployment completes.

---

## 9. Operations Checklist

Before first production deploy:

1. Verify SSH_PRIVATE_KEY is valid private key (not .pub).
2. Verify VPS deploy user can login via key.
3. Verify VPS can run ssh -T git@gitlab.com.
4. Verify GIT_REPO_SSH_URL is correct.
5. Verify PM2 is installed and available in PATH.
6. Verify DEPLOY_PATH exists and is writable.

After deploy:

1. Check process list:

```bash
pm2 ls
```

2. Check logs:

```bash
pm2 logs your-app-name --lines 100
```

3. Verify site and API connectivity from browser.

---

## 10. Troubleshooting Quick Reference

### error in libcrypto

Cause:

1. Invalid or malformed SSH private key variable.

Fix:

1. Recreate key.
2. Save in GitLab as File variable.
3. Do not paste public key.

### detected dubious ownership

Cause:

1. Git ownership/safety restriction on VPS path.

Fix:

```bash
git config --global --add safe.directory /www/wwwroot/your-app-domain.com
```

### Host key verification failed

Cause:

1. Missing gitlab.com host key in VPS known_hosts.

Fix:

```bash
ssh-keyscan -H gitlab.com >> ~/.ssh/known_hosts
```

### could not read Username for https://gitlab.com

Cause:

1. Repository remote is HTTPS on VPS.

Fix:

1. Switch origin to SSH URL.
2. Ensure deploy key exists in GitLab.

### local changes would be overwritten by checkout

Cause:

1. Manual edits on VPS working tree.

Fix:

1. Use forced sync in deployment:
   1. checkout -f
   2. reset --hard
   3. clean -fd

---

## 11. Reuse Template for Other Projects

For each new project, usually only change:

1. DEPLOY_PATH
2. APP_NAME
3. GIT_REPO_SSH_URL
4. Set of environment variables
5. Domain and reverse proxy config

Everything else can stay identical.

---

## 12. Security Recommendations

1. Prefer non-root deploy user.
2. Use Protected branches and Protected variables.
3. Keep SSH_PRIVATE_KEY as File variable, not plain text variable.
4. Do not store server-only secrets in NEXT_PUBLIC variables.
5. Rotate deploy keys periodically.
6. Restrict firewall to required ports.

