# tg-discover releases

Signed builds of tg-discover, a local Telegram channel-discovery app. Its background daemon installs and updates
itself from the releases here. Each release carries one bundle and a signed manifest. A bundle holds the compiled
server (readable JavaScript, its comments removed) and the web app (minified), with `package.json`, its lockfile
and the daemon's own scripts, and nothing else from the project.

## What a release contains

- `tg-discover-<version>.tgz`: the compiled server and web app, `package.json`/`package-lock.json` (dependencies
  are installed by `npm ci --omit=dev --ignore-scripts` from the public npm registry and checked against the
  lockfile's sha512 hashes) and the daemon's scripts.
- `manifest.json`: version, size and sha256 of the bundle. The version is the build time in UTC and the commit.
- `manifest.json.sig`: an ed25519 signature over the exact bytes of `manifest.json`.

## How the daemon checks a release

It accepts a release only when:
1. the signature verifies against the public key it was installed with (`release-key.pub` here);
2. the bundle's size and sha256 match the signed manifest;
3. the version is newer than the newest it has kept, which it remembers across reinstalls on the same PC (a daemon
   set up on a new PC accepts any signed release);
4. the bundle holds only plain files with plain names inside its folder (no link, no device name), and its lockfile
   names only registry packages with hashes.

It then installs the release into a new folder and runs its server and its updater on a scratch folder. Only then
does it switch, and it keeps the new version only after its server has run three minutes, from the moment it
listens, without stopping, and then answers. Otherwise it switches back and does not fetch that version again. A release stamped more than
a day ahead of the daemon's clock is not installed.

Manifests carry no expiry: someone who controlled this repository but not the signing key could not make a daemon
install anything unsigned or older, but could withhold newer releases from it.

Public key fingerprint (sha256 of the raw key): `cb6a ff7b ee8a c2da 5127 96f6 b6a1 a860`

To check by hand, with OpenSSL 3:

```
base64 -d manifest.json.sig > manifest.sig.bin
openssl pkeyutl -verify -pubin -inkey release-key.pub -rawin -in manifest.json -sigfile manifest.sig.bin
sha256sum tg-discover-<version>.tgz   # compare with "sha256" in manifest.json
```
