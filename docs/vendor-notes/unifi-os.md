# Vendor notes — UniFi OS / UniFi Network

Working notes for the future `rehome.adapters.unifi_os` adapter. Captured from a real Asus BE82U → UniFi Dream Router 7 migration on 2026-04-09 against **UniFi Network 10.0.162**.

The migration was driven by a one-off Python script (`udr7_configure.py`) that became the seed for this adapter. Everything below is what that script learned the hard way so the adapter doesn't have to.

> **Scope:** these notes apply specifically to UniFi Network 10.x running on a UniFi OS console (UDR-7, UDM-Pro, Cloud Gateway). The legacy v6/v7 controller and the standalone UniFi Network Application may differ.

## TL;DR for adapter authors

1. **Authenticate via API key**, not username/password. Cloud-SSO consoles often have no local-only admin account at all. Read the key from `X-API-KEY` header. Document an env-var convention (e.g. `UNIFI_API_KEY`).
2. **Don't mutate the Default LAN until everything else is applied.** Changing its subnet renumbers the interface mid-API-call and kills your HTTP session. Apply Default last, after all other steps have been written through the still-live session.
3. **The integrations API is incomplete.** `/proxy/network/integration/v1/...` only exposes sites/networks/devices/clients. For everything else (routing, firewall, traffic rules, static DNS, wlanconf, etc.) you have to use the legacy `/proxy/network/api/s/<site>/...` and `/proxy/network/v2/api/site/<site>/...` paths.
4. **Several legacy fields use unusual naming conventions.** See the schema gotchas below.
5. **Idempotency is your friend.** Every write should check for an existing object first and PUT-merge instead of POST'ing duplicates. UniFi tracks objects by `_id`, but human keys (network name, SSID name, MAC, hostname) work fine for dedup.

---

## Auth

### API key (preferred)

UniFi Cloud Gateways are typically adopted via `unifi.ui.com` cloud SSO during the first-boot wizard. After adoption, no local-only admin account exists by default — `POST /api/auth/login` returns `AUTHENTICATION_FAILED_INVALID_CREDENTIALS` for any user/password combination.

The fix is to create a local API key in the cloud portal:

> **Settings → Control Plane → Integrations → API Keys → Create API Key**

The key is then sent on every request as a header:

```http
X-API-KEY: <key>
```

No login round-trip is needed. The key works on every legacy and v2 endpoint.

Sanity check the key against a cheap known-good endpoint before running any mutating code:

```bash
curl -sk -H "X-API-KEY: $UNIFI_API_KEY" \
  https://192.168.50.1/proxy/network/api/s/default/rest/networkconf
```

A 200 with `{"meta":{"rc":"ok"},...}` means the key is good.

### Username/password (fallback)

Only works on consoles where a local admin account has been created (or on factory-fresh consoles before cloud adoption). Endpoint:

```http
POST /api/auth/login
Content-Type: application/json

{"username": "...", "password": "...", "remember": true}
```

Captures a session cookie + `X-CSRF-Token` header. UniFi rotates the CSRF token on most responses — read it back from response headers and resend on the next request.

---

## Endpoint surfaces

UniFi exposes three overlapping API surfaces. Adapter code touches all three:

| Surface | Path prefix | Notes |
|---|---|---|
| **Legacy v1 (site)** | `/proxy/network/api/s/<site>/...` | Networks, WLANs, users (DHCP reservations), routing, firewall rules, settings. Most write operations live here. |
| **v2 (site)** | `/proxy/network/v2/api/site/<site>/...` | Static DNS, traffic rules, AP groups, some newer features. Returns naked JSON arrays in many endpoints (no `data` wrapper). |
| **Integrations v1** | `/proxy/network/integration/v1/...` | Newer documented API. Currently only exposes sites/networks/devices/clients. Not yet usable for write-side adapter work — incomplete. |

The default site name is the literal string `"default"` (not the UUID). The integrations API uses a UUID site ID; the legacy/v2 APIs use the short name.

Get the application version and confirm you're talking to a real UniFi Network controller:

```bash
curl -sk -H "X-API-KEY: $KEY" \
  https://<host>/proxy/network/integration/v1/info
```

Returns `{"applicationVersion": "10.0.162"}` or similar. Adapter should branch on the major version because schema details change.

---

## Schema gotchas (the things that ate hours)

### 1. WLANs require `ap_group_ids`

Every wlanconf POST must include `ap_group_ids: [<id>, ...]` or the API returns `api.err.ApGroupMissing`.

Fetch the available groups dynamically — IDs vary per controller and per site:

```http
GET /proxy/network/v2/api/site/default/apgroups
```

```json
[{"_id": "...", "name": "All APs", "device_macs": ["..."], ...}]
```

The default group on a single-AP UDR-7 is named `"All APs"`. When creating an SSID that should broadcast on every AP, bind to all known group IDs. When creating an SSID for a specific AP only, bind to that AP's group.

### 2. SSIDs use `wlan_bands` (list), not `wlan_band` (string)

