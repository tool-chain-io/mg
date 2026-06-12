# Scenario-1 Connectivity Discovery — `az` CLI Runbook

> **Purpose:** turn the Istio scenario-1 requirement — *old cluster A must reach new cluster B's
> internal gateway LB on TCP 443* (see [istio.md](istio.md) §3, [cluster-migration-methods-eval.md](cluster-migration-methods-eval.md))
> — into concrete IPs / CIDRs / route facts, so **cloud-ops** and the **firewall team** get an
> unambiguous ask. **Every command here is read-only (`list`/`show`)** — nothing mutates.
>
> Work through the blocks in order; each block's output feeds the next.
>
> **Recap of the path** (from the earlier architecture discussion): `A sidecar ──TCP 443──► Azure
> internal LB (NEW_LB_IP, L4 passthrough) ──► gateway Service ──► Istio gateway Envoy pods (terminate
> TLS, route) ──► serviceA`. The only thing A ever talks to in B is that **L4 internal LB**. Direction
> is **unidirectional A→B**.

---

## 0. Pin the two clusters

```bash
az account show -o table                       # confirm correct subscription/tenant
az aks list -o table                           # find names + resource groups
RG_OLD=<rg>; AKS_OLD=<old-cluster>
RG_NEW=<rg>; AKS_NEW=<new-cluster>
```

---

## 1. Capture each cluster's network identity

Also answers the **CNI gating question** (Azure CNI vs Cilium) and, critically, the **source-IP
behaviour** that determines what B's NSG/firewall must allow.

```bash
for pair in "$RG_OLD $AKS_OLD" "$RG_NEW $AKS_NEW"; do set -- $pair
  echo "=== $2 ==="
  az aks show -g "$1" -n "$2" --query "{ \
    nodeRG:nodeResourceGroup, \
    networkPlugin:networkProfile.networkPlugin, \
    pluginMode:networkProfile.networkPluginMode, \
    dataplane:networkProfile.networkDataplane, \
    outboundType:networkProfile.outboundType, \
    podCidr:networkProfile.podCidr, \
    serviceCidr:networkProfile.serviceCidr, \
    privateCluster:apiServerAccessProfile.enablePrivateCluster, \
    nodeSubnets:agentPoolProfiles[].vnetSubnetId }" -o jsonc
done
```

**What each field decides:**

| Field | Decides |
|---|---|
| `networkPlugin` / `dataplane` | `azure`/`kubenet`/`none`; `dataplane=cilium` ⇒ Azure CNI Powered by Cilium (the Cilium-Mesh gate) |
| `networkPlugin` + `pluginMode` | **The source IP B must allow** (see below) |
| `outboundType` | `loadBalancer` / `managedNATGateway` / `userAssignedNATGateway` / `userDefinedRouting` — whether a firewall is in path |
| `podCidr` / `serviceCidr` | Overlap checks; the pod source range when `azure`-non-overlay |

**Source-IP rule (the most common firewall-ask mistake):**

- `azure` **without** overlay → pods have real VNet IPs ⇒ **source = OLD pod subnet CIDR**.
- `overlay` or `kubenet` → pod traffic SNATs to the node IP ⇒ **source = OLD node subnet CIDR**.
- ⚠ **The NAT-GW / public egress IP is IRRELEVANT for A→B.** Peered traffic to an *internal* LB stays
  private and is **not** SNAT'd to your internet egress IP. The public egress IP only matters if a UDR
  forces this traffic through a hub firewall (step 4).

---

## 2. Resolve the OLD source subnet → the prefix to allowlist

```bash
OLD_SUBNET_ID=$(az aks show -g "$RG_OLD" -n "$AKS_OLD" \
  --query "agentPoolProfiles[0].vnetSubnetId" -o tsv)
az network vnet subnet show --ids "$OLD_SUBNET_ID" \
  --query "{subnet:name, cidr:addressPrefix, vnet:id, routeTable:routeTable.id, nsg:networkSecurityGroup.id}" -o jsonc
```

`cidr` (plus the pod CIDR from step 1 if `azure`/non-overlay) is your **`OLD_EGRESS` source** — hand
the firewall team the **CIDR**, not a guessed single IP.

---

## 3. Resolve the NEW gateway subnet → the destination IP

The internal LB doesn't exist until the `Gateway` is applied. Two clean options:

```bash
NEW_SUBNET_ID=$(az aks show -g "$RG_NEW" -n "$AKS_NEW" \
  --query "agentPoolProfiles[0].vnetSubnetId" -o tsv)
az network vnet subnet show --ids "$NEW_SUBNET_ID" \
  --query "{subnet:name, cidr:addressPrefix, nsg:networkSecurityGroup.id}" -o jsonc
```

- **Preferred:** pre-allocate a **static private IP** for the internal LB from this subnet (set via the
  gateway Service annotation / `loadBalancerIP`) so the firewall ask is **one concrete `NEW_LB_IP`**.
- Otherwise allowlist the **gateway subnet CIDR** as destination.

If the gateway LB already exists, read its IP from the node RG:

```bash
az network lb list -g $(az aks show -g "$RG_NEW" -n "$AKS_NEW" --query nodeResourceGroup -o tsv) \
  --query "[].frontendIpConfigurations[?privateIpAddress!=null].privateIpAddress" -o tsv
```

---

## 4. Peering + routing — *is there a firewall in the path at all?*

