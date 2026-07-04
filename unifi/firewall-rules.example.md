# UniFi Inter-VLAN Firewall Rules (Sanitized Example)

The rule set as configured on the UniFi gateway, in evaluation order. This is the
implementation of [`docs/segmentation-policy.md`](../docs/segmentation-policy.md). If the two
ever disagree, the policy document wins and the rules get fixed.

All subnets, VLAN IDs, and addresses are placeholders. Network objects/groups referenced below
(`ALL_PRIVATE`, `IOT_NET`, etc.) are UniFi "IP Group" objects so rules read as intent, not
addresses.

## Groups

| Group | Members |
|---|---|
| `ALL_PRIVATE` | `10.20.0.0/16` (every internal segment) |
| `MGMT_NET` | `10.20.10.0/24` |
| `TRUSTED_NET` | `10.20.20.0/24` |
| `IOT_NET` | `10.20.30.0/24` |
| `SERVERS_NET` | `10.20.40.0/24` |
| `DMZ_NET` | `10.20.60.0/24` |
| `GUEST_NET` | `10.20.70.0/24` |
| `PIHOLE_HOST` | `10.20.40.53/32` |
| `DMZ_MEDIA_HOST` | `10.20.60.101/32` (the one self-hosted realtime media service) |

## LAN In rules (inter-VLAN), in order

Stateful: established/related return traffic is always permitted; these rules govern who may *initiate*.

| # | Name | Action | Source | Destination | Port/Proto | Note |
|---|---|---|---|---|---|---|
| 1 | Allow established/related | ACCEPT | any | any | state match | Return traffic for existing sessions |
| 2 | Trusted → Management admin | ACCEPT | `TRUSTED_NET` | `MGMT_NET` | 443/tcp, 22/tcp | Only path to network-gear UIs |
| 3 | Trusted → Servers | ACCEPT | `TRUSTED_NET` | `SERVERS_NET` | any | Personal devices use the services |
| 4 | Trusted → IoT cast/control | ACCEPT | `TRUSTED_NET` | `IOT_NET` | app-specific ports | Casting/control only; paired with mDNS reflection |
| 5 | IoT → Pi-hole DNS | ACCEPT | `IOT_NET` | `PIHOLE_HOST` | 53/tcp+udp | The single IoT→Servers allow |
| 6 | **Drop IoT → private** | DROP (log) | `IOT_NET` | `ALL_PRIVATE` | any | Logged; a probing smart plug is signal |
| 7 | **Drop Guest → private** | DROP | `GUEST_NET` | `ALL_PRIVATE` | any | Guests get internet only |
| 8 | **Drop DMZ → private** | DROP (log) | `DMZ_NET` | `ALL_PRIVATE` | any | DMZ initiates into nothing, not even DNS |
| 9 | **Drop private → DMZ** | DROP | `ALL_PRIVATE` | `DMZ_NET` | any | No internal segment initiates into the DMZ; admin rides the mesh VPN |
| 10 | **Default: drop inter-VLAN** | DROP | `ALL_PRIVATE` | `ALL_PRIVATE` | any | The backstop for anything not matched above |

> Rules 6–9 are technically redundant with rule 10. They exist anyway: (a) the high-risk drops
> get their own hit counters and logs, and (b) if someone later adds a careless broad allow
> below them, the specific drops still hold for the segments that matter most.

## WAN In (port forwards)

| # | Name | External | Forward to | Note |
|---|---|---|---|---|
| 1 | Realtime media (TCP) | `<tcp-port>/tcp` | `DMZ_MEDIA_HOST:<tcp-port>` | Real-time media can't ride the HTTP tunnel |
| 2 | Realtime media (UDP) | `<udp-port>/udp` | `DMZ_MEDIA_HOST:<udp-port>` | Single muxed UDP port, not a range |

That's the complete inbound surface. Every web-facing service is published via outbound-only
tunnels instead (zero inbound rules); see the pattern in my
[nextcloud-cloudflare-tunnel](https://github.com/brockharries/nextcloud-cloudflare-tunnel) repo.

## Notes on operating this

- **Name rules after intent, not mechanics** ("Drop IoT → private", not "Rule 14"). Six months
  later, the name is the documentation you'll actually read.
- **Log the interesting denials only.** Logging every drop buries the signal; logging IoT/DMZ
  drops toward internal segments *is* the signal.
- **Test from the untrusted side.** After a change, verify from a device on IoT/Guest/DMZ that
  the things that should fail still fail. A segmentation rule set that's only ever tested from
  the trusted side is untested.
