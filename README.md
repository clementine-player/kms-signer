# kms-signer

Signs Android app bundles and APKs with a key held in Cloud KMS. The private key never exists
outside Cloud KMS: not in CI, not on a developer machine, and not while making the signing
certificate. It signs [Clementine Remote](https://github.com/clementine-player/Android-Remote)'s
Google Play releases, and was adapted from hatstand/nowt's `mobile/kms-signer`.

## How it works

`apksig`, the library `apksigner` and the Android Gradle plugin use, has an extension point
for keys held elsewhere: `KeyConfig.Kms` and `com.android.apksig.kms.KmsSignerEngineProvider`,
found through `ServiceLoader`. It defines `KmsType.GCP` but ships no implementation; this tool
is that implementation. apksig builds the signature files exactly as it would for a local key
and hands over only the bytes to sign. `GcpKmsSignerEngine` hashes them locally and sends the
SHA-256 digest to Cloud KMS's `AsymmetricSign`, with its CRC32C integrity checks both ways.
Only a digest and a signature cross the wire.

Google Play takes app bundles, which carry a plain JAR signature (APK signature schemes v2 and
v3 only apply to APKs, and Play re-signs those itself with the app signing key). For a `.aab`
the tool enables JAR signing only; a bundle has no top-level manifest for apksig to read the
minimum SDK from, so `--min-sdk` gives it. APKs get JAR signing and schemes v2 and v3.

Clementine Remote's key, created by its `scripts/gcp_play_setup.sh`, is `RSA_SIGN_PKCS1_3072_SHA256`. Keys must be
RSA of at most 3072 bits or `EC_SIGN_P256_SHA256`: Cloud KMS keys sign SHA-256 digests only,
and APK signature schemes use SHA-512 with RSA keys larger than 3072 bits.

Authentication is Application Default Credentials: Workload Identity Federation in CI
(`google-github-actions/auth`, no stored keys), and locally `gcloud auth
application-default login --impersonate-service-account=android-play-release@clementine-data.iam.gserviceaccount.com`.

## Getting it

Each [release](https://github.com/clementine-player/kms-signer/releases) has the distribution zip
and its SHA-256. Workflows should pin a version and check the checksum before running it, as it
signs releases:

```sh
version=2026.09.25-abc1234  # a version from the releases page
gh release download "v$version" --repo clementine-player/kms-signer \
  --pattern "kms-signer-$version.zip*"
echo "<sha256 from the release>  kms-signer-$version.zip" | sha256sum --check
unzip -q "kms-signer-$version.zip"
signer=kms-signer-$version/bin/kms-signer
```

It needs Java 21. To build it yourself: `./gradlew installDist`, then
`signer=build/install/kms-signer/bin/kms-signer`.

## Usage

**Once per key**, make the signing certificate and commit it. Every release must be signed with
the same certificate, so it is never regenerated:

```sh
$signer gencert \
  --key projects/clementine-data/locations/global/keyRings/android-signing/cryptoKeys/play-upload/cryptoKeyVersions/1 \
  --subject "CN=Clementine Remote Upload,O=Clementine" \
  --out upload_cert.pem
```

**Each release**, sign the unsigned bundle and check the result:

```sh
$signer sign --key projects/.../cryptoKeyVersions/1 --cert upload_cert.pem \
  --in app-release-unsigned.aab --out app-release.aab --min-sdk 23
$signer verify --in app-release.aab
```

## Tests

`./gradlew test` signs bundles through apksig's KMS extension point with a
local key standing in for Cloud KMS, and checks them with the JDK's JAR verification and
`jarsigner`.

## Releasing

Releases are automatic, like Clementine's: every push to `main` that changes the signer (its
code or build) runs the tests and publishes version `YYYY.MM.DD-<commit>` as a GitHub release
tagged `v<version>`, with `kms-signer-<version>.zip` and its SHA-256. Changes to the docs alone
don't release. The release workflow can also be run by hand.

Projects pin a version and its checksum, so a new release changes nothing until they update
both (Clementine Remote's `.github/workflows/play.yml`).
