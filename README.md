# releaseproof

**Verify every npm release matches its source before you approve it.**

> Early release (0.1.x). The verdicts are deterministic and fail closed, but expect rough
> edges; please [report](https://github.com/ujjwalv01/releaseproof/issues) packages it
> cannot rebuild.

## The problem

In 2026, attackers started publishing malware through package maintainers' own
release pipelines. In the TanStack incident (May 2026), a stolen publish token
was used to release 84 malicious versions across 42 packages, and those
versions carried valid provenance. The maintainers learned about it from an
outside security company.

npm's answer is **staged publishing**: a release waits until a maintainer
approves it with 2FA. From January 2027, tokens that bypass 2FA can only stage
releases, not publish them. That puts every serious maintainer in front of an
**Approve** button, but npm shows no diff and no scan there. The maintainer has
to decide, on their own, whether the tarball is really theirs.

## What releaseproof does

1. **Watches** for a new release (staged or published) of your package.
2. **Rebuilds** the package from the tagged git commit in a clean container.
3. **Compares** the release with the rebuild, file by file.
4. **Checks** the things attackers add: new install scripts, new dependencies,
   unexpected files (such as `.vscode/` or `.claude/`), binaries, and
   provenance pointing to the wrong repository or workflow.
5. **Reports** a clear verdict before you click Approve:
   - ✅ Matches your source
   - 🚨 Differs from your source, with exactly what changed

No LLM, no API key, no server. Same input, same answer, every time.

## Usage

Needs Node 22.12+ and Docker. No install step:

```sh
npx releaseproof check sigstore@5.0.0
```

From a checkout, after `pnpm install && pnpm build`, the same command is
`node packages/cli/dist/index.js check sigstore@5.0.0`.

What happens: the tarball and its attestations are fetched, the provenance is verified
against the Sigstore trust root, the package is rebuilt from the certified commit inside a
hardened Docker container (clone and install with network, build without), the two
tarballs are compared file by file, and the rules decide.

```text
Package      sigstore@5.0.0 (registry.npmjs.org), 9 files, sha512 849a897e81bf…
Provenance   VERIFIED  sigstore/sigstore-js@7d2900e on refs/heads/main
             workflow .github/workflows/release.yml, trigger push, github-hosted runner
Rebuild      OK (2m 14s)  node:22-bookworm, npm 10.9.9, build: root "build" script
Files        9 identical, 0 changed, 0 only in release, 0 only in rebuild

Findings     none

Verdict: MATCHES SOURCE: 9 files identical to a clean build of sigstore/sigstore-js@7d2900e
```

Exit codes: `0` matches source, `1` differs from source, `2` could not verify (or a tool
error). "Could not verify" is never "safe".

Staged, not yet approved versions: `node packages/cli/dist/index.js check --stage <id>
--repo <url> --commit <sha>` (the stage id from `npm stage list`; needs your npm token in
`NPM_TOKEN` or `.npmrc`). The registry hides a staged version's provenance until approval,
so releaseproof looks it up in the Rekor transparency log by the tarball's digest; when a
verified entry from `--repo` is there, the source is certified and `--commit` is only the
fallback (`--no-rekor` skips the lookup).

Options: `--repo <url>` (the repository you expect; with `--commit <sha>` it is the source
to rebuild from when there is no provenance), `--node <major>` / `--image <image>` /
`--build-command <cmd>` to override the build plan, `--memory <size>` for the container
memory limit (default `4g`; big TypeScript builds may need more), `--json` for the full report
(fields are described in [docs/json-report.md](./docs/json-report.md)),
`--offline` to skip refreshing the Sigstore trust root, `--keep-volume` to inspect the
build afterwards. Every rule and severity is documented in [docs/rules.md](./docs/rules.md).

Requirements: Node 22.12+, Docker. No account, no API key, no telemetry.

Questions after a first run, such as why a package reports COULD NOT VERIFY or how to check a
staged version, are answered in the [FAQ](./docs/faq.md).

## GitHub Action

Run the same check after every release and read the verdict in the job summary:

```yaml
on:
  release:
    types: [published]

permissions: {}

jobs:
  verify:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: ujjwalv01/releaseproof@v0.1.0 # or pin the commit SHA of the tag
```

With no inputs, the package and version come from the checkout's `package.json`, the
release must have been built from this repository, and the job fails unless the verdict is
MATCHES SOURCE. Add `create-issue: "true"` (plus `issues: write`) to open an issue when it
is not. The action refuses to run on `pull_request_target`. All inputs are documented in
[action.yml](./action.yml); the security reasoning in
[docs/design-action.md](./docs/design-action.md).

## Packages

| Package | What it is |
| --- | --- |
| [`releaseproof`](https://www.npmjs.com/package/releaseproof) (CLI) | `npx releaseproof check <package>@<version>` |
| [releaseproof check](https://github.com/marketplace/actions/releaseproof-check) (Action) | Runs on every release and posts the verdict as a job summary or issue |

## Status

- Fetching a published or staged version and verifying its provenance against the Sigstore
  trust root, with the certificate (not the build's own claim) as the source of truth.
- Rebuilding a package from the certified commit in a hardened Docker container and
  comparing it with the release file by file.
- 42 rules with graded severity ([docs/rules.md](./docs/rules.md)) and a verdict that fails
  closed: a failed rebuild is "could not verify", never "safe".
- Tested against real releases: 11 popular packages rebuild bit-for-bit, 0 false alarms
  ([results](./docs/real-world-results.md)).

Not yet rebuilt for real: Yarn-based repositories and packages that need a package manager
other than npm, pnpm or Yarn (those report "could not verify").

## Development

Requires Node 22.12+ and Corepack (bundled with Node 22 and 24).

```sh
corepack enable pnpm   # uses the pnpm version pinned in package.json
pnpm install
pnpm check             # lint + typecheck + test
pnpm build
node packages/cli/dist/index.js --version
```

Tests run offline against captured registry responses. To also run the few
tests that talk to the real npm registry and Sigstore:

```sh
RELEASEPROOF_NET_TESTS=1 pnpm test
```

On Windows without admin rights, use
`corepack enable pnpm --install-directory "%APPDATA%\npm"`.

| Path | What it is |
| --- | --- |
| `packages/core` | Verification logic (private, bundled into the CLI) |
| `packages/cli` | The `releaseproof` command |

## License

[MIT](./LICENSE) © 2026 Ujjwal Verma
