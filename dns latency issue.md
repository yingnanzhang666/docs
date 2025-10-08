# Problem Statement
The dns latency from synthetic agent reports the high DNS latency issue in the large scale cluster. The P95
and P99 latency are much higher compared to other clusters.
The synthetic agent uses coredns as its resolver to do DNS lookup to expose DNS related SLO.

# Root Cause
SUMMARY: The garbage collection (GC) of CoreDNS frequently exhausted the allocated CPU
time slice within one or two cgroup periods, resulting in CPU throttling by the CPU cgroup. This
recurrent GC activity significantly impacts the P99 latency of DNS queries.

The Impact of CPU throttling to P99 latency of e2e dns queries: [Current cpu limit of coredns is 9]

Cpu limit is using cgroup to limit the amount of CPU time that a group of processes can utilize
within a given period. This scheduling period is defined by cpu.cfs_period_us, default value is
100ms. It means, in each 100ms, cgroup allocates 900ms cpu time(cpu limit is 9) for all the
threads in coredns. If the 900ms was exhausted the processes in this cgroup will be throttled
until the next scheduling period(100ms).

|cpu.cfs_period_us|100000 [100ms by default]|
|-------|-----|
|cpu.cfs_quota_us|900000|

<img width="1118" height="510" alt="image" src="https://github.com/user-attachments/assets/a16cf6de-3785-4fe4-8b83-3cc60e47197e" />
In extreme cases, during a 100ms scheduling period, a process can be throttled for almost 100ms.
For example, as shown in the figure above, the 900ms CPU quota is exhausted within the first
few milliseconds, and for the remaining time of this scheduling period, CoreDNS is unable to
execute any programs. Which is where the long tail latency came from.

<img width="2588" height="1002" alt="image" src="https://github.com/user-attachments/assets/56411a1b-8724-4e74-a375-ce0300a2f1af" />
Although the cpu usage of coredns from the dashboard is only 0.8 cpu, far away from the cpu
limit: 9 cpus. But it’s indeed throttled, which caused the **instantaneous burst of cpu usage in
100ms**. The cpu usage metrics has been averaged to 1 second.
Because the scheduling period is 100ms, then fetch the cpu usage in each 100ms by using “top
-p **-d 0.1** -b”, (set the cpu limit is unlimited) we can see, in about 10s interval, there are only once
which the cpu usage is over 3000% in one 100ms, others are almost less than 100%. So, the
average cpu usage per second is around 80%.

