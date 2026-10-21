# VLAN Inventory

## HQ VLANs

| VLAN | Name              | HQ Subnet     | Purpose / Gateway convention                                                              |
|------|-------------------|---------------|-------------------------------------------------------------------------------------------|
| 10   | MANAGEMENT        | 10.10.10.0/24 | Network device management. If HSRP works: VIP .1, CORE-1 .2, CORE-2 .3, access switches .11-.15 |
| 20   | IT-ADMIN          | 10.10.20.0/24 | Administrator workstations; DHCP .100-.199 or static                                      |
| 30   | STAFF             | 10.10.30.0/24 | Employee workstations                                                                     |
| 40   | FINANCE           | 10.10.40.0/24 | Finance endpoints; restricted from STAFF                                                  |
| 50   | SERVERS           | 10.10.50.0/24 | DNS/DHCP server .10, internal app server .11                                              |
| 60   | DMZ               | 10.10.60.0/24 | Public-facing web server .10                                                              |
| 70   | GUEST             | 10.10.70.0/24 | Guest endpoints; internet-only policy                                                     |
| 80   | LOGGING           | 10.10.80.0/24 | Optional log server .10                                                                   |
| 998  | NATIVE-BLACKHOLE  | No subnet     | Native VLAN on trunks; no endpoint access ports                                           |
| 999  | PARKING           | No subnet     | Unused access ports; shut down                                                            |

## Branch VLANs

| VLAN | Name  | Subnet        | Gateway                              |
|------|-------|---------------|--------------------------------------|
| 10   | MGMT  | 10.20.10.0/24 | R-BRANCH .1; access switch .2        |
| 30   | STAFF | 10.20.30.0/24 | R-BRANCH subinterface .1             |
| 70   | GUEST | 10.20.70.0/24 | R-BRANCH subinterface .1             |

## Notes

- HSRP is a core extension. If it doesn't pass its test, use one SVI gateway per VLAN and record the single point of failure in `feature-limitations.md`.
- VLANs 998 and 999 have no subnet because they carry no routed traffic.
