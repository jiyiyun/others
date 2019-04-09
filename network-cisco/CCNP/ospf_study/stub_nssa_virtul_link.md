- Stub Area
- Totally Stub Area
- NSSA (Not So Stub Area)
- Totally NSSA


配置Stub Area
>#area 4 stub

只需在ospf 1 进程配置界面中添加一条area 4 stub，注意要在所有stub area 路由器宣告一遍，这里area 4是stub area

配置Totally Stub Area
>#area 1 stub no-summary

stub area配置多了一个参数no-summary

配置NSSA
>area 3 nssa

只添加一条area 3 nssa，要在该area每个路由器中宣告一遍

配置Totally NSSA
>#area 5 nssa no-summary

只添加一条#area 5 nssa no-summary即可，要在该area每个路由器中宣告一遍

配置Virtual Link
```txt
R2(config)#router ospf 1
R2(config-router)#net 2.2.2.2 0.0.0.0 area 0
R2(config-router)#area 2 virtual-link 12.12.12.12
R2(config-router)#exit

R2的环回口在area 0中

R12(config)#router ospf 1
R12(config-router)#net 12.12.12.12 0.0.0.0 area 2
R12(config-router)#net 2.12.0.0 0.0.0.255 area 2
R12(config-router)#area 2 virtual-link 2.2.2.2

R12的环回口在area 2中
```