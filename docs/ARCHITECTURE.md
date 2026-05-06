# CardputerZeroRepository architecture

Design notes and tradeoffs for hosting a Debian `.deb` repository on a single
public GitHub repo.

## Why this shape

### Not LFS

GitHub Large File Storage on the free plan is:

| | Free plan |
|---|---|
| Single file | 2 GB |
| Total storage | **1 GB** |
| Bandwidth / month | **1 GB** |
| Public repo exempt from bandwidth? | **No** |

For an apt repo with 100–500 `.deb` files at 1–20 MB each and <10k downloads
per month, LFS storage would be exhausted before any download traffic, and
the first few dozen `apt install` calls would exceed the monthly bandwidth
quota. Data packs cost $5/month per 50 GB.

### GitHub Pages + Releases

- **Pages** serves the `dists/.../Packages*` + `Release` + `InRelease` text
  files. These are small (KB per package). Pages' soft bandwidth limit is
  100 GB/month, comfortable for `apt update` traffic.
- **Releases** host the `.deb` binaries as assets. Asset size limit is 2 GB,
  no documented hard quota on total asset storage, and release bandwidth is
  tracked separately from LFS.
- `Packages` indices point to `https://github.com/OWNER/REPO/releases/download/<tag>/<file>.deb`,
  so `apt` downloads the binary directly from Releases.

### Prior art

- [`AdityaGarg8/t2-ubuntu-repo`](https://github.com/AdityaGarg8/t2-ubuntu-repo)
  — flat gh-pages layout, `apt-ftparchive` + GPG sign in Actions, no LFS.
- [`ryanfortner/box64-debs`](https://github.com/ryanfortner/box64-debs)
  — 4 years of daily CI commits, nightly arm64 builds, no LFS, repo stays
  under 100 MB.

Both use `apt-ftparchive` + `dpkg-scanpackages` + `crazy-max/ghaction-import-gpg`.

## Workflow separation (security-critical)

- **`validate-submission.yml`** runs on `pull_request`. No secrets available,
  `GITHUB_TOKEN` is read-only. Can build, test, inspect. Cannot sign or push.
- **`publish.yml`** runs on `push` to `main` (i.e. after merge). Has access
  to `secrets.GPG_PRIVATE_KEY`, can push to `gh-pages` branch, can move debs
  to Releases.

**Never use `pull_request_target` with `checkout` of the PR head.** That
combination grants secrets to arbitrary PR code — multiple public projects
have been pwned this way (see
[GitHub Security Lab: "Preventing pwn requests"](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)).

## Repository size budget

Assuming 300 apps, weekly updates, metadata only in git:

- `Packages` text: ~1 KB per app × 300 = 300 KB
- `Packages.gz`: compressed, ~100 KB total
- `Release` / `InRelease`: small, ~5 KB
- `KEY.gpg`: 3 KB
- CI logs accumulate in Actions (not in repo)

Repository should stay well under 100 MB across years of history; deb assets
live in Releases and don't count toward repo size.

## Public-repo Actions economics

- **Free unlimited minutes** for public repos (official).
- **Concurrency**: ~20 concurrent jobs free tier.
- **Job timeout**: 6 hours.
- **Workflow timeout**: 35 days.

A single PR validation completes in <2 minutes; a full re-sign on merge in
<3 minutes. Room to grow.

## Upgrade path

If this repo hits growth that strains GitHub:

1. **Multi-repo split** — one repo per component (apps / SDKs / firmware).
2. **Cloudflare R2** — egress is free; mirror `.deb` assets there while
   keeping this repo as the source of truth.
3. **deb-s3** — full migration to S3-backed hosting; minimal client impact
   since the `sources.list` URL changes but signing/layout do not.
