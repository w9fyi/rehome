# Architecture

`rehome` has three layers: **adapters**, the **intermediate representation (IR)**, and the **engine**.

```
  User CLI / API
       │
       ▼
  ┌─────────────┐
  │   Engine    │   ← dry-run, diff, idempotency, rollback, logging
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │     IR      │   ← vendor-neutral YAML, validated schema
  │ (NetConfig) │
  └──────┬──────┘
         │
    ┌────┴────┬───────────┬───────────┐
    ▼         ▼           ▼           ▼
 Asus      UniFi       OpenWrt     pfSense      ← adapters
 adapter   adapter     adapter     adapter
    │         │           │           │
    ▼         ▼           ▼           ▼
 SSH+       REST API    SSH+UCI     REST+
 nvram                              XML
```

## The intermediate representation

The IR is a single YAML file describing a network's logical configuration. It is **vendor-neutral** — it describes *what* the network looks like, not *how* a specific router implements it.

### Top-level structure

```yaml
version: 1              # IR schema version
metadata:
  name: "my-home-network"
  description: "optional human description"
  captured_from: "asus-merlin"
  captured_at: "2026-04-08T17:00:00Z"
wan:
  mode: dhcp            # or "pppoe", "static"
  # ... mode-specific fields
networks:
  - name: Default
    # ... LAN / VLAN definitions
wifi:
  - ssid: "my-wifi"
    # ... SSID definitions
dhcp_reservations:
  - mac: "00:1c:2d:05:3b:50"
    # ... fixed IP assignments
dns_records:
  - host: flex8400.lan
    # ... custom DNS entries
routes:
  - dest: "10.0.0.0/8"
    # ... static routes
port_forwards:
  - name: "ssh to jumpbox"
    # ... inbound NAT rules
firewall:
  - name: "block IoT to LAN"
    # ... firewall rules
qos:
  - label: "ham radio priority"
    # ... traffic priority rules
```

See [`examples/netconf.example.yaml`](../examples/netconf.example.yaml) for a full worked example.

### Design principles

1. **Every field is optional where vendors disagree.** A field that only some vendors support gets a clear default and a note in the schema about who honors it.
2. **No vendor-specific escape hatches in core IR.** If you need something vendor-specific, it goes in a `vendor_specific:` block that only the matching adapter reads.
3. **Stable key ordering.** The YAML serializer emits keys in a documented order so diffs are meaningful.
4. **Comments in examples, not in generated output.** Human-authored YAML can have comments; `rehome export` produces clean, diffable output.
5. **Round-trip fidelity is a hard requirement.** `export → YAML → import` on the same device must produce a no-op after the first apply. Adapter PRs need round-trip tests.

## The adapter interface

```python
from typing import Protocol

class Adapter(Protocol):
    name: str                           # "asus-merlin", "unifi-os", etc.
    supported_versions: list[str]       # e.g. ["388.x", "3006.x"]

    def connect(self, host: str, creds: "Credentials") -> None: ...
    def export(self) -> "NetConfig": ...
    def import_(
        self,
        config: "NetConfig",
        *,
        dry_run: bool = False,
        delete_unmanaged: bool = False,
    ) -> "ImportReport": ...
    def supports(self, feature: "Feature") -> bool: ...
    def disconnect(self) -> None: ...
```

**`connect()` and `disconnect()`** manage the transport (SSH session, HTTP session with cookies, REST tokens).

**`export()`** reads the device's current state and returns a populated `NetConfig`. This is where vendor-specific parsing lives — reading nvram on Asus, `/proxy/network/api/...` on UniFi, UCI files on OpenWrt, etc.

**`import_()`** takes a `NetConfig` and applies it to the device. Must be idempotent. Must honor `dry_run=True` by never mutating state. Returns an `ImportReport` with one entry per action: created / updated / skipped / warned / failed.

**`supports()`** declares capability. Before an import, the engine calls `supports()` for every feature present in the target `NetConfig` and warns about anything the adapter cannot handle.

## The engine

The engine is what the user actually talks to. It:

1. **Loads and validates** the YAML IR against the schema
2. **Calls `supports()`** on the target adapter for every feature in the IR, collecting warnings
3. **Runs dry-run mode first** (unless the user explicitly passes `--apply`), emitting a structured prose diff
4. **Waits for user confirmation** before applying mutations
5. **Calls `import_()`** on the target adapter
6. **Collects the `ImportReport`** and prints a final summary
7. **Optionally commits** the applied YAML to a git repo (`--commit-to ./network-config`)

The engine is the only place that should know about CLI arguments, logging format, or user interaction. Adapters are pure library code.

## Credentials handling

- Credentials are passed in-memory only.
- No file-based secrets by default.
- Users can opt into reading from environment variables (`REHOME_<VENDOR>_PASSWORD`) or from the system keychain.
- Never written to logs. Every adapter runs its own log output through a redaction filter before printing.
- The debug log (`--debug-log rehome.log`) excludes credentials and session tokens.

## Testing strategy

- **Unit tests** for each adapter mock the transport layer (SSH via `paramiko` stub, HTTP via `responses` or `httpx.MockTransport`). Fast, deterministic, run on every PR.
- **Integration tests** marked `@pytest.mark.hardware` require real devices and environment variables. Run manually before merging adapter changes.
- **Round-trip tests** are the gold standard: `export → IR → import → export → compare`. Any difference is a bug.
- **IR schema tests** validate that example YAML files round-trip cleanly through parse → serialize.

## Non-goals

- **No GUI.** Plain-text CLI only. (A future editor plugin is welcome as a separate project.)
- **No live monitoring / state tracking.** `rehome` is a batch config tool, not a controller. It does one-shot captures and one-shot applies.
- **No cross-site orchestration.** For managing a fleet of sites, wrap `rehome` in Ansible / Terraform / Nix / your tool of choice. `rehome` stays small and does one thing well.
- **No ISP gateway support** for devices with no API, no SSH, and no way in. If the vendor locks it down, `rehome` can't help.
