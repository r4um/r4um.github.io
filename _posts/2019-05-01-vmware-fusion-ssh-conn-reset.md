---
layout: post
title: "VMware Fusion: ssh connection reset"
comments: false
description: "VMWare Fusion: ssh connection reset"
keywords: "vmware, vmware fusion, tcpdump, tshark, ssh, qos, linux"
---

After setting up a new archlinux install on vmware fusion on macOS, git clones
on ssh protocol are not working.

```
$ git clone git@github.com:r4um/dotfiles.git
Cloning into 'dotfiles'...
Warning: Permanently added the RSA host key for IP address '192.30.253.112' to the list of known hosts.
client_loop: send disconnect: Broken pipe
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

Turning on verbose logging and checking its not a auth related issue.

```
$ ssh -vv git@github.com
---removed messages----
debug1: Authentication succeeded (publickey).
Authenticated to github.com ([192.30.253.113]:22).
---removed messages----
debug2: channel 0: open confirm rwindow 32000 rmax 35000
client_loop: send disconnect: Broken pipe
```

Auth succeeds, oddly rest of the  linux virtual machines (ubuntu etc.) don't have this issue.

We should see whats happening on the wire. To make debugging easier
we will make connection attempts to just one of hosts under github.com,  `192.30.253.113`. Capturing packets inside the VM.

```
$ tshark -P -w /tmp/192.30.253.113.pcap -n -i ens33 host 192.30.253.113
Running as user "root" and group "root". This could be dangerous.
Capturing on 'ens33'
    1 0.000000000 172.16.27.131 → 192.30.253.113 TCP 74 57964 → 22 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=4096655165 TSecr=0 WS=128
    2 0.216464591 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [SYN, ACK] Seq=0 Ack=1 Win=64240 Len=0 MSS=1460
    3 0.216536516 172.16.27.131 → 192.30.253.113 TCP 54 57964 → 22 [ACK] Seq=1 Ack=1 Win=64240 Len=0
    4 0.216932907 172.16.27.131 → 192.30.253.113 SSH 75 Client: Protocol (SSH-2.0-OpenSSH_8.0)
    5 0.217164293 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=1 Ack=22 Win=64240 Len=0
    6 0.438223288 192.30.253.113 → 172.16.27.131 SSHv2 911 Server: Protocol (SSH-2.0-babeld-d125c03e), Key Exchange Init
    7 0.438247789 172.16.27.131 → 192.30.253.113 TCP 54 57964 → 22 [ACK] Seq=22 Ack=858 Win=63418 Len=0
    8 0.438726577 172.16.27.131 → 192.30.253.113 SSHv2 1446 Client: Key Exchange Init
    9 0.441217324 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=858 Ack=1414 Win=64240 Len=0
   10 0.441228638 172.16.27.131 → 192.30.253.113 SSHv2 102 Client: Diffie-Hellman Key Exchange Init
   11 0.441455806 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=858 Ack=1462 Win=64240 Len=0
   12 0.923364269 192.30.253.113 → 172.16.27.131 SSHv2 662 Server: Diffie-Hellman Key Exchange Reply
   13 0.923391020 192.30.253.113 → 172.16.27.131 SSHv2 70 Server: New Keys
   14 0.923450139 172.16.27.131 → 192.30.253.113 TCP 54 57964 → 22 [ACK] Seq=1462 Ack=1482 Win=63418 Len=0
   15 0.925596861 172.16.27.131 → 192.30.253.113 SSHv2 70 Client: New Keys
   16 0.925757240 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=1482 Ack=1478 Win=64240 Len=0
   17 0.928386942 172.16.27.131 → 192.30.253.113 SSHv2 98 Client: Encrypted packet (len=44)
   18 0.928576580 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=1482 Ack=1522 Win=64240 Len=0
   19 1.140991189 192.30.253.113 → 172.16.27.131 SSHv2 226 Server: Encrypted packet (len=172)
   20 1.182513570 172.16.27.131 → 192.30.253.113 TCP 54 57964 → 22 [ACK] Seq=1522 Ack=1654 Win=63418 Len=0
   21 1.355749087 192.30.253.113 → 172.16.27.131 SSHv2 98 Server: Encrypted packet (len=44)
   22 1.355769795 172.16.27.131 → 192.30.253.113 TCP 54 57964 → 22 [ACK] Seq=1522 Ack=1698 Win=63418 Len=0
   23 1.355923126 172.16.27.131 → 192.30.253.113 SSHv2 114 Client: Encrypted packet (len=60)
   24 1.356153167 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=1698 Ack=1582 Win=64240 Len=0
   25 1.572804908 192.30.253.113 → 172.16.27.131 SSHv2 98 Server: Encrypted packet (len=44)
   26 1.572964095 172.16.27.131 → 192.30.253.113 SSHv2 418 Client: Encrypted packet (len=364)
   27 1.573447738 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=1742 Ack=1946 Win=64240 Len=0
   28 1.795074003 192.30.253.113 → 172.16.27.131 SSHv2 378 Server: Encrypted packet (len=324)
   29 1.797953675 172.16.27.131 → 192.30.253.113 SSHv2 698 Client: Encrypted packet (len=644)
   30 1.798275441 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=2066 Ack=2590 Win=64240 Len=0
   31 2.014906486 192.30.253.113 → 172.16.27.131 SSHv2 82 Server: Encrypted packet (len=28)
   32 2.015730861 172.16.27.131 → 192.30.253.113 SSHv2 106 Client: Encrypted packet (len=52)
   33 2.015972915 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [ACK] Seq=2094 Ack=2642 Win=64240 Len=0
   34 2.231551154 192.30.253.113 → 172.16.27.131 SSHv2 98 Server: Encrypted packet (len=44)
   35 2.231718306 172.16.27.131 → 192.30.253.113 SSHv2 446 Client: Encrypted packet (len=392)
   36 2.231788379 192.30.253.113 → 172.16.27.131 TCP 60 22 → 57964 [RST] Seq=2138 Win=64240 Len=0
   37 2.334074052 192.30.253.113 → 172.16.27.131 SSHv2 98 Server: [TCP Spurious Retransmission] , Encrypted packet (len=44)
   38 2.334102359 172.16.27.131 → 192.30.253.113 TCP 54 57964 → 22 [RST] Seq=2642 Win=0 Len=0
