# Tracking Audit Rubric — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the audit tooling that runs the 18-dimension rubric defined in `docs/superpowers/specs/2026-04-18-tracking-audit-rubric-design.md` against any entry point and emits per-entry reports + an aggregate scorecard.

**Architecture:** Python package at top-level `audit/`, peer to existing `monitoring/` and `sdk/`. Extends the existing `tracking-contract.json` (additive — no breaking changes) with an `entry_points` section. 18 dimension plugins auto-discovered by the runner. Output: markdown per-entry reports + markdown+JSON scorecard. CLI: `python -m audit run|generate-registry|drift-check`.

**Tech Stack:** Python 3 (matches existing `monitoring/`), pytest + responses for tests, PostHog Query API, Playwright (already installed) for resilience dim, existing `monitoring/reconciliation.py` reused.

---

## File Structure

Everything new lives under `audit/`. Tests mirror the tree under `tests/audit/`.

```
audit/
  __init__.py
  __main__.py                    # python -m audit entry point
  cli.py                         # argparse: run, generate-registry, drift-check
  core/
    __init__.py
    models.py                    # Entry, Window, Result, Status dataclasses
    registry.py                  # load/parse tracking-contract.json entry_points
    runner.py                    # discover dimensions, orchestrate execution
    posthog_client.py            # thin HogQL wrapper
    sources.py                   # WP/ThriveCart/Ontraport source-of-truth adapters
  dimensions/
    __init__.py
    _helpers.py                  # shared test helpers (HogQL count, SoT fetch)
    dim_01_registered.py
    dim_02_server_capture.py
    dim_03_client_capture.py
    dim_04_payload_complete.py
    dim_05_payload_correct.py
    dim_06_attribution.py
    dim_07_identity.py
    dim_08_dedup.py
    dim_09_reconciliation.py
    dim_10_resilience.py
    dim_11_freshness.py
    dim_12_monitoring.py
    dim_13_architectural_fit.py
    dim_14_simplicity.py
    dim_15_consistency.py
    dim_16_code_quality.py
    dim_17_testability.py
    dim_18_doc_truth.py
  reporters/
    __init__.py
    per_entry.py                 # markdown per-entry report
    scorecard.py                 # markdown + JSON aggregate
  scanners/
    __init__.py
    html_forms.py                # HTML scrape for form discovery
    thrivecart_products.py       # ThriveCart product list
  fire_drill_log.json            # manual: history of monitoring fire drills
  architecture_review.md         # manual: one Decision per subsystem
  reports/                       # output: per-entry reports land here
  scorecards/                    # output: scorecards land here
tests/audit/
  conftest.py
  test_models.py
  test_registry.py
  test_posthog_client.py
  test_runner.py
  dimensions/
    test_dim_01_registered.py
    ... (one test file per dimension)
  reporters/
    test_per_entry.py
    test_scorecard.py
  integration/
    test_reference_run.py        # full end-to-end on tr-betrayal-form
```

Extended at repo root:
- `tracking-contract.json` — add top-level `entry_points` array (additive, existing consumers ignore)
- `requirements.txt` — add `responses` for HTTP mocking in tests

---

## Task 1: Scaffold `audit/` package + core dataclasses

**Files:**
- Create: `audit/__init__.py`, `audit/core/__init__.py`, `audit/core/models.py`
- Create: `tests/audit/__init__.py`, `tests/audit/conftest.py`, `tests/audit/test_models.py`
- Modify: `requirements.txt` — add `responses>=0.25.0`

- [ ] **Step 1: Create package skeleton**

```bash
mkdir -p audit/core audit/dimensions audit/reporters audit/scanners audit/reports audit/scorecards
mkdir -p tests/audit/dimensions tests/audit/reporters tests/audit/integration
touch audit/__init__.py audit/core/__init__.py audit/dimensions/__init__.py audit/reporters/__init__.py audit/scanners/__init__.py
touch tests/audit/__init__.py tests/audit/dimensions/__init__.py tests/audit/reporters/__init__.py tests/audit/integration/__init__.py
```

- [ ] **Step 2: Write failing test for models**

Create `tests/audit/test_models.py`:

```python
from datetime import datetime, timedelta, timezone

import pytest

from audit.core.models import Entry, Result, Status, Window


def test_status_has_three_values():
    assert {s.value for s in Status} == {"pass", "fail", "na"}


def test_window_has_start_and_end():
    now = datetime.now(timezone.utc)
    w = Window(start=now - timedelta(days=7), end=now)
    assert (w.end - w.start) == timedelta(days=7)


def test_window_from_spec_7d():
    w = Window.from_spec("7d")
    assert (w.end - w.start).days == 7


def test_window_from_spec_24h():
    w = Window.from_spec("24h")
    assert (w.end - w.start) == timedelta(hours=24)


def test_entry_required_fields():
    e = Entry(
        id="tr-betrayal-form",
        site="terryreal.com",
        type="elementor_form",
        locator={"url_pattern": "^/betrayal/$"},
        expected_events=["FormSubmit", "Lead"],
        required_properties={"FormSubmit": ["email", "site"]},
        source_of_truth="elementor_pro/forms/new_record",
        reconciliation_query="",
        applies=[1, 2, 3],
        skip=[13, 16],
    )
    assert e.id == "tr-betrayal-form"
    assert 1 in e.applies
    assert 13 in e.skip


def test_result_construction():
    r = Result(
        dimension=2,
        status=Status.PASS,
        evidence={"hogql_count": 82, "source_count": 82, "ratio": 1.0},
        remediation_hint=None,
    )
    assert r.status is Status.PASS
    assert r.evidence["ratio"] == 1.0
```

- [ ] **Step 3: Run test, verify it fails**

```bash
cd /Users/jamescox/.superset/worktrees/rli/plastic-mood
python -m pytest tests/audit/test_models.py -v
```

Expected: `ModuleNotFoundError: No module named 'audit.core.models'`

- [ ] **Step 4: Implement `audit/core/models.py`**

```python
"""Core data types for the audit runner."""

from __future__ import annotations

import re
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from enum import Enum
from typing import Any


class Status(str, Enum):
    PASS = "pass"
    FAIL = "fail"
    NA = "na"


@dataclass(frozen=True)
class Window:
    start: datetime
    end: datetime

    @classmethod
    def from_spec(cls, spec: str) -> "Window":
        """Parse '7d', '24h', '30m' into a Window ending at now (UTC)."""
        match = re.fullmatch(r"(\d+)([dhm])", spec)
        if not match:
            raise ValueError(f"Invalid window spec: {spec!r}")
        n, unit = int(match.group(1)), match.group(2)
        delta = {"d": timedelta(days=n), "h": timedelta(hours=n), "m": timedelta(minutes=n)}[unit]
        end = datetime.now(timezone.utc)
        return cls(start=end - delta, end=end)


@dataclass(frozen=True)
class Entry:
    id: str
    site: str
    type: str
    locator: dict[str, Any]
    expected_events: list[str]
    required_properties: dict[str, list[str]]
    source_of_truth: str
    reconciliation_query: str
    applies: list[int]
    skip: list[int] = field(default_factory=list)


@dataclass
class Result:
    dimension: int
    status: Status
    evidence: dict[str, Any]
    remediation_hint: str | None = None
```

- [ ] **Step 5: Run test, verify it passes**

```bash
python -m pytest tests/audit/test_models.py -v
```

Expected: 6 passed.

- [ ] **Step 6: Add `responses` to requirements and create pytest conftest**

Append to `requirements.txt`:

```
responses>=0.25.0
```

Install:

```bash
pip install responses>=0.25.0
```

Create `tests/audit/conftest.py`:

```python
"""Shared fixtures for audit tests."""

from datetime import datetime, timedelta, timezone

import pytest

from audit.core.models import Entry, Window


@pytest.fixture
def window_7d() -> Window:
    now = datetime(2026, 4, 18, 12, 0, 0, tzinfo=timezone.utc)
    return Window(start=now - timedelta(days=7), end=now)


@pytest.fixture
def betrayal_entry() -> Entry:
    return Entry(
        id="tr-betrayal-form",
        site="terryreal.com",
        type="elementor_form",
        locator={"url_pattern": r"^/betrayal(-offer|-1|-workshop-test)?/$"},
        expected_events=["FormSubmit", "Lead"],
        required_properties={
            "FormSubmit": ["email", "site", "page", "form_type", "utm_source"],
            "Lead": ["email", "site", "page", "utm_source"],
        },
        source_of_truth="elementor_pro/forms/new_record",
        reconciliation_query="",
        applies=[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14, 15, 17, 18],
        skip=[13, 16],
    )
```

- [ ] **Step 7: Commit**

```bash
git add audit/ tests/audit/ requirements.txt
git commit -m "audit: scaffold package + core dataclasses (Entry, Window, Result, Status)"
```

---

## Task 2: Extend `tracking-contract.json` + registry loader

**Files:**
- Modify: `tracking-contract.json` — add `entry_points` array
- Create: `audit/core/registry.py`, `tests/audit/test_registry.py`

- [ ] **Step 1: Write failing test**

Create `tests/audit/test_registry.py`:

```python
import json
from pathlib import Path

import pytest

from audit.core.registry import load_registry, load_entry


def test_load_registry_returns_entries(tmp_path: Path):
    contract = {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "version": "1.0.0",
        "entry_points": [
            {
                "id": "tr-betrayal-form",
                "site": "terryreal.com",
                "type": "elementor_form",
                "locator": {"url_pattern": "^/betrayal/$"},
                "expected_events": ["FormSubmit", "Lead"],
                "required_properties": {
                    "FormSubmit": ["email", "site"],
                    "Lead": ["email", "site"],
                },
                "source_of_truth": "elementor_pro/forms/new_record",
                "reconciliation_query": "",
                "applies": [1, 2, 3],
                "skip": [13, 16],
            }
        ],
    }
    path = tmp_path / "tracking-contract.json"
    path.write_text(json.dumps(contract))

    entries = load_registry(path)
    assert len(entries) == 1
    assert entries[0].id == "tr-betrayal-form"
    assert entries[0].applies == [1, 2, 3]


def test_load_registry_missing_file(tmp_path: Path):
    with pytest.raises(FileNotFoundError):
        load_registry(tmp_path / "nope.json")


def test_load_registry_rejects_duplicate_ids(tmp_path: Path):
    contract = {
        "entry_points": [
            {"id": "dup", "site": "x", "type": "t", "locator": {},
             "expected_events": [], "required_properties": {},
             "source_of_truth": "", "reconciliation_query": "",
             "applies": [], "skip": []},
            {"id": "dup", "site": "x", "type": "t", "locator": {},
             "expected_events": [], "required_properties": {},
             "source_of_truth": "", "reconciliation_query": "",
             "applies": [], "skip": []},
        ]
    }
    path = tmp_path / "tracking-contract.json"
    path.write_text(json.dumps(contract))

    with pytest.raises(ValueError, match="duplicate entry id"):
        load_registry(path)


def test_load_registry_no_entry_points_key_returns_empty(tmp_path: Path):
    path = tmp_path / "tracking-contract.json"
    path.write_text('{"version": "1.0.0"}')
    assert load_registry(path) == []


def test_load_entry_by_id(tmp_path: Path):
    contract = {
        "entry_points": [
            {"id": "a", "site": "x", "type": "t", "locator": {},
             "expected_events": [], "required_properties": {},
             "source_of_truth": "", "reconciliation_query": "",
             "applies": [], "skip": []},
            {"id": "b", "site": "y", "type": "t", "locator": {},
             "expected_events": [], "required_properties": {},
             "source_of_truth": "", "reconciliation_query": "",
             "applies": [], "skip": []},
        ]
    }
    path = tmp_path / "tracking-contract.json"
    path.write_text(json.dumps(contract))

    assert load_entry(path, "b").site == "y"
    with pytest.raises(KeyError):
        load_entry(path, "missing")
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/test_registry.py -v
```

Expected: ImportError / ModuleNotFoundError.

- [ ] **Step 3: Implement `audit/core/registry.py`**

```python
"""Load entry points from tracking-contract.json."""

from __future__ import annotations

import json
from pathlib import Path

from audit.core.models import Entry


REQUIRED_FIELDS = (
    "id", "site", "type", "locator", "expected_events",
    "required_properties", "source_of_truth", "reconciliation_query",
    "applies",
)


def _entry_from_dict(raw: dict) -> Entry:
    missing = [f for f in REQUIRED_FIELDS if f not in raw]
    if missing:
        raise ValueError(f"entry missing required fields: {missing} (entry={raw.get('id')!r})")
    return Entry(
        id=raw["id"],
        site=raw["site"],
        type=raw["type"],
        locator=raw["locator"],
        expected_events=raw["expected_events"],
        required_properties=raw["required_properties"],
        source_of_truth=raw["source_of_truth"],
        reconciliation_query=raw["reconciliation_query"],
        applies=list(raw["applies"]),
        skip=list(raw.get("skip", [])),
    )


def load_registry(path: Path) -> list[Entry]:
    """Parse tracking-contract.json and return all entry points.

    Returns [] if the file has no entry_points key (additive extension,
    older versions of the contract pre-date the audit rubric).
    """
    data = json.loads(Path(path).read_text())
    raw_entries = data.get("entry_points", [])
    entries = [_entry_from_dict(e) for e in raw_entries]
    ids = [e.id for e in entries]
    if len(ids) != len(set(ids)):
        dupes = {i for i in ids if ids.count(i) > 1}
        raise ValueError(f"duplicate entry id(s): {dupes}")
    return entries


def load_entry(path: Path, entry_id: str) -> Entry:
    for e in load_registry(path):
        if e.id == entry_id:
            return e
    raise KeyError(f"entry {entry_id!r} not found in {path}")
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/test_registry.py -v
```

Expected: 5 passed.

- [ ] **Step 5: Extend `tracking-contract.json` with entry_points array**

Add to `tracking-contract.json` at the top level (after the existing `events` section). Keep one sample entry for now — Task 28 will seed the rest via the scanner.

