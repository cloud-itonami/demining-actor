# Operator quickstart

Four things can be verified locally in this repository. Together they take about three
minutes, most of which is one JVM start. Everything else you might expect to run here
cannot be run here, for reasons given at the bottom — **that section is part of the
quickstart, not an appendix.**

Every command below was executed against commit `0f01626` on 2026-09-01. Every check was
also confirmed to *fail* when the thing it checks is broken — the exact breakages are
named — so a green result means something.

**Prerequisites**, with the versions this was verified on:

| Tool | Verified version | Needed for |
|---|---|---|
| `nbb` | 1.5.212 (Node v26.7.0) | steps 1 and 2 — no JVM |
| Clojure CLI | 1.12.5.1654 (OpenJDK 24.0.2) | steps 3 and 4 |

Steps 1 and 2 are the ones to run if you only run some. They cover the same ground as
step 3 without starting a JVM.

## 1. Run the contract tests without a JVM

`src/` and `test/` are pure `.cljc` — no interop, no reader conditionals outside one
`ExceptionInfo` reference — so the suite runs on `nbb` directly.

```sh
cat > /tmp/dm-run.cljs <<'EOF'
(ns dm-run
  (:require [clojure.test :as t]
            [demining.murakumo-test]))
(t/run-tests 'demining.murakumo-test)
EOF
kbb --backend sci --classpath "src:test" /tmp/dm-run.cljs
```

Expected — exit 0, about 2 seconds:

```
Testing demining.murakumo-test

Ran 9 tests containing 57 assertions.
0 failures, 0 errors.
```

**Confirmed to fail**: replacing the body of `missing-gates` with `[]` (so nothing is ever
reported missing) turns this into `11 failures, 0 errors`, naming
`cell-plan-blocks-when-gates-missing` first. The suite is not decorative — it is the only
thing standing between "a gate was not attested" and "the record was planned anyway".

## 2. Check that the manifest and the cljc scaffold still agree

`actor-manifest.jsonld` declares the pipelines; `cell-specs` in `src/demining/murakumo.cljk`
declares the planning cells. **Nothing generates one from the other**, so they can drift.
This is the check that catches it, and it is the check most worth running after editing
either file.

```sh
cat > /tmp/dm-agree.cljs <<'EOF'
(ns dm-agree
  (:require ["fs" :as fs]
            [demining.murakumo :as m]))
(let [manifest (js->clj (js/JSON.parse (fs.readFileSync "actor-manifest.jsonld" "utf8")))
      declared (set (map #(get % "id") (get manifest "pipelines")))
      scaffold (set (map :legacy-cell (vals m/cell-specs)))
      only-manifest (sort (remove scaffold declared))
      only-scaffold (sort (remove declared scaffold))
      did-ok? (= m/actor-did (get manifest "@id"))]
  (println "manifest pipelines:" (count declared) (pr-str (sort declared)))
  (println "cljc cell-specs   :" (count scaffold) (pr-str (sort scaffold)))
  (println "actor did         :" m/actor-did "vs" (get manifest "@id"))
  (when (seq only-manifest)
    (println "FAIL declared in manifest, no cljc cell:" (pr-str only-manifest)))
  (when (seq only-scaffold)
    (println "FAIL cljc cell with no manifest pipeline:" (pr-str only-scaffold)))
  (when-not did-ok? (println "FAIL actor-did disagrees with manifest @id"))
  (let [bad (+ (count only-manifest) (count only-scaffold) (if did-ok? 0 1))]
    (println (if (zero? bad) "ok   manifest and cljc scaffold agree"
                 (str "FAIL " bad " disagreement(s)")))
    (js/process.exit (if (zero? bad) 0 1))))
EOF
kbb --backend sci --classpath "src" /tmp/dm-agree.cljs
```

Expected — exit 0:

```
manifest pipelines: 3 ("eore-session" "land-release-to-public" "survey-to-social")
cljc cell-specs   : 3 ("eore-session" "land-release-to-public" "survey-to-social")
actor did         : did:web:demining.etzhayyim.com vs did:web:demining.etzhayyim.com
ok   manifest and cljc scaffold agree
```

**Confirmed to fail, in both directions:**

| Breakage | Result |
|---|---|
| Delete the `:eore-session` entry from `cell-specs` | `FAIL declared in manifest, no cljc cell: ("eore-session")`, exit 1 |
| Change `actor-did` to `did:web:WRONG.etzhayyim.com` | `FAIL actor-did disagrees with manifest @id`, exit 1 |

Note the asymmetry this check is built around: the tests in step 1 introspect `cell-specs`,
so **deleting a cell keeps them green**. Step 1 cannot see that kind of drift. Step 2 is
the only thing that can.

## 3. Run the same suite on the declared JVM alias

`deps.edn` declares `:test` with the cognitect test-runner. This is what CI would run.

```sh
kbb -M:test
```

Expected — exit 0, identical counts to step 1:

```
Running tests in #{"test"}

Testing demining.murakumo-test

Ran 9 tests containing 57 assertions.
0 failures, 0 errors.
```

Measured twice on a machine under load average 104: **1m03s** wall with a cold dependency
cache, **26s** warm — against 7–8s of CPU either way. Almost all of it is JVM start and
dependency resolution, which is why steps 1 and 2 exist: if you only want to know whether
the code is correct, step 1 answers the same question in about two seconds. Reach for this
one when you specifically need to know that the *declared* alias works.

## 4. Lint

```sh
kbb -M:lint
```

Expected — **exit 0 with one warning**, which is the current state of the tree and not a
failure:

```
src/demining/murakumo.cljk:75:14: warning: unused binding input
linting took 5229ms, errors: 0, warnings: 1
```

(The elapsed figure varies run to run — 5.2s and 5.7s across two runs here. The line that
matters is `errors: 0, warnings: 1`.)

The alias sets `--fail-level error`, so warnings do not fail it. Do not read the exit code
as "no findings" — read the printed count. The one standing warning is the unused `:as
input` destructuring binding in `records-for`; it is left in place because the map is
destructured for its keys and the binding documents the parameter, but it is a real finding
and not suppressed.

## What cannot be run here, and why

- **The actor itself.** This repository plans; it does not execute. `cell-plan` returns
  `:mst/put-record` effect *descriptions* as data. Nothing here opens a socket, holds a key,
  or writes to a PDS, so there is no "start the actor" command to give you. The runtime that
  consumes these plans is `k8s-langserver` (`runtime` in the manifest) and lives elsewhere.
- **The gates.** The seven `common-gates` are checked for *attestation presence* only —
  `gate-value` asks whether an attestation exists, not whether it is valid. Signature
  verification is the caller's job. A green step 1 says the blocking logic is correct; it
  does not say any real gate was satisfied.
- **The DID resolution.** `.well-known/did.json` is served from `etzhayyim.com`, not from
  this repository. Whether `did:web:etzhayyim.com:actor:demining` resolves is a DNS and
  hosting question, and a local checkout cannot answer it.
- **The design corpus checks.** IMAS crosswalks, legal instruments, and crawl seeds live in
  `cloud-itonami/demining`, which has its own `docs/operator-quickstart.md` for them.