```

From capture above seems like github host is reseting the connection after sending packet no. `35`, reset at packet no. `36`. Since this virtual machine's network is [NAT'd](https://kb.vmware.com/s/article/1022264)
we should capture the interaction from outside the VM, capturing on the host

```
$ tshark -n -i en0 host 192.30.253.113
Capturing on 'Wi-Fi: en0'
    1   0.000000 192.168.86.161 → 192.30.253.113 TCP 78 64174 → 22 [SYN, ECN, CWR] Seq=0 Win=65535 Len=0 MSS=1460 WS=64 TSval=1380790781 TSecr=0 SACK_PERM=1
    2   0.213654 192.30.253.113 → 192.168.86.161 TCP 74 22 → 64174 [SYN, ACK, ECN] Seq=0 Ack=1 Win=28480 Len=0 MSS=1414 SACK_PERM=1 TSval=67380909 TSecr=1380790781 WS=1024
------- removed entries for brevity ---------  
   29   2.001450 192.168.86.161 → 192.30.253.113 TCP 66 64174 → 22 [ACK] Seq=2590 Ack=2094 Win=131008 Len=0 TSval=1380792766 TSecr=67381356
   30   2.002266 192.168.86.161 → 192.30.253.113 SSHv2 118 Client: Encrypted packet (len=52)
   31   2.216095 192.30.253.113 → 192.168.86.161 SSHv2 110 Server: Encrypted packet (len=44)
   32   2.216187 192.168.86.161 → 192.30.253.113 TCP 66 64174 → 22 [ACK] Seq=2642 Ack=2138 Win=131008 Len=0 TSval=1380792979 TSecr=67381410
   33   2.321182 192.168.86.161 → 192.30.253.113 TCP 66 64174 → 22 [FIN, ACK] Seq=2642 Ack=2138 Win=131072 Len=0 TSval=1380793083 TSecr=67381410
   34   2.535883 192.30.253.113 → 192.168.86.161 TCP 66 22 → 64174 [FIN, ACK] Seq=2138 Ack=2643 Win=36864 Len=0 TSval=67381490 TSecr=1380793083
   35   2.535994 192.168.86.161 → 192.30.253.113 TCP 66 64174 → 22 [ACK] Seq=2643 Ack=2139 Win=131072 Len=0 TSval=1380793294 TSecr=67381490
