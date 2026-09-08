# Konzept: tubesync komplett „ruff-sauber" bekommen

*Stand: 2026-09-08 · Ruff 0.16.6 · Scope: `tubesync/` (155 Python-Dateien)*

## Kontext

Der aktuelle CI-Check (`.github/workflows/ci.yaml`, Step *„Check with ruff"*) prüft nur einen
sehr engen Regelsatz:

```
ruff check --target-version py312 --output-format github \
  --select 'C4,E4,E7,E9,F' \
  --ignore 'C408,C409,C410,E701,E722,E731,I001,UP017,UP018'
```

Dieser enge Check ist **heute bereits grün**. `tubesync/ruff.toml` enthält bewusst kein
`[lint] select`, d.h. lokale Läufe nutzen nur die Ruff-Defaults (`E4/E7/E9/F`).

Ziel dieses Dokuments: eine Einschätzung + kompletter Guide, um **das ganze
`tubesync/`-Verzeichnis** gegen einen breiteren, sinnvollen Ruff-Regelsatz **ohne
Beanstandungen** zu bekommen — als Grundlage für ein schrittweises Rework.

### Grundfragen vorab

**„GitHub-Standard-Regelset – gibt es sowas?"**
Nein. Es gibt keinen offiziellen „GitHub-Ruff-Regelsatz". GitHubs `super-linter` führt Ruff
einfach mit der Repo-Config bzw. den Ruff-Defaults (`E4,E7,E9,F`) aus. Der breit akzeptierte
De-facto-Community-Baseline ist: **Defaults + `I,UP,B,C4,SIM,RUF`**, oft ergänzt um
`RET,PERF,PIE,G,ICN,FURB,LOG`. Genau das ist unten als **„Empfohlene Stufe"** ausformuliert.

**„Welcher Quote-Stil ist im Projekt aktuell Standard (Stammentwickler)?"**
Eindeutig **einfache Anführungszeichen (`'`)**. Messung über das ganze `tubesync/`:
~18.700 `'` vs. ~1.900 `"`. In den zuletzt von `tcely`/`meeb` geänderten Dateien
(`sync/tasks.py`, `sync/models/media.py`, `common/utils.py`, `sync/choices.py`) liegt der
Single-Quote-Anteil bei ~90–99 %. Auch die CI-YAML selbst nutzt konsequent Single Quotes und
Yoda-Vergleiche (`'pull_request' != github.event_name`). → `quote-style = "single"` setzen,
`Q`-Regeln nicht aktivieren, `SIM300` (Yoda) ignorieren.

**Scope:** nur `tubesync/` (wie heutiger CI-Scope). `patches/` bleibt außen vor.

---

## Ist-Zustand (gemessen mit ruff 0.16.6, `--target-version py312`)

| Zielbild | Verstöße | Bewertung |
|---|---|---|
| Heutiger CI-Check (`C4,E4,E7,E9,F` minus Ignores) | **0** | bereits grün |
| Ruff-Defaults (`E,F`) über ganzes `tubesync/` | 19 | trivial |
| **Empfohlene Stufe** (siehe unten), Projekt-Ignores berücksichtigt | ~900 roh · ~415 auto-fixbar · Rest großteils „ignorieren" | Ziel dieses Reworks |
| `--select ALL` | ~14.830 | unrealistisch (Docstrings/Annotations/Copyright) – **nicht** das Ziel |
| `ruff format` (Single-Quote-Config) | 131 von 156 Dateien würden umformatiert | eigener, großer Commit |

### Bereits vorhandene Vorarbeit des Stammentwicklers

`tcely` hat in den letzten Wochen begonnen, künftige Regeln vorab zu unterdrücken – per
Ruff-Suppression-Kommentaren (echtes Feature ab Ruff 0.16, funktioniert **ohne** `--preview`):

- `# ruff: ignore[CODE]` – einzelne Zeile
- `# ruff: file-ignore[CODE,CODE]` – ganze Datei (muss vor dem ersten Code stehen)
- `# ruff: disable[CODE]` … `# ruff: enable[CODE]` – Bereich

Betroffene Codes quer durch den Baum: `BLE001, S110, RUF059, RUF012, SIM117, SIM118, SIM102,
PIE790, B006, B008, B010, B018, C400, C401, TRY002, TRY003, TRY004, TRY203, INT001, PLC0208`.
Diese Liste ist faktisch die **Intent-Deklaration**, welche Regeln der Maintainer akzeptiert
bzw. dauerhaft ausnimmt. Das Konzept baut darauf auf.

> ⚠️ **Audit nötig:** Einzelne dieser Kommentare greifen evtl. nicht (Leerzeichen in der
> Code-Liste, Platzierung nicht vor dem ersten Statement). Zusätzlich meldet Ruff schon heute
> **16× `RUF100` (unused-noqa)** – tote Suppressions. Beides im Rework mit aufräumen.

---

## Grundsatzentscheidungen (Empfehlung)

1. **Kein `ALL`.** Ziel ist ein kuratierter, bug- und modernisierungs-orientierter Satz ohne
   `D` (Docstrings), `ANN` (Type-Annotations), `CPY` (Copyright), `PT` (pytest – Projekt nutzt
   `unittest`), `EM`, `FBT`, `TD`/`FIX`.
2. **Quote-Stil bleibt single** – via `ruff format` + `quote-style = "single"`. `Q`-Regeln aus.
3. **`E501` (line-too-long) zunächst nicht aktivieren.** 946 Treffer, geringer Nutzen, vom
   Maintainer nie erzwungen. Optional später mit `line-length = 120` + `ruff format`.
4. **Style-Präferenzen des Maintainers respektieren** → dauerhaft ignorieren:
   `SIM300` (Yoda), `C408/C409/C410` (`dict()`/`list()`/`tuple()`-Calls), `E701/E722/E731`,
   `UP017/UP018`, `I001`+`RUF012` in Migrations (bereits in `ruff.toml`).
5. **Suppression-Strategie:** akzeptierte Einzelfälle per `# noqa: CODE` (mit Begründung) bzw.
   `# ruff: file-ignore[…]`. Ganze Regelklassen per `ignore` / `per-file-ignores` in `ruff.toml`.
6. **Schrittweise einführen** (Phasen unten), pro Regelgruppe ein PR, damit Reviews klein bleiben.

---

## Ziel-Konfiguration `tubesync/ruff.toml`

```toml
target-version = "py312"
line-length = 88          # nur relevant falls E501 später aktiviert wird

[format]
quote-style = "single"
docstring-code-format = false

[lint]
# Basis (Ruff-Defaults) + kuratierte Ergänzungen
select = [
    "E4", "E7", "E9", "F", "W",   # Fehler/Pyflakes/Warnungen
    "I",                          # Import-Sortierung (isort)
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
    "RUF",                        # Ruff-eigene Regeln (inkl. RUF100)
]

ignore = [
    # bewusste Stil-Entscheidungen des Projekts
    "SIM300",   # Yoda-Vergleiche sind Projektstil
    "C408", "C409", "C410",   # dict()/list()/tuple()-Aufrufe erlaubt
    "E701", "E722", "E731",
    "UP017", "UP018",
    # dauerhaft akzeptiert (aus den vorhandenen file-ignore-Kommentaren abgeleitet)
    "RUF059",   # unused-unpacked-variable
    "SIM117",   # verschachtelte with-Statements
    "SIM118",   # `.keys()`-Iteration (bewusst genutzt)
]

[lint.per-file-ignores]
"**/management/commands/*.py" = ["N999"]
"**/migrations/0*.py"        = ["I001", "RUF012", "E501", "UP"]
"**/tests/**"                = ["S101"]          # falls S je aktiviert wird
"shasum.py"                  = []                # siehe per-file-target-version

[lint.per-file-target-version]
"shasum.py" = "py310"

[lint.isort]
# an bestehende Import-Reihenfolge anpassen (im Rework final justieren)
known-first-party = ["common", "sync", "tubesync"]
```

> Die konkrete `ignore`-Liste wird im Rework datengetrieben finalisiert: Regel testweise
> aktivieren → wenn <5 echte Fundstellen und sinnvoll behebbar: fixen; sonst ignorieren oder
> per-file-ignore.

---

## Umsetzung in Phasen

### Phase 0 – Fundament (1 PR)
- `ruff.toml` um `target-version`, `[format]` (`quote-style = "single"`) erweitern.
- **`ruff format tubesync/`** über den ganzen Baum laufen lassen → 1 großer, rein mechanischer
  Formatierungs-Commit (131 Dateien). Separat halten, damit er die inhaltlichen PRs nicht
  verrauscht.
- `.editorconfig` bleibt kompatibel (indent 4, LF, final newline – deckt sich mit `ruff format`).
- CI-Step: `ruff format --check` ergänzen.

### Phase 1 – Auto-fixbares (1 PR)
- `select` auf die Zielliste setzen.
- `ruff check --fix tubesync/` → deckt ~415 Verstöße automatisch ab
  (I001 Imports, RET50x, UP0xx, PIE790, RUF010/022, C410, W29x …).
- `ruff check --fix --unsafe-fixes` gezielt für einzelne Regeln nach Sichtprüfung
  (~309 zusätzliche „hidden fixes").
- Ergebnis reviewen, committen.

### Phase 2 – Tote Suppressions & Kommentar-Audit (1 PR)
- 16× `RUF100` (unused-noqa) entfernen.
- Alle `# ruff: ignore[…]` / `file-ignore[…]` / `disable/enable[…]`-Kommentare prüfen:
  greift die Unterdrückung wirklich? (Leerzeichen raus, Platzierung korrigieren.)
  Nicht mehr nötige entfernen.

### Phase 3 – Manuelle Fixes nach Regelgruppe (mehrere kleine PRs)
Je Gruppe ein PR (siehe „Komplette Auflistung" unten). Grobe Reihenfolge nach Nutzen/Aufwand:
`B006/B008` → `B904` → `B905` → `UP006/UP007/UP045/UP035` → `G004` →
`SIM105/SIM102/SIM115` → `PERF*` → `RUF012` (mutable class default) → Rest.

### Phase 4 – CI umstellen (1 PR)
- Den engen `--select 'C4,E4,E7,E9,F'`-Aufruf durch `ruff check` (ohne `--select`,
  nutzt `ruff.toml`) + `ruff format --check` ersetzen.
- `continue-on-error: false` beibehalten → Merge-Blocker.
- Optional: `--output-format github` für PR-Annotations behalten.

### Phase 5 – optional/später
- `E501` mit `line-length = 120` aktivieren + `ruff format` erneut.
- Weitere Gruppen erwägen: `PL` (Pylint, ~200), `TRY` (~90), `DTZ` (Datetime-Zonen, ~30),
  `S` (bandit/Security, ~120 – v.a. `S101` in Tests, `S110`, `S603/S607`).

---

## Komplette Auflistung: jede Regelgruppe der Zielstufe + Vorgehen

Zahlen = aktuelle Treffer über `tubesync/` (Projekt-`ruff.toml` berücksichtigt).
Legende Aktion: **AUTO** = `--fix` · **AUTO\*** = `--unsafe-fixes`/Sichtprüfung ·
**MAN** = manuell · **IGN** = in `ignore` aufnehmen · **PFI** = per-file-ignore.

### Import-Sortierung
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `I001` unsorted-imports | 89 | **AUTO** | `isort`-Sektion final justieren; Migrations via PFI raus |

### pyupgrade (`UP`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `UP045` non-pep604-optional (`Optional[X]`→`X \| None`) | 13 | **AUTO** | |
| `UP017` datetime-timezone-utc | 11 | **IGN** | bereits vom Projekt ignoriert |
| `UP006` non-pep585 (`List`→`list`) | 10 | **AUTO** | |
| `UP018` native-literals | 9 | **IGN** | bereits ignoriert |
| `UP007` non-pep604-union | 5 | **AUTO** | |
| `UP035` deprecated-import | 7 | **MAN** | `typing.X` → Ersatz prüfen |
| `UP032` f-string, `UP012` encode-utf8, `UP015` redundant-open-modes, `UP041` | je 1–3 | **AUTO** | |

### flake8-bugbear (`B`) – echte Bug-Kandidaten, Priorität
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `B904` raise-without-from-in-except | 15 | **MAN** | `raise … from err` / `from None` ergänzen |
| `B006` mutable-argument-default | 13 | **MAN** | Default `None` + Init im Body |
| `B905` zip-without-explicit-strict | 12 | **MAN** | `strict=True/False` explizit |
| `B007` unused-loop-var | 2 | **AUTO** | `_`-Präfix |
| `B010` set-attr-with-constant | 6 | **AUTO** | bzw. bereits per Kommentar unterdrückt |
| `B008`, `B018` | je wenige | **MAN/IGN** | vom Maintainer teils schon adressiert/ignoriert |

### comprehensions (`C4`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `C408/C409/C410` | 108 / 2 / 5 | **IGN** | `dict()`/`tuple()`/`list()`-Calls sind Projektstil |
| übrige `C4xx` | 0 | – | |

### flake8-simplify (`SIM`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `SIM300` yoda-conditions | 126 | **IGN** | Projektstil (auch in CI-YAML) |
| `SIM105` suppressible-exception | 8 | **MAN** | `contextlib.suppress` – Fall für Fall |
| `SIM118` in-dict-keys | 6 | **IGN** | bewusst genutzt |
| `SIM102` collapsible-if | 5 | **MAN** | |
| `SIM117` multiple-with | 4 | **IGN** | |
| `SIM115` open-without-context | 4 | **MAN** | ggf. echter Ressourcen-Leak → prüfen |
| `SIM108/210/910/114/103` | je 1–3 | **AUTO/MAN** | |

### flake8-return (`RET`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `RET505` superfluous-else-return | 19 | **AUTO** | |
| `RET502` implicit-return-value | 8 | **AUTO** | |
| `RET503` implicit-return | 6 | **MAN** | explizites `return None` |
| `RET504/501/506/507` | je 1–4 | **AUTO** | |

### Logging (`G`, `LOG`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `G004` logging-f-string | 7 | **MAN** | `%`-Formatierung oder bewusst `# noqa` |
| `G010` logging-warn | 1 | **AUTO** | `.warning()` |
| `LOG*` | 0 | – | |

### perflint (`PERF`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `PERF102` incorrect-dict-iterator | 3 | **AUTO\*** | `.items()`→`.values()`/`.keys()` |
| `PERF401/402/403` manual-comprehension/copy | je 1 | **MAN** | |

### flake8-pie (`PIE`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `PIE790` unnecessary-placeholder (`pass`/`...`) | 10 | **AUTO** | |
| `PIE804` unnecessary-dict-kwargs | 1 | **AUTO** | |

### import-conventions (`ICN`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `ICN001` unconventional-import-alias | 3 | **MAN** | z.B. `import numpy as np` – oder `ignore` wenn Projekt-Aliasse abweichen |

### Ruff-eigene (`RUF`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `RUF100` unused-noqa | 16 | **AUTO** | tote Suppressions entfernen (Phase 2) |
| `RUF012` mutable-class-default | 20 | **MAN** | `ClassVar[…]` annotieren; Migrations via PFI |
| `RUF059` unused-unpacked-variable | 28 | **IGN** | vom Maintainer akzeptiert |
| `RUF022` unsorted-`__all__` | 3 | **AUTO** | |
| `RUF010` explicit-f-string-conversion | 2 | **AUTO** | |
| `RUF037` empty-iterable-in-deque | 3 | **AUTO** | |
| `RUF015` iterable-allocation-for-first-element | 1 | **MAN** | `next(iter(x))` |
| `RUF003` ambiguous-unicode-in-comment | 1 | **MAN** | Zeichen ersetzen |

### flynt (`FLY`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `FLY002` static-join-to-fstring | 0–wenige | **AUTO** | |

### pycodestyle Whitespace (`W`)
| Code | n | Aktion | Notiz |
|---|---|---|---|
| `W291` trailing-whitespace | 30 | **AUTO** (via `ruff format`) | |
| `W293` blank-line-with-whitespace | 9 | **AUTO** | |
| `W292` no-newline-at-eof | 2 | **AUTO** | |

### Nicht in der Zielstufe, aber zu entscheiden
| Gruppe | Treffer | Empfehlung |
|---|---|---|
| `E501` line-too-long | 946 | **später** (`line-length=120`) |
| `D` Docstrings | ~1.400 | **nein** |
| `ANN` Annotations | ~1.500 | **nein** |
| `PT` pytest | 338 | **nein** (unittest-Projekt) |
| `CPY001` copyright | 151 | **nein** |
| `PL` Pylint | ~200 | Phase 5 optional |
| `TRY` tryceratops | ~90 | Phase 5 optional (Maintainer ignoriert TRY002/003/004) |
| `S` bandit | ~120 | Phase 5 optional; `S101` in Tests via PFI |
| `DTZ` datetime-timezone | ~30 | Phase 5 optional (echte Bugs möglich) |
| `COM`/`ISC` commas | ~220 | über `ruff format` abgedeckt, Regeln nicht nötig |

---

## Kritische Dateien

- `tubesync/ruff.toml` – zentrale Zielkonfiguration (oben)
- `.github/workflows/ci.yaml` – Step *„Check with ruff"*: engen `--select`-Aufruf ersetzen,
  `ruff format --check` ergänzen
- `.editorconfig` – gegenprüfen, dass `ruff format` nicht dagegen arbeitet (indent 4 / LF / EOF-NL)
- Höchste Verstoß-Dichte (Fokus für manuelle Phasen):
  `sync/views/` (80) · `sync/models/` (73) · `sync/management/` (60) · `common/logs/` (47) ·
  `hat-syslog_tool.py` (42) · `common/huey.py` (37) · `sync/tasks.py` (29) ·
  `sync/youtube.py` (26) · `sync/hooks.py` (26) · `sync/matching.py` (24)
- Sonderfälle: `shasum.py` / `shasum_tests.py` (eigene `target-version = py310`, stdlib-only,
  laufen im CI separat), `yt_dlp_plugins/extractor/*.py` (Dateinamen mit `-` → `N999`/Import-Regeln
  ggf. PFI), `common/migrations/*` & `sync/migrations/*` (47 Dateien, generiert → PFI).

## Aufwandsschätzung

| Phase | Aufwand | Risiko |
|---|---|---|
| 0 Format-Commit | 0,5 Tag | niedrig (mechanisch, aber großer Diff) |
| 1 Auto-fix | 0,5 Tag | niedrig |
| 2 Suppression-Audit | 0,5 Tag | niedrig |
| 3 Manuelle Fixes (~150–200 echte, verteilt auf ~10 PRs) | 3–5 Tage | mittel (`B904`, `B006`, `SIM115` brauchen Verständnis) |
| 4 CI-Umstellung | 0,25 Tag | niedrig |
| **Summe bis „grün" auf empfohlener Stufe** | **~5–7 Tage** | |
| 5 optional (`PL`/`TRY`/`S`/`DTZ`/`E501`) | +3–5 Tage | mittel |

## Verifikation

```bash
cd tubesync

# Formatierung
uvx ruff format --check --target-version py312 .

# Linting gegen die Zielkonfiguration (nutzt ruff.toml)
uvx ruff check --target-version py312 .
# erwartete Ausgabe am Ende jeder Phase: "All checks passed!"

# Gegenprobe: keine toten Suppressions mehr
uvx ruff check --select RUF100 .

# Statistik-Überblick beim Nachjustieren der ignore-Liste
uvx ruff check --statistics .
```

- Nach **jeder** Phase Django-Testsuite laufen lassen (lokaler Harness / CI-Matrix
  3.12/3.13/3.14): `cd tubesync && TUBESYNC_DEBUG=True python3 -B -W default manage.py test --no-input --buffer`
- Format-Commit (Phase 0) getrennt reviewen; `git show --stat` prüfen, dass nur Whitespace/Quotes.
- Vor Merge von Phase 4: CI-Run auf einem Test-Branch (`test-*`) triggern, da der ruff-Step
  dort `continue-on-error: false` als echten Blocker fährt.

## Offene Punkte für die finale Config

1. `isort.known-first-party` / evtl. `force-sort-within-sections` an bestehenden Stil anpassen
   (nach Phase 1 sichtbar).
2. `ICN001` (3×): fixen oder projekt-spezifische Aliasse in `[lint.flake8-import-conventions]`
   whitelisten?
3. Ob `SIM115` (open ohne Kontextmanager, 4×) echte Leaks sind – im Rework einzeln bewerten.