```command output
top - 02:26:12 up 47 days, 1:15, 17 users, load average: 39.37, 36.46, 35.18
Tasks: 1 total, 0 running, 1 sleeping, 0 stopped, 0 zombie
%Cpu(s): 15.1 us, 2.7 sy, 0.0 ni, 80.2 id, 0.0 wa, 0.0 hi, 2.0 si, 0.0 st
MiB Mem : 515061.2 total, 12030.1 free, 234473.0 used, 268558.0 buff/cache
MiB Swap: 0.0 total, 0.0 free, 0.0 used. 227873.2 avail Mem
PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
99065 root 20 0 1618540 581856 38100 S 60.0 0.1 20031:07 coredns
------
top - 02:26:12 up 47 days, 1:15, 17 users, load average: 39.37, 36.46, 35.18
Tasks: 1 total, 0 running, 1 sleeping, 0 stopped, 0 zombie
%Cpu(s): 15.3 us, 2.4 sy, 0.0 ni, 80.5 id, 0.0 wa, 0.0 hi, 1.9 si, 0.0 st
MiB Mem : 515061.2 total, 12030.1 free, 234473.0 used, 268558.0 buff/cache
MiB Swap: 0.0 total, 0.0 free, 0.0 used. 227873.2 avail Mem
PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
99065 root 20 0 1618540 581856 38100 S 70.0 0.1 20031:08 coredns
------
top - 02:26:12 up 47 days, 1:15, 17 users, load average: 39.37, 36.46, 35.18
Tasks: 1 total, 0 running, 1 sleeping, 0 stopped, 0 zombie
%Cpu(s): 39.8 us, 3.2 sy, 0.0 ni, 54.5 id, 0.0 wa, 0.0 hi, 2.4 si, 0.0 st
MiB Mem : 515061.2 total, 12030.1 free, 234473.0 used, 268558.0 buff/cache
MiB Swap: 0.0 total, 0.0 free, 0.0 used. 227873.2 avail Mem
PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
99065 root 20 0 1618540 581856 38100 S 3280 0.1 20031:11 coredns
------
top - 02:26:12 up 47 days, 1:15, 17 users, load average: 39.37, 36.46, 35.18
Tasks: 1 total, 0 running, 1 sleeping, 0 stopped, 0 zombie
%Cpu(s): 19.6 us, 3.3 sy, 0.0 ni, 74.0 id, 0.0 wa, 0.0 hi, 3.0 si, 0.0 st
MiB Mem : 515061.2 total, 12030.1 free, 234473.0 used, 268558.0 buff/cache
MiB Swap: 0.0 total, 0.0 free, 0.0 used. 227873.2 avail Mem
PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
99065 root 20 0 1618540 581856 38100 S 70.0 0.1 20031:11 coredns
------
top - 02:26:13 up 47 days, 1:15, 17 users, load average: 39.37, 36.46, 35.18
Tasks: 1 total, 0 running, 1 sleeping, 0 stopped, 0 zombie
%Cpu(s): 24.6 us, 3.3 sy, 0.0 ni, 68.2 id, 0.0 wa, 0.0 hi, 3.9 si, 0.0 st
MiB Mem : 515061.2 total, 12030.1 free, 234473.0 used, 268558.0 buff/cache
MiB Swap: 0.0 total, 0.0 free, 0.0 used. 227873.2 avail Mem
PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
99065 root 20 0 1618540 581856 38100 S 80.0 0.1 20031:11 coredns
------
top - 02:26:13 up 47 days, 1:15, 17 users, load average: 39.37, 36.46, 35.18
Tasks: 1 total, 0 running, 1 sleeping, 0 stopped, 0 zombie
%Cpu(s): 20.0 us, 2.0 sy, 0.0 ni, 75.6 id, 0.0 wa, 0.0 hi, 2.4 si, 0.0 st
MiB Mem : 515061.2 total, 12030.1 free, 234473.0 used, 268558.0 buff/cache
MiB Swap: 0.0 total, 0.0 free, 0.0 used. 227873.2 avail Mem
PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
99065 root 20 0 1618540 581856 38100 S 60.0 0.1 20031:11 coredns
```
The cpu burst is caused by the memory GC of coredns. https://go.dev/doc/gc-guide

The **concurrent GC processes** exhausted the cpu quota in the first several milliseconds which
caused that the coredns was throttled in the remaining time of the 100ms scheduling period.

# Solutions
## Option 1 : increase the cpu limit of coredns to 40
Set the cpu limit to unlimited, and monitor the cpu usage in each 100ms, all of them are almost
3000%+.

So, in order to avoid the cpu throttling, set the cpu limit of coredns to 40.

There’s no cpu throttled after this change, and p99 latency of e2e dns query decreased from
100ms to <5ms.

## Option 2: set GOGC=-1 and GOMEMLIMIT=3GiB /memlimit of coredns/ (to reduce the gc frequency)
For the default setting of gc, GOGC=100. It will do GC when the heap size is double of the live
heap of the last GC. So, for the gc of coredns in this cluster, the heap size is increased from 200MB to
400MB and GC to 200MB, the gc interval is almost **10s**.

After setting GOGC=-1 and GOMEMLIMIT=3GiB, the gc interval becomes around **2.5 minutes**.

The cpu throttling is still happening when the gc happens, but the frequency is from 1/10s to
1/150s. The proportion of affected requests has significantly decreased.

## Option 3: set GOMAXPROCS=<cpu-limit> (to avoid cpu time exhausted by GC)
_GOMAXPROCS sets the maximum number of CPUs that can be executing simultaneously and returns
the previous setting. It defaults to the value of runtime.NumCPU._