```

Looks like we are closing the connection from our end, packet no. `33` sends  `FIN-ACK` to initiate closing the connection. It is fair to assume the VMware NAT feature is doing this, but we can confirm this via macOS tcpdump, which has an option of displaying the process ID / name via the `-k NP` option. 

```
$ tcpdump -k NP -ttt -n -i en0 host 192.30.253.113
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
 00:00:00.000000 pid vmnet-natd.70337 svc BE IP 192.168.86.161.64245 > 192.30.253.113.22: Flags [S], seq 263919761, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 1381897829 ecr 0,sackOK,eol], length 0
 00:00:00.216146 IP 192.30.253.113.22 > 192.168.86.161.64245: Flags [S.], seq 800967607, ack 263919762, win 28480, options [mss 1414,sackOK,TS val 4163052253 ecr 1381897829,nop,wscale 10], length 0
------- removed entries for brevity ---------  
  00:00:00.102167 pid vmnet-natd.70337 svc BE IP 192.168.86.161.64245 > 192.30.253.113.22: Flags [F.], seq 2642, ack 2138, win 2048, options [nop,nop,TS val 1381900149 ecr 4163052759], length 0
 00:00:00.216198 IP 192.30.253.113.22 > 192.168.86.161.64245: Flags [F.], seq 2138, ack 2643, win 36, options [nop,nop,TS val 4163052839 ecr 1381900149], length 0
 00:00:00.000100  pktflags 0x4 IP 192.168.86.161.64245 > 192.30.253.113.22: Flags [.], ack 2139, win 2048, options [nop,nop,TS val 1381900364 ecr 4163052839], length 0
```

The capture confirms, it is in fact the `vmware-natd` process responsible for this interaction.

Going back to our first packet capture from inside the virtual machine host, it is odd
that packet no. `32` goes through fine but packet no. `35` (both are `SSHv2 Client` packets) somehow irks `vmware-natd` to close connection and send a reset. 

Let's take a diff between these two packets to see if something is different.

```shell
$ tshark -r /tmp/192.30.253.113.pcap -T json 'frame.number == 32' > 32
Running as user "root" and group "root". This could be dangerous.
$ tshark -r /tmp/192.30.253.113.pcap -T json 'frame.number == 35' > 35
Running as user "root" and group "root". This could be dangerous.
$ diff -U1 32 35
--- 32	2019-05-02 00:32:08.695301950 +0530
+++ 35	2019-05-02 00:32:11.834955476 +0530
@@ -13,11 +13,11 @@
           "frame.encap_type": "1",
-          "frame.time": "May  2, 2019 00:31:57.870906798 IST",
+          "frame.time": "May  2, 2019 00:31:58.088527303 IST",
           "frame.offset_shift": "0.000000000",
-          "frame.time_epoch": "1556737317.870906798",
-          "frame.time_delta": "0.000826482",
+          "frame.time_epoch": "1556737318.088527303",
+          "frame.time_delta": "0.000154310",
           "frame.time_delta_displayed": "0.000000000",
-          "frame.time_relative": "2.023470351",
-          "frame.number": "32",
-          "frame.len": "106",
-          "frame.cap_len": "106",
+          "frame.time_relative": "2.241090856",
+          "frame.number": "35",
+          "frame.len": "446",
+          "frame.cap_len": "446",
           "frame.marked": "0",
