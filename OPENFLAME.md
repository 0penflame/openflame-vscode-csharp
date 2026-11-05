# C# extension for OpenFlame

This branch is the free C# extension (`muhammadsammy/free-vscode-csharp`,
published on Open VSX as `muhammad-sammy.csharp`) at release tag `2.160.4`
plus the small set of changes OpenFlame workspaces need. Its debugger is
netcoredbg, bundled in the platform package under `.debugger/netcoredbg`.
The OpenFlame workspace image (code-server) installs the `linux-x64` package
built from this branch instead of pulling the latest build from Open VSX.

Branch: `openflame-2.160.4` (from tag `2.160.4`).
Extension version: `2.160.4-openflame.1`; the extension id
(`muhammad-sammy.csharp`) is unchanged so settings and dependants keep working.

## How OpenFlame attaches

OpenFlame runs a C# service as `dotnet ./bin/Debug/<tfm>/Server.dll
OPENFLAME_SERVICE=<name> OPENFLAME_LANG=CSHARP OPENFLAME_PORT=<port>`. The
OpenFlame process picker extension (`openflame-vscode-process-picker`)
contributes `${command:PickOpenflameProcess}` and
`${command:RunOpenflameProcess}` for the `coreclr` debug type and returns that
process's pid. VS Code substitutes the variable before
`resolveDebugConfigurationWithSubstitutedVariables`, so the extension sees a
numeric `processId` and does not show its own picker:

```json
{ "name": "Attach", "type": "coreclr", "request": "attach", "processId": "${command:PickOpenflameProcess}" }
```

Without `pipeTransport` the extension starts its bundled netcoredbg
(`--interpreter=vscode`). netcoredbg attaches through the .NET runtime's
debugging interface, so no ptrace capability is involved.

## OpenFlame changes

1. **Version and display name** (`version.json`, `package.json`).
   `version.json` pins `2.160.4-openflame.1` and adds `^refs/heads/openflame-.*$`
   to `publicReleaseRefSpec`, so Nerdbank.GitVersioning stamps exactly that
   version (no `-g<commit>` suffix) when packaging from an `openflame-*`
   branch. The display name is "C# (OpenFlame)".
   Why: the build is distinguishable from the Open VSX release. Installed from
   a VSIX, code-server records the extension as pinned (`"pinned": true,
   "source": "vsix"` in `extensions.json`), so it is not auto-updated to the
   Open VSX build.

2. **OpenFlame attach snippets** (`package.json`, `coreclr`
   `configurationSnippets`). ".NET: Attach to OpenFlame Service" and
   ".NET: Run and Attach to OpenFlame Service" insert attach configurations
   using `${command:PickOpenflameProcess}` / `${command:RunOpenflameProcess}`
   and the bundled netcoredbg.
   Why: a developer can add a working OpenFlame attach configuration from
   "Add Configuration..." without knowing the picker's variable names.

## Checked against upstream and not carried

- **Process picking for `dotnet`-hosted services.** The OpenFlame picker
  itself handles this (its 2022-05-18 "Fix process picker for dotnet" change
  lists `dotnet Server.dll ...` processes); nothing in this extension filters
  the pid it is given.
- **7-digit PIDs / command variables treated as names.** This extension does
  not parse `ps` output for an attach with a `processId`, and only shows its
  own picker when `processId` and `processName` are both empty
  (`src/shared/configurationProvider.ts`).
- **Picker variables.** VS Code merges `debuggers[].variables` contributed by
  several extensions for the same type, so the process picker's own
  `coreclr` contribution is enough.

## Build

Packaging needs Node.js and the .NET SDK (the language server, Razor and
netcoredbg packages are downloaded during packaging):

```bash
npm ci
npm run compile
npm run vscode:prepublish
npx ts-node -e "require('./tasks/packaging/offlinePackagingTasks').vsixReleasePackageLinuxTask({prerelease: false, outputFolder: require('path').resolve('vsix'), codeExtensionPath: process.cwd()})"
# -> vsix/csharp-linux-x64-2.160.4-openflame.1.vsix (plus the other Linux targets)
```

`npm run vsix:release:package` builds every platform the same way.

## Releasing a build

The OpenFlame workspace image does not keep a copy of the `.vsix`: it
downloads the asset of a GitHub release of this repository and checks it
against a pinned sha256 (`tools/vsix-releases.txt` and
`tools/fetch-vsix.sh` in `0penflame/openflame-workspace`).
This repository is public, so the workspace build downloads the asset
from the release download URL without a token.

1. Merge the `openflame-<version>` branch into `master` (pull request,
   squash). Bump the `-openflame.<n>` suffix for an OpenFlame-only
   rebuild of the same upstream release.
2. Build the `.vsix` from exactly the head of the merged
   `openflame-<version>` branch, in a clean checkout, with the build
   steps above. Releases are tagged there rather than on `master`,
   because `master` also carries upstream commits made after the
   release (`v2.160.4-openflame.1` is on the `openflame-2.160.4` head).
3. Cut the release on that commit, with the `.vsix` as its only asset:

   ```sh
   V=<version>                       # e.g. 2.160.4-openflame.1
   SHA=$(git rev-parse HEAD)         # head of the merged openflame-<version> branch
   cp vsix/csharp-linux-x64-$V.vsix csharp-$V-linux-x64.vsix
   shasum -a 256 csharp-$V-linux-x64.vsix
   gh release create "v$V" csharp-$V-linux-x64.vsix --repo 0penflame/openflame-vscode-csharp \
     --target "$SHA" --title "C# $V" \
     --notes "Built from $SHA. sha256 of csharp-$V-linux-x64.vsix: <sha256>"
   ```

   Never replace the asset of an existing release; cut a new version.
4. In `0penflame/openflame-workspace`, update the `csharp.vsix` line of
   `tools/vsix-releases.txt` (tag, asset name, sha256) in a pull request.
   The image build fails if the downloaded asset does not match the pinned
   sha256. Running workspaces pick up the new build on their next start
   after the image is rolled out (the entrypoint resyncs bundled extensions
   whose `.vsix` changed).

## Rebasing onto the next upstream release

1. `git fetch upstream --tags`
2. `git checkout -b openflame-<version> <version>`
3. Cherry-pick the OpenFlame commit(s) from the previous `openflame-*` branch
   (they touch only `version.json`, `package.json` and this file) and set the
   version in `version.json` to `<version>-openflame.1`.
4. Re-check the "not carried" items above against the new release
   (`src/shared/configurationProvider.ts`, `src/coreclrDebug/activate.ts`).
5. Build the `linux-x64` package and release it (see "Releasing a build").
