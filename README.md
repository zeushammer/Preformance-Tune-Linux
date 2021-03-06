##Kernel Parameters

####To start, edit /etc/sysctl.conf and add these lines:
```
# /etc/sysctl.conf
# Increase max open system file descriptor limit to 100,000 from the default (typically 1024).
# Increasing this limit will ensure that lingering TIME_WAIT sockets and other consumers of file 
# descriptors don’t impact our ability to handle lots of concurrent requests.

fs.file-max = 100000

# Decrease the VM swappiness parameter (default = 60), which discourages the kernel from 
# swapping memory to disk. By default, Linux attempts to swap out idle processes fairly 
# aggressively, which is counterproductive for long-running server processes that desire low latency.

vm.swappiness = 10

# Increase the port range for ephemeral (outgoing) ports

net.ipv4.ip_local_port_range = 10000 65000

# Increase Linux autotuning TCP buffer limits
# Set max to 16MB for 1GE and 32M (33554432) or 54M (56623104) for 10GE
# This enables more data to be transferred without ACKs, increasing throughput.
# Don't set tcp_mem itself! Let the kernel scale it based on RAM.

net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 16777216
net.core.wmem_default = 16777216
net.core.optmem_max = 40960
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Make room for more TIME_WAIT sockets due to more clients,
# and allow them to be reused if we run out of sockets
# Also increase the max packet backlog

net.core.netdev_max_backlog = 50000
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.tcp_max_tw_buckets = 2000000

# Set tcp_tw_reuse to tell the kernel it can reuse sockets in the TIME_WAIT state

net.ipv4.tcp_tw_reuse = 1

# Decrease the time that sockets stay in the TIME_WAIT state by lowering 
# tcp_fin_timeout from its default of 60 seconds to 10. But too low, and 
# you can run into socket close errors in networks with lots of jitter. 

net.ipv4.tcp_fin_timeout = 10

# Disable TCP slow start on idle connections
net.ipv4.tcp_slow_start_after_idle = 0

# If your servers talk UDP, also up these limits
net.ipv4.udp_rmem_min = 8192
net.ipv4.udp_wmem_min = 8192

# Disable source routing and redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.accept_source_route = 0

# Log packets with impossible addresses for security
net.ipv4.conf.all.log_martians = 1
```

Since some of these settings can be cached by networking services, it’s best to reboot to apply them properly (sysctl -p does not work reliably).
##Open File Descriptors

In addition to the Linux fs.file-max kernel setting above, we need to edit a few more files to increase the file descriptor limits. The reason is the above just sets an absolute max, but we still need to tell the shell what our per-user session limits are.

So, first edit /etc/security/limits.conf to increase our session limits:
```
# /etc/security/limits.conf
# allow all users to open 100000 files
# alternatively, replace * with an explicit username
* soft nofile 100000
* hard nofile 100000
```
Confirm these settings have taken effect by opening a new ssh connection to the box and checking ulimit:
```
$ ulimit -n
100000
```
Why Linux has evolved to require 4 different settings in 4 different files is beyond me, but that’s a topic for a different post. :)


##TCP Congestion Window

Finally, let’s increase the TCP congestion window from 1 to 10 segments. This is done on the interface, which makes it a more manual process that our sysctl settings. First, use ip route to find the default route, shown in bold below:
```
$ ip route
default via 10.248.77.193 dev eth0 proto kernel
10.248.77.192/26 dev eth0  proto kernel  scope link  src 10.248.77.212
```
Copy that line, and paste it back to the ip route change command, adding initcwnd 10 to the end to increase the congestion window:
```
$ sudo ip route change default via 10.248.77.193 dev eth0 proto kernel initcwnd 10
```
To make this persistent across reboots, you’ll need to add a few lines of bash like the following to a startup script somewhere. Often the easiest candidate is just pasting these lines into /etc/rc.local:
```
defrt=`ip route | grep "^default" | head -1`
ip route change $defrt initcwnd 10
```
And your done!
