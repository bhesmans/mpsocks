Simple sock test with mptcp

This will create a new vm from scratch and build an mptcp kernel to run it.

Usage:

```
source mpsocks_lib

# build a debian image base
build_image

# build mptcp kernel
build_kernel

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
# You can now configure firefox to use socks server 6.6.6.2 on port 6666
# Using firefox, go check http://amiusingmptcp.de/#check should be green

# you should have less overhead with dante because it doesn't tunnel at all
# Firefox will connect straight to the vm

# Once you're done, do some clean up
clean_host_nat
clean_host_tap
```

