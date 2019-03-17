2019年3月15日在huawei eNSP上做OSPF练习，总共开启了6个路由器

![OSPF简单练习] (ospf20190315121433.png)

```txt
R1
#
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255 

#
ospf 1 
 area 0.0.0.1 
  network 1.1.1.1 0.0.0.0 
  network 16.0.0.0 0.0.0.255

R2
#
interface GigabitEthernet0/0/0
 ip address 26.0.0.2 255.255.255.0 
#
interface GigabitEthernet0/0/1
 ip address 24.0.0.2 255.255.255.0 
#
interface GigabitEthernet0/0/2
#
interface NULL0
#
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 2.2.2.2 0.0.0.0 
  network 24.0.0.0 0.0.0.255 
  network 26.0.0.0 0.0.0.255

R3
#
interface GigabitEthernet0/0/0
 ip address 34.0.0.3 255.255.255.0 
#
interface GigabitEthernet0/0/1
 ip address 36.0.0.3 255.255.255.0 
#
interface GigabitEthernet0/0/2
#
interface NULL0
#
interface LoopBack0
 ip address 3.3.3.3 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 3.3.3.3 0.0.0.0 
  network 34.0.0.0 0.0.0.255 
  network 36.0.0.0 0.0.0.255 

R4
#
interface GigabitEthernet0/0/0
 ip address 24.0.0.4 255.255.255.0 
#
interface GigabitEthernet0/0/1
 ip address 34.0.0.4 255.255.255.0 
#
interface GigabitEthernet0/0/2
 ip address 45.0.0.4 255.255.255.0 
#
interface NULL0
#
interface LoopBack0
 ip address 4.4.4.4 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 4.4.4.4 0.0.0.0 
  network 24.0.0.0 0.0.0.255 
  network 34.0.0.0 0.0.0.255 
 area 0.0.0.2 
  network 45.0.0.0 0.0.0.255 

R5
#
interface GigabitEthernet0/0/0
 ip address 45.0.0.5 255.255.255.0 
#
interface GigabitEthernet0/0/1
#
interface GigabitEthernet0/0/2
#
interface NULL0
#
interface LoopBack0
 ip address 5.5.5.5 255.255.255.255 
#
ospf 1 
 area 0.0.0.2 
  network 5.5.5.5 0.0.0.0 
  network 45.0.0.0 0.0.0.255 

R6
#
interface GigabitEthernet0/0/0
 ip address 26.0.0.6 255.255.255.0 
#
interface GigabitEthernet0/0/1
 ip address 36.0.0.6 255.255.255.0 
#
interface GigabitEthernet0/0/2
 ip address 16.0.0.6 255.255.255.0 
#
interface NULL0
#
interface LoopBack0
 ip address 6.6.6.6 255.255.255.255 
#
ospf 1 
 area 0.0.0.0 
  network 6.6.6.6 0.0.0.0 
  network 26.0.0.0 0.0.0.255 
  network 36.0.0.0 0.0.0.255 
 area 0.0.0.1 
  network 16.0.0.0 0.0.0.255 
```

开启OSPF，R6为列

```txt
[R6]ospf 1
[R6-ospf-1]area 0
[R6-ospf-1-area-0.0.0.0]netwo	
[R6-ospf-1-area-0.0.0.0]network 6.6.6.6 0.0.0.0
[R6-ospf-1-area-0.0.0.0]network 26.0.0.0 0.0.0.255
[R6-ospf-1-area-0.0.0.0]network 36.0.0.0 0.0.0.255
[R6-ospf-1-area-0.0.0.0]q
[R6-ospf-1]area 1
[R6-ospf-1-area-0.0.0.1]net 16.0.0.0 0.0.0.255
[R6-ospf-1]q
[R6]
```
验证OSPF搭建情况

```txt
<R5>ping 1.1.1.1
  PING 1.1.1.1: 56  data bytes, press CTRL_C to break
    Reply from 1.1.1.1: bytes=56 Sequence=1 ttl=252 time=150 ms
    Reply from 1.1.1.1: bytes=56 Sequence=2 ttl=252 time=40 ms
    Reply from 1.1.1.1: bytes=56 Sequence=3 ttl=252 time=40 ms
    Reply from 1.1.1.1: bytes=56 Sequence=4 ttl=252 time=40 ms
    Reply from 1.1.1.1: bytes=56 Sequence=5 ttl=252 time=50 ms

  --- 1.1.1.1 ping statistics ---
    5 packet(s) transmitted
    5 packet(s) received
    0.00% packet loss
    round-trip min/avg/max = 40/64/150 ms
<R5>
```