```json
  "entry_points": [
    {
      "id": "tr-betrayal-form",
      "site": "terryreal.com",
      "type": "elementor_form",
      "locator": {
        "url_pattern": "^/betrayal(-offer|-1|-workshop-test)?/$",
        "form_id": null
      },
      "expected_events": ["FormSubmit", "Lead"],
      "required_properties": {
        "FormSubmit": ["email", "site", "page", "form_type", "utm_source"],
        "Lead": ["email", "site", "page", "utm_source"]
      },
      "source_of_truth": "elementor_pro/forms/new_record",
      "reconciliation_query": "elementor_submissions_where_url_matches",
      "applies": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14, 15, 17, 18],
      "skip": [13, 16]
    }
  ]
```

- [ ] **Step 6: Verify real contract loads**

```bash
python -c "from audit.core.registry import load_registry; from pathlib import Path; es = load_registry(Path('tracking-contract.json')); print(f'{len(es)} entries: {[e.id for e in es]}')"
```

Expected: `1 entries: ['tr-betrayal-form']`

- [ ] **Step 7: Commit**

```bash
git add tracking-contract.json audit/core/registry.py tests/audit/test_registry.py
git commit -m "audit: registry loader + extend tracking-contract.json with entry_points"
```

---

## Task 3: PostHog HogQL client wrapper

**Files:**
- Create: `audit/core/posthog_client.py`, `tests/audit/test_posthog_client.py`

- [ ] **Step 1: Write failing test**

Create `tests/audit/test_posthog_client.py`:

```python
import os

import pytest
import responses

from audit.core.posthog_client import PostHogClient, PostHogError


@responses.activate
def test_hogql_query_sends_correct_request():
    responses.post(
        "https://us.posthog.com/api/projects/127361/query/",
        json={"results": [[42]], "columns": ["count"]},
        status=200,
    )
    client = PostHogClient(project_id="127361", api_key="phx_test", host="https://us.posthog.com")
    result = client.hogql("SELECT count() FROM events")
    assert result.rows == [[42]]
    assert result.columns == ["count"]

    sent = responses.calls[0].request
    assert sent.headers["Authorization"] == "Bearer phx_test"
    body = sent.body.decode() if isinstance(sent.body, bytes) else sent.body
    assert "SELECT count() FROM events" in body


@responses.activate
def test_hogql_raises_on_http_error():
    responses.post(
        "https://us.posthog.com/api/projects/127361/query/",
        json={"detail": "unauthorized"},
        status=401,
    )
    client = PostHogClient(project_id="127361", api_key="bad", host="https://us.posthog.com")
    with pytest.raises(PostHogError, match="401"):
        client.hogql("SELECT 1")


def test_from_env_reads_key_file(tmp_path, monkeypatch):
    key_file = tmp_path / ".posthog-key"
    key_file.write_text("phx_from_file\n")
    monkeypatch.setenv("POSTHOG_KEY_FILE", str(key_file))
    monkeypatch.setenv("POSTHOG_PROJECT_ID", "127361")

    client = PostHogClient.from_env()
    assert client.api_key == "phx_from_file"
    assert client.project_id == "127361"


def test_from_env_missing_key_raises(monkeypatch, tmp_path):
    monkeypatch.setenv("POSTHOG_KEY_FILE", str(tmp_path / "nope"))
    monkeypatch.setenv("POSTHOG_PROJECT_ID", "127361")
    with pytest.raises(PostHogError, match="POSTHOG_KEY_FILE"):
        PostHogClient.from_env()
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/test_posthog_client.py -v
```

Expected: ModuleNotFoundError.

- [ ] **Step 3: Implement `audit/core/posthog_client.py`**

```python
"""Thin HogQL client for the audit runner.

Keeps scope tight — one method (hogql) that the dimensions call.
"""

from __future__ import annotations

import os
from dataclasses import dataclass
from pathlib import Path
from typing import Any

import requests


class PostHogError(RuntimeError):
    """Raised when PostHog returns non-2xx or env config is missing."""


@dataclass
class QueryResult:
    columns: list[str]
    rows: list[list[Any]]


class PostHogClient:
    def __init__(self, project_id: str, api_key: str, host: str = "https://us.posthog.com"):
        self.project_id = project_id
        self.api_key = api_key
        self.host = host.rstrip("/")

    @classmethod
    def from_env(cls) -> "PostHogClient":
        key_file = os.environ.get("POSTHOG_KEY_FILE", ".posthog-key")
        path = Path(key_file)
        if not path.exists():
            raise PostHogError(f"POSTHOG_KEY_FILE not found: {path}")
        api_key = path.read_text().strip()
        project_id = os.environ.get("POSTHOG_PROJECT_ID", "127361")
        host = os.environ.get("POSTHOG_HOST", "https://us.posthog.com")
        return cls(project_id=project_id, api_key=api_key, host=host)

    def hogql(self, query: str, timeout: float = 30.0) -> QueryResult:
        url = f"{self.host}/api/projects/{self.project_id}/query/"
        payload = {"query": {"kind": "HogQLQuery", "query": query}}
        resp = requests.post(
            url,
            json=payload,
            headers={"Authorization": f"Bearer {self.api_key}"},
            timeout=timeout,
        )
        if resp.status_code >= 400:
            raise PostHogError(f"HogQL {resp.status_code}: {resp.text[:200]}")
        data = resp.json()
        return QueryResult(columns=data.get("columns", []), rows=data.get("results", []))
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/test_posthog_client.py -v
```

Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/core/posthog_client.py tests/audit/test_posthog_client.py
git commit -m "audit: PostHog HogQL client wrapper"
```

---

## Task 4: Dimension plugin interface + runner

**Files:**
- Create: `audit/core/runner.py`, `tests/audit/test_runner.py`

- [ ] **Step 1: Write failing test**

Create `tests/audit/test_runner.py`:

```python
from audit.core.models import Entry, Result, Status, Window
from audit.core.runner import DimensionPlugin, Runner


class FakePlugin:
    dimension = 99

    def applies_to(self, entry: Entry) -> bool:
        return 99 in entry.applies

    def run(self, entry: Entry, window: Window, ctx) -> Result:
        return Result(dimension=99, status=Status.PASS, evidence={"ok": True})


def test_runner_executes_applicable_plugin(betrayal_entry, window_7d):
    e = Entry(**{**betrayal_entry.__dict__, "applies": [99]})
    runner = Runner(plugins=[FakePlugin()], context={})
    results = runner.run_entry(e, window_7d)
    assert len(results) == 1
    assert results[0].status is Status.PASS


def test_runner_skips_non_applicable_plugin(betrayal_entry, window_7d):
    e = Entry(**{**betrayal_entry.__dict__, "applies": [1, 2], "skip": [99]})
    runner = Runner(plugins=[FakePlugin()], context={})
    results = runner.run_entry(e, window_7d)
    assert results == []


def test_runner_returns_na_for_skipped_dimension():
    # When a plugin's dim is in skip, result is NA not omitted
    # (needed so per-entry reports show "N/A" explicitly)
    class AppliesAlwaysButInSkip:
        dimension = 13

        def applies_to(self, entry):
            return True

        def run(self, entry, window, ctx):
            return Result(dimension=13, status=Status.PASS, evidence={})

    e = Entry(
        id="x", site="x", type="t", locator={},
        expected_events=[], required_properties={},
        source_of_truth="", reconciliation_query="",
        applies=[1, 13], skip=[13],
    )
    runner = Runner(plugins=[AppliesAlwaysButInSkip()], context={})
    results = runner.run_entry(e, Window.from_spec("24h"))
    assert len(results) == 1
    assert results[0].status is Status.NA
    assert results[0].evidence == {"reason": "dimension in entry.skip list"}
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/test_runner.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement `audit/core/runner.py`**

```python
"""Runner — orchestrates dimension plugins against an entry."""

from __future__ import annotations

from typing import Any, Protocol, runtime_checkable

from audit.core.models import Entry, Result, Status, Window


@runtime_checkable
class DimensionPlugin(Protocol):
    dimension: int

    def applies_to(self, entry: Entry) -> bool: ...

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result: ...


class Runner:
    def __init__(self, plugins: list[DimensionPlugin], context: dict[str, Any]):
        self.plugins = sorted(plugins, key=lambda p: p.dimension)
        self.context = context

    def run_entry(self, entry: Entry, window: Window) -> list[Result]:
        results: list[Result] = []
        for plugin in self.plugins:
            if not plugin.applies_to(entry):
                continue
            if plugin.dimension in entry.skip:
                results.append(Result(
                    dimension=plugin.dimension,
                    status=Status.NA,
                    evidence={"reason": "dimension in entry.skip list"},
                ))
                continue
            results.append(plugin.run(entry, window, self.context))
        return results
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/test_runner.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/core/runner.py tests/audit/test_runner.py
git commit -m "audit: dimension plugin interface + runner orchestration"
```

---

## Task 5: Dim #1 — Registered (drift detector)

**Files:**
- Create: `audit/scanners/html_forms.py`, `audit/scanners/thrivecart_products.py`
- Create: `audit/dimensions/dim_01_registered.py`, `tests/audit/dimensions/test_dim_01_registered.py`

- [ ] **Step 1: Write failing test**

Create `tests/audit/dimensions/test_dim_01_registered.py`:

```python
from audit.core.models import Entry, Status, Window
from audit.dimensions.dim_01_registered import Dim01Registered


def make_entry(entry_id="x"):
    return Entry(
        id=entry_id, site="terryreal.com", type="elementor_form",
        locator={"url_pattern": "^/betrayal/$"},
        expected_events=["FormSubmit"], required_properties={"FormSubmit": []},
        source_of_truth="", reconciliation_query="",
        applies=[1], skip=[],
    )


def test_registered_passes_when_entry_is_in_registry():
    entry = make_entry("tr-betrayal-form")
    ctx = {"known_entry_ids": {"tr-betrayal-form", "tr-unstuck-form"}}
    result = Dim01Registered().run(entry, Window.from_spec("24h"), ctx)
    assert result.status is Status.PASS


def test_registered_fails_when_site_has_unknown_forms():
    entry = make_entry("tr-betrayal-form")
    ctx = {
        "known_entry_ids": {"tr-betrayal-form"},
        "discovered_form_urls_by_site": {
            "terryreal.com": ["/betrayal/", "/mystery-new-form/"],
        },
        "registered_url_patterns_by_site": {
            "terryreal.com": [r"^/betrayal(-offer|-1|-workshop-test)?/$"],
        },
    }
    result = Dim01Registered().run(entry, Window.from_spec("24h"), ctx)
    assert result.status is Status.FAIL
    assert "/mystery-new-form/" in result.evidence["unregistered_urls"]
    assert result.remediation_hint is not None


def test_registered_passes_when_all_discovered_match_a_pattern():
    entry = make_entry("tr-betrayal-form")
    ctx = {
        "known_entry_ids": {"tr-betrayal-form"},
        "discovered_form_urls_by_site": {
            "terryreal.com": ["/betrayal/", "/betrayal-offer/"],
        },
        "registered_url_patterns_by_site": {
            "terryreal.com": [r"^/betrayal(-offer|-1|-workshop-test)?/$"],
        },
    }
    result = Dim01Registered().run(entry, Window.from_spec("24h"), ctx)
    assert result.status is Status.PASS
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_01_registered.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement dimension**

Create `audit/dimensions/dim_01_registered.py`:

```python
"""Dimension 1 — Registered: no unknown forms on any site."""

from __future__ import annotations

import re
from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim01Registered:
    dimension = 1

    def applies_to(self, entry: Entry) -> bool:
        return 1 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        discovered = ctx.get("discovered_form_urls_by_site", {}).get(entry.site, [])
        patterns = ctx.get("registered_url_patterns_by_site", {}).get(entry.site, [])
        compiled = [re.compile(p) for p in patterns]
        unknown = [url for url in discovered if not any(c.search(url) for c in compiled)]
        if unknown:
            return Result(
                dimension=1,
                status=Status.FAIL,
                evidence={"unregistered_urls": unknown, "site": entry.site},
                remediation_hint=(
                    f"Add entry_points in tracking-contract.json for {len(unknown)} "
                    f"unregistered form URL(s) on {entry.site}."
                ),
            )
        return Result(
            dimension=1,
            status=Status.PASS,
            evidence={"site": entry.site, "discovered_count": len(discovered)},
        )
```

Create `audit/scanners/html_forms.py`:

```python
"""Scrape a site for `<form>` elements that look like trackable entry points.

Lightweight — just enough to detect unknown forms. Full spidering is out of
scope for Spec 1; this hits a curated list of seed URLs per site.
"""

from __future__ import annotations

from dataclasses import dataclass
from html.parser import HTMLParser
from urllib.parse import urljoin, urlparse

import requests


SEED_PATHS_BY_SITE = {
    "terryreal.com": [
        "/", "/betrayal/", "/betrayal-offer/", "/couple-therapists-workshop/",
        "/unstuck/", "/masculinity-workshop/", "/stay-or-go/",
        "/stop-earning-love/", "/20-essential-practices-of-relational-life-therapy/",
    ],
    "relationallife.com": [
        "/", "/the-certification-guide/", "/guide/", "/certification-guide-v2/",
        "/5-common-mistakes-webinar/", "/5cmw/", "/5cm-landing/",
        "/second-session/", "/certification/", "/apply/",
        "/parenting-team/", "/parenting-webinar-1/",
    ],
    "summit.terryreal.com": ["/", "/us/", "/var/"],
    "quiz.terryreal.com": ["/"],
    "grid.terryreal.com": ["/quiz-v2", "/new-quiz"],
}


@dataclass
class DiscoveredForm:
    site: str
    path: str
    action: str | None
    inputs: list[str]


class _FormFinder(HTMLParser):
    def __init__(self):
        super().__init__()
        self.forms: list[dict] = []
        self._current: dict | None = None

    def handle_starttag(self, tag, attrs):
        ad = dict(attrs)
        if tag == "form":
            self._current = {"action": ad.get("action"), "inputs": []}
            self.forms.append(self._current)
        elif tag == "input" and self._current is not None:
            name = ad.get("name")
            if name:
                self._current["inputs"].append(name)

    def handle_endtag(self, tag):
        if tag == "form":
            self._current = None


