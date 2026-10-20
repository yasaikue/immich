# Standalone Android APK for Immich

This fork provides a manually triggered GitHub Actions workflow that builds an
Android APK installable **alongside the official Immich app**.

The workflow does **not** modify the checked-in Immich application source.
It changes the app's ID and name in its temporary CI workspace:

| Setting | Official app | Standalone build |
| --- | --- | --- |
| Application ID | `app.alextran.immich` | `app.alextran.immich.fork` |
| Display name | Immich | Immich Fork |

## Build

1. In the GitHub repository, select **Actions**.
2. Open **Build standalone Immich Android APK** and click **Run workflow**.
3. Select **Source**:
   - **upstream** (default) checks out `immich-app/immich` at the selected
     upstream ref. With the default ref of `main`, every run fetches the
     current official main branch, regardless of the fork's sync status.
   - **fork** checks out the fork's `main` branch as it exists at run time.
     It does **not** sync upstream automatically.
4. Optionally set **upstream_ref** to an official release tag or exact commit
   for a reproducible build. This field is ignored for fork builds.
5. Open the completed workflow run, download the `immich-fork-apk-...` artifact,
   unzip it, and transfer `app-release.apk` to the Android device.
6. Open the APK on the device to install it. Android may ask you to allow
   installs from the app used to open the APK.

The run summary shows the source commit that was built. Building the latest
upstream `main` can yield an unreleased version: ensure that the client is
compatible with your Immich server, or pin an appropriate upstream tag.

## Signing and updates

When no GitHub Actions signing secrets are configured, Gradle signs the
release APK with the runner's **temporary debug key**. This is sufficient
for a one-off test but does not provide a stable upgrade path. A later
build may require uninstalling the previous standalone app, and app data
will be lost on uninstall. It does not affect the separately installed
official Immich app.

For reliable in-place APK updates, create a private signing keystore once
on a trusted local machine, back it up securely, and add these values under
**Settings > Secrets and variables > Actions > Repository secrets**:

- `KEY_JKS`: entire keystore file encoded as Base64 (single line).
- `ALIAS`: key alias.
- `ANDROID_KEY_PASSWORD`: password for the key.
- `ANDROID_STORE_PASSWORD`: password for the keystore.

For example, generate a new keystore locally:

```bash
keytool -genkeypair -v \
  -keystore immich-fork.jks \
  -alias immich-fork \
  -keyalg RSA -keysize 3072 \
  -validity 10000
```

On Linux/macOS, convert the file to a single-line secret value:

```bash
base64 < immich-fork.jks | tr -d '\n'
```

On Windows PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("immich-fork.jks"))
```

Do not commit the keystore, the Base64 value, or passwords. Keep a secure
backup of the original keystore and its passwords. Use **the same keystore**
for all subsequent builds. Set these secrets before installing the APK you
intend to keep long term, otherwise the initial debug-signed app will need
to be uninstalled once.

## Important limitations

- Changes to upstream source may break code generation or the Android build.
  If the upstream Gradle configuration changes, the APK identity patch
  intentionally fails rather than silently producing an APK that replaces
  the official app.
- Both installations have separate app data, accounts, and backup settings.
- Certain Android link handlers (including the `immich://` and
  `app.immich` OAuth callback schemes) are shared with the official app.
  Android may ask which app should open a link, and third-party OAuth
  callbacks can be ambiguous when both are installed.
- The workflow creates an APK for manual installation. It does not publish
  to Google Play or update the APK automatically.
