# what is node_filesystem_avail_bytes and node_filesystem_size_bytes metric?
* these metrics come from Prometheus Node Exporter (a system monitoring agent that exposes linux/unix host metrics for prometheus).
* you’ll typically see them when scraping disk/filesystem usage.


## 🧩 what they are: from prometheus node exporter:
### 1. node_filesystem_size_bytes
* this is the total size of a filesystem (in bytes).
** it represents the full capacity of a mounted filesystem.
** it is reported per mount point (e.g., /, /var, /mnt/data).
** it does not change frequently unless the disk/partition is resized.
** example: node_filesystem_size_bytes{mountpoint="/"} = 500000000000 → root filesystem is ~500 GB total


### 2. node_filesystem_avail_bytes
* this is the available space for non-root users (in bytes).
** it is the space that normal processes can still use.
** it excludes space reserved for root (on ext filesystems, typically ~5% by default).
** it changes as files are created/deleted.
** example: node_filesystem_avail_bytes{mountpoint="/"} = 120000000000 → ~120 GB available for general use


## ⚠️ Important distinction: “avail” vs “free”
* avail is what you usually want for alerting dashboards
* free may include reserved blocks (not safely usable)


## 📊 How they’re used
* disk usage percentage
** most dashboards compute usage like:
** 1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
* -or-
* in percent:
** (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100


## 🧠 Labels you’ll usually see
* these metrics are broken down by labels like:
** device → disk device (e.g., /dev/sda1)
** mountpoint → where it is mounted (e.g., /, /data)
** fstype → filesystem type (ext4, xfs, tmpfs, etc.)
* example:
** node_filesystem_avail_bytes{mountpoint="/",fstype="ext4"}


## 🚨 common gotcha
* you should usually filter out pseudo filesystems:
* node_filesystem_avail_bytes and fstype !~ "tmpfs|overlay|squashfs"
* otherwise you’ll see misleading values from container/system mounts.

## 🧾 Summary
* node_filesystem_size_bytes → total disk capacity
* node_filesystem_avail_bytes → usable free space (excluding reserved blocks)
* used together to compute disk utilization
* exposed by prometheus node exporter for prometheus monitoring




# what is node_disk_io_time_seconds_total?
* node_disk_io_time_seconds_total is a metric exposed by prometheus’ node_exporter that represents: the total amount of time (in seconds) that a disk has spent performing I/O operations since the system started.


## what it actually measures
* it is a counter (monotonically increasing).
* it accumulates the time the disk was busy doing reads or writes.
* unit: seconds (but fractional).
* scope: per disk device (e.g., sda, nvme0n1, etc.).
* so if a disk has:
** been actively reading/writing for 2 hours total since boot,
** the metric might show ~7200 seconds.


## important nuance
* this is not wall-clock time the system has been running.
* instead, it is: “how long was the disk busy doing I/O work?”
* if multiple I/O operations overlap, it still counts the busy time, not number of operations.


## how to use it properly
* since it’s a cumulative counter, you almost always use it with rate() or irate():


## disk utilization approximation
* rate(node_disk_io_time_seconds_total[5m])
* this gives:
** fraction of time per second the disk is busy
** often interpreted as utilization (0.0 → 1.0)
** example interpretation:
*** 0.25 → disk busy ~25% of the time
*** 1.0 → disk fully saturated (100% busy)


## why it’s useful
* you use it to detect:
** disk saturation / bottlenecks
** high I/O load periods
** storage performance issues


## related metrics - you’ll often combine it with:
* node_disk_read_bytes_total
* node_disk_written_bytes_total
* node_disk_io_time_weighted_seconds_total (more advanced latency-sensitive variant)
* node_disk_io_now (current outstanding I/O operations)


## quick mental model
* think of it like: “out of all the time since boot, how many seconds was this disk actively doing work?”




# what is node_network_receive_bytes_total and node_network_transmit_bytes_total?
* in prometheus node exporter, the metrics: node_network_receive_bytes_total and node_network_transmit_bytes_total are cumulative network I/O counters per network interface.


## what they represent
* node_network_receive_bytes_total - total number of bytes received (downloaded) by a network interface since the system (or interface) started.
* node_network_transmit_bytes_total - total number of bytes transmitted (uploaded) by a network interface since the system (or interface) started.


## key characteristics
### 1. they are counters (monotonically increasing)
* they only go up over time.
* they reset only if:
** the machine reboots, or
** the network interface is reset/recreated.


### 2. they are per network interface
* each metric has labels like:
** device="eth0"
** device="ens33"
** device="lo" (loopback)
* so you’ll see separate values per NIC.


### 3. units: bytes
* not bits. If you want bits/sec, you multiply by 8.


## how you actually use them
* these metrics are not useful as raw numbers. you typically convert them into rates.
** bytes per second (receive): rate(node_network_receive_bytes_total[5m])
** bytes per second (transmit): rate(node_network_transmit_bytes_total[5m])


## common real-world PromQL patterns
* total network throughput per interface:
** rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m])
* ingress traffic (Mbps):
** rate(node_network_receive_bytes_total[5m]) * 8 / 1e6
* egress traffic (Mbps):
** rate(node_network_transmit_bytes_total[5m]) * 8 / 1e6