The legacy `wlan_band: "both"` field still works for 2.4 + 5 GHz SSIDs. But:

- `wlan_band: "6g"` → rejected as `api.err.InvalidValue`
- `wlan_band: "all"` → not a thing
- The newer schema is `wlan_bands: ["2g", "5g", "6g"]` (a list)

A single SSID can broadcast on all three bands simultaneously via `wlan_bands`. You do **not** need a separate WPA3-only companion SSID for 6 GHz, even though the Wi-Fi Alliance spec forbids WPA2 on 6 GHz. The controller handles this transparently — it switches to WPA3-only on the 6 GHz radio while keeping WPA2/WPA3 transition mode on 2.4 and 5. This is a behavior change from older controllers where dual-SSID workarounds were required.

Minimal working POST shape:

```json
{
  "name": "MyWifi",
  "x_passphrase": "...",
  "security": "wpapsk",
  "wpa_mode": "wpa2",
  "wpa_enc": "ccmp",
  "wpa3_support": true,
  "wpa3_transition": true,
  "pmf_mode": "optional",
  "wlan_bands": ["2g", "5g", "6g"],
  "networkconf_id": "<network id>",
  "ap_group_ids": ["<ap group id>"],
  "enabled": true,
  "is_guest": false,
  "hide_ssid": false
}
```

### 3. Static routes use fully-hyphenated field names

The field naming convention here is **inconsistent with everything else** in the API:

- `static-route_network` (not `static_network`, not `staticRouteNetwork`)
- `static-route_nexthop`
- `static-route_type` (value: `"nexthop-route"`)
- `static-route_distance` (integer, default 1)

The hyphen sits between `static` and `route`, then there's an underscore before the suffix. Mixing `static_network` with `static-route_type` (which the original udr7_configure.py did) returns `api.err.InvalidDestinationNetwork`.

There is **no v2 routing endpoint** — POSTs go through:

```http
POST /proxy/network/api/s/default/rest/routing
```

```json
{
  "name": "Tailscale CGNAT via Pi",
  "type": "static-route",
  "enabled": true,
  "static-route_type": "nexthop-route",
  "static-route_network": "100.64.0.0/10",
  "static-route_nexthop": "192.168.50.250",
  "static-route_distance": 1
}
```

### 4. Firewall rules require `src_networkconf_type` and `dst_networkconf_type`

When a firewall rule references a network by ID (for inter-VLAN isolation, IoT-blocking, etc.), both type fields must be set to the literal string `"NETv4"`. Without them the API returns `api.err.FirewallRuleNetworkConfTypeRequired`.

`NETv4` here refers to "the IPv4 subnet of the named network." There is presumably a `NETv6` variant for IPv6 — not tested.

```json
{
  "name": "Block IoT to Default",
  "ruleset": "LAN_IN",
  "rule_index": 20000,
  "action": "drop",
  "protocol": "all",
  "src_networkconf_id":   "<IoT network _id>",
  "src_networkconf_type": "NETv4",
  "dst_networkconf_id":   "<Default network _id>",
  "dst_networkconf_type": "NETv4",
  "enabled": true,
  "logging": false,
  "state_new":  false,
  "state_established": false,
  "state_invalid": false,
  "state_related": false
}
```

### 5. Traffic Rules schema for source-IP matching is broken/undocumented

The legacy approach (`target_devices: [{client_mac: "", ...}]` placeholder + `source.ips` matching) is rejected by Network 10.x with a Spring validation error on `targetDevices[0].clientMac` ("rejected value [Optional[]]"). The UX has been overhauled to be device/client-centric rather than source-IP-centric.

For "match by source IP only, no specific client MAC," the JSON shape is undocumented and inconsistent across releases. Until the new schema is reverse-engineered, treat this as an **unsupported feature on Network 10.x** and emit an advisory in the IR import output telling the user to either:

- Enable Smart Queues (Settings → Internet → WAN → Smart Queues) — handles QoS adaptively without per-IP rules, and is what UDR-7 actually wants you to use
- Or hand-create the rules in the UI (Settings → Security → Traffic Rules)

This is the right call for UDR-7 specifically — Smart Queues is dramatically better than the old TOS-marker scheme it replaces.

### 6. The Default LAN cannot be updated mid-script

This is the single most important rule for any UniFi adapter and the one that bit udr7_configure.py first.

When you PUT a `networkconf` update that changes the subnet of the LAN your script is talking to, **the controller renumbers the interface synchronously**. Your script's existing DHCP lease becomes invalid. Your existing TCP socket dies. The PUT may or may not return a response before the connection drops; treat connection-reset / timeout / max-retries errors as a successful apply, not a failure.

Adapter design rule: **the Default LAN is always the LAST mutation in any apply pass.** Create new VLANs first, write all wlanconf / DHCP / DNS / routing / firewall / etc. through the still-live session, then update Default last. Catch the expected network error and treat it as success.

If the new subnet matches the existing one (idempotent rerun), skip the PUT entirely — there's no point writing a no-op that incurs a session-killing risk.

