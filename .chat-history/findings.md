# Findings not acted on

Things observed while fixing the CI pipeline that **no observed failure implicated**, so
they were logged rather than changed. Recorded against branch `feature/ci-fixes`, PR #1.

---

## 1. CI runs Node 14 while the project declares Node 15

`.github/workflows/ci.yaml` pins `node-version: '14'` in all four jobs. The project
declares the opposite:

- `codebase/rdicidr-0.1.0/package.json:43-47` — `"node": ">=15.0.0 <16.0.0"`, `"npm": ">=7.0.0 <8.0.0"`
- `codebase/rdicidr-0.1.0/.npmrc:1` — `engine-strict=true`
- `codebase/rdicidr-0.1.0/.nvmrc:1` — `15.5.1`

I predicted this would fail the `install` job. **It did not.** Run 33567936009 installed,
linted and tested cleanly on `node v14.21.3` / `npm 6.14.18`.

Why the prediction was wrong: `engine-strict` in npm 6 is only applied to *dependencies'*
engine ranges, not to the root package's own. Verified locally — `npm config get
engine-strict` returns `true` under npm 6.14.18, and `npm ci` still proceeds. Root-package
engine enforcement arrived in npm 7. So the declared contract is unenforced on Node 14 by
accident of tooling, not by design.

**Why left alone:** no failure implicates it. **Why it still matters:** CI is validating on
a runtime the project declares unsupported, and the guard meant to catch that is inert. The
moment anything bumps npm to 7+ while Node stays at 14, `install` breaks. Changing
`node-version` to `15.5.1` would align CI with `.nvmrc` and activate `engine-strict`
properly. Note the ceiling is real, not stale: `node-sass@5.0.0` (`package.json:9`) tops
out at Node 15, which is why `engines` caps below 16.

## 2. `npm ci` kept instead of the brief's `npm install`

The brief lists step 1 as `npm install`; the workflow runs `npm ci`. Once the lockfile was
regenerated (commit 81dc80a) `npm ci` succeeds, so no failure implicates the command.

Kept `npm ci` deliberately: it installs exactly the committed tree and fails loudly on
lock drift, which is the behaviour that surfaced defect #1 in the first place. Swapping to
`npm install` would have silently repaired the drift on every run and hidden it.

## 3. `npm test -- --watchAll=false` kept instead of `CI=true npm run test`

The brief specifies `CI=true npm run test`. The workflow's form already disables watch
mode, and all 11 tests pass under it (run 33568131605). Functionally equivalent here;
`CI=true` would additionally treat warnings as errors in `react-scripts build`, which is a
behavioural change nothing observed calls for.

## 4. Evidence-bot pushes create CI runs stuck in `action_required`

Runs 33566398740 and 33568018015 both sit at `action_required`, 0s, never executed. Both
were triggered by `github-actions[bot]` pushing a `.ci-evidence/` commit onto the PR
branch; GitHub holds bot-authored PR runs for manual approval.

**Why left alone:** `.github/workflows/evidence.yaml` is explicitly off-limits, and these
runs are harmless — the real runs on the same commits succeed. Flagged because they look
like failures in the run list and are not.

## 5. Deprecated action versions

Every run annotates:

    Node.js 20 is deprecated. The following actions target Node.js 20 but are being
    forced to run on Node.js 24: actions/cache/save@v3, actions/checkout@v3,
    actions/setup-node@v3

Annotations only, not failures. Bumping to `@v4` is unrelated to any observed defect.

## 6. `setup-node` does not use its built-in npm cache

The workflow hand-rolls `actions/cache/save` + `restore` around `node_modules` instead of
`actions/setup-node`'s `cache: 'npm'` (which caches `~/.npm` and is keyed automatically).
The hand-rolled version is what produced defect #4 — a mistyped key that missed silently.
It works now. Replacing it is a design change, not a fix.

## 7. `npm run prettier` fails on 4 unformatted files

Not part of the CI pipeline (which runs install/lint/test/build only), so it never fails a
run. Under prettier 3.3.1:

    [warn] src/index.js
    [warn] src/IPv4Addr.js
    [warn] src/lib/ipv4.js
    [warn] src/SubnetNumbersInput.js

Related to commit fe00016: these four files are also why wiring `plugin:prettier/recommended`
back in would not have been a one-line fix — it would have turned each into a lint error and
required reformatting source that no failure implicates.

## 8. 211 npm audit advisories

10 low / 115 moderate / 67 high / 19 critical, inherent to the `react-scripts@4.0.3`
dependency era. `npm audit fix --force` would replace react-scripts wholesale. Out of scope.

## 9. Out of scope for this pass

- `k8s/deployment.yaml` — a `Deployment` (brief requires StatefulSet), `replicas: 2` (requires
  3+), no `namespace: production`, `containerPort: 3000` and probes on `:3000` against an nginx
  image that listens on `:80`, and `requests: cpu 4000m / memory 4Gi` that will not schedule on
  a laptop.
- `k8s/service.yaml` — no namespace, `port: 80` (brief requires 8080), and
  `selector: app: rdicidr-web` which matches no pod label (`app: rdicidr`).
- `codebase/rdicidr-0.1.0/Dockerfile` — no `ARG` for the commit SHA or workflow run id, and
  `nginx.conf` has no `/version` location, both of which the CD evidence requirement needs.
