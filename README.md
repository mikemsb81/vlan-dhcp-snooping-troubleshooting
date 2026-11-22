# DHCP Snooping Configuration and Troubleshooting

Cisco Packet Tracer lab by Michael Bessard.

## Overview

Enabled DHCP snooping for VLANs 10 and 20 on two switches, configured trusted ports toward the legitimate DHCP server, and verified client leases and snooping bindings. Deliberately removed trust from SW2's uplink to reproduce a DHCP failure, then restored trust and verified recovery.

**Lab file:** [vlan-dhcp-snooping-troubleshooting.pkt](vlan-dhcp-snooping-troubleshooting.pkt)

## Topology and addressing

| Connection | Role |
|---|---|
| R1 G0/0 to SW1 G0/2 | Router-on-a-stick trunk for VLANs 10 and 20 |
| SW1 G0/1 to SW2 G0/1 | Inter-switch trunk for VLANs 10 and 20 |
| PC1 to SW1 Fa0/1 | VLAN10 DHCP client |
| PC2 to SW1 Fa0/2 | VLAN20 static client |
| PC3 to SW2 Fa0/1 | VLAN10 static client |
| PC4 to SW2 Fa0/2 | VLAN20 DHCP client |

R1 is the legitimate DHCP server and provides gateways 192.168.10.1/24 and 192.168.20.1/24 through subinterfaces G0/0.10 and G0/0.20. The existing DHCP pools exclude .1 through .20 in each subnet.

| Client | VLAN | Baseline address | Default gateway |
|---|---|---|---|
| PC1 | 10, USERS | 192.168.10.21/24, DHCP | 192.168.10.1 |
| PC2 | 20, SERVERS | 192.168.20.12/24, static | 192.168.20.1 |
| PC3 | 10, USERS | 192.168.10.13/24, static | 192.168.10.1 |
| PC4 | 20, SERVERS | 192.168.20.21/24, DHCP | 192.168.20.1 |

## Trust design

DHCP snooping checks incoming DHCP traffic. Server replies entering an untrusted port are blocked. Trust follows the path from the legitimate server into each switch; it is not automatically required on both ends of every trunk.

| Switch port | Trust | Reason |
|---|---|---|
| SW1 G0/2 | Trusted | Receives legitimate server replies from R1 |
| SW1 G0/1 | Untrusted | Receives downstream client requests; server replies leave SW1 here |
| SW2 G0/1 | Trusted | Receives legitimate server replies through SW1 |
| PC-facing ports | Untrusted | Endpoints should not supply DHCP server replies |

Trust is a port policy, not cryptographic authentication of a server. Only ports with a justified server-facing role should be trusted.

## Configuration

On SW1:

```text
enable
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
interface gigabitEthernet0/2
 ip dhcp snooping trust
exit
end
show ip dhcp snooping
```

On SW2:

```text
enable
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
interface gigabitEthernet0/1
 ip dhcp snooping trust
exit
end
show ip dhcp snooping
```

Option 82 insertion was disabled for this exercise. Both switches displayed hardware-address verification enabled. DHCP rate limits were not configured.

## Baseline verification

On PC1 and PC4:

```text
ipconfig /release
ipconfig /renew
ipconfig
```

Both clients obtained the correct addresses, /24 subnet masks, and gateways with snooping enabled.

On both switches:

```text
show ip dhcp snooping binding
```

Observed bindings:

| Switch | Client MAC | Address | VLAN | Interface |
|---|---|---|---|---|
| SW1 | 00:03:E4:85:EB:AA | 192.168.10.21 | 10 | FastEthernet0/1 |
| SW2 | 00:02:16:C9:70:62 | 192.168.20.21 | 20 | FastEthernet0/2 |

Each captured switch table contained one binding. The displayed lease field was 0 seconds; no lease-duration behavior was tested. Static clients PC2 and PC3 were not expected to create dynamic DHCP snooping bindings through a DHCP exchange.

## Controlled fault: untrusted uplink

Removed trust from SW2 G0/1 only:

```text
configure terminal
interface gigabitEthernet0/1
 no ip dhcp snooping trust
end
show ip dhcp snooping
```

