## IP Addressing
 
| Site | VLAN | Purpose | Network | Mask |
|---|---|---|---|---|
| HQ | 10 | Sales | 10.1.1.0 | /24 |
| HQ | 20 | Engineering | 20.1.1.0 | /24 |
| HQ | 30 | Exec | 30.1.1.0 | /24 |
| HQ | 40 | Servers | 40.1.1.0 | /24 (sized for future DC growth; 4 servers currently at .10/.20/.30/.40) |
| HQ | 100 | Mgmt | 100.1.1.0 | /28 |
| BR1 | 10 | Sales | 10.2.1.0 | /24 |
| BR1 | 20 | Engineering | 20.2.1.0 | /24 |
| BR1 | 100 | Mgmt | 100.2.1.0 | /30 |
| BR2 | 10 | Sales | 10.2.2.0 | /24 |
| BR2 | 30 | Exec | 30.2.2.0 | /24 |
| BR2 | 100 | Mgmt | 100.2.2.0 | /30 |
 
**WAN / Transit Links**
 
| Link | Network | Mask |
|---|---|---|
| Core1 ↔ HQ-RTR | 200.1.1.0 | /30 |
| HQ-RTR ↔ BR1-RTR (serial) | 200.2.1.0 | /30 |
| HQ-RTR ↔ BR2-RTR (serial) | 200.2.2.0 | /30 |
 
**Loopbacks (Router Management)**
 
| Device | Address | Mask |
|---|---|---|
| HQ-RTR | 1.1.1.1 | /32 |
| BR1-RTR | 3.3.3.3 | /32 |
| BR2-RTR | 2.2.2.2 | /32 |
 