## important nuances
### 1. includes all traffic
* this includes:
** application traffic
** system traffic
** container/VM traffic (if sharing NIC)
** loopback traffic (lo) unless filtered out


### 2. can include errors only indirectly
* these metrics are not error-aware. for errors you’d use:
** node_network_receive_errs_total
** node_network_transmit_errs_total


### 3. virtual interfaces matter
* in Kubernetes / Docker environments you may see:
** cni0
** veth*
** docker0
* each will have its own counters.


## simple mental model
* think of them as: a car odometer for network traffic
** receive = miles coming in
** transmit = miles going out
* but you don’t care about total miles—you care about speed (rate).




# what is node_load1 metric?
* node_load1 in prometheus node exporter is a metric that represents the system load average over the last 1 minute.


## what it actually measures
* it comes from the linux kernel’s load average, which reflects how many processes are:
** actively running on CPU, or
** waiting for CPU time (runnable queue), or
** in some cases, waiting on uninterruptible I/O (less emphasized in modern interpretations, but still included)
* so:
** node_load1 = 1-minute load average of the system


## where it comes from
* node Exporter reads this from the system (typically /proc/loadavg) and exposes it to prometheus.
* you’ll usually see related metrics too:
** node_load1 → 1 minute average
** node_load5 → 5 minute average
** node_load15 → 15 minute average


## how to interpret it - the key thing is: load is relative to CPU cores
* if you have 1 CPU core:
** load of 1.0 = fully utilized
** load > 1.0 = processes are waiting
* if you have 4 CPU cores:
** load of 4.0 ≈ fully utilized system
** oad of 8.0 = heavily overloaded (roughly 2x demand vs capacity)
* so load alone isn’t enough—you usually compare it to:
** node_load1 / number_of_cpus
* example PromQL:
** node_load1 / count(node_cpu_seconds_total{mode="idle"})


## important nuance
* load is not CPU percentage. it’s a queue depth metric:
** CPU % = how busy CPU is
** load average = how many tasks are competing for CPU (or stuck waiting)


## when it’s useful
* detecting CPU saturation trends
* identifying system pressure (especially if load rises while CPU is already high)
* alerting when load exceeds core capacity for sustained periods




# what is node_memory_MemTotal_bytes and node_memory_MemFree_bytes and node_memory_Cached_bytes and node_memory_Buffers_bytes?
* These metrics come from the prometheus node exporter (node_exporter), which exposes linux system memory information from /proc/meminfo. they represent different parts of how the OS accounts for RAM. here’s what each one means:


## 🧠 node_memory_MemTotal_bytes
* this is the total physical RAM installed on the machine.
* equivalent to MemTotal in /proc/meminfo
* includes all usable RAM (minus a small amount reserved by the kernel/hardware)
* does not change unless memory is added/removed or hotplugged
* 👉 think of it as: “how much RAM does this system have in total?”


## 🆓 node_memory_MemFree_bytes
* this is the amount of RAM that is completely unused right now.
* equivalent to MemFree
* memory not used by anything at all (no processes, no cache, nothing)
* on linux, this is usually very low, even on idle systems
* 👉 important: low MemFree is normal on linux.



## 💾 node_memory_Cached_bytes
* this is memory used by the kernel for page cache (file caching).
* stores data from disk reads so future reads are faster
* can be quickly reclaimed if applications need memory
* includes cached file contents, metadata, etc.
* 👉 think of it as: “RAM being used to speed up disk access, but not permanently occupied”


## 📦 node_memory_Buffers_bytes
* this is memory used for block device buffers (filesystem metadata / I/O buffers).
* used for temporary storage of raw disk I/O
* smaller than cache in modern systems
* also reclaimable if needed
* 👉 think of it as: “short-term staging area for disk operations”


## 🧩 how they relate
* linux memory is not simply “used vs free”. A more useful view is:
* available RAM ≈ MemFree + Buffers + Cached (minus reclaimable parts)
* but Node Exporter also provides a better metric for that:
** node_memory_MemAvailable_bytes ← ⭐ best metric for “how much RAM can I actually use?”


## ⚠️ Common confusion
* MemFree being low is not a problem
* Cached memory is not wasted
* linux aggressively uses free RAM for cache to improve performance