golang will spin up the OS threads to run the goroutines, if GOMAXPROCS is not set, the number
of threads is similar to the cpu cores on this node. Which is around 130. The concurrent GC can
use these threads at the same time, which causes the cpu quota to be exhausted rapidly. If set
GOMAXPROCS=9 (the current cpu-limit of coredns), concurrent GC processes cannot execute
more than 9 at the same time, which leaves the cpu quota for dns queries handling.
```spec
env:
- name: GOMAXPROCS
  valueFrom:
    resourceFieldRef:
    resource: limits.cpu
```
The cpu throttled nr is 0 after setting GOMAXPROCS=9.
```command output
root@node:/sys/fs/cgroup/cpu/kubepods.slice/kubepods-burstable.slice/kub
epods-burstable-pod9118fc9a_5622_43c2_a6a9_ea0a582ea247.slice/cri-containerd-0c8cd1b
2ec1d8877d7a6a412759d20646f39aec47ff5e3bfd6cfe96f7c9bcfff.scope# cat cpu.stat
nr_periods 97605
nr_throttled 0
throttled_time 0
```
_Decreasing the maximum number of CPUs that can be executing simultaneously might impact the pause
duration of go gc cycle(which is the duration of stop-the-world). From the result, there’s no downgrade of
performance of GC pause duration._

The pause duration of GC after setting GOMAXPROCS to 9.
```command output
root@node:~# nsenter -t 165650 -n curl -s localhost:9153/metrics | grep -E 'gc|heap'
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 6.3322e-05           <---
go_gc_duration_seconds{quantile="0.25"} 0.000104902       <---
go_gc_duration_seconds{quantile="0.5"} 0.000153445        <---
go_gc_duration_seconds{quantile="0.75"} 0.000264325       <---
go_gc_duration_seconds{quantile="1"} 0.001312075          <---
go_gc_duration_seconds_sum 0.261237248
go_gc_duration_seconds_count 860                          <---
# HELP go_memstats_gc_sys_bytes Number of bytes used for garbage collection system metadata.
# TYPE go_memstats_gc_sys_bytes gauge
go_memstats_gc_sys_bytes 3.9909952e+07
# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 2.08494536e+08
# HELP go_memstats_heap_idle_bytes Number of heap bytes waiting to be used.
# TYPE go_memstats_heap_idle_bytes gauge
go_memstats_heap_idle_bytes 3.70352128e+08
# HELP go_memstats_heap_inuse_bytes Number of heap bytes that are in use.
# TYPE go_memstats_heap_inuse_bytes gauge
go_memstats_heap_inuse_bytes 3.70302976e+08
# HELP go_memstats_heap_objects Number of allocated objects.
# TYPE go_memstats_heap_objects gauge
go_memstats_heap_objects 3.07753e+06
# HELP go_memstats_heap_released_bytes Number of heap bytes released to OS.
# TYPE go_memstats_heap_released_bytes gauge
go_memstats_heap_released_bytes 3.05135616e+08
# HELP go_memstats_heap_sys_bytes Number of heap bytes obtained from system.
# TYPE go_memstats_heap_sys_bytes gauge
go_memstats_heap_sys_bytes 7.40655104e+08
# HELP go_memstats_last_gc_time_seconds Number of seconds since 1970 of last garbage collection.
# TYPE go_memstats_last_gc_time_seconds gauge
go_memstats_last_gc_time_seconds 1.7302047446302154e+09
# HELP go_memstats_next_gc_bytes Number of heap bytes when next garbage collection will take place.
# TYPE go_memstats_next_gc_bytes gauge
go_memstats_next_gc_bytes 4.16897848e+08                  <---
```
Before setting GOMAXPROCS to 9, the threads for coredns is around the cpu cores(128) on the
node.
```command output
root@node:~# nsenter -t 139471 -n curl http://localhost:9153/metrics -s | grep -E 'gc|heap'
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0.000182547            <---
go_gc_duration_seconds{quantile="0.25"} 0.000304236         <---
go_gc_duration_seconds{quantile="0.5"} 0.000361129          <---
go_gc_duration_seconds{quantile="0.75"} 0.000441626         <---
go_gc_duration_seconds{quantile="1"} 0.001587149            <---
go_gc_duration_seconds_sum 40.283932797
go_gc_duration_seconds_count 94355                          <---
# HELP go_memstats_gc_sys_bytes Number of bytes used for garbage collection system metadata.
# TYPE go_memstats_gc_sys_bytes gauge
go_memstats_gc_sys_bytes 4.0917832e+07
# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 2.2199e+08
# HELP go_memstats_heap_idle_bytes Number of heap bytes waiting to be used.
# TYPE go_memstats_heap_idle_bytes gauge
go_memstats_heap_idle_bytes 3.7965824e+08
# HELP go_memstats_heap_inuse_bytes Number of heap bytes that are in use.
# TYPE go_memstats_heap_inuse_bytes gauge
go_memstats_heap_inuse_bytes 4.40000512e+08
# HELP go_memstats_heap_objects Number of allocated objects.
# TYPE go_memstats_heap_objects gauge
go_memstats_heap_objects 3.196202e+06
# HELP go_memstats_heap_released_bytes Number of heap bytes released to OS.
# TYPE go_memstats_heap_released_bytes gauge
go_memstats_heap_released_bytes 3.20872448e+08
# HELP go_memstats_heap_sys_bytes Number of heap bytes obtained from system.
# TYPE go_memstats_heap_sys_bytes gauge
go_memstats_heap_sys_bytes 8.19658752e+08
# HELP go_memstats_last_gc_time_seconds Number of seconds since 1970 of last garbage collection.
# TYPE go_memstats_last_gc_time_seconds gauge
go_memstats_last_gc_time_seconds 1.730204964758413e+09
# HELP go_memstats_next_gc_bytes Number of heap bytes when next garbage collection will take place.
# TYPE go_memstats_next_gc_bytes gauge
go_memstats_next_gc_bytes 4.4114172e+08                  <---
```
So, for the coredns case, we can also select this solution.

