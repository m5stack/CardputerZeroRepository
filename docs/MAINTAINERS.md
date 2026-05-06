# Maintainers runbook

## One-time setup (before first real publication)

### 1. Generate a GPG signing key

On a trusted machine (not the CI runner):

```bash
gpg --batch --gen-key <<EOF
%no-protection
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: CardputerZero Repository
Name-Email: repo@m5stack.local
Expire-Date: 3y
%commit
EOF

# Grab the fingerprint
FPR=$(gpg --list-keys --with-colons repo@m5stack.local | awk -F: '/^fpr/ {print $10; exit}')
echo "$FPR"

# Export the public key (commit this file)
gpg --armor --export "$FPR" > KEY.gpg

# Export the private key (store as secret, never commit)
gpg --armor --export-secret-keys "$FPR" > gpg-private.asc
```

Add `KEY.gpg` to the repo (normal commit). Set repo secrets:

- `GPG_PRIVATE_KEY` = contents of `gpg-private.asc`
- `GPG_PASSPHRASE` = passphrase if you used one (leave unset if `%no-protection`)

Shred `gpg-private.asc` once the secret is saved.

### 2. Enable GitHub Pages

- Settings → Pages → Source: `gh-pages` branch, `/` root.
- The first `publish.yml` run will create the branch; rerun the workflow if
  Settings doesn't offer `gh-pages` yet.

### 3. Flip the Pages URL into README

Once the site is live (check `https://<owner>.github.io/<repo>/`), add its URL
to `README.md` so users can copy-paste the `sources.list` line.

## Accepting a submission

1. CI must be green on the PR (`validate-submission.yml`).
2. Inspect the `.deb` metadata in the PR diff (GitHub renders `dpkg-deb -I`
   output from the action logs).
3. Merge with "Squash and merge".
4. `publish.yml` runs automatically:
   - uploads the `.deb` to the `apt-pool` release,
   - removes it from `incoming/`,
   - rebuilds `Packages` / `Release` / `InRelease`,
   - pushes metadata to `gh-pages`.

## Removing a package

1. `gh release delete-asset apt-pool <file>.deb` (or via UI).
2. `gh workflow run publish.yml` to rebuild the index without it.

## Rotating the GPG key

1. Generate a new key, commit the new `KEY.gpg`.
2. Update `GPG_PRIVATE_KEY` + `GPG_PASSPHRASE` secrets.
3. Re-run `publish.yml`; clients will see a new `InRelease` signature.
4. Announce the key change — users need to re-import `KEY.gpg`.

## Monitoring

- GitHub → Insights → Traffic: watch Pages bandwidth.
- GitHub → Releases → apt-pool: sum of asset sizes = total deb storage.
- If the repo crosses 1 GB: audit `pool/` (should be empty in committed tree —
  a trailing `.deb` there means `publish.yml` didn't clean up).
- If Pages bandwidth gets close to 100 GB/month: plan migration to Cloudflare
  R2 (see `ARCHITECTURE.md` — apt client impact is only the URL change).
