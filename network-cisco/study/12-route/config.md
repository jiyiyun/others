R1

conf t
int loo 0
ip add 1.1.1.1 255.255.255.255
no shut
int e1/0
ip add 1.2.0.1 255.255.255.0
no shut
int e1/1
ip add 1.5.0.1 255.255.255.0
no shut
int e1/2
ip add 1.7.0.1 255.255.255.0
no shut
end

show cdp ne
show ip int br

-----------------------------------
R2

conf t
int loo 0
ip add 2.2.2.2 255.255.255.255
no shut
int e1/0
ip add 2.4.0.2 255.255.255.0
no shut
int e1/1
ip add 1.2.0.2 255.255.255.0
no shut
int e1/2
ip add 2.8.0.2 255.255.255.0
no shut
int e1/3
ip add 2.3.0.2 255.255.255.0
no shut
end

show cdp ne
show ip int br

----------------------------------------------
R3

conf t
int loo 0
ip add 3.3.3.3 255.255.255.255
no shut
int e1/0
ip add 2.3.0.3 255.255.255.0
no shut
int e1/1
ip add 3.6.0.3 255.255.255.0
no shut
int e1/2
ip add 3.5.0.3 255.255.255.0
no shut
end

show cdp ne
show ip int br

-----------------------------------------
R4

conf t
int loo 0
ip add 4.4.4.4 255.255.255.255
no shut
int e1/0
ip add 2.4.0.4 255.255.255.0
no shut
int e1/1
ip add 4.11.0.4 255.255.255.0
no shut
int e1/2
ip add 4.9.0.4 255.255.255.0
no shut
int e1/3
ip add 4.6.0.4 255.255.255.0
no shut
end

show cdp ne
show ip int br

------------------------------------------
R5

conf t
int loo 0
ip add 5.5.5.5 255.255.255.255
no shut
int e1/0
ip add 3.5.0.5 255.255.255.0
no shut
int e1/1
ip add 1.5.0.5 255.255.255.0
no shut
int e1/2
ip add 5.12.0.5 255.255.255.0
no shut
int e1/3
ip add 5.7.0.5 255.255.255.0
no shut
end

show cdp ne
show ip int br

------------------------------------------
R6

conf t
int loo 0
ip add 6.6.6.6 255.255.255.255
no shut
int e1/0
ip add 4.6.0.6 255.255.255.0
no shut
int e1/1
ip add 6.12.0.6 255.255.255.0
no shut
int e1/2
ip add 3.6.0.6 255.255.255.0
no shut
end

show cdp ne
show ip int br

-------------------------------------
R7

conf t
int loo 0
ip add 7.7.7.7 255.255.255.255
no shut
int e1/0
ip add 7.8.0.7 255.255.255.0
no shut
int e1/1
ip add 1.7.0.7 255.255.255.0
no shut
int e1/2
ip add 7.10.0.7 255.255.255.0
no shut
int e1/3
ip add 5.7.0.7 255.255.255.0
no shut
end

show cdp ne
show ip int br

----------------------------------------
R8

conf t
int loo 0
ip add 8.8.8.8 255.255.255.255
no shut
int e1/0
ip add 2.8.0.8 255.255.255.0
no shut
int e1/1
ip add 7.8.0.8 255.255.255.0
no shut
int e1/2
ip add 8.9.0.8 255.255.255.0
no shut
end

show cdp ne
show ip int br

--------------------------------------------
R9

conf t
int loo 0
ip add 9.9.9.9 255.255.255.255
no shut
int e1/0
ip add 8.9.0.9 255.255.255.0
no shut
int e1/1
ip add 9.10.0.9 255.255.255.0
no shut
int e1/2
ip add 4.9.0.9 255.255.255.0
no shut
end

show cdp ne
show ip int br

----------------------------------------------
R10

conf t
int loo 0
ip add 10.10.10.10 255.255.255.255
no shut
int e1/0
ip add 9.10.0.10 255.255.255.0
no shut
int e1/1
ip add 10.11.0.10 255.255.255.0
no shut
int e1/2
ip add 10.12.0.10 255.255.255.0
no shut
int e1/3
ip add 7.10.0.10 255.255.255.0
no shut
end

show cdp ne
show ip int br

---------------------------------------
R11

conf t
int loo 0
ip add 11.11.11.11 255.255.255.255
no shut
int e1/0
ip add 4.11.0.11 255.255.255.0
no shut
int e1/1
ip add 10.11.0.11 255.255.255.0
no shut
int e1/2
ip add 11.12.0.11 255.255.255.0
no shut
end

show cdp ne
show ip int br

-----------------------------------------------
R12

conf t
int loo 0
ip add 12.12.12.12 255.255.255.255
no shut
int e1/0
ip add 6.12.0.12 255.255.255.0
no shut
int e1/1
ip add 11.12.0.12 255.255.255.0
no shut
int e1/2
ip add 5.12.0.12 255.255.255.0
no shut
int e1/3
ip add 10.12.0.12 255.255.255.0
no shut
end

show cdp ne
show ip int br
