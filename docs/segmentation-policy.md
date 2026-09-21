# Segmentation Policy: Who May Talk to Whom, and Why

The complete inter-VLAN traffic policy. The firewall implements this table; the table is the
source of truth. **Anything not explicitly allowed is denied.** The value of the policy is
as much in the empty cells as the filled ones.

All subnets and VLAN IDs are sanitized placeholders.

## Segments

| Segment | VLAN | Subnet | What lives there |
|---|---|---|---|
| Management | 10 | `10.20.10.0/24` | Gateway, switch, APs (the network's own gear) |
| Trusted | 20 | `10.20.20.0/24` | Personal laptops, phones |
| IoT | 30 | `10.20.30.0/24` | TVs, cameras, plugs, voice assistants |
| Servers | 40 | `10.20.40.0/24` | Hypervisors, NAS, self-hosted services, Pi-hole |
| DMZ | 60 | `10.20.60.0/24` | Internet-facing services (game/chat servers) |
| Guest | 70 | `10.20.70.0/24` | Visitor devices |

## Traffic matrix

Rows = who initiates. Columns = destination. ✅ = explicit allow rule exists. ❌ = default-deny (no rule).
Return traffic for established sessions is always allowed (stateful firewall); this matrix is about who may *initiate*.

| From \ To | Management | Trusted | IoT | Servers | DMZ | Guest | Internet |
|---|---|---|---|---|---|---|---|
| **Management** | n/a | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (updates) |
| **Trusted** | ✅ admin UIs | n/a | ✅ cast/control¹ | ✅ all ports⁶ | ❌ ² | ❌ | ✅ |
| **IoT** | ❌ | ❌ | n/a | ✅ DNS only³ | ❌ | ❌ | ✅ |
| **Servers** | ❌ | ❌ | ❌ | n/a | ❌ ² | ❌ | ✅ |
| **DMZ** | ❌ | ❌ | ❌ | ❌ ⁴ | n/a | ❌ | ✅ |
| **Guest** | ❌ | ❌ | ❌ | ❌ | ❌ | n/a | ✅ |
| **Internet** | ❌ | ❌ | ❌ | ❌ | ✅ 2 ports⁵ | ❌ | n/a |

### The footnotes are the policy

1. **Trusted → IoT, narrow:** casting and device-control protocols only (e.g. mDNS reflection +
   the specific ports the TV/speaker needs). Not "all traffic": the point of an IoT segment
   dies the day it gets a blanket allow.
2. **Nothing internal initiates into the DMZ, including me.** Admin access to DMZ hosts rides
   the identity-based mesh VPN (Tailscale), not an inter-VLAN allow. Rationale: the DMZ is the
   most-likely-compromised segment. A Servers→DMZ allow would not let a DMZ host open
   connections back; the firewall is stateful, so an admin connection permits replies from
   the DMZ host and nothing unsolicited. The reason to refuse it is that it makes "has a
   Servers address" enough to reach the most exposed hosts on the network. Either admin path
   still needs containment, because a session into a compromised host is exposed to that
   host however it arrived (see the limits below). This rule has personally inconvenienced
   me mid-deployment. It stays.
3. **IoT → Servers, DNS only:** one allow, to one host (Pi-hole), on port 53. IoT devices get
   name resolution and filtering; they do not get to browse the server segment.
4. **DMZ initiates into nothing internal, not even DNS.** DMZ hosts use public resolvers.
   Name resolution is reconnaissance; a compromised public host doesn't get to enumerate the
   internal namespace.
5. **Internet → DMZ, two ports:** one TCP + one UDP forward, to a single DMZ host, for one
   realtime-media workload that genuinely cannot ride an outbound tunnel. Every web-facing
   service instead uses outbound-only tunnels (zero inbound ports). Each forward exists
   because it survived the question "can this ride the tunnel instead?"
6. **Trusted → Servers, every port.** This is the one broad allow in the table, and it fails
   the "specific" test in the standing rules below. It exists so personal devices can use
   whatever service I stand up without a rule change. The cost: the firewall does not enforce
   VPN-only administration of the server segment. SSH and the hypervisor UI are reachable
   from any device on Trusted. Listed under the limits too.

## Standing rules

- **Default-deny is the baseline.** New segments start with no allows. New devices start in
  the least-trusted segment that lets them function.
- **Every allow rule is directional, specific, and written down here first.** Rule name in the
  firewall matches the row in this document.
- **Exceptions are time-boxed.** A temporary allow for a migration or debugging session gets
  removed the same week, not "eventually."
- **The blocked *attempt* is signal.** Denials from IoT or DMZ toward internal segments are
  logged; a smart plug probing the NAS is exactly the event this design exists to catch.

## What this policy assumes (honest limits)

- **It contains segments, not hosts.** Two servers on the Servers VLAN can reach each other
  freely; VLAN segmentation does nothing about lateral movement *within* a segment. That's a
  microsegmentation/zero-trust problem, called out in the README's "at scale" section.
- **It trusts the gateway.** Every rule is enforced by one device; a gateway compromise is
  game over. Mitigated by keeping the Management segment reachable only from Trusted, MFA on
  the controller, and timely firmware updates. Not eliminated.
- **VLAN assignment is by SSID/port, not identity.** A hostile device that gets onto the
  Trusted SSID is trusted. At home the Wi-Fi credential is the control; at scale this is why
  802.1X/NAC exists.
- **Trusted → Servers is a blanket allow.** I use the mesh VPN for administration, but the
  firewall would let a compromised laptop on Trusted reach every port on every server. The
  fix is to narrow that rule to the service ports and leave admin ports to the mesh. Not done
  yet.
- **These are IPv4 rules for traffic between VLANs.** Traffic addressed to the gateway itself
  (its management UI, SSH, the DNS and DHCP it offers on each VLAN) is handled by a separate
  rule set and needs its own drops for IoT, Guest and DMZ. IPv6 needs an equivalent rule set,
  or has to stay disabled on these networks. A policy that only exists for IPv4 transit
  traffic has holes on both sides.
- **The admin plane is a second network, and it needs its own policy.** Admin access over the
  mesh VPN means DMZ hosts run a mesh agent, so a compromised DMZ host doesn't just hold a
  DMZ address; it holds a mesh identity. The inter-VLAN firewall never sees that overlay
  path. Containment there depends on the mesh ACLs scoping what each identity may reach, so
  a DMZ node's identity gets the narrowest possible reach, and those ACLs have to be reviewed
  with the same suspicion as the firewall table. Two enforcement points, two policies to keep
  honest.
- **DNS filtering only binds devices that actually use my DNS.** An IoT device with a
  hardcoded resolver baked into its firmware, or one that speaks DNS-over-HTTPS, walks right
  past the Pi-hole. Mitigation: NAT-redirect all outbound port-53 traffic to the Pi-hole
  regardless of the destination the device asked for, and block the known DoH endpoints at
  the firewall. DoH to an endpoint I haven't identified still gets through; that's a real
  gap, narrowed rather than closed.
- **Recursive DNS is a privacy improvement, not privacy.** Unbound keeps any one resolver
  operator from seeing every query, but recursion is unencrypted and the ISP can read it on
  the wire. I used to list a public resolver as Pi-hole's second upstream for availability.
  Pi-hole does not hold a second upstream in reserve, so that leaked ordinary queries. A
  second identical resolver box replaced it.
