# Concept: getting all of tubesync "ruff-clean"

*As of 2026-09-08 · Ruff 0.16.6 · Scope: `tubesync/` (155 Python files)*

## Context

The current CI check (`.github/workflows/ci.yaml`, step *"Check with ruff"*) only enforces a
very narrow rule set:

```
ruff check --target-version py312 --output-format github \
  --select 'C4,E4,E7,E9,F' \
  --ignore 'C408,C409,C410,E701,E722,E731,I001,UP017,UP018'
```

That narrow check is **already green today**. `tubesync/ruff.toml` deliberately has no
`[lint] select`, so local runs only use the ruff defaults (`E4/E7/E9/F`).

Goal of this document: an assessment plus a complete guide for getting the **entire `tubesync/`
directory** to pass a broader, sensible ruff rule set **with no complaints** — as the basis for
an incremental rework.

### Up-front questions

**"Is there a 'GitHub standard' rule set?"**
No. There is no official "GitHub ruff rule set". GitHub's `super-linter` simply runs ruff with
the repo config or the ruff defaults (`E4,E7,E9,F`). The widely accepted de-facto community
baseline is: **defaults + `I,UP,B,C4,SIM,RUF`**, often extended with
`RET,PERF,PIE,G,ICN,FURB,LOG`. That is exactly what is spelled out below as the
**"Recommended tier"**.

**"What quote style is the project's current standard (core developer)?"**
Clearly **single quotes (`'`)**. Measured across all of `tubesync/`: ~18,700 `'` vs. ~1,900 `"`.
In the files most recently touched by `tcely`/`meeb` (`sync/tasks.py`, `sync/models/media.py`,
`common/utils.py`, `sync/choices.py`) the single-quote share is ~90–99%. The CI YAML itself also
consistently uses single quotes and Yoda comparisons (`'pull_request' != github.event_name`).
→ set `quote-style = "single"`, do not enable the `Q` rules, ignore `SIM300` (Yoda).

