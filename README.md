# Something helpful to run DPDK inside a container

## Motivation
- As a NFV Infra developer, my mission is providing well-shaped kubernetes infra for telco applications like 5G things such as vRAN and UPF. Someguys say that now is the age of kubernetes and cloud native for 5G, but many telcos and vendors are still running their time critical packet processing applications inside VM and they just started to shift from VM to Container. For serveral months, I struggled to run DPDK inside a container based on kubernetes infra. Eventually, I found lots of things we("infra manager/developer, DPDK application developer, operator") have to know to run DPDK inside a container. So I'm ganna share it. Take a breath, here we go.


