
# FortiGate Multi-Site Network Lab (EVE-NG)

A two-site network built in EVE-NG with FortiGate firewalls, designed as the foundation for an IPsec and SD-WAN architecture. This repository documents the design, addressing, configuration, validation and lessons learned.

> **Lab only.** All addresses, credentials and keys in this repo are for a private lab. No vendor images or licenses are included.

---

## Objectives

- Segment each site behind a FortiGate with least-privilege policies
- Provide internet access through NAT on each firewall
- Connect two sites with routed, policy-controlled connectivity
- Prepare the underlay for dual-path IPsec and Fortinet SD-WAN (Phase 2)
- Add an out-of-band IDS for visibility (Phase 3)

## Project status

| Phase | Scope | Status |
|---|---|---|
| 1 | Two sites, static routing, NAT, DHCP, firewall policies | Complete |
| 2 | Dual IPsec tunnels, SD-WAN zone, SLA health checks, failover test | Planned |
| 3 | Third site, out-of-band IDS (Suricata) on a mirrored segment | Planned |

---

## Topology

<img width="843" height="506" alt="image" src="https://github.com/user-attachments/assets/d1c15996-7889-42a9-ada7-905806b2c21e" />

```
   [Cloud0]                                        [Cloud0]
      |                                                |
   port1 (DHCP)                                     port1 (DHCP)
 [ FGT1 ]--------- port2 (172.16.2.0/24) ---------[ FGT2 ]
   port3                                            port3
     |                                                |
 [Switch1]                                        [Server1]
   /    \
[PC1]  [PC2]
```

## Environment

| Item | Detail |
|---|---|
| Platform | EVE-NG Community, running as a VM in VMware |
| Firewalls | FortiGate-VM (KVM build, FortiOS 8.0.0), evaluation license |
| Node resources | 1 vCPU, 2048 MB RAM per FortiGate (evaluation license limit) |
| Endpoints | VPCS (PC1, PC2), Linux node (Server1) |
| Switching | EVE bridge object |
| Internet | EVE Management (Cloud0) bridged to the host network |

## Addressing plan

| Link | Subnet | FGT1 | FGT2 |
|---|---|---|---|
| WAN (port1) | DHCP from lab network | DHCP | DHCP |
| Inter-site (port2) | 172.16.2.0/24 | .1 | .2 |
| Site 1 LAN (port3) | 10.1.1.0/24 | 10.1.1.1 | n/a |
| Site 2 LAN (port3) | 10.1.2.0/24 | n/a | 10.1.2.1 |

| Host | Address | Method |
|---|---|---|
| PC1, PC2 | 10.1.1.100-200 | DHCP from FGT1 |
| Server1 | 10.1.2.10/24, gw 10.1.2.1 | Static |

## Design decisions

- **Static IP on the LAN interface, DHCP server on the firewall.** The FortiGate serves DHCP on the interface it owns, so users need no extra infrastructure.
- **Outbound NAT only on the WAN policy.** Traffic between sites is routed without NAT, so source addresses stay visible for logging and future IDS correlation.
- **Explicit policy in each direction.** FortiGate is default-deny, so each inter-site flow has its own named policy.
- **Direct inter-site link first.** The simplest working baseline comes before IPsec, which makes later failures easier to isolate.

## Firewall policy summary

| # | Name | From | To | Action | NAT |
|---|---|---|---|---|---|
| 1 | LAN-to-WAN | port3 | port1 | Accept | Enabled |
| 2 | LAN-to-Site (other site) | port3 | port2 | Accept | Disabled |
| 3 | Site-to-LAN (other site) | port2 | port3 | Accept | Disabled |

Current policies use `service ALL` for the lab baseline. In Phase 2 these will be narrowed to specific services and security profiles.

Sanitized configurations are in [`configs/`](configs/).

## Validation

| Test | Expected | Result |
|---|---|---|
| FGT1 port1 receives DHCP lease | IP in lab subnet | Pass |
| FGT1 and FGT2 reach each other on port2 | Ping replies | Pass |
| PC1 receives DHCP lease | 10.1.1.x | Pass |
| PC1 to gateway 10.1.1.1 | Ping replies | Pass |
| PC1 to internet (8.8.8.8) | Ping replies via NAT | Pass |
| PC1 to Server1 (10.1.2.10) | Ping replies across sites | Pass |

Evidence (screenshots, command output) is in [`docs/evidence/`](docs/evidence/).

## Lessons learned

Real issues hit during the build and how they were resolved:

| Symptom | Root cause | Fix |
|---|---|---|
| Admin password rejected on first login | Default policy requires 12+ characters with upper, lower, number and symbol | Set a compliant password |
| `License invalid due to exceeding allowed 1 CPUs and 2048 MB RAM` | Node resources above the evaluation license limit | Set the node to 1 vCPU and 2048 MB |
| `fork() failed: Cannot allocate memory` | FortiGate under-provisioned, host short on RAM | Raised node and host memory |
| `dns-server1` parse error in DHCP config | Option only appears when `dns-service specify` is set | Set `dns-service` first, or use the default |
| Clients get no DHCP lease although the server is enabled | `vci-match enable` limits replies to FortiSwitch and FortiExtender clients | `set vci-match disable` on the DHCP server |
| Console would not open from the browser | No telnet handler on the host | Use the HTML5 console |
| "Switch" gave no connectivity | Node was a firewall image, not a switch | Replaced with an EVE bridge object |
| "ISP" had no internet | Node was a VPCS, not a cloud | Replaced with Management (Cloud0) |

## Roadmap

- [ ] Replace the direct link with two IPsec tunnels over separate underlay paths
- [ ] SD-WAN zone with SLA health checks and rule-based steering
- [ ] Failover demonstration with induced latency and loss
- [ ] Add a third site
- [ ] Out-of-band IDS (Suricata) on a mirrored segment
- [ ] Narrow policies to specific services and apply security profiles
- [ ] Syslog forwarding to a monitoring host

## Repository layout

```
.
├── README.md
├── configs/          # sanitized FortiGate configs (no passwords, keys or license data)
├── docs/
│   ├── topology.png
│   └── evidence/     # test screenshots and command output
└── eve/
    └── fortigate-lab.unl   # exported EVE-NG lab file
```

## Reproducing the lab

1. Install EVE-NG and obtain a FortiGate-VM KVM image from Fortinet (not included here).
2. Place the image in `/opt/unetlab/addons/qemu/fortinet-FGT-<version>/virtioa.qcow2` and run `/opt/unetlab/wrappers/unl_wrapper -a fixpermissions`.
3. Import `eve/fortigate-lab.unl` and start the nodes.
4. Apply the configs from `configs/`, replacing placeholders.

## Disclaimer

This is a personal learning and portfolio project built in a private lab. It is not based on any employer or customer environment.
