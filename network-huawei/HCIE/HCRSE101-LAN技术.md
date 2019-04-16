MAC地址表分为：动态表项、静态表项、黑洞表项
- 动态表项：接口通过报文中源MAC地址学习获得，表项可老化，在系统复位，接口板热插拔或接口版复位后，动态表项丢失。可通过查看动态MAC地址表项，可以判断两台设备之间是否有数据转发；也可以通过指定动态MAC地址表项个数，可以获取接口下通讯的用户数
- 静态表项：用户手动添加，并下发到各接口板，表项不可老化。在系统复位、接口板热插拔或接口板复位后，保存的表项不会丢失。一条静态MAC地址表项只能绑定一个出接口。一个接口和MAC地址静态绑定后，不影响该接口动态MAC地址表项的学习，通过绑定静态MAC地址表项，可以保证合法用户的使用，防止其它用户使用该MAC进行攻击
- 黑洞表项：由用户手工配置，并下发到各接口板，表项不可老化。在系统复位、接口板热插拔或接口复位后，保存的表项不会丢失。通过配置黑洞MAC地址表项，可以过滤非法用户。

清除端口配置
```txt
[~SW1-GE1/0/5]clear configuration ?
  candidate  Uncommitted configuration on the current session
  this       This view's current configuration
	
[~SW1-GE1/0/5]clear configuration this 
Warning: All running configurations of the interface will be cleared immediately
. Continue? [Y/N]:y
Info: Total 2 commands executed, 2 successful, 0 failed.
[~SW1-GE1/0/5]clear configuration candidate 
```
设置mac-address黑洞
```txt
[~SW1]mac-address blackhole 5489-98e1-1b1c vlan 10

[~SW1]dis mac-address
Flags: * - Backup  
BD   : bridge-domain   Age : dynamic MAC learned time in seconds
-------------------------------------------------------------------------------
MAC Address    VLAN/VSI   Learned-From        Type                Age
-------------------------------------------------------------------------------
5489-98e1-1b1c 10/-          -                   blackhole             -
5489-9828-136a 10/-          Eth-Trunk1          dynamic               -
5489-9846-7953 10/-          GE1/0/3             dynamic               -
-------------------------------------------------------------------------------
Total items: 3
```

MAC老化时间(默认300秒)
```txt
[~SW2]mac-address aging-time ?
  <0,60-1000000>  Aging time, in seconds. The default is 300. The value 0
                  indicates that the MAC address aging function is disabled
```

查看交换机CPU使用率
```txt
[~SW2]dis cpu
CPU utilization statistics at 2019-04-12 17:37:27 338 ms
System CPU Using Percentage :  17%
CPU utilization for five seconds: 0%, one minute: 0%, five minutes: 0%.
Max CPU Usage :                86%
Max CPU Usage Stat. Time : 2019-04-12 16:12:36 637 ms
State: Non-overload
Overload threshold:  90%, Overload clear threshold:  75%, Duration:  480s
---------------------------
ServiceName  UseRate   
---------------------------
SYSTEM           17%
---------------------------
CPU Usage Details
----------------------------------------------------------------
CPU     Current  FiveSec   OneMin  FiveMin  Max MaxTime         
----------------------------------------------------------------
cpu0        28%       0%       0%       0%   0% 2019-04-12 16:11:32
cpu1         6%       0%       0%       0%   0% 2019-04-12 16:11:32
----------------------------------------------------------------
```