## Another try to leverage the cpu.cfs_burst_us
https://lore.kernel.org/lkml/162452036714.395.9249272896491500398.tip-bot2@tip-bot2/

Kernel introduced cfs_burst from 5.14 to try to solve this unnecessary CPU throttling case.
```diff
diff --git a/kernel/sched/fair.c b/kernel/sched/fair.c
index 7b8990f..4a3e61a 100644
--- a/kernel/sched/fair.c
+++ b/kernel/sched/fair.c
@@ -4626,8 +4626,11 @@ static inline u64 sched_cfs_bandwidth_slice(void)
*/
void __refill_cfs_bandwidth_runtime(struct cfs_bandwidth *cfs_b)
{
- if (cfs_b->quota != RUNTIME_INF)
- cfs_b->runtime = cfs_b->quota;
+ if (unlikely(cfs_b->quota == RUNTIME_INF))
+ return;
+
+ cfs_b->runtime += cfs_b->quota;
+ cfs_b->runtime = min(cfs_b->runtime, cfs_b->quota + cfs_b->burst);
}
```

The unused cpu quota in the previous cfs periods can be left for the current period. But the max
cannot over quota+burst.

Also, there’s a limitation for burst, it cannot be set over the quota.

It means, for the “cpu-limit=9” case.

|||
|----------|-------|
|cpu.cfs_period_us |100000|
|cpu.cfs_quota_us |900000|
|cpu.cfs_burst_us |0|

The maximum value for burst can only be set equal to the quota. If we leverage the cfs burst, set
it to 900000. Then in one scheduling period(100ms), it can use up to a maximum of twice the
quota. 1800% is not enough for our case, there’s still throttling.