def scan_site(site: str, timeout: float = 10.0) -> list[DiscoveredForm]:
    paths = SEED_PATHS_BY_SITE.get(site, ["/"])
    out: list[DiscoveredForm] = []
    for path in paths:
        url = f"https://{site}{path}"
        try:
            resp = requests.get(url, timeout=timeout, allow_redirects=True)
            if resp.status_code != 200:
                continue
        except requests.RequestException:
            continue
        parser = _FormFinder()
        parser.feed(resp.text)
        for f in parser.forms:
            if not f["inputs"]:
                continue
            out.append(DiscoveredForm(site=site, path=path, action=f["action"], inputs=f["inputs"]))
    return out


def discovered_urls_by_site(sites: list[str]) -> dict[str, list[str]]:
    return {site: sorted({f.path for f in scan_site(site)}) for site in sites}
```

Create stub `audit/scanners/thrivecart_products.py`:

```python
"""ThriveCart product scanner — stub for Spec 1.

Real implementation calls ThriveCart /products API. For now returns
the registry-recorded product IDs only — wired up in Spec 2.
"""

from __future__ import annotations


def list_product_ids(api_key: str | None = None) -> list[str]:
    return []
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_01_registered.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_01_registered.py audit/scanners/ tests/audit/dimensions/test_dim_01_registered.py
git commit -m "audit: dim #1 Registered + HTML form scanner"
```

---

## Task 6: Dim #2 — Server-side capture

**Files:**
- Create: `audit/core/sources.py`
- Create: `audit/dimensions/dim_02_server_capture.py`, `tests/audit/dimensions/test_dim_02_server_capture.py`

- [ ] **Step 1: Write failing test**

Create `tests/audit/dimensions/test_dim_02_server_capture.py`:

```python
from unittest.mock import MagicMock

from audit.core.models import Status, Window
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_02_server_capture import Dim02ServerCapture


def test_server_capture_passes_at_99pct_ratio(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["c"], rows=[[99]])
    sources = MagicMock()
    sources.source_of_truth_count.return_value = 100

    ctx = {"posthog": ph, "sources": sources}
    result = Dim02ServerCapture().run(betrayal_entry, window_7d, ctx)
    assert result.status is Status.PASS
    assert result.evidence["ratio"] == 0.99


def test_server_capture_fails_below_threshold(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["c"], rows=[[90]])
    sources = MagicMock()
    sources.source_of_truth_count.return_value = 100

    ctx = {"posthog": ph, "sources": sources}
    result = Dim02ServerCapture().run(betrayal_entry, window_7d, ctx)
    assert result.status is Status.FAIL
    assert result.evidence["ratio"] == 0.90
    assert "10 missing" in (result.remediation_hint or "")


def test_server_capture_na_when_source_count_is_zero(betrayal_entry, window_7d):
    # No activity in window = cannot judge capture
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["c"], rows=[[0]])
    sources = MagicMock()
    sources.source_of_truth_count.return_value = 0

    ctx = {"posthog": ph, "sources": sources}
    result = Dim02ServerCapture().run(betrayal_entry, window_7d, ctx)
    assert result.status is Status.NA
    assert result.evidence["reason"] == "no activity in window"
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_02_server_capture.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement `audit/core/sources.py`**

```python
"""Source-of-truth adapters for reconciliation dimensions.

Each adapter knows how to count records in the authoritative system
for a given entry type. For Spec 1 we ship a dispatch shell;
real per-type queries are wired up one at a time as dims need them.
"""

from __future__ import annotations

from typing import Protocol

from audit.core.models import Entry, Window


class SourceOfTruth(Protocol):
    def count(self, entry: Entry, window: Window) -> int: ...


class SourceRegistry:
    """Dispatch: entry.type → counter implementation.

    Counters return int. If a counter for a type isn't registered, count
    returns -1 (sentinel meaning "unknown" — callers treat as NA).
    """

    def __init__(self):
        self._counters: dict[str, SourceOfTruth] = {}

    def register(self, entry_type: str, counter: SourceOfTruth) -> None:
        self._counters[entry_type] = counter

    def source_of_truth_count(self, entry: Entry, window: Window) -> int:
        counter = self._counters.get(entry.type)
        if counter is None:
            return -1
        return counter.count(entry, window)
```

- [ ] **Step 4: Implement dimension**

Create `audit/dimensions/dim_02_server_capture.py`:

```python
"""Dimension 2 — Server-side capture matches source-of-truth at ≥99%."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


THRESHOLD = 0.99


class Dim02ServerCapture:
    dimension = 2

    def applies_to(self, entry: Entry) -> bool:
        return 2 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        sources = ctx["sources"]

        source_count = sources.source_of_truth_count(entry, window)
        if source_count <= 0:
            return Result(
                dimension=2,
                status=Status.NA,
                evidence={"reason": "no activity in window", "source_count": source_count},
            )

        # Count server-side events: tracking_layer='server'
        event_names = "', '".join(entry.expected_events)
        query = f"""
            SELECT count()
            FROM events
            WHERE event IN ('{event_names}')
              AND properties.site = '{entry.site}'
              AND properties.tracking_layer = 'server'
              AND timestamp >= '{window.start.isoformat()}'
              AND timestamp <  '{window.end.isoformat()}'
        """
        result = ph.hogql(query)
        posthog_count = result.rows[0][0] if result.rows else 0

        ratio = round(posthog_count / source_count, 4)
        status = Status.PASS if ratio >= THRESHOLD else Status.FAIL
        hint = None
        if status is Status.FAIL:
            hint = (
                f"{source_count - posthog_count} missing server events "
                f"({(1 - ratio) * 100:.1f}% gap). Check PHP hook for {entry.type}."
            )
        return Result(
            dimension=2,
            status=status,
            evidence={
                "posthog_count": posthog_count,
                "source_count": source_count,
                "ratio": ratio,
                "threshold": THRESHOLD,
            },
            remediation_hint=hint,
        )
```

- [ ] **Step 5: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_02_server_capture.py -v
```

Expected: 3 passed.

- [ ] **Step 6: Commit**

```bash
git add audit/core/sources.py audit/dimensions/dim_02_server_capture.py tests/audit/dimensions/test_dim_02_server_capture.py
git commit -m "audit: dim #2 Server-side capture + source-of-truth dispatch"
```

---

## Task 7: Dim #3 — Client-side capture

**Files:**
- Create: `audit/dimensions/dim_03_client_capture.py`, `tests/audit/dimensions/test_dim_03_client_capture.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_03_client_capture import Dim03ClientCapture


def _ph(returning_counts):
    """HogQL called twice: first server count, then client count."""
    ph = MagicMock()
    ph.hogql.side_effect = [
        QueryResult(columns=["c"], rows=[[returning_counts[0]]]),
        QueryResult(columns=["c"], rows=[[returning_counts[1]]]),
    ]
    return ph


def test_client_capture_passes_at_75pct(betrayal_entry, window_7d):
    ctx = {"posthog": _ph([100, 75])}
    r = Dim03ClientCapture().run(betrayal_entry, window_7d, ctx)
    assert r.status is Status.PASS
    assert r.evidence["ratio"] == 0.75


def test_client_capture_fails_below_75pct(betrayal_entry, window_7d):
    ctx = {"posthog": _ph([100, 60])}
    r = Dim03ClientCapture().run(betrayal_entry, window_7d, ctx)
    assert r.status is Status.FAIL
    assert r.evidence["ratio"] == 0.60


def test_client_capture_na_when_no_server_events(betrayal_entry, window_7d):
    ctx = {"posthog": _ph([0, 0])}
    r = Dim03ClientCapture().run(betrayal_entry, window_7d, ctx)
    assert r.status is Status.NA
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_03_client_capture.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 3 — Client-side capture is ≥75% of server-side count."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


THRESHOLD = 0.75


class Dim03ClientCapture:
    dimension = 3

    def applies_to(self, entry: Entry) -> bool:
        return 3 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        events = "', '".join(entry.expected_events)

        server_q = f"""
            SELECT count() FROM events
            WHERE event IN ('{events}')
              AND properties.site = '{entry.site}'
              AND properties.tracking_layer = 'server'
              AND timestamp >= '{window.start.isoformat()}'
              AND timestamp <  '{window.end.isoformat()}'
        """
        server_count = ph.hogql(server_q).rows[0][0] if ph.hogql.__self__ else 0  # read below
        # Re-fetch cleanly (drop previous line's noise):
        server_count = ph.hogql(server_q).rows[0][0]

        if server_count == 0:
            return Result(
                dimension=3,
                status=Status.NA,
                evidence={"reason": "no server events in window"},
            )

        client_q = f"""
            SELECT count() FROM events
            WHERE event IN ('{events}')
              AND properties.site = '{entry.site}'
              AND (properties.tracking_layer = 'client' OR properties.sdk_version IS NOT NULL)
              AND timestamp >= '{window.start.isoformat()}'
              AND timestamp <  '{window.end.isoformat()}'
        """
        client_count = ph.hogql(client_q).rows[0][0]

        ratio = round(client_count / server_count, 4)
        status = Status.PASS if ratio >= THRESHOLD else Status.FAIL
        hint = None
        if status is Status.FAIL:
            hint = f"Client SDK coverage {ratio * 100:.1f}% < 75%. Check SDK load failures / ad blockers on {entry.site}."
        return Result(
            dimension=3,
            status=status,
            evidence={
                "server_count": server_count,
                "client_count": client_count,
                "ratio": ratio,
                "threshold": THRESHOLD,
            },
            remediation_hint=hint,
        )
```

**NOTE for implementer:** Remove the `ph.hogql.__self__` line from the test-writing draft — it's a drafting artifact. The implementation above already calls `ph.hogql(server_q)` cleanly.

Clean version (use this):

```python
server_count = ph.hogql(server_q).rows[0][0]

if server_count == 0:
    return Result(
        dimension=3,
        status=Status.NA,
        evidence={"reason": "no server events in window"},
    )

client_count = ph.hogql(client_q).rows[0][0]
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_03_client_capture.py -v
```

Expected: 3 passed. (Each test sets side_effect with 2 elements because both server and client queries run; the NA test sets both to 0 — only the server query gets consumed before we return NA.)

Adjust the NA test to use `side_effect=[QueryResult(columns=["c"], rows=[[0]])]` if you hit a StopIteration — only one query is consumed in the NA path.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_03_client_capture.py tests/audit/dimensions/test_dim_03_client_capture.py
git commit -m "audit: dim #3 Client-side capture"
```

---

## Task 8: Dim #4 — Payload completeness

**Files:**
- Create: `audit/dimensions/dim_04_payload_complete.py`, `tests/audit/dimensions/test_dim_04_payload_complete.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_04_payload_complete import Dim04PayloadComplete


def test_complete_passes_when_no_nulls(betrayal_entry, window_7d):
    ph = MagicMock()
    # One row per required property, (total, null_count). All 100/0.
    ph.hogql.return_value = QueryResult(
        columns=["property", "total", "null_count"],
        rows=[
            ["email", 100, 0], ["site", 100, 0], ["page", 100, 0],
            ["form_type", 100, 0], ["utm_source", 100, 0],
        ],
    )
    ctx = {"posthog": ph}
    r = Dim04PayloadComplete().run(betrayal_entry, window_7d, ctx)
    assert r.status is Status.PASS
    assert r.evidence["missing_fields"] == {}


def test_complete_fails_when_any_required_is_null(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["property", "total", "null_count"],
        rows=[
            ["email", 82, 0], ["site", 82, 0], ["page", 82, 0],
            ["form_type", 82, 0], ["utm_source", 82, 6],
        ],
    )
    ctx = {"posthog": ph}
    r = Dim04PayloadComplete().run(betrayal_entry, window_7d, ctx)
    assert r.status is Status.FAIL
    assert r.evidence["missing_fields"] == {"utm_source": {"null": 6, "total": 82}}


def test_complete_na_when_no_events(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["property", "total", "null_count"],
        rows=[["email", 0, 0], ["site", 0, 0]],
    )
    r = Dim04PayloadComplete().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.NA
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_04_payload_complete.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 4 — All required properties are present on all events."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim04PayloadComplete:
    dimension = 4

    def applies_to(self, entry: Entry) -> bool:
        return 4 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        # Build a UNION ALL query: one row per (event, required_property)
        unions = []
        for event, props in entry.required_properties.items():
            for prop in props:
                unions.append(f"""
                    SELECT '{prop}' AS property,
                           count() AS total,
                           countIf(JSONExtractString(properties, '{prop}') = '' OR properties.{prop} IS NULL) AS null_count
                    FROM events
                    WHERE event = '{event}'
                      AND properties.site = '{entry.site}'
                      AND timestamp >= '{window.start.isoformat()}'
                      AND timestamp <  '{window.end.isoformat()}'
                """)
        if not unions:
            return Result(dimension=4, status=Status.NA, evidence={"reason": "no required_properties defined"})

        query = " UNION ALL ".join(unions)
        rows = ph.hogql(query).rows

        total_events = sum(r[1] for r in rows)
        if total_events == 0:
            return Result(dimension=4, status=Status.NA, evidence={"reason": "no events in window"})

        missing = {r[0]: {"null": r[2], "total": r[1]} for r in rows if r[2] > 0}
        if missing:
            return Result(
                dimension=4,
                status=Status.FAIL,
                evidence={"missing_fields": missing},
                remediation_hint=(
                    f"Required properties missing: {list(missing)}. "
                    f"Check SDK property population and server-side hook for {entry.type}."
                ),
            )
        return Result(dimension=4, status=Status.PASS, evidence={"missing_fields": {}, "rows_checked": len(rows)})
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_04_payload_complete.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_04_payload_complete.py tests/audit/dimensions/test_dim_04_payload_complete.py
git commit -m "audit: dim #4 Payload completeness"
```

---

## Task 9: Dim #5 — Payload correctness (sampled)