The output showed G0/1 as untrusted while snooping remained enabled for VLANs 10 and 20.

On PC4:

```text
ipconfig /release
ipconfig /renew
ipconfig
```

PC4 failed to obtain a DHCP lease and showed the self-assigned address **169.254.112.98**, mask **255.255.0.0**, and default gateway **0.0.0.0**. This link-local address was not allocated by R1.

The uplink trust change caused legitimate server replies to arrive on an untrusted port. Releasing the existing lease exposed the failure; an already configured client can otherwise temporarily hide a DHCP acquisition problem. Packet-level drop counters and a packet capture were not collected.

## Recovery

Restored trust on SW2:

```text
configure terminal
interface gigabitEthernet0/1
 ip dhcp snooping trust
end
```

Then on PC4:

```text
ipconfig /renew
ipconfig
ping 192.168.10.21
```

PC4 again showed **192.168.20.21/24**, gateway **192.168.20.1**. The ping to PC1 returned **four replies with 0% loss**.

On SW2:

```text
show ip dhcp snooping binding
```

The binding for PC4 appeared on VLAN20, FastEthernet0/2 again.

Saved the final configuration on both switches:

```text
copy running-config startup-config
```

Saved the Packet Tracer project with the corrected trust configuration.

## Observed results

| Stage | Check | Result |
|---|---|---|
| Baseline | Snooping enabled on both switches | VLANs 10 and 20 |
| Baseline | Trusted server-facing ports | SW1 G0/2 and SW2 G0/1 |
| Baseline | PC1 and PC4 DHCP renewal | Correct addresses and gateways |
| Baseline | Client bindings | One local client binding captured on each switch |
| Fault | SW2 G0/1 trust | Untrusted |
| Fault | PC4 DHCP acquisition | Failed; 169.254.112.98/16, no gateway |
| Recovery | PC4 DHCP settings | 192.168.20.21/24, gateway 192.168.20.1 |
| Recovery | PC4 to PC1 | 4/4 ping replies |
| Recovery | SW2 client binding | PC4 on VLAN20, Fa0/2 |

## Screenshot evidence

| Screenshot | Evidence |
|---|---|
| [01-sw1-snooping-baseline.png](01-sw1-snooping-baseline.png) | SW1 snooping configuration and trusted G0/2 |
| [02-sw2-snooping-baseline.png](02-sw2-snooping-baseline.png) | SW2 snooping configuration and trusted G0/1 |
| [03-pc1-dhcp-baseline.png](03-pc1-dhcp-baseline.png) | PC1 renewal and baseline settings |
| [04-pc4-dhcp-baseline.png](04-pc4-dhcp-baseline.png) | PC4 renewal and baseline settings |
| [05-sw1-client-binding.png](05-sw1-client-binding.png) | PC1 binding on SW1 |
| [06-sw2-client-binding.png](06-sw2-client-binding.png) | PC4 binding on SW2 |
| [07-sw2-untrusted-uplink.png](07-sw2-untrusted-uplink.png) | Uplink trust fault |
| [08-pc4-dhcp-failure.png](08-pc4-dhcp-failure.png) | Self-assigned address after DHCP failure |
| [09-pc4-recovery-verified.png](09-pc4-recovery-verified.png) | Correct DHCP settings and successful cross-VLAN ping |
| [10-sw2-binding-restored.png](10-sw2-binding-restored.png) | PC4 binding after recovery |

## Lessons learned and scope

- DHCP snooping can help block unauthorized DHCP server replies entering untrusted ports.
- Incorrect trust placement can also block a legitimate DHCP server.
- Port trust is directional: evaluate traffic entering each switch.
- Inspect client settings and switch bindings together when diagnosing DHCP failures.
- A 169.254 address is a useful clue that DHCP acquisition failed; it does not identify the cause by itself.
- Verify both lease acquisition and connectivity after restoring the configuration.

This lab demonstrated legitimate DHCP operation, a deliberate uplink trust fault, and recovery. No rogue DHCP server was deployed, so rogue-server blocking was not directly tested. DHCP starvation, rate limiting, persistent binding storage, Dynamic ARP Inspection, and IP Source Guard were outside the exercise.
