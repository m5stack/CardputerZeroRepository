# CardputerZero Repository

The official Debian `.deb` repository for [M5 CardputerZero](https://docs.m5stack.com/)
applications — hosted on GitHub, served via GitHub Pages.

## Quick start (on the device)

```bash
# Import the signing key
curl -fsSL https://m5stack.github.io/CardputerZeroRepository/KEY.gpg \
    | sudo tee /etc/apt/trusted.gpg.d/cardputer.asc > /dev/null

# Add the repository
echo 'deb [arch=arm64] https://m5stack.github.io/CardputerZeroRepository stable main' \
    | sudo tee /etc/apt/sources.list.d/cardputer.list

sudo apt update
sudo apt install <appname>
```

## How it works

- **Metadata** (`dists/stable/main/binary-arm64/Packages*`, `Release`, `InRelease`)
  lives on the `main` branch and is republished to `gh-pages` on every push. This
  is small text (~KB per app).
- **`.deb` binaries** live as **GitHub Release assets**, not in the git tree. The
  `Packages` index points to `https://github.com/m5stack/CardputerZeroRepository/releases/download/<tag>/<pkg>.deb`.
  This avoids LFS quotas entirely.
- **Signing** happens inside GitHub Actions using a GPG key stored as a repo secret.
  The public key is committed as `KEY.gpg` so clients can verify `InRelease`.
- **Submissions** come in as Pull Requests containing an uploaded `.deb` under
  `incoming/`. The `validate-submission.yml` workflow runs on PR without
  secrets (safe). On merge, `publish.yml` moves the file to a Release, rebuilds
  the index, signs it, and pushes to `gh-pages`.

## Architecture rationale

Why GitHub Pages + Releases and **not** LFS? LFS on free plan is 1 GB storage /
1 GB bandwidth per month — and **bandwidth counts even for public repos**. The
Pages+Releases split avoids LFS entirely; see `docs/ARCHITECTURE.md` for the
full writeup.

## Submission flow

Developers either:

1. **`czdev upload <file.deb>`** from CardputerZero-AppBuilder — opens a PR in
   this repo with the `.deb` dropped under `incoming/`.
2. **Manual PR** — drop a `.deb` into `incoming/`, open a PR. CI validates
   dpkg metadata + architecture + filename. Maintainer reviews, merges.

Auth / signing for submitters is not wired yet — maintainer merge gates the
publication.

## Layout

```
CardputerZeroRepository/
├── dists/stable/main/binary-arm64/   # apt metadata (Packages, Release, InRelease)
├── pool/main/                        # reserved; small debs may land here later
├── incoming/                         # PR landing zone, emptied on merge
├── KEY.gpg                           # public signing key
└── .github/workflows/
    ├── validate-submission.yml       # PR safety: verify deb format only
    └── publish.yml                   # on merge to main: release + reindex + sign
```

## Status

- [x] Repo structure bootstrapped (this PR)
- [x] `validate-submission.yml` — checks deb header, architecture=arm64
- [x] `publish.yml` — builds Packages/Release, signs, pushes to gh-pages
- [ ] GPG signing key added as secret (see `docs/MAINTAINERS.md`)
- [ ] GitHub Pages enabled on `gh-pages` branch
- [ ] `czdev upload` subcommand wired to this flow (CardputerZero-AppBuilder)
