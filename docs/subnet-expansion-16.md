# Subnet Expansion: /24 → /16 (10.242.0.0/16)

## Overview

This change expands the WireGuard VPN internal subnet from a `/24` (`10.7.0.0/24`)
to a `/16` (`10.242.0.0/16`). The new address space supports up to **65,533 VPN
clients**, compared to the previous maximum of **253 clients**.

---

## Motivation

The original `/24` subnet caps client capacity at 253 peers (`.2` through `.254`,
with `.1` reserved for the server and `.255` as broadcast). For deployments
requiring hundreds or thousands of concurrent VPN users, this is a hard ceiling
that cannot be worked around without a schema change. A `/16` provides 256× more
address space with no changes to the WireGuard protocol or tunnel behaviour.

---

## Address Allocation

| Address | Role |
|---------|------|
| `10.242.0.0` | Network address (unusable) |
| `10.242.0.1` | VPN server gateway |
| `10.242.0.2` – `10.242.0.254` | Clients (first block, 253 addresses) |
| `10.242.X.0` | Network address of each /24 block — skipped |
| `10.242.X.255` | Broadcast of each /24 block — skipped |
| `10.242.1.1` – `10.242.255.254` | Clients (remaining blocks, 65,280 addresses) |
| `10.242.255.255` | Subnet broadcast (unusable) |
| **Total client slots** | **65,533** |

---

## Files Changed

### `wireguard-install.sh`

#### 1. Server interface address (line ~774)

The WireGuard server's internal interface address and prefix length.

```diff
-Address = 10.7.0.1/24
+Address = 10.242.0.1/16
```

#### 2. `create_firewall_rules()` — firewalld backend (lines ~786–791)

Four firewalld commands that trust and NAT the VPN subnet.

```diff
-firewall-cmd -q --zone=trusted --add-source=10.7.0.0/24
-firewall-cmd -q --permanent --zone=trusted --add-source=10.7.0.0/24
-firewall-cmd -q --direct --add-rule ipv4 nat POSTROUTING 0 -s 10.7.0.0/24 ! -d 10.7.0.0/24 -j MASQUERADE
-firewall-cmd -q --permanent --direct --add-rule ipv4 nat POSTROUTING 0 -s 10.7.0.0/24 ! -d 10.7.0.0/24 -j MASQUERADE
+firewall-cmd -q --zone=trusted --add-source=10.242.0.0/16
+firewall-cmd -q --permanent --zone=trusted --add-source=10.242.0.0/16
+firewall-cmd -q --direct --add-rule ipv4 nat POSTROUTING 0 -s 10.242.0.0/16 ! -d 10.242.0.0/16 -j MASQUERADE
+firewall-cmd -q --permanent --direct --add-rule ipv4 nat POSTROUTING 0 -s 10.242.0.0/16 ! -d 10.242.0.0/16 -j MASQUERADE
```

#### 3. `create_firewall_rules()` — iptables backend (lines ~813–819)

Four iptables `ExecStart`/`ExecStop` lines written into `wg-iptables.service`.

```diff
-ExecStart=... -t nat -A POSTROUTING -s 10.7.0.0/24 ! -d 10.7.0.0/24 -j MASQUERADE
-ExecStart=... -I FORWARD -s 10.7.0.0/24 -j ACCEPT
-ExecStop=...  -t nat -D POSTROUTING -s 10.7.0.0/24 ! -d 10.7.0.0/24 -j MASQUERADE
-ExecStop=...  -D FORWARD -s 10.7.0.0/24 -j ACCEPT
+ExecStart=... -t nat -A POSTROUTING -s 10.242.0.0/16 ! -d 10.242.0.0/16 -j MASQUERADE
+ExecStart=... -I FORWARD -s 10.242.0.0/16 -j ACCEPT
+ExecStop=...  -t nat -D POSTROUTING -s 10.242.0.0/16 ! -d 10.242.0.0/16 -j MASQUERADE
+ExecStop=...  -D FORWARD -s 10.242.0.0/16 -j ACCEPT
```

