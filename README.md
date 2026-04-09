# rehome

**Move your network so easily it's scary.**

Network migration as code — capture your router config once, version it, diff it, apply it anywhere. Cross-vendor. Scriptable. Safe.

> **Status: pre-alpha.** Architecture in progress. First target pair: **Asus Merlin → UniFi OS** — proven end-to-end on 2026-04-09 via a one-off seed script (Asus BE82U → UniFi Dream Router 7, UniFi Network 10.0.162). Adapter code lands next, seeded by the schema notes in [`docs/vendor-notes/unifi-os.md`](docs/vendor-notes/unifi-os.md). Not yet usable as a CLI — ⭐ the repo to follow along.

---

## What is this?

Every time you replace a router — vendor upgrade, ISP swap, firmware migration, hardware failure — you re-enter the same config by hand. DHCP reservations. Port forwards. Wi-Fi SSIDs and PSKs. Firewall rules. Static routes. Custom DNS. QoS. Hours of clicking through web UIs that were never designed for bulk operations.

`rehome` reads your existing router's config, writes it to a single human-readable YAML file, and applies that file to your new router. One command to capture. One command to apply. Diff between them. Commit the YAML to git. Roll back with `git revert`.

```bash
# Capture everything from the old router
rehome export --from asus-merlin --host 192.168.1.1 \
              --user admin --out mynetwork.yaml

# See what would change on the new router (no mutations)
rehome import --to unifi-os --host 192.168.1.1 \
              --user admin --in mynetwork.yaml --dry-run

# Apply for real
rehome import --to unifi-os --host 192.168.1.1 \
              --user admin --in mynetwork.yaml
```

The YAML file is the product. It's yours. It's readable. It's diffable. It lives in your git repo alongside the rest of your infrastructure.

## Why you'll actually use it

**For homelab nerds and DevOps folks:**

- **Your network as code.** `git diff` your router. `git revert` a bad change. Commit every tweak. No more "I think I changed something last Tuesday."
- **Cross-vendor migration.** Moving from Asus to UniFi? pfSense to OpenWrt? Mikrotik to anything? Capture once, apply anywhere. No rebuild-from-scratch.
- **Reproducible setup.** Spin up a new homelab or deploy a fleet of sites from one template. Clone a working network in seconds.
- **Disaster recovery.** Factory reset → `rehome import` → back online. Your config is in git, not locked inside a dying router.
- **Documentation that never goes stale.** The config file *is* the documentation. If something's wrong, you fix the file and reapply.
- **Dry-run before apply.** See exactly what will change before it changes. No surprises.
- **Idempotent.** Re-run safely. Partial failures don't leave you in a weird half-state.

**For managed service providers:**

- Version every client's network in git. Audit changes. Roll back on demand.
- Bootstrap a new customer site from a template.
- Move a client from one ISP gateway to another without a full rebuild.

**For accessibility:**

- Because the config is plain text and the CLI output is structured prose, `rehome` is inherently usable by blind and low-vision admins who cannot navigate modern router web UIs. No drag-drop. No icon-only buttons. No color-coded status. The text file is the interface.

`rehome` isn't an accessibility tool with a CLI tacked on. It's a config-as-code tool whose architecture happens to make it accessible to everyone — including people who've been locked out of network admin by bad web UIs for years.

## How it works

`rehome` is a compiler. Each router vendor gets an **adapter** with two methods: `export()` and `import()`. Adapters translate between the vendor's native API and a vendor-neutral **intermediate representation** — a YAML file with a documented schema.

```
  Asus Merlin  ─┐                        ┌─  UniFi OS
                │                        │
  pfSense      ─┤                        ├─  OpenWrt
                ├──▶  netconf.yaml  ──▶  │
  OpenWrt      ─┤       (the IR)         ├─  pfSense
                │                        │
  Mikrotik     ─┘                        └─  Mikrotik
```

Adding a new router = writing one adapter. Every adapter works with every other adapter through the IR, with no N×N matrix explosion.

See [docs/architecture.md](docs/architecture.md) for the adapter interface and IR schema. See [examples/netconf.example.yaml](examples/netconf.example.yaml) for what the YAML looks like.

## Vendor support matrix

| Vendor          | Export (read) | Import (apply) | Version target       | Notes                                |
|-----------------|---------------|----------------|----------------------|--------------------------------------|
| Asus Merlin     | planned v0.1  | —              | 388.x+               | SSH + nvram + iptables + JFFS        |
| UniFi OS        | planned v0.2  | planned v0.1   | Network 8.x+         | REST API at `/proxy/network/api`     |
| OpenWrt         | planned v0.3  | planned v0.3   | 22.03+               | UCI over SSH                         |
| pfSense         | planned v0.4  | planned v0.4   | 2.7+                 | `config.xml` + REST (where enabled)  |
| OPNsense        | future        | future         | 24.x+                | REST API                             |
| Mikrotik        | future        | future         | RouterOS 7+          | API or SSH                           |
| DD-WRT          | future        | future         | —                    | nvram + telnet/SSH                   |
| ISP gateways    | out of scope  | out of scope   | —                    | No API, no SSH, no way               |

## Roadmap

- **v0.1** — Asus Merlin → UniFi OS one-way migration. Networks/VLANs, Wi-Fi SSIDs, DHCP reservations, custom DNS, static routes, port forwards, basic QoS. Dry-run, idempotent, rollback. Proves the IR on a real migration.
- **v0.2** — UniFi OS `export()` — symmetric with import. Round-trip tests (export → diff → import) to catch lossy mappings.
- **v0.3** — OpenWrt adapter (both directions). Validates the IR is vendor-neutral, not UniFi-biased.
- **v0.4** — pfSense source + target. Cross-vendor 2×2 matrix complete.
- **v0.5** — Accessibility polish pass: structured logging format, prose diff output, narrated demo video, contributor docs, accessibility statement.
- **v1.0** — Community launch. GitHub Pages site, vendor support matrix, contribution guide, blog post, first real-world case studies.

## Prerequisites

- Python 3.10+
- SSH access to source router (if the adapter uses SSH)
- Admin credentials for target router
- **Dry-run first, always.** Never apply a config you haven't diff'd.

## Safety

`rehome` touches network infrastructure. The safety model:

1. **Dry-run is the default mode of thought.** Every `import` should start with `--dry-run`. CI/CD pipelines should gate actual applies on human review of the dry-run output.
2. **Idempotent by construction.** Re-running a partial apply continues where it left off, no duplicates, no corrupted state.
3. **Never destructive without consent.** `rehome` will not delete existing config without an explicit `--delete-unmanaged` flag. Anything in the YAML is created or updated; anything not in the YAML is ignored, not removed.
4. **Rollback is just `git revert`.** Since the YAML is in git, rollback is a diff away.
5. **Bricking is possible.** If you apply a broken config to a router, you can lock yourself out. Every vendor's adapter documents its specific recovery path (factory reset button, serial console, etc.). Read the docs before you apply.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The cleanest way to help: pick a vendor you use, write an adapter, submit a PR. The adapter interface is small and the IR is documented.

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, embed it in your MSP tooling, ship it in your distro. Credit appreciated, not required.

## Acknowledgments

This project started as a script to migrate one specific home network from an Asus BE82U to a Ubiquiti UDR-7 on 2026-04-09 by [@w9fyi](https://github.com/w9fyi) (AI5OS). It turned into this when it became obvious the same problem hits everyone who touches a router more than once.
