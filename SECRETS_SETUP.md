# GitHub Secrets Setup

## 🔐 Security Setup

Your Shopify OAuth credentials are protected and won't be committed to GitHub.

## How It Works

All stores share **one** Shopify OAuth app (same client ID/secret installed on every store), so there's a single credential pair, not one per store.

### Local Development
- Uses `config.local.yaml` (which is in `.gitignore`)
- This file only needs to set top-level `client_id`/`client_secret` with your real values
- `load_config()` merges this over `config.yaml`, inheriting the store list from there automatically — no need to duplicate stores locally
- **Never gets committed to Git**

### GitHub Actions (Production)
- Uses `config.yaml` with environment variable placeholders: `${SHOPIFY_CLIENT_ID}` / `${SHOPIFY_CLIENT_SECRET}`
- The workflow sets these once as job-level env vars, read from GitHub Secrets
- Secrets are encrypted and never exposed in logs

## 🚀 Setup Instructions

### Step 1: Add GitHub Secrets

1. Go to your GitHub repository
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Add exactly two secrets (shared across all stores, no per-store suffix):
   - **Name**: `SHOPIFY_CLIENT_ID` / **Value**: Your client ID
   - **Name**: `SHOPIFY_CLIENT_SECRET` / **Value**: Your client secret
5. Click **Add secret** for each

### Step 2: Push to GitHub

Now you can safely push:

```bash
git add .
git commit -m "Feed Manager with GitHub Pages and secrets"
git push
```

GitHub's push protection will no longer block you because `config.yaml` only contains placeholders, and `config.local.yaml` is ignored.

## 📁 File Structure

```
Feed Manager/
├── config.yaml              # Safe to commit (uses ${VARIABLES}); also the store list
├── config.local.yaml        # NEVER committed (has real credentials only, no store list)
├── .gitignore               # Ignores config.local.yaml
└── .github/workflows/
    └── generate-feeds.yml   # Uses secrets.SHOPIFY_CLIENT_ID / secrets.SHOPIFY_CLIENT_SECRET
```

## 🧪 Testing

### Local Testing
```bash
# Works automatically - uses config.local.yaml
python generate_feeds.py
```

### Testing with Environment Variables
```bash
# Simulates GitHub Actions environment
export SHOPIFY_CLIENT_ID="your_client_id"
export SHOPIFY_CLIENT_SECRET="your_client_secret"
mv config.local.yaml config.local.yaml.bak  # Temporarily hide local config
python generate_feeds.py
mv config.local.yaml.bak config.local.yaml  # Restore
unset SHOPIFY_CLIENT_ID SHOPIFY_CLIENT_SECRET
```

## 🔄 Adding More Stores

Since credentials are shared across all stores, adding a store doesn't need new secrets or workflow changes — it's a single block added to `config.yaml`'s `stores:` list. See the "Adding More Stores" section in `README.md` for the exact steps.

## ✅ Verification

Check that secrets are working:

1. **Local**: Run `python generate_feeds.py` - should work using `config.local.yaml`
2. **GitHub**: Push code and check Actions tab - workflow should succeed
3. **Logs**: GitHub Action logs will show `***` instead of actual credentials

## 🚨 Important Notes

- **NEVER** commit `config.local.yaml`
- **NEVER** put real credentials in `config.yaml`
- **ALWAYS** use `${VARIABLE}` syntax in `config.yaml`
- **DO** add new secrets in GitHub Settings before using them in workflows

## 🔒 Security Best Practices

✅ Credentials in GitHub Secrets (encrypted)
✅ Local credentials in .gitignore'd file
✅ Placeholders in committed config
✅ No credentials in logs or history
✅ Short-lived tokens (24h) via OAuth client credentials

## 🆘 If You Already Committed Credentials

1. **Regenerate credentials immediately**:
   - Go to [Shopify Partners](https://partners.shopify.com) → Apps
   - Find your app → API access → Regenerate client secret

2. **Update everywhere**:
   - `config.local.yaml` (local)
   - GitHub Secrets (GitHub Actions)

3. **Clear Git history** (optional but recommended):
   ```bash
   # Remove sensitive file from history
   git filter-branch --force --index-filter \
     "git rm --cached --ignore-unmatch config.yaml" \
     --prune-empty --tag-name-filter cat -- --all

   # Force push
   git push origin --force --all
   ```

You're now secure and ready to push! 🎉