@@ -48,9 +48,9 @@
           "ip.hdr_len": "20",
-          "ip.dsfield": "0x00000000",
+          "ip.dsfield": "0x00000048",
           "ip.dsfield_tree": {
-            "ip.dsfield.dscp": "0",
+            "ip.dsfield.dscp": "18",
             "ip.dsfield.ecn": "0"
           },
-          "ip.len": "92",
-          "ip.id": "0x00002118",
+          "ip.len": "432",
+          "ip.id": "0x00002119",
           "ip.flags": "0x00004000",
@@ -64,3 +64,3 @@
           "ip.proto": "6",
-          "ip.checksum": "0x00009460",
+          "ip.checksum": "0x000092c3",
           "ip.checksum.status": "2",
@@ -81,6 +81,6 @@
           "tcp.stream": "0",
-          "tcp.len": "52",
-          "tcp.seq": "2590",
-          "tcp.nxtseq": "2642",
-          "tcp.ack": "2094",
+          "tcp.len": "392",
+          "tcp.seq": "2642",
+          "tcp.nxtseq": "3034",
+          "tcp.ack": "2138",
           "tcp.hdr_len": "20",
@@ -103,3 +103,3 @@
           "tcp.window_size_scalefactor": "-2",
-          "tcp.checksum": "0x00008572",
+          "tcp.checksum": "0x000086c6",
           "tcp.checksum.status": "2",
@@ -107,13 +107,13 @@
           "tcp.analysis": {
-            "tcp.analysis.acks_frame": "31",
-            "tcp.analysis.ack_rtt": "0.000826482",
+            "tcp.analysis.acks_frame": "34",
+            "tcp.analysis.ack_rtt": "0.000154310",
             "tcp.analysis.initial_rtt": "0.217767932",
-            "tcp.analysis.bytes_in_flight": "52",
-            "tcp.analysis.push_bytes_sent": "52"
+            "tcp.analysis.bytes_in_flight": "392",
+            "tcp.analysis.push_bytes_sent": "392"
           },
           "Timestamps": {
-            "tcp.time_relative": "2.023470351",
-            "tcp.time_delta": "0.000826482"
+            "tcp.time_relative": "2.241090856",
+            "tcp.time_delta": "0.000154310"
           },
-          "tcp.payload": "be:55:58:b7:eb:f9:36:d3:6a:4a:33:d6:f3:79:db:cc:45:88:29:88:4e:07:68:9c:b2:7c:8a:49:85:74:fa:34:f2:f8:5e:ee:b8:07:15:77:13:10:aa:be:b0:3c:ae:1e:4b:5a:e7:5b"
+          "tcp.payload": "d6:6a:97:0a:32:82:75:94:37:bd:25:09:1a:9c:0c:fd:fb:4a:23:0b:72:a1:af:f7:46:6d:03:e6:a4:de:e9:5a:43:aa:85:26:48:25:a1:ef:01:d2:e4:22:55:d8:ca:e6:2b:ee:7a:74:19:f9:84:e9:68:8b:b9:74:d6:f5:46:fc:0f:5c:0f:e0:bc:70:59:3b:04:9f:62:be:93:41:e1:a4:22:e0:2f:3f:ce:2e:ee:79:b1:0d:8b:14:0a:83:bd:cd:15:86:71:75:dc:f8:bf:d9:81:61:75:41:27:90:80:9d:a0:85:77:83:f9:0f:37:29:7e:64:8f:80:46:63:ff:6e:f2:b8:67:91:6a:e6:c2:74:74:72:f5:b0:10:0d:8c:73:9a:f4:67:bc:e4:b8:d9:8e:48:d3:79:90:e2:d0:38:ad:26:5c:9c:42:60:3c:0c:4a:e6:5a:37:11:62:9e:30:3a:f1:57:3a:10:87:52:09:34:a2:6b:89:75:f1:aa:40:c3:27:b0:8b:e4:94:f4:0e:a4:fe:4f:db:b8:89:ca:ce:15:70:d2:e2:07:b7:66:1a:ba:68:31:8a:a1:2d:93:d5:54:3f:7b:0e:2e:6a:16:7d:fe:ef:d1:8d:3f:c7:d7:6c:71:17:d8:30:3e:86:67:d9:b8:2a:a1:20:c5:ae:1e:16:2f:ca:fe:8d:4c:5d:c7:81:f5:07:c6:8e:6a:bb:c9:ff:4f:e9:98:28:90:81:2e:c5:4f:f0:77:5a:bb:67:27:ad:ca:fd:a9:ef:17:81:36:12:b7:81:46:ee:2f:22:9a:f9:11:b2:3e:cf:b6:bb:0a:f4:d6:8a:53:2b:6f:aa:63:cb:50:fa:fa:e8:6b:63:6e:c1:0d:0d:a3:09:b3:eb:1e:0a:ae:91:a5:d2:50:b8:ac:13:79:b7:9f:73:c8:8d:91:e2:5b:1b:f1:a6:52:7d:bb:d7:1b:18:20:b0:98:d3:6d:91:f9:42:ba:b1:f4:67:89:24:42:25:5d:94:8b:40:59:07:cc:c8:e6:bb:ac:9d:04:de:0a"
         },