# How to find the root cause?
## Figure out where the latency came from.
Tracepkt to hook the network stack layer 2, 3, 4(udp_rcv, udp_sendmsg) and socket layer.
```
[02:34:51.812271] [4026532072] eth0 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:napi_gro_receive
[02:34:51.812359] [4026532072] eth0 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:trace_ip_rcv_core
[02:34:51.812386] [4026532072] cali654b97c3c11 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:ip_finish_output
[02:34:51.812409] [4026532072] cali654b97c3c11 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:__ip_finish_output
[02:34:51.812430] [4026532072] cali654b97c3c11 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:__dev_queue_xmit
[02:34:51.812454] [4026537161] eth0 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:netif_rx
[02:34:51.812476] [4026537161] eth0 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:__netif_receive_skb
[02:34:51.812496] [4026537161] eth0 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:ip_rcv
[02:34:51.812517] [4026537161] eth0 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:trace_ip_rcv_core
[02:34:51.812538] [4026537161] eth0 U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:udp_rcv
*[02:34:51.812556] [0 ] none U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:RET_udp_rcv
*[02:34:51.881658] [0 ] none U:10.167.8.212:49511->10.37.92.17:53 len:129 cksum:5467 FUNC:kfree_skbmem
*[02:34:51.881745] [4026537161] U:0.0.0.0:53->10.167.8.212:49511 len:0 cksum:0 FUNC:RET_udpv6_recvmsg
*[02:34:51.881773] [4026537161] U:0.0.0.0:53->10.167.8.212:49511 len:0 cksum:0 FUNC:RET_sock_recvmsg
[02:34:51.882375] [4026537161] U:0.0.0.0:53->10.167.8.212:49511 len:0 cksum:0 FUNC:sock_sendmsg
[02:34:51.882427] [4026537161] U:0.0.0.0:53->10.167.8.212:49511 len:0 cksum:0 FUNC:udp_sendmsg
[02:34:51.882445] [4026537161] eth0 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:ip_finish_output
[02:34:51.882461] [4026537161] eth0 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:__ip_finish_output
[02:34:51.882473] [4026537161] eth0 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:__dev_queue_xmit
[02:34:51.882486] [4026532072] cali654b97c3c11 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:netif_rx
[02:34:51.882498] [4026532072] cali654b97c3c11 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:__netif_receive_skb
[02:34:51.882511] [4026532072] cali654b97c3c11 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:ip_rcv
[02:34:51.882523] [4026532072] cali654b97c3c11 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:trace_ip_rcv_core
[02:34:51.882536] [4026532072] eth0 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:ip_finish_output
[02:34:51.882548] [4026532072] eth0 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:__ip_finish_output
[02:34:51.882560] [4026532072] eth0 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:__dev_queue_xmit
[02:34:51.882572] [4026532072] eth0 U:10.37.92.17:53->10.167.8.212:49511 len:216 cksum:31386 FUNC:kfree_skbmem|
```
RET_udp_rcv is the function ret hook of udp_rcv, which handles the udp packet receipt and puts
the skb to the sock queue. After this function returns, the kernel stack part is mostly finished. For
the socket layer, it’s more related to the user space.

Userspace calls syscall to receive msg, syscall->sock_recvmsg->udpv6_recvmsg to read msg
from the skb of the current socket’ skb queue and copy to userspace.

From the trace result, the kernel stack has already put the skb to the queue of sock, the **70ms
latency comes from the socket read.**

There are **probably two reasons**:

**One** is the udp socket read/write burst. (such as dns requests/responses burst in a short time).

Coredns listen on :53, all the requests to coredns and responses return from coredns are sharing
this socket. All the read/write operations with :53 are executed serially.
- So if we meet this issue, we can add more listeners and reuse the port “SO_REUSEPORT”.
The socket read/write can be simultaneously.

**The other** is that there are not enough CPUs to handle the socket read operation.

Firstly, check whether there’s requests/response bursts. According to the results from tcpdump,
there is **no significant request-response burst** when latency increases.

Then consider there’s not enough CPU for that time. From the common top command and cpu
usage dashboard, we didn’t see the larger cpu usage than the cpu limit, but the cpu throttled
time is increased.

Considering that the CFS (Completely Fair Scheduler) scheduling period is 100ms, attempt to
monitor the CPU usage every 100ms by using the command `top -p xxx -d 0.1`. This will help
determine if there are any bursts in CPU usage within each 100ms interval.

Then indeed found the cpu burst within 100ms interval. A brief burst occurs approximately every
around 10 seconds. In the absence of any apparent request bursts, the frequent yet
non-continuous high CPU usage is suspected to be caused by Golang's garbage collection.
```command output
nsenter -t 99065 -n curl -s localhost:9153/metrics | grep go_gc_duration_seconds_count
go_gc_duration_seconds_count 140936
```

When garbage collection occurs, the metric "go_gc_duration_seconds_count" will increment.

It was then observed that the garbage collection count increased simultaneously with the burst in
CPU usage.