## 📊 Practical monitoring tip
* instead of relying on these individually, most dashboards use:
** Memory usage %
** MemAvailable
** Working set (used - reclaimable cache)




# what is node_network_receive_drop_total and node_network_transmit_drop_total?
* in prometheus (via node_exporter), the metrics: node_network_receive_drop_total and node_network_transmit_drop_total are cumulative counters that track dropped network packets at the kernel/network interface level.


## what they mean
* node_network_receive_drop_total
** this is the total number of incoming packets that were dropped by a network interface.
** it increases when packets are received by the NIC but discarded before being delivered to the application stack.
* node_network_transmit_drop_total
** this is the total number of outgoing packets that were dropped by a network interface.
** it increases when the system fails to send packets out successfully.


## important characteristics
* they are monotonic counters (always increase or reset on reboot)
* they are exposed per network interface (e.g. eth0, ens33)
* you typically query them using rate() or increase() in PromQL
* example: rate(node_network_receive_drop_total[5m])


## what “drop” actually means here
* drops are not always “bad network problems.” They can happen due to:
** receive drops (RX)
** kernel socket buffers full
** NIC ring buffer overflow
** high traffic bursts exceeding processing capacity
** firewall / iptables early drops (sometimes accounted differently depending on system)
** driver limitations
** transmit drops (TX)
** outgoing queue full
** nterface congestion
** NIC or driver cannot enqueue packets fast enough
** QoS or shaping rules discarding packets
** hardware buffer exhaustion


## how to interpret them

### 1. occasional small increases
* usually normal under burst traffic.

### 2. Sustained growth or high rate
* this is a red flag:
** network saturation
** CPU bottlenecks (packet processing)
** insufficient NIC buffers
** misconfigured queue sizes


## common PromQL checks
* RX drop rate - rate(node_network_receive_drop_total[5m])
* TX drop rate - rate(node_network_transmit_drop_total[5m])
* total drops per interface - increase(node_network_receive_drop_total{device="eth0"}[1h])


## how this differs from errors
* it’s important not to confuse drops with errors:
** drops → packets intentionally or passively discarded (resource/queue issues)
** errors → malformed, corrupted, or invalid packets
* you’ll often see:
** node_network_receive_errs_total
** node_network_transmit_errs_total


## quick intuition
* drops = “couldn’t keep up”
* errors = “packet was broken”




# what is node_network_receive_errs_total and node_network_transmit_errs_total?
* in prometheus node exporter, the metrics: node_network_receive_errs_total and node_network_transmit_errs_total are network interface error counters exposed by the linux kernel via /proc/net/dev.


## what they mean
* both are cumulative counters (since boot) and are labeled per network interface (e.g., eth0, ens33, lo).

### 1. node_network_receive_errs_total
* this is the total number of receive errors on a network interface.
* it increases when the NIC encounters problems while receiving packets, such as:
** CRC/frame errors (corrupted packets)
** FIFO overruns (driver or hardware can't keep up)
** alignment errors
** dropped malformed packets at the hardware/driver level
** 👉 think: “packets came in, but the NIC couldn’t properly accept them.”


### 2. node_network_transmit_errs_total
* this is the total number of transmit errors on a network interface.
* it increases when the NIC fails to send packets, such as:
** transmission failures at the driver/hardware level
** carrier issues (link instability)
** buffer or queue failures
** hardware retry exhaustion
** 👉 think: “we tried to send packets, but something prevented successful transmission.”


## important characteristics
* counters, not gauges
* they only go up (unless the interface resets or the node reboots).
* per-interface metric
* example:
** node_network_receive_errs_total{device="eth0"} 12
** node_network_transmit_errs_total{device="eth0"} 3
* usually paired with:
** node_network_receive_drop_total
** node_network_transmit_drop_total
** node_network_receive_packets_total
** node_network_transmit_packets_total


## how to interpret them
### 1. any non-zero value is a signal
* occasional small errors might be normal on busy or noisy networks, but:
** consistently increasing values = problem
** sudden spikes = event worth investigating

### 2. convert to rate (recommended in Prometheus)
* instead of raw counters, you usually want:
* rate(node_network_receive_errs_total[5m]) and rate(node_network_transmit_errs_total[5m])
* this shows errors per second, which is much more useful.


## common causes
* receive errors
* bad cable / faulty switch port
* duplex mismatch
* NIC driver issues
* overloaded receive ring buffer
* Transmit errors
* network congestion / queue drops
* hardware offload issues
* link instability
* MTU mismatches (sometimes indirect symptoms)


## when to worry
* you should investigate if:
** error rate is continuously > 0
** errors grow steadily over time
** only one interface is affected (isolated hardware/cable issue)
** errors correlate with latency or packet loss


## quick mental model
* receive_errs_total → “bad packets coming in”
* transmit_errs_total → “failed packets going out”