@@ -121,5 +121,5 @@
           "SSH Version 2 (encryption:chacha20-poly1305@openssh.com mac:<implicit> compression:none)": {
-            "ssh.packet_length_encrypted": "be:55:58:b7",
-            "ssh.encrypted_packet": "eb:f9:36:d3:6a:4a:33:d6:f3:79:db:cc:45:88:29:88:4e:07:68:9c:b2:7c:8a:49:85:74:fa:34:f2:f8:5e:ee",
-            "ssh.mac": "b8:07:15:77:13:10:aa:be:b0:3c:ae:1e:4b:5a:e7:5b"
+            "ssh.packet_length_encrypted": "d6:6a:97:0a",
+            "ssh.encrypted_packet": "32:82:75:94:37:bd:25:09:1a:9c:0c:fd:fb:4a:23:0b:72:a1:af:f7:46:6d:03:e6:a4:de:e9:5a:43:aa:85:26:48:25:a1:ef:01:d2:e4:22:55:d8:ca:e6:2b:ee:7a:74:19:f9:84:e9:68:8b:b9:74:d6:f5:46:fc:0f:5c:0f:e0:bc:70:59:3b:04:9f:62:be:93:41:e1:a4:22:e0:2f:3f:ce:2e:ee:79:b1:0d:8b:14:0a:83:bd:cd:15:86:71:75:dc:f8:bf:d9:81:61:75:41:27:90:80:9d:a0:85:77:83:f9:0f:37:29:7e:64:8f:80:46:63:ff:6e:f2:b8:67:91:6a:e6:c2:74:74:72:f5:b0:10:0d:8c:73:9a:f4:67:bc:e4:b8:d9:8e:48:d3:79:90:e2:d0:38:ad:26:5c:9c:42:60:3c:0c:4a:e6:5a:37:11:62:9e:30:3a:f1:57:3a:10:87:52:09:34:a2:6b:89:75:f1:aa:40:c3:27:b0:8b:e4:94:f4:0e:a4:fe:4f:db:b8:89:ca:ce:15:70:d2:e2:07:b7:66:1a:ba:68:31:8a:a1:2d:93:d5:54:3f:7b:0e:2e:6a:16:7d:fe:ef:d1:8d:3f:c7:d7:6c:71:17:d8:30:3e:86:67:d9:b8:2a:a1:20:c5:ae:1e:16:2f:ca:fe:8d:4c:5d:c7:81:f5:07:c6:8e:6a:bb:c9:ff:4f:e9:98:28:90:81:2e:c5:4f:f0:77:5a:bb:67:27:ad:ca:fd:a9:ef:17:81:36:12:b7:81:46:ee:2f:22:9a:f9:11:b2:3e:cf:b6:bb:0a:f4:d6:8a:53:2b:6f:aa:63:cb:50:fa:fa:e8:6b:63:6e:c1:0d:0d:a3:09:b3:eb:1e:0a:ae:91:a5:d2:50:b8:ac:13:79:b7:9f:73:c8:8d:91:e2:5b:1b:f1:a6:52:7d:bb:d7:1b:18:20:b0:98:d3:6d:91:f9:42:ba:b1:f4:67:89:24:42",
+            "ssh.mac": "25:5d:94:8b:40:59:07:cc:c8:e6:bb:ac:9d:04:de:0a"
           }
