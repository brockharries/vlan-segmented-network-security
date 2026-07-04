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
| **Trusted** | ✅ admin UIs | n/a | ✅ cast/control¹ | ✅ use services | ❌ ² | ❌ | ✅ |
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
   most-likely-compromised segment; an "admin convenience" hole from Servers→DMZ becomes a
   pivot path pointing at the segment with the family photos. This rule has personally
   inconvenienced me mid-deployment. It stays.
3. **IoT → Servers, DNS only:** one allow, to one host (Pi-hole), on port 53. IoT devices get
   name resolution and filtering; they do not get to browse the server segment.
4. **DMZ initiates into nothing internal, not even DNS.** DMZ hosts use public resolvers.
   Name resolution is reconnaissance; a compromised public host doesn't get to enumerate the
   internal namespace.
5. **Internet → DMZ, two ports:** one TCP + one UDP forward, to a single DMZ host, for one
   realtime-media workload that genuinely cannot ride an outbound tunnel. Every web-facing
   service instead uses outbound-only tunnels (zero inbound ports). Each forward exists
   because it survived the question "can this ride the tunnel instead?"

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