#### 4. `remove_firewall_rules()` — firewalld teardown (lines ~842–849)

The grep pattern and five remove commands used during uninstall.

```diff
-grep '\-s 10.7.0.0/24 ! -d 10.7.0.0/24'
-firewall-cmd -q --zone=trusted --remove-source=10.7.0.0/24
-firewall-cmd -q --permanent --zone=trusted --remove-source=10.7.0.0/24
-firewall-cmd -q --direct --remove-rule ... -s 10.7.0.0/24 ! -d 10.7.0.0/24 ...
-firewall-cmd -q --permanent --direct --remove-rule ... -s 10.7.0.0/24 ! -d 10.7.0.0/24 ...
+grep '\-s 10.242.0.0/16 ! -d 10.242.0.0/16'
+firewall-cmd -q --zone=trusted --remove-source=10.242.0.0/16
+firewall-cmd -q --permanent --zone=trusted --remove-source=10.242.0.0/16
+firewall-cmd -q --direct --remove-rule ... -s 10.242.0.0/16 ! -d 10.242.0.0/16 ...
+firewall-cmd -q --permanent --direct --remove-rule ... -s 10.242.0.0/16 ! -d 10.242.0.0/16 ...
```

#### 5. `select_client_ip()` — complete rewrite (lines ~933–944)

This is the most significant logic change. The original function tracked only a
single variable (`octet`, the 4th octet) and iterated it from `2` to `254`.
That design is fundamentally limited to a single `/24` block.

The new implementation tracks **two variables** — `third` (3rd octet) and
`fourth` (4th octet) — and iterates across the entire `/16` address space:

```diff
 select_client_ip() {
-	# available octet. Important to start looking at 2, because 1 is our gateway.
-	octet=2
-	while grep AllowedIPs "$WG_CONF" | cut -d "." -f 4 | cut -d "/" -f 1 | grep -q "^$octet$"; do
-		(( octet++ ))
-	done
-	if [[ "$octet" -eq 255 ]]; then
-		exiterr "253 clients are already configured. The WireGuard internal subnet is full!"
-	fi
+	# Track 3rd and 4th octets for 10.242.0.0/16.
+	# Server is 10.242.0.1, clients start at 10.242.0.2.
+	third=0
+	fourth=2
+	while grep -q "AllowedIPs = 10\.242\.$third\.$fourth/32" "$WG_CONF"; do
+		(( fourth++ ))
+		if [[ "$fourth" -eq 255 ]]; then
+			# Skip .X.255 broadcast; move to next /24 block starting at .1
+			(( third++ ))
+			fourth=1
+		fi
+	done
+	if [[ "$third" -eq 256 ]]; then
+		exiterr "65533 clients are already configured. The WireGuard internal subnet is full!"
+	fi
 }
```

Key design decisions:
- **`fourth` resets to `1` (not `2`) after the first block.** Only `10.242.0.2`
  must start at `.2` because `.0.1` is the server. For all subsequent blocks
  (`.1.x`, `.2.x`, …) the `.X.1` address is a valid client slot.
- **`.X.0` is never assigned.** The iteration never produces `fourth=0` because
  it starts at `2` in the first block and resets to `1` (not `0`) in subsequent
  blocks.
- **`.X.255` is never assigned.** When `fourth` reaches `255` the code
  increments `third` and resets `fourth` to `1`, skipping the broadcast address.
- **Conflict detection** changed from a pipeline of `cut | grep` (which matched
  any peer with the same 4th octet, regardless of 3rd octet) to a direct
  `grep -q` on the full `10.242.X.Y/32` string, which is both correct and more
  efficient.

#### 6. Auto-assign display message (line ~957)

```diff
-echo "Using auto assigned IP address 10.7.0.$octet."
+echo "Using auto assigned IP address 10.242.$third.$fourth."
```

#### 7. Manual IP input block (lines ~961–973)

Updated prompt text, parsing, validation regex, and conflict check.