**Files:**
- Create: `audit/dimensions/dim_05_payload_correct.py`, `tests/audit/dimensions/test_dim_05_payload_correct.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_05_payload_correct import Dim05PayloadCorrect


def test_correct_passes_when_sampled_rows_match_sot(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["order_id", "product_name", "order_total"],
        rows=[["40001", "Betrayal", "47.00"], ["40002", "Betrayal", "47.00"]],
    )
    sources = MagicMock()
    # Resolver returns {order_id: {product_name: ..., order_total: ...}}
    sources.resolve_sample.return_value = {
        "40001": {"product_name": "Betrayal", "order_total": "47.00"},
        "40002": {"product_name": "Betrayal", "order_total": "47.00"},
    }

    r = Dim05PayloadCorrect().run(betrayal_entry, window_7d, {"posthog": ph, "sources": sources})
    assert r.status is Status.PASS
    assert r.evidence["sampled"] == 2
    assert r.evidence["mismatches"] == []


def test_correct_fails_when_sampled_row_mismatches(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["order_id", "product_name", "order_total"],
        rows=[["40001", "WRONG_BUMP", "47.00"]],
    )
    sources = MagicMock()
    sources.resolve_sample.return_value = {
        "40001": {"product_name": "Betrayal", "order_total": "47.00"},
    }

    r = Dim05PayloadCorrect().run(betrayal_entry, window_7d, {"posthog": ph, "sources": sources})
    assert r.status is Status.FAIL
    assert len(r.evidence["mismatches"]) == 1
    assert r.evidence["mismatches"][0]["field"] == "product_name"
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_05_payload_correct.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 5 — Sampled events match source-of-truth exactly."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


SAMPLE_SIZE = 20


class Dim05PayloadCorrect:
    dimension = 5

    def applies_to(self, entry: Entry) -> bool:
        return 5 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        sources = ctx["sources"]

        # Sample by primary key — order_id for Purchases, email+timestamp for Leads.
        # For v1 scope: only Purchase entries (they have order_id). Others NA.
        if "Purchase" not in entry.expected_events:
            return Result(dimension=5, status=Status.NA, evidence={"reason": "no sampleable primary key for this event type"})

        query = f"""
            SELECT properties.order_id, properties.product_name, properties.order_total
            FROM events
            WHERE event = 'Purchase'
              AND properties.site = '{entry.site}'
              AND timestamp >= '{window.start.isoformat()}'
              AND timestamp <  '{window.end.isoformat()}'
            ORDER BY rand()
            LIMIT {SAMPLE_SIZE}
        """
        posthog_rows = ph.hogql(query).rows
        if not posthog_rows:
            return Result(dimension=5, status=Status.NA, evidence={"reason": "no events to sample"})

        ids = [r[0] for r in posthog_rows]
        sot = sources.resolve_sample(entry, ids)

        mismatches = []
        for order_id, product_name, order_total in posthog_rows:
            expected = sot.get(order_id)
            if expected is None:
                mismatches.append({"order_id": order_id, "field": "<missing in source>", "expected": None, "actual": product_name})
                continue
            if expected.get("product_name") != product_name:
                mismatches.append({"order_id": order_id, "field": "product_name", "expected": expected.get("product_name"), "actual": product_name})
            if str(expected.get("order_total")) != str(order_total):
                mismatches.append({"order_id": order_id, "field": "order_total", "expected": expected.get("order_total"), "actual": order_total})

        status = Status.PASS if not mismatches else Status.FAIL
        hint = None
        if mismatches:
            hint = f"{len(mismatches)} field mismatches in sample of {len(posthog_rows)}. Check order-item iteration in Purchase tracker snippet."
        return Result(
            dimension=5,
            status=status,
            evidence={"sampled": len(posthog_rows), "mismatches": mismatches},
            remediation_hint=hint,
        )
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_05_payload_correct.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_05_payload_correct.py tests/audit/dimensions/test_dim_05_payload_correct.py
git commit -m "audit: dim #5 Payload correctness (sampled)"
```

---

## Task 10: Dim #6 — Attribution chain

**Files:**
- Create: `audit/dimensions/dim_06_attribution.py`, `tests/audit/dimensions/test_dim_06_attribution.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_06_attribution import Dim06Attribution


def test_attribution_passes_at_90pct(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["total", "with_touchpoint"],
        rows=[[100, 90]],
    )
    r = Dim06Attribution().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.PASS
    assert r.evidence["ratio"] == 0.90


def test_attribution_fails_below_90pct(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["total", "with_touchpoint"],
        rows=[[100, 73]],
    )
    r = Dim06Attribution().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.FAIL
    assert r.evidence["ratio"] == 0.73


def test_attribution_na_when_no_purchases_or_leads(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["total", "with_touchpoint"], rows=[[0, 0]])
    r = Dim06Attribution().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.NA
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_06_attribution.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 6 — Lead/Purchase events have a non-direct Touchpoint within 90d."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


THRESHOLD = 0.90
LOOKBACK_DAYS = 90


class Dim06Attribution:
    dimension = 6

    def applies_to(self, entry: Entry) -> bool:
        return 6 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        target_events = [e for e in entry.expected_events if e in ("Lead", "Purchase")]
        if not target_events:
            return Result(dimension=6, status=Status.NA, evidence={"reason": "attribution applies to Lead/Purchase only"})

        events_csv = "', '".join(target_events)
        query = f"""
            WITH conversions AS (
                SELECT distinct_id, timestamp AS ts
                FROM events
                WHERE event IN ('{events_csv}')
                  AND properties.site = '{entry.site}'
                  AND timestamp >= '{window.start.isoformat()}'
                  AND timestamp <  '{window.end.isoformat()}'
            )
            SELECT count() AS total,
                   countIf(EXISTS (
                       SELECT 1 FROM events t
                       WHERE t.distinct_id = conversions.distinct_id
                         AND t.event = 'Touchpoint'
                         AND t.properties.touchpoint_source != 'direct'
                         AND t.timestamp >= conversions.ts - INTERVAL {LOOKBACK_DAYS} DAY
                         AND t.timestamp <= conversions.ts
                   )) AS with_touchpoint
            FROM conversions
        """
        row = ph.hogql(query).rows[0]
        total, with_touchpoint = row[0], row[1]
        if total == 0:
            return Result(dimension=6, status=Status.NA, evidence={"reason": "no conversions in window"})

        ratio = round(with_touchpoint / total, 4)
        status = Status.PASS if ratio >= THRESHOLD else Status.FAIL
        hint = None
        if status is Status.FAIL:
            hint = f"Only {ratio * 100:.1f}% of conversions have a non-direct touchpoint ({total - with_touchpoint} attributed to direct). Check UTM cookie persistence and referrer parsing."
        return Result(
            dimension=6,
            status=status,
            evidence={"total": total, "with_touchpoint": with_touchpoint, "ratio": ratio, "threshold": THRESHOLD},
            remediation_hint=hint,
        )
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_06_attribution.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_06_attribution.py tests/audit/dimensions/test_dim_06_attribution.py
git commit -m "audit: dim #6 Attribution chain"
```

---

## Task 11: Dim #7 — Identity stitching

**Files:**
- Create: `audit/dimensions/dim_07_identity.py`, `tests/audit/dimensions/test_dim_07_identity.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_07_identity import Dim07Identity


def test_identity_passes_when_no_fragments(betrayal_entry, window_7d):
    ph = MagicMock()
    # Query returns: total distinct emails with events, emails with >1 distinct_id
    ph.hogql.return_value = QueryResult(
        columns=["total_emails", "fragmented_emails"],
        rows=[[42, 0]],
    )
    r = Dim07Identity().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.PASS


def test_identity_fails_when_fragments_exist(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["total_emails", "fragmented_emails"], rows=[[42, 3]])
    r = Dim07Identity().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.FAIL
    assert r.evidence["fragmented_emails"] == 3


def test_identity_na_when_no_emails(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["total_emails", "fragmented_emails"], rows=[[0, 0]])
    r = Dim07Identity().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.NA
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_07_identity.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 7 — No email is fragmented across multiple distinct_ids."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim07Identity:
    dimension = 7

    def applies_to(self, entry: Entry) -> bool:
        return 7 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        query = f"""
            SELECT
                count(DISTINCT properties.email) AS total_emails,
                countIf(distinct_id_count > 1) AS fragmented_emails
            FROM (
                SELECT properties.email AS email,
                       count(DISTINCT distinct_id) AS distinct_id_count
                FROM events
                WHERE properties.email != ''
                  AND properties.site = '{entry.site}'
                  AND timestamp >= '{window.start.isoformat()}'
                  AND timestamp <  '{window.end.isoformat()}'
                GROUP BY email
            )
        """
        row = ph.hogql(query).rows[0]
        total, fragmented = row[0], row[1]
        if total == 0:
            return Result(dimension=7, status=Status.NA, evidence={"reason": "no emails in window"})

        status = Status.PASS if fragmented == 0 else Status.FAIL
        hint = None
        if status is Status.FAIL:
            hint = f"{fragmented} email(s) map to multiple distinct_ids — $create_alias not firing. Check cross-domain identity bootstrap."
        return Result(
            dimension=7,
            status=status,
            evidence={"total_emails": total, "fragmented_emails": fragmented},
            remediation_hint=hint,
        )
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_07_identity.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_07_identity.py tests/audit/dimensions/test_dim_07_identity.py
git commit -m "audit: dim #7 Identity stitching"
```

---

## Task 12: Dim #8 — Dedup

**Files:**
- Create: `audit/dimensions/dim_08_dedup.py`, `tests/audit/dimensions/test_dim_08_dedup.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_08_dedup import Dim08Dedup


def test_dedup_passes_when_no_duplicate_order_ids(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["order_id", "fires"], rows=[])
    r = Dim08Dedup().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.PASS
    assert r.evidence["duplicate_count"] == 0


def test_dedup_fails_when_duplicates_exist(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["order_id", "fires"],
        rows=[["40573780", 6], ["40581234", 2]],
    )
    r = Dim08Dedup().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.FAIL
    assert r.evidence["duplicate_count"] == 2
    assert "40573780" in [d["order_id"] for d in r.evidence["duplicates"]]
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_08_dedup.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 8 — No duplicate Purchase events per order_id."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim08Dedup:
    dimension = 8

    def applies_to(self, entry: Entry) -> bool:
        return 8 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        if "Purchase" not in entry.expected_events:
            return Result(dimension=8, status=Status.NA, evidence={"reason": "dedup applies to Purchase events"})

        query = f"""
            SELECT properties.order_id AS order_id, count() AS fires
            FROM events
            WHERE event = 'Purchase'
              AND properties.site = '{entry.site}'
              AND properties.order_id != ''
              AND timestamp >= '{window.start.isoformat()}'
              AND timestamp <  '{window.end.isoformat()}'
            GROUP BY order_id
            HAVING fires > 1
            ORDER BY fires DESC
        """
        rows = ph.hogql(query).rows
        dupes = [{"order_id": r[0], "fires": r[1]} for r in rows]
        status = Status.PASS if not dupes else Status.FAIL
        hint = None
        if dupes:
            hint = f"{len(dupes)} duplicate order_id(s). Check cookie dedup (_rli_oid_/_tr_oid_) + server-side check-order endpoint."
        return Result(
            dimension=8,
            status=status,
            evidence={"duplicate_count": len(dupes), "duplicates": dupes[:10]},
            remediation_hint=hint,
        )
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_08_dedup.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_08_dedup.py tests/audit/dimensions/test_dim_08_dedup.py
git commit -m "audit: dim #8 Dedup"
```

---

## Task 13: Dim #9 — Reconciliation (wraps existing script)

**Files:**
- Create: `audit/dimensions/dim_09_reconciliation.py`, `tests/audit/dimensions/test_dim_09_reconciliation.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.dimensions.dim_09_reconciliation import Dim09Reconciliation


def test_reconciliation_passes_at_99pct(betrayal_entry, window_7d):
    reconciler = MagicMock()
    reconciler.run.return_value = {"matched": 100, "expected": 101, "missing": [101]}
    r = Dim09Reconciliation().run(betrayal_entry, window_7d, {"reconciler": reconciler})
    assert r.status is Status.PASS
    assert r.evidence["match_rate"] == 100 / 101


def test_reconciliation_fails_below_99pct(betrayal_entry, window_7d):
    reconciler = MagicMock()
    reconciler.run.return_value = {"matched": 90, "expected": 100, "missing": list(range(10))}
    r = Dim09Reconciliation().run(betrayal_entry, window_7d, {"reconciler": reconciler})
    assert r.status is Status.FAIL
    assert r.evidence["match_rate"] == 0.9


def test_reconciliation_na_when_expected_is_zero(betrayal_entry, window_7d):
    reconciler = MagicMock()
    reconciler.run.return_value = {"matched": 0, "expected": 0, "missing": []}
    r = Dim09Reconciliation().run(betrayal_entry, window_7d, {"reconciler": reconciler})
    assert r.status is Status.NA
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_09_reconciliation.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 9 — Reconcile PostHog counts against source-of-truth at ≥99%.

Wraps monitoring/reconciliation.py functionality via a thin Reconciler
protocol so the dim is testable in isolation. Real wiring happens in
audit/cli.py where the context is assembled.
"""

from __future__ import annotations

from typing import Any, Protocol

from audit.core.models import Entry, Result, Status, Window


THRESHOLD = 0.99


class Reconciler(Protocol):
    def run(self, entry: Entry, window: Window) -> dict: ...


class Dim09Reconciliation:
    dimension = 9

    def applies_to(self, entry: Entry) -> bool:
        return 9 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        reconciler = ctx["reconciler"]
        out = reconciler.run(entry, window)
        expected = out.get("expected", 0)
        matched = out.get("matched", 0)
        if expected == 0:
            return Result(dimension=9, status=Status.NA, evidence={"reason": "no expected records in window"})

        rate = matched / expected
        status = Status.PASS if rate >= THRESHOLD else Status.FAIL
        missing = out.get("missing", [])
        hint = None
        if status is Status.FAIL:
            hint = f"{len(missing)} record(s) in source-of-truth missing from PostHog. See monitoring/reconciliation.py output."
        return Result(
            dimension=9,
            status=status,
            evidence={"matched": matched, "expected": expected, "match_rate": rate, "missing_sample": missing[:10]},
            remediation_hint=hint,
        )
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_09_reconciliation.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_09_reconciliation.py tests/audit/dimensions/test_dim_09_reconciliation.py
git commit -m "audit: dim #9 Reconciliation (wraps monitoring/reconciliation.py)"
```

