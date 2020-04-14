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
# boot vm and compile dante inside the vm then shut it down
boot_vm 0 yes &
sleep 5 # Let qemu open the fwd port for ssh
install_dante
ssh_vm shutdown -h now
```

### Run a socks server on the vm

```
# Comma-separated list of interface to use for mptcp subflows
# Could be the same if your system does not have multiple itf
boot_socks wlp1s0,wlp1s0,wlp1s0

# Configure your app to use 10.66.6.2 as a sock proxy
# ^C to finish the socks server and the vm.
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