```diff
-read -rp "Enter IP address for the new client (e.g. 10.7.0.X): " client_ip
-octet=$(printf '%s' "$client_ip" | cut -d "." -f 4)
-until [[ $client_ip =~ ^10\.7\.0\.([2-9]|...|25[0-4])$ ]] \
-    && ! grep AllowedIPs "$WG_CONF" | cut -d "." -f 4 | cut -d "/" -f 1 | grep -q "^$octet$"; do
-    if [[ ! $client_ip =~ ^10\.7\.0\....$ ]]; then
-        echo "Invalid IP address. Must be within the range 10.7.0.2 to 10.7.0.254."
+read -rp "Enter IP address for the new client (e.g. 10.242.0.X): " client_ip
+third=$(printf '%s' "$client_ip" | cut -d "." -f 3)
+fourth=$(printf '%s' "$client_ip" | cut -d "." -f 4)
+until [[ $client_ip =~ ^10\.242\.(0\.([2-9]|...|25[0-4])|([1-9]|...|25[0-5])\.([1-9]|...|25[0-4]))$ ]] \
+    && ! grep -q "AllowedIPs = 10\.242\.$third\.$fourth/32" "$WG_CONF"; do
+    if [[ ! $client_ip =~ ^10\.242\.... ]]; then
+        echo "Invalid IP address. Must be within the range 10.242.0.2 to 10.242.255.254 (excluding .X.0 and .X.255)."
```

The validation regex has two branches:
- `0\.([2-9]|…|25[0-4])` — third octet is `0`, fourth must be `2`–`254`
  (server occupies `.0.1`)
- `([1-9]|…|25[0-5])\.([1-9]|…|25[0-4])` — third octet is `1`–`255`, fourth
  must be `1`–`254` (skip `.X.0` network and `.X.255` broadcast)

#### 8. Server peer `AllowedIPs` template (line ~983)

Written into the server's `wg0.conf` for each new peer.

```diff
-AllowedIPs = 10.7.0.$octet/32
+AllowedIPs = 10.242.$third.$fourth/32
```

#### 9. Client config `Address` template (line ~990)

Written into the client's `.conf` file.

```diff
-Address = 10.7.0.$octet/24
+Address = 10.242.$third.$fourth/16
```

---

## Capacity Comparison

| | Before | After |
|-|--------|-------|
| Subnet | `10.7.0.0/24` | `10.242.0.0/16` |
| Server IP | `10.7.0.1` | `10.242.0.1` |
| Client range | `10.7.0.2` – `10.7.0.254` | `10.242.0.2` – `10.242.255.254` |
| Max clients | **253** | **65,533** |
| Prefix in client config | `/24` | `/16` |

---

## Upgrade / Migration Notes

- **Existing installations are not affected** by this change. The script does not
  modify a running WireGuard server's configuration. The new subnet is only
  applied during a fresh `--auto` or interactive install.
- **Existing clients** created with the old script will retain their `10.7.0.x`
  addresses and will continue to work normally against an old server config.
- **To migrate** an existing deployment to the new subnet, uninstall WireGuard
  (`sudo bash wireguard.sh --uninstall`) and reinstall. All client profiles
  will need to be regenerated and redistributed.

---

## Testing

A test suite (`/tmp/test_subnet.sh`) was run against the modified script and all
10 tests passed:

| # | Test | Result |
|---|------|--------|
| 1 | Server address is `10.242.0.1/16` | PASS |
| 2 | First client auto-assigned `10.242.0.2` | PASS |
| 3 | Second client auto-assigned `10.242.0.3` | PASS |
| 4 | Rollover after filling `.0.2`–`.0.254` → `.1.1` | PASS |
| 5 | Skip in-use `.1.1`, assign `.1.2` | PASS |
| 6 | No legacy `10.7.0.x` references remaining | PASS |
| 7 | No legacy `X.X.X.0/24` subnet references remaining | PASS |
| 8 | Peer `AllowedIPs` template uses `10.242.$third.$fourth/32` | PASS |
| 9 | Client `Address` template uses `10.242.$third.$fourth/16` | PASS |
| 10 | Capacity error message updated to 65,533 | PASS |