---

## Task 14: Dim #10 — Resilience (Playwright synthetic paths)

**Files:**
- Create: `audit/dimensions/dim_10_resilience.py`, `tests/audit/dimensions/test_dim_10_resilience.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.dimensions.dim_10_resilience import Dim10Resilience


def test_resilience_passes_when_all_paths_pass(betrayal_entry, window_7d):
    runner = MagicMock()
    runner.run_path.side_effect = [
        {"path": "cache_hit", "passed": True},
        {"path": "consent_denied", "passed": True},
        {"path": "mobile_ua", "passed": True},
        {"path": "cross_domain_hop", "passed": True},
    ]
    r = Dim10Resilience().run(betrayal_entry, window_7d, {"playwright_runner": runner})
    assert r.status is Status.PASS
    assert len(r.evidence["paths"]) == 4


def test_resilience_fails_when_any_path_fails(betrayal_entry, window_7d):
    runner = MagicMock()
    runner.run_path.side_effect = [
        {"path": "cache_hit", "passed": True},
        {"path": "consent_denied", "passed": False, "error": "event did not fire"},
        {"path": "mobile_ua", "passed": True},
        {"path": "cross_domain_hop", "passed": True},
    ]
    r = Dim10Resilience().run(betrayal_entry, window_7d, {"playwright_runner": runner})
    assert r.status is Status.FAIL
    failed_paths = [p["path"] for p in r.evidence["paths"] if not p["passed"]]
    assert failed_paths == ["consent_denied"]
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_10_resilience.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 10 — Tracking survives cache, consent, mobile, cross-domain.

Uses a Playwright runner injected via ctx["playwright_runner"]. Concrete
runner lives in audit/core/playwright_runner.py and is wired in Task 26
(integration). Tests here use a mock.
"""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


SYNTHETIC_PATHS = ["cache_hit", "consent_denied", "mobile_ua", "cross_domain_hop"]


class Dim10Resilience:
    dimension = 10

    def applies_to(self, entry: Entry) -> bool:
        return 10 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        runner = ctx["playwright_runner"]
        results = [runner.run_path(entry, path) for path in SYNTHETIC_PATHS]
        failed = [r for r in results if not r.get("passed")]
        status = Status.PASS if not failed else Status.FAIL
        hint = None
        if failed:
            hint = f"Failed synthetic paths: {[r['path'] for r in failed]}. Re-run tests/audit/integration locally to reproduce."
        return Result(
            dimension=10,
            status=status,
            evidence={"paths": results},
            remediation_hint=hint,
        )
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_10_resilience.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_10_resilience.py tests/audit/dimensions/test_dim_10_resilience.py
git commit -m "audit: dim #10 Resilience (Playwright synthetic paths via injected runner)"
```

---

## Task 15: Dim #11 — Freshness

**Files:**
- Create: `audit/dimensions/dim_11_freshness.py`, `tests/audit/dimensions/test_dim_11_freshness.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_11_freshness import Dim11Freshness


def test_freshness_passes_when_latency_under_threshold(betrayal_entry, window_7d):
    ph = MagicMock()
    # p95_client_sec, p95_webhook_sec
    ph.hogql.return_value = QueryResult(columns=["p95_client", "p95_webhook"], rows=[[3.1, 45.0]])
    r = Dim11Freshness().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.PASS


def test_freshness_fails_when_client_over_5s(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["p95_client", "p95_webhook"], rows=[[8.4, 20.0]])
    r = Dim11Freshness().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.FAIL
    assert r.evidence["p95_client_sec"] == 8.4


def test_freshness_fails_when_webhook_over_60s(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["p95_client", "p95_webhook"], rows=[[3.0, 90.0]])
    r = Dim11Freshness().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.FAIL
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_11_freshness.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 11 — Event latency p95 within thresholds (client <5s, webhook <60s)."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


CLIENT_P95_SEC = 5.0
WEBHOOK_P95_SEC = 60.0


class Dim11Freshness:
    dimension = 11

    def applies_to(self, entry: Entry) -> bool:
        return 11 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        events = "', '".join(entry.expected_events)
        # Uses _timestamp (ingestion time) - timestamp (event time) in seconds.
        # PostHog schema exposes _timestamp via HogQL.
        query = f"""
            SELECT
                quantile(0.95)(dateDiff('second', timestamp, _timestamp)) FILTER (WHERE properties.tracking_layer = 'client') AS p95_client,
                quantile(0.95)(dateDiff('second', timestamp, _timestamp)) FILTER (WHERE properties.source_type = 'webhook') AS p95_webhook
            FROM events
            WHERE event IN ('{events}')
              AND properties.site = '{entry.site}'
              AND timestamp >= '{window.start.isoformat()}'
              AND timestamp <  '{window.end.isoformat()}'
        """
        row = ph.hogql(query).rows[0]
        p95_client, p95_webhook = row[0], row[1]
        p95_client = float(p95_client) if p95_client is not None else 0.0
        p95_webhook = float(p95_webhook) if p95_webhook is not None else 0.0

        client_ok = p95_client <= CLIENT_P95_SEC
        webhook_ok = p95_webhook <= WEBHOOK_P95_SEC

        if client_ok and webhook_ok:
            status = Status.PASS
            hint = None
        else:
            status = Status.FAIL
            reasons = []
            if not client_ok:
                reasons.append(f"client p95 {p95_client:.1f}s > {CLIENT_P95_SEC}s")
            if not webhook_ok:
                reasons.append(f"webhook p95 {p95_webhook:.1f}s > {WEBHOOK_P95_SEC}s")
            hint = "; ".join(reasons)

        return Result(
            dimension=11,
            status=status,
            evidence={
                "p95_client_sec": p95_client,
                "p95_webhook_sec": p95_webhook,
                "threshold_client_sec": CLIENT_P95_SEC,
                "threshold_webhook_sec": WEBHOOK_P95_SEC,
            },
            remediation_hint=hint,
        )
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_11_freshness.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_11_freshness.py tests/audit/dimensions/test_dim_11_freshness.py
git commit -m "audit: dim #11 Freshness (p95 latency)"
```

---

## Task 16: Dim #12 — Monitoring/alerting fire-drill log

**Files:**
- Create: `audit/fire_drill_log.json`, `audit/dimensions/dim_12_monitoring.py`, `tests/audit/dimensions/test_dim_12_monitoring.py`

- [ ] **Step 1: Write failing test**

```python
import json
from datetime import datetime, timedelta, timezone
from pathlib import Path

from audit.core.models import Status
from audit.dimensions.dim_12_monitoring import Dim12Monitoring


def _write_log(path: Path, days_ago: int):
    now = datetime.now(timezone.utc)
    path.write_text(json.dumps([{
        "drill_type": "form_hook_disable",
        "performed_at": (now - timedelta(days=days_ago)).isoformat(),
        "alert_fired_within_sla": True,
    }]))


def test_monitoring_passes_when_drill_within_90d(tmp_path, betrayal_entry, window_7d):
    log = tmp_path / "fire_drill_log.json"
    _write_log(log, days_ago=30)
    r = Dim12Monitoring().run(betrayal_entry, window_7d, {"fire_drill_log_path": log})
    assert r.status is Status.PASS


def test_monitoring_fails_when_drill_overdue(tmp_path, betrayal_entry, window_7d):
    log = tmp_path / "fire_drill_log.json"
    _write_log(log, days_ago=120)
    r = Dim12Monitoring().run(betrayal_entry, window_7d, {"fire_drill_log_path": log})
    assert r.status is Status.FAIL
    assert "overdue" in (r.remediation_hint or "").lower()


def test_monitoring_fails_when_log_empty(tmp_path, betrayal_entry, window_7d):
    log = tmp_path / "fire_drill_log.json"
    log.write_text("[]")
    r = Dim12Monitoring().run(betrayal_entry, window_7d, {"fire_drill_log_path": log})
    assert r.status is Status.FAIL


def test_monitoring_fails_when_last_drill_alert_did_not_fire(tmp_path, betrayal_entry, window_7d):
    log = tmp_path / "fire_drill_log.json"
    now = datetime.now(timezone.utc)
    log.write_text(json.dumps([{
        "drill_type": "form_hook_disable",
        "performed_at": (now - timedelta(days=10)).isoformat(),
        "alert_fired_within_sla": False,
    }]))
    r = Dim12Monitoring().run(betrayal_entry, window_7d, {"fire_drill_log_path": log})
    assert r.status is Status.FAIL
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_12_monitoring.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 12 — Monitoring/alerting verified by quarterly fire drill."""

from __future__ import annotations

import json
from datetime import datetime, timedelta, timezone
from pathlib import Path
from typing import Any

from audit.core.models import Entry, Result, Status, Window


SLA_DAYS = 90


class Dim12Monitoring:
    dimension = 12

    def applies_to(self, entry: Entry) -> bool:
        return 12 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        log_path = Path(ctx["fire_drill_log_path"])
        if not log_path.exists():
            return Result(dimension=12, status=Status.FAIL,
                          evidence={"reason": "fire-drill log does not exist"},
                          remediation_hint="Create audit/fire_drill_log.json and record quarterly alert-test drills.")

        entries = json.loads(log_path.read_text() or "[]")
        if not entries:
            return Result(dimension=12, status=Status.FAIL,
                          evidence={"reason": "no fire drills recorded"},
                          remediation_hint="Run a monitoring fire drill and record it in audit/fire_drill_log.json.")

        entries.sort(key=lambda e: e["performed_at"], reverse=True)
        latest = entries[0]
        performed = datetime.fromisoformat(latest["performed_at"])
        age_days = (datetime.now(timezone.utc) - performed).days

        if age_days > SLA_DAYS:
            return Result(dimension=12, status=Status.FAIL,
                          evidence={"last_drill_days_ago": age_days, "sla_days": SLA_DAYS},
                          remediation_hint=f"Fire drill overdue ({age_days}d > {SLA_DAYS}d). Run a new drill.")

        if not latest.get("alert_fired_within_sla", False):
            return Result(dimension=12, status=Status.FAIL,
                          evidence={"last_drill": latest},
                          remediation_hint="Latest drill did not trigger alerts. Monitoring is broken.")

        return Result(dimension=12, status=Status.PASS,
                      evidence={"last_drill_days_ago": age_days, "last_drill": latest})
```

Create `audit/fire_drill_log.json` with an empty array so the file exists:

```json
[]
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_12_monitoring.py -v
```

Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/fire_drill_log.json audit/dimensions/dim_12_monitoring.py tests/audit/dimensions/test_dim_12_monitoring.py
git commit -m "audit: dim #12 Monitoring/alerting (fire-drill log)"
```

---

## Task 17: Dim #13 — Architectural fit (repo-wide)

**Files:**
- Create: `audit/architecture_review.md`, `audit/dimensions/dim_13_architectural_fit.py`, `tests/audit/dimensions/test_dim_13_architectural_fit.py`

- [ ] **Step 1: Write failing test**

```python
from pathlib import Path

from audit.core.models import Status
from audit.dimensions.dim_13_architectural_fit import Dim13ArchitecturalFit


def _write_md(path: Path, subsystems: dict[str, str]):
    # subsystems is {name: decision_line or ""}
    lines = []
    for name, decision in subsystems.items():
        lines.append(f"## {name}")
        if decision:
            lines.append(f"Decision: {decision}")
        lines.append("")
    path.write_text("\n".join(lines))


def test_arch_fit_passes_when_all_subsystems_have_decision(tmp_path, betrayal_entry, window_7d):
    p = tmp_path / "architecture_review.md"
    _write_md(p, {"SDK": "Keep custom SDK for now.", "Server tracking": "Keep PHP hook pattern."})
    r = Dim13ArchitecturalFit().run(betrayal_entry, window_7d, {"architecture_review_path": p})
    assert r.status is Status.PASS


def test_arch_fit_fails_when_any_subsystem_missing_decision(tmp_path, betrayal_entry, window_7d):
    p = tmp_path / "architecture_review.md"
    _write_md(p, {"SDK": "Keep.", "Server tracking": ""})
    r = Dim13ArchitecturalFit().run(betrayal_entry, window_7d, {"architecture_review_path": p})
    assert r.status is Status.FAIL
    assert "Server tracking" in r.evidence["missing_decisions"]


