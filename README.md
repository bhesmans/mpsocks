Simple sock test with mptcp

This will create a new vm from scratch and build an mptcp kernel to run it.

## Quick start:

### Build vm (First time only)

```
source mpsocks_lib

# build a debian image base
build_image

# build mptcp kernel
build_kernel

# install dante sock server (https://www.inet.no/dante/)
# install redsock server (https://darkk.net.ru/redsocks/)
# boot vm and dl/compile dante/redsocks inside the vm then shut it down
install_vm_extra
```

### Run a socks server on the vm

```
# Comma-separated list of interface to use for mptcp subflows
# Could be the same if your system does not have multiple itf
boot_socks wlp1s0,wlp1s0,wlp1s0

# Configure your app to use 10.66.6.2 as a sock proxy
# ^C to finish the socks server and the vm.

# If you don't have an option to add a socks proxy on your application, the vm
# runs a "socksifier", you just need to route traffic through the vm and it will
# be "socksified"
sudo ip route add 130.104.228.140 via 10.66.6.2 # multipath-tcp.org
wget -4 -O- -U curl -q multipath-tcp.org
# Yay, you are MPTCP-capable! You can now rest in peace.
```

## More detailed steps ...:

The process can be split is smaller tasks to understand what's going on and
do it step by step.

```
# Create tap for nw
configure_host_tap

# Nat out the tap
configure_host_nat

# boot the vm
boot_vm

# wait for the vm to boot, this should be fast and leave this open in a console
# OPTIONNAL:
# Install dante to use as a socks server.
# https://www.inet.no/dante/
# install_dante

# Configure vm nw
configure_guest_nw

# Check that the vm is indeed using mptcp
check_mptcp
# should print "Yay, you are MPTCP-capable! You can now rest in peace."

# Enable local socks server
run_socks
# alternative without ssh : run_socks dante

# if you use ssh :
# You can now configure firefox to use socks server localhost on port 6666
# if you use dante :
# You can now configure firefox to use socks server 10.66.6.2 on port 6666
# Using firefox, go check http://amiusingmptcp.de/#check should be green

# you should have less overhead with dante because it doesn't tunnel at all
# Firefox will connect straight to the vm

# Once you're done, do some clean up
clean_host_nat
clean_host_tap
```

Dvlp:

Mptcp with one subflow is kind of sad. Let's configure a second itf.

The first interface configured before is still there and use the default table.
Second interface is configured with source routing. The second interface need
to be configured in the call to configure\_host\_source\_routing (see bellow)

In the example bellow, I used two interfaces on my laptop. My first interface
is a tethering interface that use my smartphone to get out. My second interface
is the wifi interface (wlp1s0).

```
# Same as the first one
configure_host_tap 1
configure_host_nat 1
# Need source routing to get out from the second tap
configure_host_source_routing wlp1s0 # replace itf by your outgoing itf

configure_guest_nw 1

run_socks dante
```

You can check on http://amiusingmptcp.de the number of subflows, if everything
went fine, you should have 2.

Let's see a what's on the "wire".

```
sudo tcpdump -i wlp1s0 -s 120 -w /tmp/wlan.pcap &
sudo tcpdump -i enp0s20f0u2 -s 120 -w /tmp/tether.pcap &
# Go to http://amiusingmptcp.de for a small speedtest.
sudo killall tcpdump
cd /tmp
mergecap -w all.pcap wlan.pcap tether.pcap
mptcptrace -f all.pcap -s # Output
# MPTCP trace V0.0 alpha : says Hello.
# MPTCP connection 0 with id 1
#        Subflow 0 with wscale : 6 7 IPv4 sport 58865 dport 80 saddr 192.168.1.21 daddr 18.194.8.156
#        Subflow 1 with wscale : 6 7 IPv4 sport 49694 dport 80 saddr 192.168.42.204 daddr 18.194.8.156
#MPTCP connection 0 with id 2
#        Subflow 0 with wscale : 6 7 IPv4 sport 57563 dport 80 saddr 192.168.1.21 daddr 18.194.8.156
#        Subflow 1 with wscale : 6 7 IPv4 sport 49696 dport 80 saddr 192.168.42.204 daddr 18.194.8.156
xplot.org s2c_seq_1.xpl # see img bellow
```

![MPTCP DSS](/img/test1.png?raw=true "MPTCP DSS")

## Bonus

The internet is sometimes funny.

From the vms i tested the following scenarii, traces are present in
`traces/net_neutrality/`

* mptcp off / ttl = 10
  * Commands

```
# sysctl net.mptcp.mptcp_enabled=0
net.mptcp.mptcp_enabled = 0
# sysctl net.ipv4.ip_default_ttl=10
net.ipv4.ip_default_ttl = 10
# wget -O- -U curl -q --bind-address 10.66.7.2 multipath-tcp.org
Nay, Nay, Nay, your have an old computer that does not speak MPTCP. Shame on you!
```

  * Observations

Even using a low ttl we are able to join the server.

* mptcp on / ttl = 10
```
# sysctl net.mptcp.mptcp_enabled=1
net.mptcp.mptcp_enabled = 1
# sysctl net.ipv4.ip_default_ttl=10
net.ipv4.ip_default_ttl = 10
# wget -O- -U curl -q --bind-address 10.66.7.2 multipath-tcp.org
^C
```

  * Observations

Using the same low TTL but with mptcp enabled, we can't reach it anymore :-/

* mptcp on / ttl = 64