```bash
# Peering state must be Connected BOTH directions
az network vnet peering list --ids "$(az network vnet subnet show --ids "$OLD_SUBNET_ID" --query 'id' -o tsv | sed 's#/subnets/.*##')" \
  --query "[].{name:name, state:peeringState, fwd:allowForwardedTraffic, remote:remoteVirtualNetwork.id}" -o jsonc

# UDR on the OLD node subnet — does any route send the NEW prefix to a VirtualAppliance (= Azure Firewall)?
RT_ID=$(az network vnet subnet show --ids "$OLD_SUBNET_ID" --query routeTable.id -o tsv)
[ -n "$RT_ID" ] && az network route-table route list --ids "$RT_ID" \
  --query "[].{prefix:addressPrefix, nextHop:nextHopType, ip:nextHopIpAddress}" -o jsonc
```

- `nextHopType=VirtualAppliance` for the NEW prefix (or `0.0.0.0/0`) ⇒ traffic **transits a firewall**;
  the firewall team owns an explicit rule, and **the source B sees becomes the firewall's private SNAT
  IP** (ask them for it).
- No matching UDR ⇒ direct peered delivery; **no firewall team needed** for A→B, only the NSG (step 5).

---

## 5. Existing NSG state on the NEW gateway subnet

```bash
NEW_NSG_ID=$(az network vnet subnet show --ids "$NEW_SUBNET_ID" --query networkSecurityGroup.id -o tsv)
az network nsg rule list --ids "$NEW_NSG_ID" \
  --query "sort_by([].{name:name,prio:priority,dir:direction,access:access,proto:protocol,src:sourceAddressPrefix,dport:destinationPortRange}, &prio)" -o table
```

---

## 6. The parallel blocker — NEW cluster → managed Azure SQL / Postgres

This is the [istio-prompt.md](istio-prompt.md) symptom and often a **separate** firewall / private-
endpoint ask (eval Gap #2):

```bash
az postgres flexible-server list -o table
az sql server list -o table

# Private endpoints + which subnet/VNet they live in:
az network private-endpoint list \
  --query "[].{name:name, subnet:subnet.id, conn:privateLinkServiceConnections[].privateLinkServiceId}" -o jsonc

# Private DNS zones must be linked to the NEW VNet or name resolution fails:
az network private-dns zone list -o table
az network private-dns link vnet list -g <dns-rg> -z privatelink.postgres.database.azure.com -o table

# PG public firewall rules (only relevant if NOT using a private endpoint):
az postgres flexible-server firewall-rule list -g <rg> -n <server> -o table
```

---

## What to hand over

### To cloud-ops (networking / platform)

1. **VNet peering** `Connected` **both** directions, OLD VNet ↔ NEW VNet (with `allowForwardedTraffic`
   if a firewall NVA is in path) — *step 4*.
2. **NSG rule on the NEW gateway subnet:** `Inbound Allow TCP 443` from **`<OLD source CIDR — step 2>`**
   → **`<NEW_LB_IP or gateway-subnet CIDR — step 3>`**, priority above any deny — *step 5 shows current
   state*.
3. **Reserved static private IP** for the new internal gateway LB (so the rule targets one IP).
4. **DB side (step 6):** private endpoint for the managed DB reachable from the NEW node subnet **+** the
   `privatelink.*` private-DNS zone **linked to the NEW VNet**.

### To the firewall team — *only if step 4 shows `VirtualAppliance` / UDR in path*

1. **Network rule:** source `<OLD source CIDR>` → destination `<NEW_LB_IP>` TCP **443**.
2. Their firewall's **private SNAT IP** — becomes the source the NEW NSG must allow (feeds cloud-ops
   item 2).
3. If DB egress also transits the firewall: source `<NEW node/pod CIDR>` → DB private endpoint / FQDN,
   port **5432** (PostgreSQL) or **1433** (SQL).

> **Direction:** unidirectional **A→B** — no B→A rule is needed (responses ride the established TCP
> connection).

---

## Fill-in summary (paste the discovered values here)

| Item | Value | From |
|---|---|---|
| OLD cluster CNI / dataplane | | step 1 |
| OLD `outboundType` | | step 1 |
| **OLD source CIDR** (pod or node subnet) | | steps 1–2 |
| NEW gateway subnet CIDR | | step 3 |
| **NEW_LB_IP** (reserved static) | | step 3 |
| Peering state (both directions) | | step 4 |
| Firewall in path? (UDR → VirtualAppliance) | | step 4 |
| Firewall SNAT IP (if applicable) | | firewall team |
| Existing NEW-NSG 443 rules | | step 5 |
| DB private endpoint + DNS-zone link to NEW VNet | | step 6 |

---

### Notes / provenance

- `az` field names (`networkProfile.networkDataplane`, `agentPoolProfiles[].vnetSubnetId`, etc.) are
  current stable CLI as of writing; if a query returns `null`, drop the `--query` and inspect the full
  object — Azure occasionally renames/relocates fields.
- The **source-IP determination** (pod vs node CIDR vs firewall SNAT IP) is the one genuinely
  topology-dependent decision — confirm it from live output before sending the ask; getting it wrong
  is the usual cause of a "correct-looking" firewall rule that still blocks the traffic.
- This runbook covers the **network/firewall** prerequisites only. Mesh-side config (ServiceEntry,
  DestinationRule SNI, weighted VirtualService, cert placement) is in [istio.md](istio.md) §2–3.
