# Contributing

Thanks for helping. A few rules keep this tool trustworthy; please follow them.

## Setup

Use an EditorConfig-aware editor to apply the repository's `.editorconfig` formatting defaults.

Node 22.12+, Corepack and Docker. Then:

```sh
corepack enable pnpm
pnpm install
pnpm check            # lint + typecheck + tests (offline, hermetic)
pnpm build
```

Opt-in tests that talk to the real world: `RELEASEPROOF_NET_TESTS=1` (registry, Sigstore)
and `RELEASEPROOF_DOCKER_TESTS=1` (real rebuilds). CI runs them weekly.

## Principles that every change must keep

- **Deterministic.** No LLMs, no heuristics that depend on time or environment. Same input,
  same verdict.
- **Fail closed.** Anything that cannot be verified is "could not verify", never "safe".
- **Untrusted by default.** Tarballs, manifests, attestations and repository contents are
  data. Never execute them on the host; validate before they touch a shell, a URL or a path.
- **No telemetry.** Nothing about the user's package leaves their machine.

## Adding a rule

Rules live in `packages/core/src/rules/` and are pure functions of the release tarball, the
rebuild, the diff and the provenance result. Every rule needs:

1. a stable kebab-case id and a severity with a one-line justification in
   `docs/rules.md`;
2. an "attacked" fixture in `fixtures/` that must trigger it and the `clean` fixture that
   must not;
3. a finding whose `detail` explains, in plain English, what was seen and why it matters.

## Changing the rebuild

`docs/design-rebuild.md` records the decisions and why. Changes to the stage scripts, the
Docker flags or the planner need a unit test and, when they affect real packages, a row in
`docs/real-world-results.md`.

## Finding something to work on

- Issues labelled [good first issue](https://github.com/ujjwalv01/releaseproof/labels/good%20first%20issue)
  are small, self-contained and name the files to touch and the tests to add.
- [help wanted](https://github.com/ujjwalv01/releaseproof/labels/help%20wanted) issues are
  bigger and need Docker and some reading of `docs/design-rebuild.md`.
- Claim an issue before starting so two people do not do the same work: comment `/assign`
  (or just "I'd like to work on this") on an open, unassigned issue and it is assigned to you
  automatically. You can hold up to three open issues at a time; comment `/unassign` to give
  one back. If you want to change something that has no issue yet, open one first and
  describe the change; for anything security-relevant, read SECURITY.md.

## Pull requests, step by step

1. **Fork and clone.** Click "Fork" on GitHub, then:

   ```sh
   git clone https://github.com/<your-user>/releaseproof.git
   cd releaseproof
   git remote add upstream https://github.com/ujjwalv01/releaseproof.git
   ```

2. **Set up.** Node 22.12+ with Corepack, and Docker if you will touch the rebuild:

   ```sh
   corepack enable pnpm
   pnpm install          # uses the pinned pnpm from package.json; install scripts stay blocked
   pnpm check            # lint + typecheck + tests, must be green before you start
   ```

3. **Branch from an up-to-date main.** Never work on `main` itself:

   ```sh
   git fetch upstream
   git switch -c fix/short-description upstream/main
   ```

   Use a prefix that says what the branch does: `feat/`, `fix/`, `docs/`, `test/`.

4. **Make the change, with tests.** Every behaviour change needs a test next to the code
   (`*.test.ts`). Security logic needs both an "attacked" case that must be caught and a
   "clean" case that must pass (see `fixtures/README.md`). Keep the principles above.

5. **Run everything locally.**

   ```sh
   pnpm check            # lint + typecheck + tests
   pnpm build            # CLI and Action bundles
   ```

   `pnpm lint` reports formatting too; `pnpm exec biome check --write .` fixes it. If you
   changed anything under `action/` or `packages/core/`, `pnpm build` updates
   `action/dist`; commit it, CI fails when the bundle is stale.

6. **Add a changeset** when users will notice the change (CLI, Action, verdicts, rules):

   ```sh
   pnpm changeset
   ```

   Pick `patch` for fixes, `minor` for new options or rules, and write one or two lines
   as they should appear in the changelog. Pure docs, tests and CI changes need none.

7. **Commit** in small steps with conventional messages, for example
   `feat(core): resolve dist-tags before checking`, `fix(action): reject a relative
   report path`, `docs: describe the JSON report`. The first line stays under 72
   characters; the body says why, not what.

8. **Push and open the PR:**

   ```sh
   git push -u origin fix/short-description
   ```

   GitHub prints a link to open the pull request against `ujjwalv01/releaseproof:main`.
   In the description: what changed, why, how you tested it, and `Closes #<issue>` when
   it resolves an issue. The template checklist is short; tick what applies.

9. **CI and review.** CI runs lint, typecheck, tests on Node 22 and 24, the packed
   manifest check and the Action self-test (a real rebuild of `sigstore@5.0.0` in
   Docker). First-time contributors need a maintainer to approve the workflow run; that
   is normal. Address review comments with new commits; the PR is squash-merged, so no
   need to rewrite history. If `main` moved, `git fetch upstream && git rebase
   upstream/main` and push again.

Rules that apply to every PR:

- Third-party GitHub Actions in workflows are pinned to full commit SHAs, never tags.
- Dependabot keeps these action pins up to date in a weekly grouped PR, reviewed like any other PR.
- No new runtime dependency without an issue discussing it first; the CLI is bundled and
  every dependency ships to every user.
- Nothing in the repository may execute package contents on the host, phone home, or
  turn a "could not verify" into a "matches".

## Reporting a bug

Use the [bug report form](https://github.com/ujjwalv01/releaseproof/issues/new?template=bug_report.yml).
It asks for what happened, the command or workflow step, the verdict and exit code, the
`--json` report (remove tokens first), the package and version, and your releaseproof, OS,
Node and Docker versions. Ideas go in the
[feature request form](https://github.com/ujjwalv01/releaseproof/issues/new?template=feature_request.yml).

If you think you found a way to make releaseproof say MATCHES SOURCE for a tampered
package, escape the build container, or leak a token, do not open a public issue:
[report it privately](https://github.com/ujjwalv01/releaseproof/security/advisories/new)
as described in [SECURITY.md](./SECURITY.md).