```

The following section of diff looks interesting, there is [ip.dsfield](https://en.wikipedia.org/wiki/Differentiated_services) field being set to value `0x48` in packet no. `35` but is `0` in no. `32`.

```
@@ -48,9 +48,9 @@
           "ip.hdr_len": "20",
-          "ip.dsfield": "0x00000000",
+          "ip.dsfield": "0x00000048",
           "ip.dsfield_tree": {
-            "ip.dsfield.dscp": "0",
+            "ip.dsfield.dscp": "18",
             "ip.dsfield.ecn": "0"
           },
-          "ip.len": "92",
-          "ip.id": "0x00002118",
+          "ip.len": "432",
+          "ip.id": "0x00002119",
           "ip.flags": "0x00004000",
```

Is this triggering a reset ? Putting together a quick python script to try it out, since this is IP layer interaction independent of TCP port, we pick 80 (HTTP) to interact with. 

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.IPPROTO_IP, socket.IP_TOS, bytes([0x48]))
s.connect(("192.30.253.113" , 80))
s.send(b"GET / HTTP/1.0\r\n\r\n")
r = s.recv(4096)
s.close
print(r)
```

Running it

```
$ python ip_ds_48.py
b'HTTP/1.1 301 Moved Permanently\r\nContent-length: 0\r\nLocation: https:///\r\nConnection: close\r\n\r\n'
```

We got a response, no resets. Perhaps its switching of the feild in between that's causing it ? Changing script to set TOS after connect

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.30.253.113" , 80))
s.setsockopt(socket.IPPROTO_IP, socket.IP_TOS, bytes([0x48]))
s.send(b"GET / HTTP/1.0\r\n\r\n")
r = s.recv(4096)
s.close
print(r)
```

Running it

```
$ python ip_ds_48.py
Traceback (most recent call last):
  File "ip_ds_48.py", line 7, in <module>
    r = s.recv(4096)
ConnectionResetError: [Errno 104] Connection reset by peer
```

Connection reset, confirmed. 

We can see `ip.dsfield.dscp` field is set to `18`, going through the [wikipedia page](https://en.wikipedia.org/wiki/Differentiated_services#Commonly_used_DSCP_values) ssh is setting [Assured Forwarding](https://en.wikipedia.org/wiki/Differentiated_services#Assured_Forwarding) `AF21` DSCP value. Going through change log and commit 
history it was done with [this change](https://github.com/openssh/openssh-portable/commit/5ee8448ad7c306f05a9f56769f95336a8269f379) starting with version 7.8p1. This VM
has version 8.0p1, rest of the VMs have older version of openssh hence don't see this problem.

Going through the [ssh_config](https://man.openbsd.org/ssh_config#IPQoS) man page, this can be controlled via the `IPQoS` option. Setting it to `lowdelay` for example, things work fine.

```
$ ssh -o IPQoS=lowdelay git@192.30.253.113
PTY allocation request failed on channel 0
Hi r4um! You've successfully authenticated, but GitHub does not provide shell access.
Connection to 192.30.253.113 closed.
```

Follow up interesting excercises

*  Reverse engineer `vmware-natd` to see why it does that, patch to turn off this behaviour.
*  Why doesn't tcpdump on linux have a `-k` like option, what would it take to add it ?