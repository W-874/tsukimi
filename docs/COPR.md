# Fedora COPR packaging

## Build requirements and source layout

Use the SCM source type with the **make srpm** method, not a plain spec upload
or the default `rpkg` method. `.copr/Makefile` archives the selected Git commit,
runs `cargo vendor --locked`, and builds an SRPM containing both source and
vendor archives. The generated `.cargo/config.toml` uses a relative `vendor`
path and enforces offline Cargo operation during the binary build.

The current application version is **26.11.1**. Its version lives in
`crates/tsukimi/Cargo.toml`; the repository root is a Cargo workspace.

| Dependency | Minimum |
| --- | --- |
| Rust | 1.92 |
| GTK4 | 4.22 |
| libadwaita | 1.9 |
| GLib / GIO | 2.76 |
| libmpv | 0.38 |
| glycin native library / loaders | 2.0 |

Both specs also declare the native dependencies needed by the Rust crates,
including fontconfig, lcms2, libseccomp, Wayland and X11. `blueprint-compiler`
builds the player's UI. `bubblewrap` and `glycin-loaders` are runtime
requirements: linking the application alone does not install image decoders.

Fedora 44 and 45 x86_64 repository queries currently satisfy the GTK4,
libadwaita and Rust constraints. Check availability when enabling another
release or architecture; a successful repository query is not a binary build.
No additional repository was needed for the queried Fedora 44/45 versions.

## 1. Update and publish the packaging branch

```sh
git switch copr
git fetch --prune upstream
git merge --no-edit upstream/main
```

Resolve any conflicts before building. Review and commit packaging changes,
then publish them to your fork:

```sh
git add .copr/Makefile tsukimi.spec tsukimi-git.spec docs/COPR.md docs/build_on_linux.md
git commit -m "Update COPR packaging for current upstream"
git push origin copr
```

The remote is `https://github.com/W-874/tsukimi.git`. Use this public HTTPS
clone URL in COPR, not the local SSH remote. COPR cannot see unpushed changes.
The source archive uses `git archive HEAD`, so uncommitted application changes
are not included in local SRPMs either.

## 2. Configure COPR authentication