```
# sysctl net.mptcp.mptcp_enabled=1
net.mptcp.mptcp_enabled = 1
# sysctl net.ipv4.ip_default_ttl=64
net.ipv4.ip_default_ttl = 64
# wget -O- -U curl -q --bind-address 10.66.7.2 multipath-tcp.org
Yay, you are MPTCP-capable! You can now rest in peace.
```

  * Observations

Using a high enough TTL we can reach it again using mptcp and the server
confirms the usage of MTPCP.

* Analysis desk:

Let's focus on TTL (Thanks Yves Vanaubel for the idea! https://dl.acm.org/doi/abs/10.1145/2504730.2504761 )

```
$ tshark -Y 'tcp.flags.syn == 1' -T fields -e tcp.flags.syn -e tcp.flags.ack -e ip.ttl -E header=yes -r traces/net_neutrality/nomptcp_ttl10
tcp.flags.syn   tcp.flags.ack   ip.ttl
1       0       9
1       1       124
$ tshark -Y 'tcp.flags.syn == 1' -T fields -e tcp.flags.syn -e tcp.flags.ack -e ip.ttl -E header=yes -r traces/net_neutrality/mptcp_ttl10
tcp.flags.syn   tcp.flags.ack   ip.ttl
1       0       9
1       0       9
1       0       9
1       0       9
1       0       9
$ tshark -Y 'tcp.flags.syn == 1' -T fields -e tcp.flags.syn -e tcp.flags.ack -e ip.ttl -E header=yes -r traces/net_neutrality/mptcp_ttl64
tcp.flags.syn   tcp.flags.ack   ip.ttl
1       0       63
1       1       51
```

Each line shows SYN SYN+ACK TTL. We can see the TTL for the SYN are 9, 9 and 63.
This is the TTL that we set, because I did the trace on my host, -1. What seems
to be suspicious is the TTL difference in SYN+ACK whether MPTCP is enabled or
not. When MPTCP is not enabled the SYN+ACK has a TTL of 124. But when MPTCP is
enabled we have 51. It suggests that the connection is handled differently based
on the presence of MPTCP. The connections is probably intercepted before
reaching the end server when MPTCP is not present  which means a lower TTL is
acceptable because the connection will be terminated earlier by an intermediate
hop and a new SYN with a new initial TTL will be generated.  Since here I'm
testing on a 4G connection I would bet on some kind of network optimizer that is
not capable of handling MPTCP correctly.

At this point we've guest that MTPCP option in the SYN is probably the only
thing that our ISP is lookin at, so it does not matter if the server actually
speaks MPTCP or not. Let's try to talk to a server that is not mptcp enabled and
see if there is a difference if we use the MPTCP option in the SYN or not :

(traces are available in the trace directory)

```
root@bhesmans-Precision-7520:~# sysctl net.mptcp.mptcp_enabled=1
net.mptcp.mptcp_enabled = 1
root@bhesmans-Precision-7520:~# wget -O /dev/null -U curl --bind-address 10.66.7.2 http://ipv4.download1.thinkbroadband.com/100MB.zip
--2020-04-16 11:30:02--  http://ipv4.download1.thinkbroadband.com/100MB.zip
Resolving ipv4.download1.thinkbroadband.com (ipv4.download1.thinkbroadband.com)... 80.249.99.148
Connecting to ipv4.download1.thinkbroadband.com (ipv4.download1.thinkbroadband.com)|80.249.99.148|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 104857600 (100M) [application/zip]
Saving to: '/dev/null'

/dev/null                  100%[=====================================>] 100.00M  4.78MB/s    in 33s

2020-04-16 11:30:36 (3.00 MB/s) - '/dev/null' saved [104857600/104857600]
```

```
root@bhesmans-Precision-7520:~# sysctl net.mptcp.mptcp_enabled=0
net.mptcp.mptcp_enabled = 0
root@bhesmans-Precision-7520:~# wget -O /dev/null -U curl --bind-address 10.66.7.2 http://ipv4.download1.thinkbroadband.com/100MB.zip
--2020-04-16 11:31:00--  http://ipv4.download1.thinkbroadband.com/100MB.zip
Resolving ipv4.download1.thinkbroadband.com (ipv4.download1.thinkbroadband.com)... 80.249.99.148
Connecting to ipv4.download1.thinkbroadband.com (ipv4.download1.thinkbroadband.com)|80.249.99.148|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 104857600 (100M) [application/zip]
Saving to: '/dev/null'

/dev/null                  100%[=====================================>] 100.00M  8.66MB/s    in 12s

2020-04-16 11:31:12 (8.56 MB/s) - '/dev/null' saved [104857600/104857600]
```

redoing our TTL check:

```
$ tshark -Y 'tcp.flags.syn == 1' -T fields -e tcp.flags.syn -e tcp.flags.ack -e
ip.ttl -E header=yes -r traces/net_neutrality/mptcp_100MB
tcp.flags.syn   tcp.flags.ack   ip.ttl
1       0       63
1       1       49
$ tshark -Y 'tcp.flags.syn == 1' -T fields -e tcp.flags.syn -e tcp.flags.ack -e
ip.ttl -E header=yes -r traces/net_neutrality/nomptcp_100MB
tcp.flags.syn   tcp.flags.ack   ip.ttl
1       0       63
1       1       124

```

We also note a difference in speed :/ I did a few tests and it seems that it was
consistently slower when MPTCP is present in the SYN.

(This might be a good time to call the net neutrality police.)

There is a possible consequence for all the devices that use MPTCP.
Because they seem to be treated differently. In particular some Apple devices
are known to use MPTCP for some of their services.
