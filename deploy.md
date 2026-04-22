# Deploy to GitHub Pages

## Current state
- Repo: `git@github.com:odisei369/blog.git` (branch: `main`)
- Hugo: v0.160.1+extended (Homebrew)
- Theme: Blowfish v2 (Hugo module)
- Domain: `nidobit.com` (Cloudflare Registrar)
- No GitHub Actions workflow yet

## Steps

### 1. Create GitHub Actions workflow

Create `.github/workflows/hugo.yml` that:
- Triggers on push to `main`
- Installs Hugo extended
- Installs Go (needed for Hugo modules)
- Builds the site with `hugo --minify`
- Deploys to GitHub Pages using `actions/deploy-pages`

### 2. Enable GitHub Pages in repo settings

1. Go to `github.com/odisei369/blog/settings/pages`
2. Set **Source** to **GitHub Actions** (not "Deploy from a branch")

### 3. Configure custom domain in GitHub

1. In the same Pages settings, set **Custom domain** to `nidobit.com`
2. GitHub will verify DNS — this requires a DNS record in Cloudflare

### 4. Add DNS records in Cloudflare

In Cloudflare dashboard > `nidobit.com` > DNS:

| Type  | Name | Target                    | Proxy  |
|-------|------|---------------------------|--------|
| CNAME | @    | odisei369.github.io       | DNS only (grey cloud) |
| CNAME | www  | odisei369.github.io       | DNS only (grey cloud) |

**Important:** These must be "DNS only" (grey cloud off) — GitHub needs to see the real CNAME to issue the TLS certificate. Cloudflare proxy would interfere with GitHub's certificate verification.

### 5. Push and verify

1. Commit the workflow file
2. Push to `main`
3. Check the Actions tab for build status
4. Verify `https://nidobit.com` loads the blog

### 6. Enforce HTTPS

Once GitHub finishes DNS verification, check **Enforce HTTPS** in Pages settings.
