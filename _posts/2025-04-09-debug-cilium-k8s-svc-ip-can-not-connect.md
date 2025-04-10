---
layout: post
title: Debugging Cilium: Why Pod Connection to SVC IP Failed
tags:
- k8s
- cilium
- network
---

Last week, a platform developer reported an issue with the new Kubernetes (k8s) cluster we prepared. Some node's daemon set (ds) pods can't connect to certain service IPs (most service IPs still work). So, I started debugging to find out why.

First, I identified a node with the problem and a ds pod equipped with debug tools. Let's say the service is `172.17.93.4:8080`. I started the tcpdump with the command `tcpdump -l -vv 'dst 172.17.93.4 and port 8080'`. However, it didn't capture any packets. Knowing that Cilium implements many BPF hooks, I realized I might have been listening on the wrong device. I changed the tcpdump to `tcpdump -l -vv 'port 8080'`, and now I could see some results:

```plaintext
...
  172.16.168.150.56306 > 172.16.149.220.http-alt: Flags [S] ...
...
```

It seems that Cilium had already translated the service IP to the pod IP. Is this the correct IP for this service? Using `cilium service list | grep '172.17.93.4'`, we can confirm the correct backend pod IP should be `172.16.109.127:8080` (this service only has one endpoint). The translated pod ID does not exist in the cluster. How could this happen? Where did this `149.220` IP come from?

We know that Cilium uses BPF maps to store the service and pod IP mappings. Using `cilium bpf lb list`, we can dump all the entries for k8s service load balancer information. However, the service/backend stored in the BPF map is correct, and no `149.220` entry is found in the map! Additionally, using `cilium bpf endpoint list` confirms that there is no `149.220` endpoint present. So, is there a real BPF map that doesn’t store information like what `cilium bpf lb list` shows?

We can use bpftool to dump the BPF map in the system. This tool is separate from the Cilium project. The Cilium project pins their BPF maps in the `/sys/fs/bpf` directory, so the map can persist during the BPF program's execution. Using `bpftool map dump pinned /sys/fs/bpf/tc/globals/cilium_lb4_services_v2`, we can retrieve the map info.

```
# Cilium uses a structure lb4_key like
# https://github.com/cilium/cilium/blob/f6369099d053607330eaaea790ed792d3d784c6e/bpf/lib/common.h#L1024-L1031
# struct lb4_key {
#	__be32 address;		/* Service virtual IPv4 address */
#	__be16 dport;		/* L4 port filter, if unset, all ports apply */
#	__u16 backend_slot;	/* Backend iterator, 0 indicates the svc frontend */
#	__u8 proto;		/* L4 protocol, 0 indicates any protocol */
#	__u8 scope;		/* LB_LOOKUP_SCOPE_* for externalTrafficPolicy=Local */
#	__u8 pad[2];
# };
# Therefore, 172.17.93.4 should translate to __be32 as ac(172) 11(17) 5d(93) 04(4) which leads us to
...
key: ac 11 5d 04 46 50 00 00  06 00 00 00  value: 00 00 00 01 01 00 01 97  00 00 00 00
key: ac 11 5d 04 46 50 01 00  06 00 00 00  value: e3 03 00 00 00 00 01 97  00 00 00 00
key: ac 11 5d 04 1f 90 00 00  06 00 00 00  value: 00 00 00 01 01 00 01 98  00 00 00 00
key: ac 11 5d 04 1f 90 01 00  06 00 00 00  value: e4 03 00 00 00 00 01 98  00 00 00 00
...
```

The value key corresponds to the ID of the `bpftool map dump pinned /sys/fs/bpf/tc/globals/cilium_lb4_backends_v3` map key, leading us to:

```
...
key: e3 03 00 00  value: ac 10 6d 7f 46 50 06 00  00 00 00 00
...
```

The `ac 10 6d 7f` is `172.16.109.127`, which is the correct IP in the map, consistent with the `cilium bpf lb` output. So, where does `149.220` come from? Furthermore, the BPF map is mounted in the tc path. Is this map used by the tc BPF program? It turns out the knowledge within the Cilium project is outdated. Cilium version 1.6+ already supports socket load balancing, which uses the cgroupv2 `cgroup_sock_addr` type BPF hook to change the `connect/recvmsg/getpeername/bind/sendmsg` network system calls. When a client attempts to connect to the service IP, it is transparently changed to the backend IP in the BPF program. That's why we can't trace it with tcpdump using `dst 172.17.93.4`. The destination IP has already been changed (and the system call tap AF_PACKET — used by tcpdump — never recognizes that the client is using `dst 172.17`).

If Cilium employs tc ingress BPF to modify the destination IP, tcpdump should show the original service IP because:

![network_packet_flow](https://raw.githubusercontent.com/dispensable/blog/refs/heads/master/_posts/images/packet_flow_in_netfilter_and_genetic_network.png)

The TAP AF_PACKET occurs before the tc ingress in the packet flow of Linux.

Now we know the "hijack" happens at the cgroupv2 socket level. However, socket load balancing uses the same BPF map as the tc ingress BPF. Looking at the socket BPF code [bpf_sock.c](https://github.com/cilium/cilium/blob/main/bpf/bpf_sock.c) confirms this. Examining the maps they use with bpftool:

```
# bpftool map | grep cilium_lb4_serv
27: hash name cilium_lb4_serv flags 0x1
144: hash name cilium_lb4_serv flags 0x1
```

Wait... why are there two BPF maps with the same name? tc and cgroup BPF share the same map; only one map is needed! Using the dump to inspect the map data, we finally find the incorrect IP `172.16.149.200` stored in the smaller numbered map (which means it was mounted before the larger one). The BPF program in cgroups must be reading the data from the old map! That's why it can't connect to the service IP!

But... the 144 map has the correct data in it. Why didn't the program fix the connection with the right data? What happens when you attach two BPF programs to the same hook? First, let's find the programs that use that map:

```
# bpftool prog --json --pretty | jq -r '.[] | select(.map_ids | select ( . != null ) | index(27))'
# bpftool prog --json --pretty | jq -r '.[] | select(.map_ids | select ( . != null ) | index(144))'
```

The output shows it's two different programs. When I used the Cilium node-specific config with:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  namespace: kube-system
  name: enable-debug
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/hostname: mynode
  defaults:
    clean-cilium-bpf-state: 'true'
    clean-cilium-state: 'true'
```

The 144 map was cleaned, while the 27 map still exists. Consulting the bpftool-cgroup manual, we learn:

```
Multiple programs are allowed to be attached to a cgroup with multi. They are executed in FIFO order (those that were attached first run first).
```

FIFO order... so when socket load balancing takes effect, it is first handled by the old outdated map, and when the correct map comes into play, the destination IP has already changed to the wrong pod IP, resulting in it doing nothing but allowing packets to proceed. Even when I set `bpf-lb-sock: 'false'` to disable socket load balancing, the outdated map still persist. Cilium somehow forgets to clean it, leading to this confusing connectivity issue!

It turns out the incorrect BPF map cannot be detached using bpftool. The `clean-cilium-*` options also do not work. The only way to clear it is to reboot the system. Now all service IPs can be connected.
