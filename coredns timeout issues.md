# Udp packet drop
phenomenon: Intermittent in-cluster dns query timeout.

Tracing dns packets can found the root cause.
Reason: There's the udp packet drop on coredns instances.
  
After scaling coredns instances from 13 up to 30 instances, there are still udp receive buffer errors. Taken twice netstat on one coredns instance, the drop rate is about 0.3% at that time.
```
-bash-4.2# nsenter --net=/proc/276244/ns/net netstat -su
IcmpMsg:
   InType3: 28878
   OutType3: 1997
Udp:
   1419512570 packets received
   3046 packets to unknown port received.
   7598078 packet receive errors
   1419512573 packets sent
   7598078 receive buffer errors
   0 send buffer errors
UdpLite:
IpExt:
   InOctets: 159340884996
   OutOctets: 670716882321
   InNoECTPkts: 1586236116
```
```
-bash-4.2# nsenter --net=/proc/276244/ns/net netstat -su
IcmpMsg:
   InType3: 28878
   OutType3: 1997
Udp:
   1419620567 packets received
   3046 packets to unknown port received.
   7598407 packet receive errors
   1419620570 packets sent
   7598407 receive buffer errors
   0 send buffer errors
UdpLite:
IpExt:
   InOctets: 159353028962
   OutOctets: 670767728339
   InNoECTPkts: 1586357040
```
Solution: Change socket receive buffer size for coredns.

The default value of rmem_max and rmem_default is as follow:
```
-bash-4.2# cat /proc/sys/net/core/rmem_max
524288
-bash-4.2# cat /proc/sys/net/core/rmem_default
212992
```
For the receive buffer size, one option to change in coredns code. Such as:
```
diff --git a/plugin/pkg/reuseport/listen_go111.go b/plugin/pkg/reuseport/listen_go111.go
index fa6f365d..f8fdcd45 100644
--- a/plugin/pkg/reuseport/listen_go111.go
+++ b/plugin/pkg/reuseport/listen_go111.go
@@ -18,6 +18,9 @@ func control(network, address string, c syscall.RawConn) error {
               if err := unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1); err != nil {
                       log.Warningf("Failed to set SO_REUSEPORT on socket: %s", err)
               }
+               if err := unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_RCVBUF, 524288); err != nil {
+                       log.Warningf("Failed to set SO_RCVBUF on socket: %s", err)
+               }
       })
       return nil
}
```
But if want to set it larger than 524288(default value in /proc/sys/net/core/rmem_max) in socketopt. the system parameter /proc/sys/net/core/rmem_max should be increased first. Because, if no set unix.SO_RCVBUF in sockopt, it will use the value `/proc/sys/net/core/rmem_default`. If set unix.SO_RCVBUF in sockopt, it will use the min(/proc/sys/net/core/rmem_max, unix.SO_RCVBUF).