def test_arch_fit_fails_when_file_missing(tmp_path, betrayal_entry, window_7d):
    r = Dim13ArchitecturalFit().run(betrayal_entry, window_7d, {"architecture_review_path": tmp_path / "nope.md"})
    assert r.status is Status.FAIL
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_13_architectural_fit.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 13 — Every subsystem has a documented architectural decision."""

from __future__ import annotations

import re
from pathlib import Path
from typing import Any

from audit.core.models import Entry, Result, Status, Window


H2_RE = re.compile(r"^##\s+(.+?)\s*$", re.MULTILINE)


class Dim13ArchitecturalFit:
    dimension = 13

    def applies_to(self, entry: Entry) -> bool:
        # Repo-wide — applies to every entry; result is identical for all.
        return 13 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        path = Path(ctx["architecture_review_path"])
        if not path.exists():
            return Result(dimension=13, status=Status.FAIL,
                          evidence={"reason": "architecture_review.md not found"},
                          remediation_hint="Create audit/architecture_review.md with one H2 per subsystem.")

        text = path.read_text()
        missing = []
        sections = list(H2_RE.finditer(text))
        for i, match in enumerate(sections):
            start = match.end()
            end = sections[i + 1].start() if i + 1 < len(sections) else len(text)
            body = text[start:end]
            if not re.search(r"(?im)^decision:\s*\S", body):
                missing.append(match.group(1))

        if missing:
            return Result(dimension=13, status=Status.FAIL,
                          evidence={"missing_decisions": missing},
                          remediation_hint=f"Add 'Decision: <text>' line to subsystems: {missing}")
        if not sections:
            return Result(dimension=13, status=Status.FAIL,
                          evidence={"reason": "no subsystems documented"},
                          remediation_hint="Populate audit/architecture_review.md.")
        return Result(dimension=13, status=Status.PASS,
                      evidence={"subsystems_reviewed": [s.group(1) for s in sections]})
```

Create `audit/architecture_review.md` with starter subsystems (all flagged as needing a decision in the first real audit):

```markdown
# Tracking Architecture Review

One H2 per subsystem. Each must have a `Decision:` line stating whether
the current approach stays, is refactored, or is replaced by a native
platform feature. Reviewed during Spec 2 runs.

## Client SDK

Location: `sdk/`
Current approach: Custom ~25KB SDK builds to WP snippet; wraps PostHog JS.
Decision:

## Server-side Form Tracking (PHP)

Location: WP `mu-plugins/rli-server-tracking.php`
Current approach: Elementor/Fluent/EverWebinar hooks → PostHog capture API.
Decision:

## ThriveCart Webhook

Location: WP `mu-plugins/rli-thrivecart-webhook.php`
Current approach: Webhook → REST → PostHog capture.
Decision:

## PostHog Proxy (Cloudflare Worker)

Location: `cloudflare-worker/`
Current approach: Worker proxies events under same-domain path.
Decision:

## Cross-Domain Identity

Current approach: SDK attaches ph_user_id to outbound links; $create_alias on arrival.
Decision:

## Attribution Cookies

Current approach: Server-set cookies `_rli_utm_source`, `_rli_gclid`, etc.
Decision:

## Reconciliation

Location: `monitoring/reconciliation.py`
Current approach: Hourly GitHub Action runs Python script, alerts via Slack.
Decision:
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_13_architectural_fit.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/architecture_review.md audit/dimensions/dim_13_architectural_fit.py tests/audit/dimensions/test_dim_13_architectural_fit.py
git commit -m "audit: dim #13 Architectural fit + starter architecture_review.md"
```

---

## Task 18: Dim #14 — Simplicity / YAGNI

**Files:**
- Create: `audit/dimensions/dim_14_simplicity.py`, `tests/audit/dimensions/test_dim_14_simplicity.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.dimensions.dim_14_simplicity import Dim14Simplicity


def test_simplicity_passes_when_all_properties_are_read(betrayal_entry, window_7d):
    usage = MagicMock()
    # Returns set of (event, property) pairs that are referenced by insights/dashboards.
    usage.read_properties.return_value = {
        ("FormSubmit", "email"), ("FormSubmit", "site"), ("FormSubmit", "page"),
        ("FormSubmit", "form_type"), ("FormSubmit", "utm_source"),
        ("Lead", "email"), ("Lead", "site"), ("Lead", "page"), ("Lead", "utm_source"),
    }
    r = Dim14Simplicity().run(betrayal_entry, window_7d, {"posthog_usage": usage})
    assert r.status is Status.PASS


def test_simplicity_fails_when_captured_but_unread(betrayal_entry, window_7d):
    usage = MagicMock()
    usage.read_properties.return_value = {
        ("FormSubmit", "email"), ("FormSubmit", "site"),
        ("Lead", "email"), ("Lead", "site"), ("Lead", "page"), ("Lead", "utm_source"),
    }
    r = Dim14Simplicity().run(betrayal_entry, window_7d, {"posthog_usage": usage})
    assert r.status is Status.FAIL
    unread = {tuple(u) for u in r.evidence["unread_properties"]}
    assert ("FormSubmit", "page") in unread
    assert ("FormSubmit", "form_type") in unread
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_14_simplicity.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 14 — Every required property is read by at least one insight/dashboard."""

from __future__ import annotations

from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim14Simplicity:
    dimension = 14

    def applies_to(self, entry: Entry) -> bool:
        return 14 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        usage = ctx["posthog_usage"]
        read_pairs = usage.read_properties()

        unread = []
        for event, props in entry.required_properties.items():
            for p in props:
                if (event, p) not in read_pairs:
                    unread.append((event, p))

        if unread:
            return Result(
                dimension=14,
                status=Status.FAIL,
                evidence={"unread_properties": [list(u) for u in unread]},
                remediation_hint=(
                    f"{len(unread)} captured properties are not referenced by any insight. "
                    "Either build a report that consumes them or drop them from required_properties."
                ),
            )
        return Result(dimension=14, status=Status.PASS, evidence={"unread_properties": []})
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_14_simplicity.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_14_simplicity.py tests/audit/dimensions/test_dim_14_simplicity.py
git commit -m "audit: dim #14 Simplicity/YAGNI"
```

---

## Task 19: Dim #15 — Consistency (taxonomy across sites)

**Files:**
- Create: `audit/dimensions/dim_15_consistency.py`, `tests/audit/dimensions/test_dim_15_consistency.py`

- [ ] **Step 1: Write failing test**

```python
from unittest.mock import MagicMock

from audit.core.models import Status
from audit.core.posthog_client import QueryResult
from audit.dimensions.dim_15_consistency import Dim15Consistency


def test_consistency_passes_when_taxonomy_identical(betrayal_entry, window_7d):
    ph = MagicMock()
    # For each event, list of (site, property_set)
    ph.hogql.return_value = QueryResult(
        columns=["event", "site", "prop"],
        rows=[
            ["FormSubmit", "terryreal.com", "email"],
            ["FormSubmit", "terryreal.com", "form_type"],
            ["FormSubmit", "relationallife.com", "email"],
            ["FormSubmit", "relationallife.com", "form_type"],
        ],
    )
    r = Dim15Consistency().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.PASS


def test_consistency_fails_when_property_missing_on_one_site(betrayal_entry, window_7d):
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(
        columns=["event", "site", "prop"],
        rows=[
            ["FormSubmit", "terryreal.com", "email"],
            ["FormSubmit", "terryreal.com", "form_type"],
            ["FormSubmit", "relationallife.com", "email"],
            # missing form_type on rli
        ],
    )
    r = Dim15Consistency().run(betrayal_entry, window_7d, {"posthog": ph})
    assert r.status is Status.FAIL
    assert "form_type" in r.evidence["inconsistent_properties"]
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_15_consistency.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 15 — Same event has the same property set on every site that fires it."""

from __future__ import annotations

from collections import defaultdict
from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim15Consistency:
    dimension = 15

    def applies_to(self, entry: Entry) -> bool:
        return 15 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        ph = ctx["posthog"]
        events_csv = "', '".join(entry.expected_events)
        query = f"""
            SELECT event, properties.site AS site, key AS prop
            FROM events
            ARRAY JOIN JSONExtractKeys(properties) AS key
            WHERE event IN ('{events_csv}')
              AND timestamp >= '{window.start.isoformat()}'
              AND timestamp <  '{window.end.isoformat()}'
            GROUP BY event, site, prop
        """
        rows = ph.hogql(query).rows

        # For each (event, property), which sites have it?
        by_pair: dict[tuple[str, str], set[str]] = defaultdict(set)
        sites_by_event: dict[str, set[str]] = defaultdict(set)
        for event, site, prop in rows:
            by_pair[(event, prop)].add(site)
            sites_by_event[event].add(site)

        inconsistent: list[str] = []
        for (event, prop), sites_with in by_pair.items():
            expected_sites = sites_by_event[event]
            if sites_with != expected_sites and len(expected_sites) > 1:
                missing_on = sorted(expected_sites - sites_with)
                inconsistent.append(f"{event}.{prop} (missing on: {missing_on})")

        if inconsistent:
            return Result(
                dimension=15,
                status=Status.FAIL,
                evidence={"inconsistent_properties": inconsistent},
                remediation_hint="Align property names across sites — same event should emit same property set.",
            )
        return Result(dimension=15, status=Status.PASS, evidence={"inconsistent_properties": []})
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_15_consistency.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_15_consistency.py tests/audit/dimensions/test_dim_15_consistency.py
git commit -m "audit: dim #15 Consistency (taxonomy across sites)"
```

---

## Task 20: Dim #16 — Code quality (dead files)

**Files:**
- Create: `audit/dimensions/dim_16_code_quality.py`, `tests/audit/dimensions/test_dim_16_code_quality.py`

- [ ] **Step 1: Write failing test**

```python
from pathlib import Path

from audit.core.models import Status
from audit.dimensions.dim_16_code_quality import Dim16CodeQuality


def test_code_quality_passes_when_all_deploy_scripts_referenced(tmp_path, betrayal_entry, window_7d):
    (tmp_path / "deploy.sh").write_text("bash deploy-fix-thing.php\nbash deploy-other.php\n")
    (tmp_path / "deploy-fix-thing.php").write_text("// ...")
    (tmp_path / "deploy-other.php").write_text("// ...")

    r = Dim16CodeQuality().run(betrayal_entry, window_7d, {"repo_root": tmp_path})
    assert r.status is Status.PASS


def test_code_quality_fails_when_deploy_script_unreferenced(tmp_path, betrayal_entry, window_7d):
    (tmp_path / "deploy.sh").write_text("bash deploy-fix-thing.php\n")
    (tmp_path / "deploy-fix-thing.php").write_text("// ...")
    (tmp_path / "deploy-orphaned.php").write_text("// ...")

    r = Dim16CodeQuality().run(betrayal_entry, window_7d, {"repo_root": tmp_path})
    assert r.status is Status.FAIL
    assert "deploy-orphaned.php" in r.evidence["dead_files"]


def test_code_quality_passes_when_no_deploy_files(tmp_path, betrayal_entry, window_7d):
    r = Dim16CodeQuality().run(betrayal_entry, window_7d, {"repo_root": tmp_path})
    assert r.status is Status.PASS
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_16_code_quality.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 16 — No orphaned deploy-*.php scripts unreferenced by deploy.sh."""

from __future__ import annotations

from pathlib import Path
from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim16CodeQuality:
    dimension = 16

    def applies_to(self, entry: Entry) -> bool:
        return 16 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        root = Path(ctx["repo_root"])
        deploy_sh = root / "deploy.sh"
        referenced_text = deploy_sh.read_text() if deploy_sh.exists() else ""

        deploy_scripts = list(root.glob("deploy-*.php"))
        dead = [p.name for p in deploy_scripts if p.name not in referenced_text]

        if dead:
            return Result(
                dimension=16,
                status=Status.FAIL,
                evidence={"dead_files": dead, "total_deploy_scripts": len(deploy_scripts)},
                remediation_hint=(
                    f"{len(dead)} deploy scripts not referenced by deploy.sh. "
                    "Either reference them or delete."
                ),
            )
        return Result(dimension=16, status=Status.PASS,
                      evidence={"total_deploy_scripts": len(deploy_scripts)})
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_16_code_quality.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_16_code_quality.py tests/audit/dimensions/test_dim_16_code_quality.py
git commit -m "audit: dim #16 Code quality (orphaned deploy scripts)"
```

---

## Task 21: Dim #17 — Testability (smoke test presence)

**Files:**
- Create: `audit/dimensions/dim_17_testability.py`, `tests/audit/dimensions/test_dim_17_testability.py`

- [ ] **Step 1: Write failing test**

```python
from pathlib import Path

from audit.core.models import Status
from audit.dimensions.dim_17_testability import Dim17Testability


def test_testability_passes_when_smoke_test_exists(tmp_path, betrayal_entry, window_7d):
    tests_dir = tmp_path / "tests"
    tests_dir.mkdir()
    (tests_dir / "smoke-elementor-form.spec.ts").write_text("// test")
    r = Dim17Testability().run(betrayal_entry, window_7d, {"tests_root": tests_dir})
    assert r.status is Status.PASS


def test_testability_fails_when_no_smoke_test_for_type(tmp_path, betrayal_entry, window_7d):
    tests_dir = tmp_path / "tests"
    tests_dir.mkdir()
    (tests_dir / "smoke-other.spec.ts").write_text("// test")
    r = Dim17Testability().run(betrayal_entry, window_7d, {"tests_root": tests_dir})
    assert r.status is Status.FAIL
    assert "elementor_form" in (r.remediation_hint or "")


def test_testability_fails_when_tests_root_missing(tmp_path, betrayal_entry, window_7d):
    r = Dim17Testability().run(betrayal_entry, window_7d, {"tests_root": tmp_path / "nope"})
    assert r.status is Status.FAIL
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_17_testability.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 17 — A smoke/Playwright test exists for this entry's type."""

from __future__ import annotations

from pathlib import Path
from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim17Testability:
    dimension = 17

    def applies_to(self, entry: Entry) -> bool:
        return 17 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        tests_root = Path(ctx["tests_root"])
        if not tests_root.exists():
            return Result(dimension=17, status=Status.FAIL,
                          evidence={"reason": "tests directory missing"},
                          remediation_hint="Create tests/ with smoke tests per entry type.")

        # Hyphenated form of the type, e.g. 'elementor-form'
        type_slug = entry.type.replace("_", "-")
        matches = list(tests_root.rglob(f"smoke-{type_slug}*.spec.*"))
        if not matches:
            return Result(
                dimension=17,
                status=Status.FAIL,
                evidence={"type_slug": type_slug, "matched": []},
                remediation_hint=f"No smoke test found for type {entry.type!r}. Add tests/smoke-{type_slug}.spec.ts.",
            )
        return Result(dimension=17, status=Status.PASS,
                      evidence={"type_slug": type_slug, "matched": [str(m) for m in matches]})
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_17_testability.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_17_testability.py tests/audit/dimensions/test_dim_17_testability.py
git commit -m "audit: dim #17 Testability (smoke test presence)"
```

---

## Task 22: Dim #18 — Doc truth (FUNNELS.md drift)

**Files:**
- Create: `audit/dimensions/dim_18_doc_truth.py`, `tests/audit/dimensions/test_dim_18_doc_truth.py`

- [ ] **Step 1: Write failing test**

```python
from pathlib import Path

from audit.core.models import Status
from audit.dimensions.dim_18_doc_truth import Dim18DocTruth


