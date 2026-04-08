# Contributing to rehome

Thanks for your interest! `rehome` is in pre-alpha — the fastest way to help right now is to **pick a vendor you use and write an adapter**.

## The adapter model

Every router vendor is represented by an `Adapter` class with a minimal interface:

```python
class Adapter(Protocol):
    name: str                       # "asus-merlin", "unifi-os", "openwrt", etc.
    supported_versions: list[str]

    def connect(self, host: str, creds: Credentials) -> None: ...
    def export(self) -> NetConfig: ...
    def import_(self, config: NetConfig, *, dry_run: bool = False) -> ImportReport: ...
    def supports(self, feature: Feature) -> bool: ...
    def disconnect(self) -> None: ...
```

`export()` reads the current state of the device and returns a `NetConfig` — the intermediate representation. `import_()` takes a `NetConfig` and applies it to the device. `supports()` lets the engine ask whether a given feature (e.g. "per-SSID PMK caching") is handled by this adapter, so it can warn on lossy mappings.

## Writing a new adapter

1. Create `src/rehome/adapters/<vendor_name>.py`
2. Subclass `Adapter` and implement the interface
3. Use the vendor's official API where available; fall back to SSH + config parsing if not
4. Add a unit test under `tests/adapters/test_<vendor_name>.py` with mocked responses
5. Add the vendor to the support matrix in `README.md`
6. Document any quirks, caveats, or feature gaps in `docs/adapters/<vendor_name>.md`

## What "feature support" means

The IR defines a superset of features across all vendors. Every adapter declares which of those features it supports via `supports()`. When importing a config that includes a feature the target doesn't support, `rehome` emits a loud `WARN` and skips that item. This is how we handle lossy mappings honestly instead of silently dropping data.

## Style

- Python 3.10+
- Type hints everywhere — checked with `mypy --strict`
- Formatted with `ruff format`
- Linted with `ruff check`
- Tests with `pytest`
- No third-party dependencies in core; adapters may depend on vendor SDKs

## Testing against real hardware

Adapters should include **two** tests:

1. **Unit test** with mocked HTTP/SSH responses — runs in CI, fast, deterministic
2. **Integration test** marked with `@pytest.mark.hardware` — runs only when you set `REHOME_TEST_<VENDOR>_HOST` + credentials, hits a real device, and is expected to be run manually before merging changes to that adapter

If you can't test against real hardware for a vendor, that's OK — just label the adapter "community-contributed, untested" in the support matrix and we'll catch issues as users run into them.

## Accessibility expectations

`rehome` is designed to be fully usable by blind and low-vision admins. That means:

- **All output is plain text.** No ANSI boxes, no spinners, no progress bars that redraw. Status updates are individual lines.
- **Errors are scannable.** Each error has a one-line summary followed by detail indented below, not a wall of stack trace.
- **Diff output is prose.** Not a colorized Unix diff. Sentence-form descriptions of what will change: "The DHCP reservation for 00:1c:2d:05:3b:50 will change from 192.168.1.117 to 192.168.50.117."
- **The YAML schema is friendly to text editors.** Logical grouping, stable key ordering, comments in examples.

PRs that regress on accessibility will be asked to change before merge.

## Code of conduct

Be kind. Assume good intent. Explain things without condescension. This project attracts first-time contributors who are excited to finally be able to do network admin on their own terms — treat their PRs accordingly.

## License

By contributing, you agree that your contributions are licensed under the MIT License, same as the rest of the project.