1. Create a [Fedora account](https://accounts.fedoraproject.org/) and sign in to
   [COPR](https://copr.fedorainfracloud.org/).
2. Install the client and local SRPM prerequisites:

   ```sh
   sudo dnf install copr-cli cargo rust git-core make python3 rpm-build zstd
   ```

3. Copy the configuration shown at <https://copr.fedorainfracloud.org/api/>
   into `~/.config/copr`. Create the directory if needed and protect the file:

   ```sh
   mkdir -p ~/.config
   chmod 600 ~/.config/copr
   copr-cli whoami
   ```

Never commit the API credentials to Git.

## 3. Create or configure the project

The README uses `walker874/tsukimi`. For a new project owned by your authenticated
account:

```sh
copr-cli create tsukimi \
  --chroot fedora-44-x86_64 \
  --chroot fedora-45-x86_64
```

If that project already exists, use its **Settings** page instead. Enable the
desired chroots. Binary-build network access can remain disabled: downloading
crates belongs to the separate SRPM-generation stage, where COPR permits
network access. Do not use network access to work around incomplete vendoring.

## 4. Submit the latest merged development snapshot

Recommended for tracking upstream `main`:

```sh
copr-cli buildscm walker874/tsukimi \
  --clone-url https://github.com/W-874/tsukimi.git \
  --commit copr \
  --type git \
  --method make_srpm \
  --spec tsukimi-git.spec
```

Equivalent web settings under **New Build → SCM**:

| Field | Value |
| --- | --- |
| Type | git |
| Clone URL | `https://github.com/W-874/tsukimi.git` |
| Committish | `copr` |
| Subdirectory | empty (repository root) |
| Spec File | `tsukimi-git.spec` |
| SRPM build method | make srpm |

The package is named `tsukimi-git`. Its version comes from
`crates/tsukimi/Cargo.toml`, and its release is
`0.<YYYYMMDD>git<10-character-checkout-sha>`. The SHA identifies the selected
fork commit, which may be a merge or packaging commit, not the upstream SHA.

The existing date-only snapshot scheme does not guarantee RPM upgrade ordering
between multiple snapshots on the same day; compare NEVRAs before publishing
such rebuilds. This guide does not change that versioning policy.

For a package named `tsukimi`, use the same command with `--spec tsukimi.spec`.
This does **not** make a `copr` branch checkout a tagged stable release: both
specs package the selected checkout.

### Tagged release builds

For a real release package, use a packaging branch based on the upstream
release tag, carrying the corrected packaging files. For example, starting
from the committed `copr` branch:

```sh
git switch -c copr-release-26.11.1 v26.11.1
git restore --source=copr -- .copr/Makefile tsukimi.spec tsukimi-git.spec
git add .copr/Makefile tsukimi.spec tsukimi-git.spec
git commit -m "Package Tsukimi 26.11.1 for COPR"
git push -u origin copr-release-26.11.1
git switch copr

copr-cli buildscm walker874/tsukimi \
  --clone-url https://github.com/W-874/tsukimi.git \
  --commit copr-release-26.11.1 \
  --type git --method make_srpm --spec tsukimi.spec
```

Use a name such as `copr-release-26.11.1`, not `copr/release-26.11.1`:
Git cannot have both the existing `copr` branch and branches below `copr/`.

Do not select the unmodified upstream tag and assume newer packaging fixes are
present there. When HEAD is exactly a tag, the generator removes its leading
`v` and optional `-copr` suffix for the RPM version; otherwise it uses the
application manifest version. The release packaging commit above uses the
latter path.

## 5. Inspect builds and install

The build command reports a build ID and URL. Check the COPR build page or:

```sh
copr-cli status BUILD_ID
copr-cli list-builds walker874/tsukimi
```

Inspect `builder-live.log` for SRPM generation, and each chroot's `root.log`
and `build.log` for dependency installation and compilation respectively.
Only advertise a chroot after its binary build succeeds.

After a successful build:

```sh
sudo dnf copr enable walker874/tsukimi
sudo dnf install tsukimi-git
```

For the release package, install `tsukimi` instead. They install the same
binary and resources and cannot coexist; to switch from an installed release
package, use `sudo dnf swap tsukimi tsukimi-git` (reverse the names to switch
back). Launch `tsukimi` from a graphical session and check startup, artwork,
and playback; a successful SRPM is not a GUI or playback test.

## 6. Generate and rebuild an SRPM locally (optional)

With the prerequisites from step 2 installed, no root access is needed for
source packaging. `DNF=true` skips only the Makefile's package-install command:

```sh
outdir="$(mktemp -d)"
make -f .copr/Makefile srpm DNF=true \
  spec=tsukimi-git.spec outdir="$outdir"
```

Use `spec=tsukimi.spec` for the release package name. To test the binary build
in a clean Fedora environment:

```sh
sudo dnf install mock
sudo mock -r fedora-45-x86_64 --rebuild "$outdir"/*.src.rpm
```

Alternatively upload that SRPM with
`copr-cli build walker874/tsukimi "$outdir"/*.src.rpm` or the COPR upload form.
Do not upload just the spec: its two `Source` entries are generated locally,
not download URLs.

## 7. Keep tracking upstream

Repeat step 1 whenever upstream changes, then rebuild the SCM package in COPR.
You can subsequently use:

```sh
copr-cli build-package walker874/tsukimi --name tsukimi-git
```

A webhook can rebuild the fork's `copr` branch after pushes, but it does not
fetch or merge upstream automatically. Configure it only after the manual
build works.

## Troubleshooting and verification scope

- `cargo vendor` fails: fix the reported lockfile, source or network issue in
  SRPM generation. The Makefile now stops immediately instead of publishing an
  incomplete SRPM.
- GTK4, libadwaita or Rust constraints cannot be satisfied: choose a supported
  newer chroot or explicitly package the missing dependency; do not lower the
  spec version below the application's feature requirements.
- Offline Cargo cannot resolve a crate: check that `Source1` contains both
  `.cargo/config.toml` and `vendor/`, and that `%autosetup -a 1` extracted them
  at the source root.
- No artwork: check `glycin-loaders` and `bubblewrap`, including the loader
  configuration under `/usr/share/glycin-loaders/2+/conf.d/`.

The packaging update was smoke-tested by generating both SRPMs, running both
RPM `%prep` stages, and resolving the complete locked Cargo dependency graph
with `--offline` and an empty `CARGO_HOME`. Injected vendor failure now makes
`make` fail without producing an SRPM. Full binary builds and GUI playback
were not verified in that environment.

References: [COPR SCM and make srpm](https://docs.copr.fedorainfracloud.org/user_documentation.html#scm),
[COPR CLI reference](https://www.mankier.com/1/copr-cli).