### 7. UniFi tracks "users" (clients) separately from "DHCP reservations"

A DHCP reservation is **not** a separate object — it's a user record with `use_fixedip: true` and `fixed_ip: "<ip>"` set. Every Wi-Fi/wired client UniFi has ever seen lives in the same `/rest/user` table. To create a reservation:

- If the MAC already has a user record (because the device has previously been on the LAN), PUT a merge that flips `use_fixedip` to `true` and sets `fixed_ip`
- If not, POST a new user record with `mac`, `name`, `use_fixedip`, `fixed_ip`, and `network_id`

The dedup key is the MAC address (lowercase). Don't dedup by name — multiple devices can share a name.

### 8. Idempotency: PUT-merge, don't POST blind

For every object class (network, wlanconf, user, routing, firewallrule, static-dns), the safe pattern is:

1. GET the full list
2. Build a map by human key (name / MAC / hostname)
3. For each desired object:
   - If the key already exists: `merged = {**existing, **desired_fields}` and PUT to `/rest/<class>/<_id>`
   - If not: POST to `/rest/<class>`

The `{**existing, **desired_fields}` merge preserves UniFi's internal fields (like `_id`, `site_id`, `external_id`, `setting_preference`, hidden flags) which the API will reject the PUT for if they're missing.

---

## Endpoint reference (write-side, what the adapter actually needs)

| What | Method | Path |
|---|---|---|
| List networks | GET | `/proxy/network/api/s/default/rest/networkconf` |
| Create network | POST | `/proxy/network/api/s/default/rest/networkconf` |
| Update network | PUT | `/proxy/network/api/s/default/rest/networkconf/<_id>` |
| List WLANs | GET | `/proxy/network/api/s/default/rest/wlanconf` |
| Create WLAN | POST | `/proxy/network/api/s/default/rest/wlanconf` |
| Update WLAN | PUT | `/proxy/network/api/s/default/rest/wlanconf/<_id>` |
| List AP groups | GET | `/proxy/network/v2/api/site/default/apgroups` |
| List users (incl. reservations) | GET | `/proxy/network/api/s/default/rest/user` |
| Create user/reservation | POST | `/proxy/network/api/s/default/rest/user` |
| Update user/reservation | PUT | `/proxy/network/api/s/default/rest/user/<_id>` |
| List static DNS records | GET | `/proxy/network/v2/api/site/default/static-dns` |
| Create static DNS record | POST | `/proxy/network/v2/api/site/default/static-dns` |
| List static routes | GET | `/proxy/network/api/s/default/rest/routing` |
| Create static route | POST | `/proxy/network/api/s/default/rest/routing` |
| Delete static route | DELETE | `/proxy/network/api/s/default/rest/routing/<_id>` |
| List firewall rules | GET | `/proxy/network/api/s/default/rest/firewallrule` |
| Create firewall rule | POST | `/proxy/network/api/s/default/rest/firewallrule` |
| List traffic rules | GET | `/proxy/network/v2/api/site/default/trafficrules` |
| Application version | GET | `/proxy/network/integration/v1/info` |
| List sites | GET | `/proxy/network/integration/v1/sites` |

All endpoints support `X-API-KEY` auth.

---

## Things still to figure out (open questions for the adapter)

- **Force-DNS through router** — the API path for "block outbound DNS to non-router DNS servers" is unstable across versions and a bad apply breaks DNS for every LAN client. udr7_configure.py marks this as a manual UI step. Adapter should probably do the same for v0.1, then revisit once a stable shape is found.
- **Port forwards** — the legacy `/rest/portforward` endpoint exists but the schema for the newer "Port Forwarding" UI section is different from older controllers. Untested in this migration (the source Asus had zero port forwards).
- **WAN configuration** — udr7_configure.py didn't touch WAN at all. The adapter will need to handle DHCP / PPPoE / static WAN modes, MAC clone, and MTU. Schema TBD.
- **Traffic rules new schema** — for IP-source matching without a specific client MAC. Reverse engineering needed; capturing browser HTTP traffic from the UniFi admin UI is the fastest path.
- **IPv6** — completely untested. The presence of `NETv4` in firewall rule fields suggests `NETv6` exists too. The `dhcpd_v6_*` family of fields exists on networkconf but their semantics weren't exercised.
- **Multi-site** — udr7_configure.py hardcodes `default` as the site name. Real adapter needs to enumerate `/proxy/network/api/self/sites` and apply to a chosen site.
- **AP-specific WLAN binding** — udr7_configure.py binds new SSIDs to all known AP groups. A more sophisticated adapter would respect a per-WLAN AP-group spec from the IR.

## Source

The script that became the seed for this adapter is `udr7_configure.py`, kept in the user's local working tree (not in this repo). All seven lessons above came from real `api.err.*` failures during the 2026-04-09 swap, fixed live, and then re-verified by a clean idempotent rerun.

When the actual adapter gets written, this doc should be the starting point. Update it whenever a new lesson is learned.