def test_doc_truth_passes_when_entry_mentioned_in_funnels_md(tmp_path, betrayal_entry, window_7d):
    funnels = tmp_path / "FUNNELS.md"
    funnels.write_text("## Funnel 1: Betrayal Workshop\n\n**Entry pages:** /betrayal/, /betrayal-offer/\n")
    r = Dim18DocTruth().run(betrayal_entry, window_7d, {"funnels_md_path": funnels})
    assert r.status is Status.PASS


def test_doc_truth_fails_when_entry_not_mentioned(tmp_path, betrayal_entry, window_7d):
    funnels = tmp_path / "FUNNELS.md"
    funnels.write_text("## Funnel X: Other\n\n**Entry pages:** /other/\n")
    r = Dim18DocTruth().run(betrayal_entry, window_7d, {"funnels_md_path": funnels})
    assert r.status is Status.FAIL


def test_doc_truth_fails_when_funnels_md_missing(tmp_path, betrayal_entry, window_7d):
    r = Dim18DocTruth().run(betrayal_entry, window_7d, {"funnels_md_path": tmp_path / "nope.md"})
    assert r.status is Status.FAIL
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/dimensions/test_dim_18_doc_truth.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Dimension 18 — Entry's URL pattern is referenced in FUNNELS.md."""

from __future__ import annotations

import re
from pathlib import Path
from typing import Any

from audit.core.models import Entry, Result, Status, Window


class Dim18DocTruth:
    dimension = 18

    def applies_to(self, entry: Entry) -> bool:
        return 18 in entry.applies

    def run(self, entry: Entry, window: Window, ctx: dict[str, Any]) -> Result:
        path = Path(ctx["funnels_md_path"])
        if not path.exists():
            return Result(dimension=18, status=Status.FAIL,
                          evidence={"reason": "FUNNELS.md not found"},
                          remediation_hint="Create/restore FUNNELS.md at repo root.")

        text = path.read_text()
        pattern = entry.locator.get("url_pattern") or ""
        # Extract a representative URL from the regex: the first literal path segment
        segments = re.findall(r"/[A-Za-z0-9_-]+/", pattern)
        if not segments:
            return Result(dimension=18, status=Status.NA,
                          evidence={"reason": "no extractable URL from locator"})
        # Any segment present = pass. All missing = fail.
        found = [s for s in segments if s in text]
        if not found:
            return Result(
                dimension=18,
                status=Status.FAIL,
                evidence={"expected_any_of": segments},
                remediation_hint=f"Entry {entry.id} not documented in FUNNELS.md. Add a section referencing {segments[0]}.",
            )
        return Result(dimension=18, status=Status.PASS, evidence={"matched_in_doc": found})
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/dimensions/test_dim_18_doc_truth.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/dimensions/dim_18_doc_truth.py tests/audit/dimensions/test_dim_18_doc_truth.py
git commit -m "audit: dim #18 Doc truth (FUNNELS.md drift)"
```

---

## Task 23: Per-entry report formatter

**Files:**
- Create: `audit/reporters/per_entry.py`, `tests/audit/reporters/test_per_entry.py`

- [ ] **Step 1: Write failing test**

```python
from audit.core.models import Entry, Result, Status
from audit.reporters.per_entry import format_per_entry_report


def _entry():
    return Entry(
        id="tr-betrayal-form", site="terryreal.com", type="elementor_form",
        locator={}, expected_events=["FormSubmit"], required_properties={},
        source_of_truth="", reconciliation_query="",
        applies=[1, 2, 4], skip=[13, 16],
    )


def test_report_has_header_and_score_line():
    results = [
        Result(dimension=1, status=Status.PASS, evidence={"site": "terryreal.com"}),
        Result(dimension=2, status=Status.PASS, evidence={"ratio": 1.0}),
        Result(dimension=4, status=Status.FAIL, evidence={"missing_fields": {"utm_source": {"null": 6, "total": 82}}}, remediation_hint="Check SDK"),
    ]
    md = format_per_entry_report(_entry(), results, window_label="2026-04-11 → 2026-04-18")
    assert "ENTRY: tr-betrayal-form" in md
    assert "Window: 2026-04-11 → 2026-04-18" in md
    assert "SCORE: 2/3" in md
    assert " 1. " in md
    assert "PASS" in md
    assert "FAIL" in md
    assert "Check SDK" in md


