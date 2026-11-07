# Popamp apt repository

Builds and mirrors <https://apt.popamp.app>, the apt repository for
Popamp, its skins and its plug-ins.

```
release in popamp, skin-*, plugin-* (.deb assets)
   │  gh workflow run index.yml -R popamp/apt   (and once a day)
   ▼
Index workflow: bin/build-index ─► signs with APT_SIGNING_KEY
   ─► release "index": apt-index.tar.gz (dists/ + pool.txt), pool.txt
   │
   ▼  server, cron: bin/mirror (no secrets)
checks InRelease against site/popamp.gpg ─► downloads new .debs from
GitHub ─► checks each against the signed index ─► switches dists/
```

The index lists the `.deb` files of the latest release of every public
repository of popamp, so a new skin needs no change here: it publishes
a release, and the index picks it up. The packages stay on GitHub and are
never uploaded again; only the index, a few kilobytes, is rebuilt. Two
repositories may not ship the same package for the same architecture.

One suite, `stable`, component `main`, for amd64 and arm64 (and `all`,
for skins and plug-ins). Indexes are plain and `.xz`, never `.gz`:
HestiaCP's nginx and Cloudflare cache `*.gz` for a long time.

## Files

| Path | What |
| --- | --- |
| `bin/build-index` | builds and signs the index; run by CI |
| `bin/mirror` | mirrors the index and its packages into a web root; for cron |
| `.github/workflows/index.yml` | the Index workflow |
| `site/` | copied into the web root on every mirror run: the public key, the sources file |

## Signing key

The secret part exists only in the repository secret `APT_SIGNING_KEY`
(armored, no passphrase) and in an offline backup. `site/popamp.gpg` is
its public part; the Index workflow refuses a secret that does not match
it, and `bin/mirror` accepts only indexes it signed.

## A release elsewhere starts the Index workflow

Each repository whose releases carry `.deb` files ends its release job
with:

```yaml
      - name: Rebuild the apt index
        env:
          GH_TOKEN: ${{ secrets.APT_INDEX_TOKEN }}
        run: |
          if [ -z "$GH_TOKEN" ]; then
            echo "::warning::APT_INDEX_TOKEN is not set; the daily Index run will pick this release up"
            exit 0
          fi
          gh workflow run index.yml -R popamp/apt --ref main
```

`APT_INDEX_TOKEN` is a fine-grained token for popamp/apt only, with
Actions: read and write. popamp is a user, not an organisation, so there
are no shared secrets: set it in each repository, e.g.

```sh
gh secret set APT_INDEX_TOKEN -R popamp/skin-steel
```

## Server (HestiaCP)

The web domain `apt.popamp.app` needs nothing special: it is all static
files, and none of them has an extension HestiaCP caches for long.

```sh
git clone https://github.com/popamp/apt ~/apt
~/apt/bin/mirror ~/web/apt.popamp.app/public_html
```

Then, as root: `v-add-cron-job Popamp '*/10' '*' '*' '*' '*' '/home/Popamp/apt/bin/mirror /home/Popamp/web/apt.popamp.app/public_html'`.
A run that finds nothing new prints nothing. A change to `site/` reaches
the server with `git -C ~/apt pull`.

## For users

```sh
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSLo /etc/apt/keyrings/popamp.gpg https://apt.popamp.app/popamp.gpg
sudo curl -fsSLo /etc/apt/sources.list.d/popamp.sources https://apt.popamp.app/popamp.sources
sudo apt update
sudo apt install popamp           # or popamp-plus
```