**Scope:** `tubesync/` only (same as today's CI scope). `patches/` stays out of scope.

---

## Current state (measured with ruff 0.16.6, `--target-version py312`)

| Target | Violations | Assessment |
|---|---|---|
| Today's CI check (`C4,E4,E7,E9,F` minus ignores) | **0** | already green |
| Ruff defaults (`E,F`) across all of `tubesync/` | 19 | trivial |
| **Recommended tier** (see below), project ignores applied | ~900 raw · ~415 auto-fixable · rest mostly "ignore" | goal of this rework |
| `--select ALL` | ~14,830 | unrealistic (docstrings/annotations/copyright) – **not** the goal |
| `ruff format` (single-quote config) | 131 of 156 files would be reformatted | its own large commit |

### Existing groundwork by the core developer

Over the past weeks `tcely` has begun pre-suppressing future rules via ruff suppression
comments (a real feature since ruff 0.16, works **without** `--preview`):

- `# ruff: ignore[CODE]` – single line
- `# ruff: file-ignore[CODE,CODE]` – whole file (must precede the first code)
- `# ruff: disable[CODE]` … `# ruff: enable[CODE]` – range

Codes seen across the tree: `BLE001, S110, RUF059, RUF012, SIM117, SIM118, SIM102, PIE790,
B006, B008, B010, B018, C400, C401, TRY002, TRY003, TRY004, TRY203, INT001, PLC0208`.
That list is effectively the maintainer's **intent declaration** of which rules are accepted
or permanently exempted. This concept builds on it.

> ⚠️ **Audit needed:** some of these comments may not take effect (spaces in the code list,
> placement not before the first statement). Ruff also already reports **16× `RUF100`
> (unused-noqa)** – dead suppressions. Clean both up during the rework.

---

## Guiding decisions (recommendation)

1. **No `ALL`.** The goal is a curated, bug- and modernization-oriented set without
   `D` (docstrings), `ANN` (type annotations), `CPY` (copyright), `PT` (pytest – the project
   uses `unittest`), `EM`, `FBT`, `TD`/`FIX`.
2. **Quote style stays single** – via `ruff format` + `quote-style = "single"`. `Q` rules off.
3. **Do not enable `E501` (line-too-long) initially.** 946 hits, low value, never enforced by
   the maintainer. Optionally later with `line-length = 120` + `ruff format`.
4. **Respect the maintainer's style preferences** → permanently ignore:
   `SIM300` (Yoda), `C408/C409/C410` (`dict()`/`list()`/`tuple()` calls), `E701/E722/E731`,
   `UP017/UP018`, `I001`+`RUF012` in migrations (already in `ruff.toml`).
5. **Suppression strategy:** accepted individual cases via `# noqa: CODE` (with a reason) or
   `# ruff: file-ignore[…]`. Whole rule classes via `ignore` / `per-file-ignores` in `ruff.toml`.
6. **Introduce incrementally** (phases below), one PR per rule group so reviews stay small.

---

## Target configuration `tubesync/ruff.toml`

```toml
target-version = "py312"
line-length = 88          # only relevant if E501 is enabled later

[format]
quote-style = "single"
docstring-code-format = false

[lint]
# base (ruff defaults) + curated additions
select = [
    "E4", "E7", "E9", "F", "W",   # errors/pyflakes/warnings
    "I",                          # import sorting (isort)
    "UP",                         # pyupgrade
    "B",                          # flake8-bugbear
    "C4",                         # comprehensions
    "SIM",                        # flake8-simplify
    "RET",                        # flake8-return
    "PIE",                        # flake8-pie
    "PERF",                       # perflint
    "G",                          # flake8-logging-format
    "LOG",                        # flake8-logging
    "ICN",                        # import-conventions
    "FLY",                        # flynt
    "RUF",                        # ruff-native rules (incl. RUF100)
]

ignore = [
    # deliberate project style choices
    "SIM300",   # Yoda comparisons are project style
    "C408", "C409", "C410",   # dict()/list()/tuple() calls allowed
    "E701", "E722", "E731",
    "UP017", "UP018",
    # permanently accepted (derived from the existing file-ignore comments)
    "RUF059",   # unused-unpacked-variable
    "SIM117",   # nested with statements
    "SIM118",   # `.keys()` iteration (used intentionally)
]

[lint.per-file-ignores]
"**/management/commands/*.py" = ["N999"]
"**/migrations/0*.py"        = ["I001", "RUF012", "E501", "UP"]
"**/tests/**"                = ["S101"]          # in case S is ever enabled
"shasum.py"                  = []                # see per-file-target-version

[lint.per-file-target-version]
"shasum.py" = "py310"

[lint.isort]
# align with the existing import ordering (finalize during the rework)
known-first-party = ["common", "sync", "tubesync"]
```

> The concrete `ignore` list is finalized data-driven during the rework: enable a rule
> tentatively → if <5 real occurrences and sensibly fixable: fix; otherwise ignore or
> per-file-ignore.

---

## Rollout in phases

### Phase 0 – Foundation (1 PR)
- Extend `ruff.toml` with `target-version`, `[format]` (`quote-style = "single"`).
- Run **`ruff format tubesync/`** across the whole tree → one large, purely mechanical
  formatting commit (131 files). Keep it separate so it does not add noise to the substantive
  PRs.
- `.editorconfig` stays compatible (indent 4, LF, final newline – matches `ruff format`).
- CI step: add `ruff format --check`.

### Phase 1 – Auto-fixable (1 PR)
- Set `select` to the target list.
- `ruff check --fix tubesync/` → covers ~415 violations automatically
  (I001 imports, RET50x, UP0xx, PIE790, RUF010/022, C410, W29x …).
- `ruff check --fix --unsafe-fixes` selectively per rule after visual review
  (~309 additional "hidden fixes").
- Review the result, commit.

### Phase 2 – Dead suppressions & comment audit (1 PR)
- Remove 16× `RUF100` (unused-noqa).
- Check every `# ruff: ignore[…]` / `file-ignore[…]` / `disable/enable[…]` comment: does the
  suppression actually take effect? (remove spaces, fix placement). Delete ones no longer needed.

### Phase 3 – Manual fixes by rule group (several small PRs)
One PR per group (see "Complete list" below). Rough order by value/effort:
`B006/B008` → `B904` → `B905` → `UP006/UP007/UP045/UP035` → `G004` →
`SIM105/SIM102/SIM115` → `PERF*` → `RUF012` (mutable class default) → rest.

### Phase 4 – Switch the CI over (1 PR)
- Replace the narrow `--select 'C4,E4,E7,E9,F'` invocation with `ruff check` (no `--select`,
  uses `ruff.toml`) + `ruff format --check`.
- Keep `continue-on-error: false` → merge blocker.
- Optionally keep `--output-format github` for PR annotations.

### Phase 5 – optional/later
- Enable `E501` with `line-length = 120` + re-run `ruff format`.
- Consider more groups: `PL` (Pylint, ~200), `TRY` (~90), `DTZ` (datetime zones, ~30),
  `S` (bandit/security, ~120 – mainly `S101` in tests, `S110`, `S603/S607`).

---

## Complete list: every rule group of the target tier + how to handle it

Numbers = current hits across `tubesync/` (project `ruff.toml` applied).
Action legend: **AUTO** = `--fix` · **AUTO\*** = `--unsafe-fixes`/visual review ·
**MAN** = manual · **IGN** = add to `ignore` · **PFI** = per-file-ignore.

### Import sorting
| Code | n | Action | Note |
|---|---|---|---|
| `I001` unsorted-imports | 89 | **AUTO** | finalize `isort` section; exclude migrations via PFI |

### pyupgrade (`UP`)
| Code | n | Action | Note |
|---|---|---|---|
| `UP045` non-pep604-optional (`Optional[X]`→`X \| None`) | 13 | **AUTO** | |
| `UP017` datetime-timezone-utc | 11 | **IGN** | already ignored by the project |
| `UP006` non-pep585 (`List`→`list`) | 10 | **AUTO** | |
| `UP018` native-literals | 9 | **IGN** | already ignored |
| `UP007` non-pep604-union | 5 | **AUTO** | |
| `UP035` deprecated-import | 7 | **MAN** | check `typing.X` replacement |
| `UP032` f-string, `UP012` encode-utf8, `UP015` redundant-open-modes, `UP041` | 1–3 each | **AUTO** | |

### flake8-bugbear (`B`) – real bug candidates, priority
| Code | n | Action | Note |
|---|---|---|---|
| `B904` raise-without-from-in-except | 15 | **MAN** | add `raise … from err` / `from None` |
| `B006` mutable-argument-default | 13 | **MAN** | default `None` + init in body |
| `B905` zip-without-explicit-strict | 12 | **MAN** | explicit `strict=True/False` |
| `B007` unused-loop-var | 2 | **AUTO** | `_` prefix |
| `B010` set-attr-with-constant | 6 | **AUTO** | or already suppressed by comment |
| `B008`, `B018` | a few each | **MAN/IGN** | partly already addressed/ignored by the maintainer |

### comprehensions (`C4`)
| Code | n | Action | Note |
|---|---|---|---|
| `C408/C409/C410` | 108 / 2 / 5 | **IGN** | `dict()`/`tuple()`/`list()` calls are project style |
| other `C4xx` | 0 | – | |

### flake8-simplify (`SIM`)
| Code | n | Action | Note |
|---|---|---|---|
| `SIM300` yoda-conditions | 126 | **IGN** | project style (also in CI YAML) |
| `SIM105` suppressible-exception | 8 | **MAN** | `contextlib.suppress` – case by case |
| `SIM118` in-dict-keys | 6 | **IGN** | used intentionally |
| `SIM102` collapsible-if | 5 | **MAN** | |
| `SIM117` multiple-with | 4 | **IGN** | |
| `SIM115` open-without-context | 4 | **MAN** | possibly a real resource leak → check |
| `SIM108/210/910/114/103` | 1–3 each | **AUTO/MAN** | |

### flake8-return (`RET`)
| Code | n | Action | Note |
|---|---|---|---|
| `RET505` superfluous-else-return | 19 | **AUTO** | |
| `RET502` implicit-return-value | 8 | **AUTO** | |
| `RET503` implicit-return | 6 | **MAN** | explicit `return None` |
| `RET504/501/506/507` | 1–4 each | **AUTO** | |

### Logging (`G`, `LOG`)
| Code | n | Action | Note |
|---|---|---|---|
| `G004` logging-f-string | 7 | **MAN** | `%` formatting or deliberate `# noqa` |
| `G010` logging-warn | 1 | **AUTO** | `.warning()` |
| `LOG*` | 0 | – | |

### perflint (`PERF`)
| Code | n | Action | Note |
|---|---|---|---|
| `PERF102` incorrect-dict-iterator | 3 | **AUTO\*** | `.items()`→`.values()`/`.keys()` |
| `PERF401/402/403` manual-comprehension/copy | 1 each | **MAN** | |

### flake8-pie (`PIE`)
| Code | n | Action | Note |
|---|---|---|---|
| `PIE790` unnecessary-placeholder (`pass`/`...`) | 10 | **AUTO** | |
| `PIE804` unnecessary-dict-kwargs | 1 | **AUTO** | |

### import-conventions (`ICN`)
| Code | n | Action | Note |
|---|---|---|---|
| `ICN001` unconventional-import-alias | 3 | **MAN** | e.g. `import numpy as np` – or `ignore` if project aliases differ |

### ruff-native (`RUF`)
| Code | n | Action | Note |
|---|---|---|---|
| `RUF100` unused-noqa | 16 | **AUTO** | remove dead suppressions (phase 2) |
| `RUF012` mutable-class-default | 20 | **MAN** | annotate `ClassVar[…]`; migrations via PFI |
| `RUF059` unused-unpacked-variable | 28 | **IGN** | accepted by the maintainer |
| `RUF022` unsorted-`__all__` | 3 | **AUTO** | |
| `RUF010` explicit-f-string-conversion | 2 | **AUTO** | |
| `RUF037` empty-iterable-in-deque | 3 | **AUTO** | |
| `RUF015` iterable-allocation-for-first-element | 1 | **MAN** | `next(iter(x))` |
| `RUF003` ambiguous-unicode-in-comment | 1 | **MAN** | replace the character |

### flynt (`FLY`)
| Code | n | Action | Note |
|---|---|---|---|
| `FLY002` static-join-to-fstring | 0–few | **AUTO** | |

### pycodestyle whitespace (`W`)
| Code | n | Action | Note |
|---|---|---|---|
| `W291` trailing-whitespace | 30 | **AUTO** (via `ruff format`) | |
| `W293` blank-line-with-whitespace | 9 | **AUTO** | |
| `W292` no-newline-at-eof | 2 | **AUTO** | |

### Not in the target tier, but to be decided
| Group | Hits | Recommendation |
|---|---|---|
| `E501` line-too-long | 946 | **later** (`line-length=120`) |
| `D` docstrings | ~1,400 | **no** |
| `ANN` annotations | ~1,500 | **no** |
| `PT` pytest | 338 | **no** (unittest project) |
| `CPY001` copyright | 151 | **no** |
| `PL` Pylint | ~200 | phase 5 optional |
| `TRY` tryceratops | ~90 | phase 5 optional (maintainer ignores TRY002/003/004) |
| `S` bandit | ~120 | phase 5 optional; `S101` in tests via PFI |
| `DTZ` datetime-timezone | ~30 | phase 5 optional (real bugs possible) |
| `COM`/`ISC` commas | ~220 | covered by `ruff format`, rules not needed |

---

## Critical files

- `tubesync/ruff.toml` – central target configuration (above)
- `.github/workflows/ci.yaml` – step *"Check with ruff"*: replace the narrow `--select`
  invocation, add `ruff format --check`
- `.editorconfig` – verify `ruff format` does not conflict with it (indent 4 / LF / EOF-NL)
- Highest violation density (focus for the manual phases):
  `sync/views/` (80) · `sync/models/` (73) · `sync/management/` (60) · `common/logs/` (47) ·
  `hat-syslog_tool.py` (42) · `common/huey.py` (37) · `sync/tasks.py` (29) ·
  `sync/youtube.py` (26) · `sync/hooks.py` (26) · `sync/matching.py` (24)
- Special cases: `shasum.py` / `shasum_tests.py` (own `target-version = py310`, stdlib-only,
  run separately in CI), `yt_dlp_plugins/extractor/*.py` (filenames with `-` → `N999`/import
  rules possibly PFI), `common/migrations/*` & `sync/migrations/*` (47 files, generated → PFI).

## Effort estimate

| Phase | Effort | Risk |
|---|---|---|
| 0 formatting commit | 0.5 day | low (mechanical, but large diff) |
| 1 auto-fix | 0.5 day | low |
| 2 suppression audit | 0.5 day | low |
| 3 manual fixes (~150–200 real ones, spread over ~10 PRs) | 3–5 days | medium (`B904`, `B006`, `SIM115` need understanding) |
| 4 CI switch-over | 0.25 day | low |
| **Total to "green" on the recommended tier** | **~5–7 days** | |
| 5 optional (`PL`/`TRY`/`S`/`DTZ`/`E501`) | +3–5 days | medium |

## Verification

```bash
cd tubesync

# formatting
uvx ruff format --check --target-version py312 .

# linting against the target configuration (uses ruff.toml)
uvx ruff check --target-version py312 .
# expected output at the end of each phase: "All checks passed!"

# cross-check: no dead suppressions left
uvx ruff check --select RUF100 .

# statistics overview while tuning the ignore list
uvx ruff check --statistics .
```

- After **every** phase run the Django test suite (local harness / CI matrix 3.12/3.13/3.14):
  `cd tubesync && TUBESYNC_DEBUG=True python3 -B -W default manage.py test --no-input --buffer`
- Review the formatting commit (phase 0) separately; check `git show --stat` that it is only
  whitespace/quotes.
- Before merging phase 4: trigger a CI run on a test branch (`test-*`), since the ruff step
  there runs `continue-on-error: false` as a real blocker.

## Open items for the final config

1. Align `isort.known-first-party` / possibly `force-sort-within-sections` with the existing
   style (visible after phase 1).
2. `ICN001` (3×): fix, or whitelist project-specific aliases in
   `[lint.flake8-import-conventions]`?
3. Whether `SIM115` (open without context manager, 4×) are real leaks – assess individually
   during the rework.