def test_report_shows_na_for_skipped_dims():
    results = [
        Result(dimension=1, status=Status.PASS, evidence={}),
        Result(dimension=2, status=Status.PASS, evidence={}),
        Result(dimension=4, status=Status.NA, evidence={"reason": "no events"}),
    ]
    md = format_per_entry_report(_entry(), results, window_label="x")
    assert "N/A" in md
    # Score should exclude N/A from denominator
    assert "SCORE: 2/2" in md
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/reporters/test_per_entry.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Markdown per-entry report formatter."""

from __future__ import annotations

import json

from audit.core.models import Entry, Result, Status


DIMENSION_NAMES = {
    1: "Registered",
    2: "Server capture",
    3: "Client capture",
    4: "Payload complete",
    5: "Payload correct",
    6: "Attribution chain",
    7: "Identity stitching",
    8: "Dedup",
    9: "Reconciliation",
    10: "Resilience",
    11: "Freshness",
    12: "Monitoring",
    13: "Architectural fit",
    14: "Simplicity",
    15: "Consistency",
    16: "Code quality",
    17: "Testability",
    18: "Doc truth",
}


def _status_label(status: Status) -> str:
    return {Status.PASS: "PASS", Status.FAIL: "FAIL", Status.NA: "N/A "}[status]


def format_per_entry_report(entry: Entry, results: list[Result], window_label: str) -> str:
    lines = [
        f"ENTRY: {entry.id}",
        f"Window: {window_label}",
        "─────────────────────────────",
    ]
    for r in sorted(results, key=lambda x: x.dimension):
        name = DIMENSION_NAMES.get(r.dimension, f"dim {r.dimension}")
        label = _status_label(r.status)
        line = f"{r.dimension:>2}. {name:<22} {label}"
        evidence_summary = _evidence_line(r)
        if evidence_summary:
            line += f"  {evidence_summary}"
        lines.append(line)
        if r.remediation_hint and r.status is Status.FAIL:
            lines.append(f"     → {r.remediation_hint}")

    applicable = [r for r in results if r.status is not Status.NA]
    passes = sum(1 for r in applicable if r.status is Status.PASS)
    lines.append("─────────────────────────────")
    lines.append(f"SCORE: {passes}/{len(applicable)}")
    return "\n".join(lines)


def _evidence_line(r: Result) -> str:
    ev = r.evidence or {}
    if r.dimension == 2 and "ratio" in ev:
        return f"{ev.get('posthog_count', '?')}/{ev.get('source_count', '?')} ({ev['ratio'] * 100:.1f}%)"
    if r.dimension == 3 and "ratio" in ev:
        return f"{ev.get('client_count', '?')}/{ev.get('server_count', '?')} ({ev['ratio'] * 100:.1f}%)"
    if r.dimension == 4 and ev.get("missing_fields"):
        fields = list(ev["missing_fields"].keys())
        return f"missing: {fields}"
    if r.dimension == 8 and "duplicate_count" in ev:
        return f"{ev['duplicate_count']} duplicate order_id(s)"
    if r.dimension == 9 and "match_rate" in ev:
        return f"{ev.get('matched', '?')}/{ev.get('expected', '?')} ({ev['match_rate'] * 100:.1f}%)"
    if r.status is Status.NA and ev.get("reason"):
        return ev["reason"]
    # Fallback: compact JSON
    if ev and len(json.dumps(ev)) < 80:
        return json.dumps(ev, separators=(",", ":"))
    return ""
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/reporters/test_per_entry.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/reporters/per_entry.py tests/audit/reporters/test_per_entry.py
git commit -m "audit: per-entry markdown report formatter"
```

---

## Task 24: Master scorecard formatter

**Files:**
- Create: `audit/reporters/scorecard.py`, `tests/audit/reporters/test_scorecard.py`

- [ ] **Step 1: Write failing test**

```python
import json

from audit.core.models import Entry, Result, Status
from audit.reporters.scorecard import format_scorecard


def _entry(entry_id, site):
    return Entry(id=entry_id, site=site, type="elementor_form",
                 locator={}, expected_events=[], required_properties={},
                 source_of_truth="", reconciliation_query="",
                 applies=[1, 2, 4], skip=[])


def test_scorecard_includes_all_entries_and_pass_rates():
    data = {
        _entry("tr-a", "terryreal.com"): [
            Result(dimension=1, status=Status.PASS, evidence={}),
            Result(dimension=2, status=Status.PASS, evidence={}),
            Result(dimension=4, status=Status.FAIL, evidence={}),
        ],
        _entry("rli-b", "relationallife.com"): [
            Result(dimension=1, status=Status.PASS, evidence={}),
            Result(dimension=2, status=Status.FAIL, evidence={}),
            Result(dimension=4, status=Status.PASS, evidence={}),
        ],
    }
    md, blob = format_scorecard(data, window_label="7d")
    assert "tr-a" in md
    assert "rli-b" in md
    assert "Dimension pass rate" in md

    parsed = json.loads(blob)
    assert parsed["window"] == "7d"
    assert len(parsed["entries"]) == 2
    by_id = {e["id"]: e for e in parsed["entries"]}
    assert by_id["tr-a"]["passes"] == 2
    assert by_id["tr-a"]["fails"] == 1


def test_scorecard_ranks_worst_offenders_first():
    data = {
        _entry("good", "x"): [Result(dimension=i, status=Status.PASS, evidence={}) for i in (1, 2, 4)],
        _entry("bad", "x"): [
            Result(dimension=1, status=Status.FAIL, evidence={}),
            Result(dimension=2, status=Status.FAIL, evidence={}),
            Result(dimension=4, status=Status.PASS, evidence={}),
        ],
    }
    md, _ = format_scorecard(data, window_label="7d")
    bad_idx = md.find("bad")
    good_idx = md.find("good")
    assert bad_idx < good_idx  # worst first in the offenders section
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/reporters/test_scorecard.py -v
```

Expected: ImportError.

- [ ] **Step 3: Implement**

```python
"""Master scorecard — aggregate markdown + JSON across all entries."""

from __future__ import annotations

import json
from collections import Counter

from audit.core.models import Entry, Result, Status


def format_scorecard(
    data: dict[Entry, list[Result]],
    window_label: str,
) -> tuple[str, str]:
    """Return (markdown, json_blob).

    Markdown: human scorecard. JSON: machine-readable for Command Centre.
    """
    entry_rows = []
    per_dim_pass = Counter()
    per_dim_total = Counter()

    for entry, results in data.items():
        applicable = [r for r in results if r.status is not Status.NA]
        passes = sum(1 for r in applicable if r.status is Status.PASS)
        fails = sum(1 for r in applicable if r.status is Status.FAIL)
        for r in results:
            if r.status is Status.NA:
                continue
            per_dim_total[r.dimension] += 1
            if r.status is Status.PASS:
                per_dim_pass[r.dimension] += 1
        entry_rows.append({
            "id": entry.id,
            "site": entry.site,
            "type": entry.type,
            "passes": passes,
            "fails": fails,
            "applicable": len(applicable),
            "fail_dims": [r.dimension for r in results if r.status is Status.FAIL],
        })

    entry_rows.sort(key=lambda r: (-r["fails"], r["id"]))

    md_lines = [
        f"# Rubric Scorecard — {window_label}",
        "",
        f"Entries audited: {len(entry_rows)}",
        "",
        "## Worst offenders",
        "",
        "| Entry | Site | Fails | Passes | Applicable | Failed dims |",
        "| --- | --- | ---: | ---: | ---: | --- |",
    ]
    for r in entry_rows:
        md_lines.append(
            f"| {r['id']} | {r['site']} | {r['fails']} | {r['passes']} | {r['applicable']} | {r['fail_dims']} |"
        )
    md_lines += ["", "## Dimension pass rate", "", "| Dim | Pass | Total | Rate |", "| ---: | ---: | ---: | ---: |"]
    for dim in sorted(per_dim_total):
        pr = per_dim_pass[dim]
        tot = per_dim_total[dim]
        rate = pr / tot if tot else 0
        md_lines.append(f"| {dim} | {pr} | {tot} | {rate * 100:.1f}% |")

    blob = {
        "window": window_label,
        "entries": entry_rows,
        "per_dimension": [
            {"dimension": d, "pass": per_dim_pass[d], "total": per_dim_total[d]}
            for d in sorted(per_dim_total)
        ],
    }
    return "\n".join(md_lines), json.dumps(blob, indent=2)
```

- [ ] **Step 4: Run test, verify it passes**

```bash
python -m pytest tests/audit/reporters/test_scorecard.py -v
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add audit/reporters/scorecard.py tests/audit/reporters/test_scorecard.py
git commit -m "audit: master scorecard (markdown + JSON)"
```

---

## Task 25: CLI (`python -m audit`)

**Files:**
- Create: `audit/__main__.py`, `audit/cli.py`, `tests/audit/test_cli.py`

- [ ] **Step 1: Write failing test**

```python
import subprocess
import sys
from pathlib import Path

REPO_ROOT = Path(__file__).resolve().parents[2]


def test_cli_help_lists_subcommands():
    result = subprocess.run(
        [sys.executable, "-m", "audit", "--help"],
        cwd=REPO_ROOT, capture_output=True, text=True,
    )
    assert result.returncode == 0
    assert "run" in result.stdout
    assert "generate-registry" in result.stdout
    assert "drift-check" in result.stdout


def test_cli_run_requires_window(monkeypatch):
    result = subprocess.run(
        [sys.executable, "-m", "audit", "run"],
        cwd=REPO_ROOT, capture_output=True, text=True,
    )
    # argparse returns 2 for missing required args
    assert result.returncode != 0
    assert "window" in result.stderr.lower()
```

- [ ] **Step 2: Run test, verify it fails**

```bash
python -m pytest tests/audit/test_cli.py -v
```

Expected: ModuleNotFoundError on `audit.__main__`.

- [ ] **Step 3: Implement `audit/__main__.py`**

```python
from audit.cli import main

if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Implement `audit/cli.py`**

```python
"""CLI entry point: python -m audit {run,generate-registry,drift-check}"""

from __future__ import annotations

import argparse
import importlib
import pkgutil
import sys
from pathlib import Path
from typing import Any

from audit.core.models import Window
from audit.core.registry import load_registry, load_entry
from audit.core.runner import Runner
from audit.reporters.per_entry import format_per_entry_report
from audit.reporters.scorecard import format_scorecard


def discover_plugins() -> list:
    """Auto-import audit.dimensions.dim_NN_* modules and instantiate the Dim class."""
    import audit.dimensions as package
    plugins = []
    for info in pkgutil.iter_modules(package.__path__):
        if not info.name.startswith("dim_"):
            continue
        mod = importlib.import_module(f"audit.dimensions.{info.name}")
        # Each dim module exposes one class named Dim<NN><Rest>
        for attr in dir(mod):
            if attr.startswith("Dim") and attr != "Dim":
                cls = getattr(mod, attr)
                if hasattr(cls, "dimension"):
                    plugins.append(cls())
                    break
    return plugins


def build_context(repo_root: Path) -> dict[str, Any]:
    """Assemble ctx dict for plugins. Spec 2 fills in real adapters."""
    from audit.core.posthog_client import PostHogClient
    # Placeholders for Spec 1 reference run — can be None unless the
    # specific dim is being tested. Real adapters ship in Spec 2.
    ctx = {
        "repo_root": repo_root,
        "tests_root": repo_root / "tests",
        "funnels_md_path": repo_root / "FUNNELS.md",
        "architecture_review_path": repo_root / "audit" / "architecture_review.md",
        "fire_drill_log_path": repo_root / "audit" / "fire_drill_log.json",
    }
    try:
        ctx["posthog"] = PostHogClient.from_env()
    except Exception as e:
        ctx["posthog"] = None
        ctx["_posthog_unavailable_reason"] = str(e)
    return ctx


def cmd_run(args) -> int:
    repo_root = Path(args.repo_root).resolve()
    registry_path = repo_root / "tracking-contract.json"
    window = Window.from_spec(args.window)
    window_label = f"{window.start.date()} → {window.end.date()}"

    entries = [load_entry(registry_path, args.entry)] if args.entry else load_registry(registry_path)
    plugins = discover_plugins()
    if args.dimension:
        plugins = [p for p in plugins if p.dimension == args.dimension]
    ctx = build_context(repo_root)
    runner = Runner(plugins, ctx)

    reports_dir = repo_root / "audit" / "reports"
    reports_dir.mkdir(parents=True, exist_ok=True)
    scorecards_dir = repo_root / "audit" / "scorecards"
    scorecards_dir.mkdir(parents=True, exist_ok=True)

    data = {}
    for entry in entries:
        results = runner.run_entry(entry, window)
        data[entry] = results
        md = format_per_entry_report(entry, results, window_label)
        out = reports_dir / f"{entry.id}-{window.end.date()}.md"
        out.write_text(md)
        print(f"Wrote {out}")

    scorecard_md, scorecard_json = format_scorecard(data, window_label)
    (scorecards_dir / f"scorecard-{window.end.date()}.md").write_text(scorecard_md)
    (scorecards_dir / f"scorecard-{window.end.date()}.json").write_text(scorecard_json)
    print(f"Wrote scorecard for {len(data)} entries")
    return 0


def cmd_generate_registry(args) -> int:
    from audit.scanners.html_forms import discovered_urls_by_site
    sites = ["terryreal.com", "relationallife.com", "summit.terryreal.com", "quiz.terryreal.com", "grid.terryreal.com"]
    discovered = discovered_urls_by_site(sites)
    print("Discovered forms (for review, not auto-committed):")
    for site, urls in discovered.items():
        print(f"\n  {site}:")
        for u in urls:
            print(f"    {u}")
    return 0


def cmd_drift_check(args) -> int:
    from audit.scanners.html_forms import discovered_urls_by_site
    repo_root = Path(args.repo_root).resolve()
    entries = load_registry(repo_root / "tracking-contract.json")
    by_site: dict[str, list[str]] = {}
    for e in entries:
        by_site.setdefault(e.site, []).append(e.locator.get("url_pattern", ""))

    sites = sorted(by_site)
    discovered = discovered_urls_by_site(sites)
    import re
    has_drift = False
    for site, urls in discovered.items():
        patterns = [re.compile(p) for p in by_site.get(site, []) if p]
        unknown = [u for u in urls if not any(p.search(u) for p in patterns)]
        if unknown:
            has_drift = True
            print(f"DRIFT on {site}: unregistered URL(s): {unknown}")
    return 1 if has_drift else 0


def main() -> None:
    parser = argparse.ArgumentParser(prog="audit", description="Tracking audit runner")
    sub = parser.add_subparsers(dest="cmd", required=True)

    p_run = sub.add_parser("run", help="Run the rubric against one or all entries")
    p_run.add_argument("--window", required=True, help="e.g. 24h, 7d")
    p_run.add_argument("--entry", help="Single entry id (default: all)")
    p_run.add_argument("--dimension", type=int, help="Run only one dimension")
    p_run.add_argument("--repo-root", default=".")
    p_run.set_defaults(func=cmd_run)

    p_gen = sub.add_parser("generate-registry", help="Scan sites for forms (preview)")
    p_gen.add_argument("--repo-root", default=".")
    p_gen.set_defaults(func=cmd_generate_registry)

    p_drift = sub.add_parser("drift-check", help="Detect forms not in registry; exit 1 if drift found")
    p_drift.add_argument("--repo-root", default=".")
    p_drift.set_defaults(func=cmd_drift_check)

    args = parser.parse_args()
    sys.exit(args.func(args))
```

- [ ] **Step 5: Run test, verify it passes**

```bash
python -m pytest tests/audit/test_cli.py -v
```

Expected: 2 passed.

- [ ] **Step 6: Smoke-test CLI by hand**

```bash
python -m audit --help
python -m audit run --help
```

Expected: argparse usage text; no tracebacks.

- [ ] **Step 7: Commit**

```bash
git add audit/__main__.py audit/cli.py tests/audit/test_cli.py
git commit -m "audit: CLI (run / generate-registry / drift-check)"
```

---

## Task 26: Reference integration run for `tr-betrayal-form`

**Files:**
- Create: `tests/audit/integration/test_reference_run.py`

- [ ] **Step 1: Write integration test**

This test exercises the full runner end-to-end against the registered `tr-betrayal-form` entry with mocked PostHog / reconciler / playwright_runner / sources. It verifies a per-entry report file and a scorecard are produced.

```python
import json
from datetime import datetime, timedelta, timezone
from pathlib import Path
from unittest.mock import MagicMock

from audit.core.models import Window
from audit.core.posthog_client import QueryResult
from audit.core.registry import load_entry
from audit.core.runner import Runner
from audit.reporters.per_entry import format_per_entry_report
from audit.reporters.scorecard import format_scorecard


REPO_ROOT = Path(__file__).resolve().parents[3]


def test_reference_run_tr_betrayal_form(tmp_path):
    contract = REPO_ROOT / "tracking-contract.json"
    entry = load_entry(contract, "tr-betrayal-form")

    now = datetime(2026, 4, 18, tzinfo=timezone.utc)
    window = Window(start=now - timedelta(days=7), end=now)

    # Assemble mocked context — every dim gets a plausible response.
    ph = MagicMock()
    ph.hogql.return_value = QueryResult(columns=["c"], rows=[[100]])
    sources = MagicMock()
    sources.source_of_truth_count.return_value = 100
    sources.resolve_sample.return_value = {}
    reconciler = MagicMock()
    reconciler.run.return_value = {"matched": 100, "expected": 100, "missing": []}
    playwright_runner = MagicMock()
    playwright_runner.run_path.side_effect = [
        {"path": n, "passed": True} for n in ["cache_hit", "consent_denied", "mobile_ua", "cross_domain_hop"]
    ]
    posthog_usage = MagicMock()
    posthog_usage.read_properties.return_value = {
        (e, p) for e, ps in entry.required_properties.items() for p in ps
    }

    fire_drill_log = tmp_path / "fire_drill_log.json"
    fire_drill_log.write_text(json.dumps([{
        "drill_type": "form_hook_disable",
        "performed_at": (now - timedelta(days=10)).isoformat(),
        "alert_fired_within_sla": True,
    }]))
    arch_review = tmp_path / "architecture_review.md"
    arch_review.write_text("## Subsystem A\nDecision: keep.\n")

    ctx = {
        "posthog": ph, "sources": sources, "reconciler": reconciler,
        "playwright_runner": playwright_runner, "posthog_usage": posthog_usage,
        "repo_root": REPO_ROOT, "tests_root": REPO_ROOT / "tests",
        "funnels_md_path": REPO_ROOT / "FUNNELS.md",
        "architecture_review_path": arch_review,
        "fire_drill_log_path": fire_drill_log,
        "known_entry_ids": {entry.id},
        "discovered_form_urls_by_site": {"terryreal.com": ["/betrayal/"]},
        "registered_url_patterns_by_site": {"terryreal.com": [entry.locator["url_pattern"]]},
    }

    from audit.cli import discover_plugins
    plugins = discover_plugins()
    # Filter plugins that apply to this entry
    plugins = [p for p in plugins if p.applies_to(entry)]
    runner = Runner(plugins, ctx)

    results = runner.run_entry(entry, window)
    assert len(results) >= 14  # all applicable dims produce a Result

    md = format_per_entry_report(entry, results, "7d")
    assert "tr-betrayal-form" in md
    assert "SCORE:" in md

    scorecard_md, blob = format_scorecard({entry: results}, "7d")
    assert "tr-betrayal-form" in scorecard_md
    data = json.loads(blob)
    assert data["entries"][0]["id"] == "tr-betrayal-form"
```

- [ ] **Step 2: Run the integration test**

```bash
python -m pytest tests/audit/integration/test_reference_run.py -v
```

Expected: 1 passed. (If any dim fails to find its ctx key, fix that dim — the integration test surfaces integration bugs across dims.)

- [ ] **Step 3: Run the full pytest suite**

```bash
python -m pytest tests/audit/ -v
```

Expected: all green. Fix any cross-dim regressions surfaced.

- [ ] **Step 4: Commit**

```bash
git add tests/audit/integration/test_reference_run.py
git commit -m "audit: integration test for end-to-end run on tr-betrayal-form"
```

---

## Task 27: Docs — README + TODO updates

**Files:**
- Create: `audit/README.md`
- Modify: `TODO.md` — add audit rollout items

- [ ] **Step 1: Write `audit/README.md`**

```markdown
# `audit/` — Tracking Audit Runner

Implementation of the rubric defined in `docs/superpowers/specs/2026-04-18-tracking-audit-rubric-design.md`.

## Usage

```bash
# Run all entries, last 7 days
python -m audit run --window 7d

# Run one entry
python -m audit run --entry tr-betrayal-form --window 24h

# Run one dimension across all entries
python -m audit run --dimension 8 --window 7d

# Scan sites for new forms (preview only)
python -m audit generate-registry

# Exit non-zero if there are registered-form drifts
python -m audit drift-check
```

## Outputs

- `audit/reports/<entry-id>-<date>.md` — one per entry
- `audit/scorecards/scorecard-<date>.md` — aggregate markdown
- `audit/scorecards/scorecard-<date>.json` — aggregate JSON for Command Centre

## Environment

- `POSTHOG_KEY_FILE` — default `.posthog-key` at repo root
- `POSTHOG_PROJECT_ID` — default `127361`
- `POSTHOG_HOST` — default `https://us.posthog.com`

## Adding a dimension

1. Create `audit/dimensions/dim_NN_name.py` with a class `DimNNName` exposing
   `dimension: int`, `applies_to(entry)`, and `run(entry, window, ctx)`.
2. Add tests in `tests/audit/dimensions/test_dim_NN_name.py`.
3. Update `audit/reporters/per_entry.py::DIMENSION_NAMES`.
4. The CLI auto-discovers the new module.

## Adding an entry point

Edit `tracking-contract.json` → `entry_points` array. Each entry needs
`id`, `site`, `type`, `locator`, `expected_events`, `required_properties`,
`source_of_truth`, `reconciliation_query`, `applies`, `skip` (see
`audit/core/registry.py` for the validation).
```

- [ ] **Step 2: Update `TODO.md`**

Append to `TODO.md` after the existing "Next Session" section:

```markdown

## Audit Rubric Rollout (Spec 2)

- [ ] Populate `tracking-contract.json` entry_points from FUNNELS.md (39+ entries)
- [ ] Wire real WP / ThriveCart / Ontraport source-of-truth counters in `audit/core/sources.py`
- [ ] Wire real Playwright synthetic paths runner (`audit/core/playwright_runner.py`)
- [ ] Wire PostHog insight-usage scanner for dim #14
- [ ] Schedule nightly `python -m audit run --window 24h` via GitHub Action
- [ ] Fill in `audit/architecture_review.md` with a Decision per subsystem
- [ ] First pass of quarterly fire drill → record in `audit/fire_drill_log.json`
- [ ] Add Rubric tab to `tracking-command-centre/` reading `audit/scorecards/scorecard-latest.json`
```

- [ ] **Step 3: Commit**

```bash
git add audit/README.md TODO.md
git commit -m "audit: README + TODO updates for Spec 2 rollout"
```

---

## Success Criteria

- `python -m pytest tests/audit/ -v` passes green (all 25+ test files).
- `python -m audit run --entry tr-betrayal-form --window 7d` produces:
  - `audit/reports/tr-betrayal-form-<date>.md`
  - `audit/scorecards/scorecard-<date>.md` + `.json`
- `python -m audit drift-check` exits 0 when registry matches discovered forms.
- `tracking-contract.json` has an `entry_points` array with at least the `tr-betrayal-form` sample.
- `audit/README.md` documents usage and extension points.
- All 27 tasks committed to the branch.
