# Skill Sheet 1 Node.js 24 cutover

The frozen frontend lockfile installs Next.js 15.5.29, the November 2026
security backport for Next.js 15. Use the frozen lockfile when rehearsing the
release on EC2. [Next.js 15.5.29 release notes](https://github.com/vercel/next.js/releases/tag/v15.5.29)

Use the authenticated CloudShell in account `889002527739`. Confirm the
account with `aws sts get-caller-identity --query Account --output text`,
then connect to the Skill Sheet 1 instance.

```sh
aws ssm start-session --region ap-northeast-1 --target i-084234a0e46ff109f
```

## Record the current deployment

```sh
sudo systemctl show skill-sheet.service \
  -p FragmentPath -p DropInPaths -p User -p Group \
  -p WorkingDirectory -p EnvironmentFiles -p MainPID
pid=$(sudo systemctl show skill-sheet.service -p MainPID --value)
sudo readlink -f "/proc/$pid/exe"
workdir=$(sudo systemctl show skill-sheet.service -p WorkingDirectory --value)
sudo git -C "$workdir" rev-parse HEAD
sudo ss -ltnp
uname -m
```

Read the unit and its drop-ins privately to determine the startup command,
bind address, port, environment files, and any required startup flags. Record
the Git revision of the deployed tree and compare its startup command and app
routing to this release. Leave its runtime, working tree, environment files,
and build output in place. Confirm the existing reverse proxy passes the
`/skill-sheet` prefix through to Next.js for pages and static assets. Resolve
any routing or PDF companion configuration differences before cutover.

## Stage and test on EC2

After the local changes are reviewed and committed, create a release archive
from the approved commit. Run from the repository root.

```sh
git archive --format=tar.gz --output=skill-sheet-1-source.tar.gz HEAD .nvmrc frontend
```

Transfer that archive into a new release directory,
for example `/opt/skill-sheet/releases/<release-id>`. Keep the new release
readable by the existing service account and writable where the app needs
scratch storage. Extract it there with
`tar -xzf skill-sheet-1-source.tar.gz` and confirm the archive layout
contains `frontend/package.json`.

Run these commands in a fresh scratch directory. Choose the archive for the
EC2 architecture. Install to a Skill Sheet 1-specific directory with a new
path.

```sh
set -eu
case "$(uname -m)" in
  x86_64) node_arch=x64 ;;
  aarch64) node_arch=arm64 ;;
  *) echo 'Unsupported architecture' >&2; exit 1 ;;
esac
version=v24.22.3
archive="node-$version-linux-$node_arch.tar.xz"
distribution="https://nodejs.org/dist/$version"
curl -fSLO "$distribution/$archive"
curl -fSLO "$distribution/SHASUMS256.txt"
awk -v archive="$archive" '$2 == archive { print; found++ } END { if (found != 1) exit 1 }' \
  SHASUMS256.txt > skill-sheet-node.sha256
sha256sum --check skill-sheet-node.sha256
runtime="/opt/skill-sheet/runtimes/node-$version-linux-$node_arch"
sudo mkdir -p /opt/skill-sheet/runtimes
sudo mkdir "$runtime"
sudo tar -xJf "$archive" --strip-components=1 -C "$runtime"
"$runtime/bin/node" --version
```

As the release build user, load a Skill Sheet-specific build configuration
with the correct public settings for the target environment. Use isolated
backing services and dummy data for the rehearsal. Install dependencies and
build on Linux using the new runtime. Run in the new release's `frontend/`
directory.

```sh
export PATH="/opt/skill-sheet/runtimes/node-v24.22.3-linux-<arch>/bin:$PATH"
export NEXT_PUBLIC_BASE_PATH=/skill-sheet
corepack pnpm install --frozen-lockfile
corepack pnpm lint
corepack pnpm build
corepack pnpm start --hostname 127.0.0.1 --port 38425
```

Use a free loopback port and the same browser and native dependencies needed
by the service. Confirm a headless browser renders the employee list under
`/skill-sheet/`, loads its CSS and JavaScript, opens a dummy profile, saves
an edit, and renders the PDF. Check production build-time configuration
separately if dummy public settings differ, because Next.js embeds those
settings in the build. Keep each build in its own release directory.
The current public frontend bundle points to
`https://dalla-production-1b00.up.railway.app/api/resume` for PDF generation.
Confirm the expected endpoint and set `NEXT_PUBLIC_RESUME_API_URL` at build
time for the production release. Use the local companion for dummy-data
testing.

```sh
export NEXT_PUBLIC_BASE_PATH=/skill-sheet
export NEXT_PUBLIC_RESUME_API_URL=https://dalla-production-1b00.up.railway.app/api/resume
corepack pnpm build
```

## Cut over after the notification window is approved

A service restart can interrupt requests. Confirm Chiiho-san has been
notified before taking this step. Pre-stage a new drop-in named
`90-skill-sheet-node24.conf` outside the service directory. Substitute the
staged runtime, release path, and verified production port. Carry forward the
running service's required flags and environment. This example assumes a
direct Next.js start. If the current ExecStart uses a wrapper, reproduce
its behavior in the staged release with the isolated runtime before using
the drop-in.

```ini
[Service]
WorkingDirectory=/opt/skill-sheet/releases/<release-id>/frontend
Environment=NEXT_PUBLIC_BASE_PATH=/skill-sheet
ExecStart=
ExecStart=/opt/skill-sheet/runtimes/node-v24.22.3-linux-<arch>/bin/node /opt/skill-sheet/releases/<release-id>/frontend/node_modules/next/dist/bin/next start --hostname <verified-bind-address> --port <verified-port>
```

From the directory containing the prepared drop-in, confirm the target
filename is free, install it, and restart only Skill Sheet 1.

```sh
set -eu
dropin=/etc/systemd/system/skill-sheet.service.d/90-skill-sheet-node24.conf
sudo test ! -e "$dropin"
sudo install -d -m 0755 /etc/systemd/system/skill-sheet.service.d
sudo install -m 0644 ./90-skill-sheet-node24.conf "$dropin"
sudo systemctl daemon-reload
sudo systemctl restart skill-sheet.service
sudo systemctl is-active skill-sheet.service
pid=$(sudo systemctl show skill-sheet.service -p MainPID --value)
runtime_exe=$(sudo readlink -f "/proc/$pid/exe")
"$runtime_exe" --version
```

Verify the executable points to the staged runtime and reports `v24.22.3`.
Use a headless browser against `https://dx.morabu.com/skill-sheet/` to verify
the rendered interface, assets, access controls, and primary workflows with
an approved test record. Roll back immediately if the service fails to start,
the frontend is blank, access checks fail, or a critical workflow breaks.

## Roll back

```sh
sudo rm -- /etc/systemd/system/skill-sheet.service.d/90-skill-sheet-node24.conf
sudo systemctl daemon-reload
sudo systemctl restart skill-sheet.service
sudo systemctl is-active skill-sheet.service
pid=$(sudo systemctl show skill-sheet.service -p MainPID --value)
runtime_exe=$(sudo readlink -f "/proc/$pid/exe")
"$runtime_exe" --version
```

Confirm the original runtime, working directory, and frontend assets are back
and repeat the browser smoke test.
