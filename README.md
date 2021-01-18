# iptables

[![Build Status](https://drone.osshelp.ru/api/badges/ansible/iptables/status.svg?ref=refs/heads/master)](https://drone.osshelp.ru/ansible/iptables)

## Description

Ansible role for iptables setup.

## Settings

### Enable icmp

``` yaml
accept_icmp: {v4: true, v6: true}
```

Creates typical icmp rules for IPv4 and IPv6.

### Supported chains

- `input`. enemy_input chain for input traffic
- `docker_forward`. docker_forward chain for forward traffic
- `dnat`. nat table. DNAT rules
- `snat`. nat table. SNAT rules
- `masq`. nat table. MASQUERADE rules
- `postrouting`. postrouting chain, nat table.
- `forward`. filter, mangle tables.

All of these chains

### Common settings for rules

Setting|Default|iptables|Description
---|---|---|---|
`i: eth1`|-|`-i eth1`|input interface
`ni: eth2`|-|`! -i eth2`|**not** input interface
`dport: 1234, p: tcp`|`p:tcp` if dport is defined|`-p tcp -m tcp --dport 1234`|port and protocol
`p: udp`|-|`-p udp`|protocol only
`dport: "80,443"`|`p:tcp` if dport is defined|`-p tcp -m multiport --dports 80,443`|multiport. doesn't work for snat/masq rules
`s: 123.123.123.123`|-|`-s 123.123.123.123`|source address
`ns: 123.123.123.123`|-|`! -s 123.123.123.123`|**not** source address
`d: 234.234.234.234`|-|`-d 234.234.234.234`|destination address
`nd: 234.234.234.234`|-|`! -d 234.234.234.234`|**not** destination address
`state: NEW`|-|`-m state --state NEW`|connection state
`list: oss-v4`|-|`-m set --match-set oss-v4 src`|ipset list
`comment: "test rule"`|-|`-m comment --comment "test rule"`|rule comment

These settings work for any supported chain.

### Settings for input and docker_forward

Setting|Default|iptables|Description
---|---|---|---|
`action: DROP`|`ACCEPT`|`-j DROP`|action for matched packet

input examples:

role rule|iptables rule
---|---
`{s: 234.234.234.234, action: DROP}`|`-A enemy_input -s 234.234.234.234 -j DROP`
`{i: eth0, ni: eth1, dport: 80, s: 123.123.123.0/24, comment: 'test rule'}`|`-A enemy_input -i eth0 ! -i eth1 -p tcp -m tcp --dport 80 -s 123.123.123.0/24 -m comment --comment "test rule"`

docker_forward examples:

role rule|iptables rule
---|---
`{p: tcp, drport 443}`|`-A docker_forward -p tcp -m tcp --dport 443 -j ACCEPT`
`{action: DROP'}`|`-A docker_forward -j DROP`

### Settings for dnat

Setting|Default|iptables|Description
---|---|---|---|
`o: eth1`|-|`-o eth1`|output interface
`dport: 61000-62000`|`p:tcp` if dport is defined|`-p tcp -m tcp --dport 61000:62000`|same as common but with port range support
`dstaddr: 10.10.10.10`|required|`-j DNAT --to-destination 10.10.10.10`|destination address
`dstport: 61000-62000`|-|`-j DNAT --to-destination 10.10.10.10:61000-62000`|destination port or port range

### Settings for snat

Setting|Default|iptables|Description
---|---|---|---|
`o: eth1`|-|`-o eth1`|output interface
`srcaddr: 123.123.123.123`|required|`-j SNAT --to-source 123.123.123.123`|source address

### Settings for masq

Setting|Default|iptables|Description
---|---|---|---|
`o: eth1`|-|`-o eth1`|output interface

example:

role rule|iptables rule
---|---
`{s: 10.10.10.0/24, nd: 10.10.10.0/24}`|`-A PREROUTING -s 10.10.10.0/24 ! -d 10.10.10.0/24 -j MASQUERADE`

### Disable interface check

``` yaml
disable_interface_check: true
```

### Settings for modules policy, ipv6header, tcpmss

Setting|Default|iptables|Description
---|---|---|---|
`m: policy`|-|`-m policy --dir in --pol ipsec`|policy module, you can override default `--dir` with `dir:` and `--pol` with `pol:`
`m: ipv6header`|-|`-m ipv6header --header esp`|ipv6header module, you can override `--header` with `header:`
`mss: 123`|-|`-m tcpmss --mss 123`| tcpmss module
`action: TCPMSS`|-|`-j TCPMSS --clamp-mss-to-pmtu`| TCPMSS action, instead `--clamp-mss-to-pmtu` you can set `--set-mss` with `setmss:`

## Usage example

``` yaml
- role: iptables
  ext_ifaces: [eno1, wlp1s0]
  accept_icmp: {v4: true, v6: true}
  input:
    v4:
      - {list: oss-v4}
      - {dport: 22, comment: "ssh host for clients access"}
    v6:
      - {list: oss-v6}
  docker_forward:
    v4:
      - {state: 'RELATED,ESTABLISHED'}
      - {dport: 80, comment: "nginx container"}
      - {dport: 443, comment: "nginx container"}
      - {action: DROP}
    v6:
      - {state: 'RELATED,ESTABLISHED'}
      - {action: DROP}
  dnat:
    v4:
      - {dport: 8888, dstaddr: 10.10.10.10, comment: "dnat for smth", dstport: 8888}
  snat:
    v4:
      - {s: 10.10.10.20, nd: 10.10.10.0/24, comment: "snat for smth", srcaddr: 123.123.123.123}
  masq:
    v4:
      - {s: 10.10.10.0/24, nd: 10.10.10.0/24, comment: "default masq rule"}
  postrouting:
    v6:
      - {o: eth0, m: policy, dir: out}
  mangle_forward:
    v4:
      - {o: eth0, p: tcp, flags: 'SYN,RST', mss: '1361:1536', setmss: 1360, action: TCPMSS}
```

## To-do

- tasks. use service_facts for services instead of the command module
- tasks. add the ability to turn off/on autorestart services. Otherwise it can be dangerous for Docker for example.
- tests. add OS dependency (for example: Ubuntu 16.04 or Ubuntu 18.04. Different ways to autoload rules)
- tests. add service test (for ubuntu 18.04)
- add support for using + in ext_ifaces names. e.g. eth+. For details see [this](https://oss.help/73364).
- add costom modules support (with params), hardcoded in templates for now
