---
title: 我的第一篇博文
published: 2023-10-01
description: 这是使用Astro创建的第一篇博文
# image: ./cover.jpg
tags: [Astro, 教程]
category: 前端开发
---

> `la as` OR `x/5i $pc`
# phase_1
```sh
(gdb) disas phase_1
# mov    $0x402400,%esi
# call   0x401338 <strings_not_equal>
# test   %eax,%eax
# je     0x400ef7 <phase_1+23>
# call   0x40143a <explode_bomb>
(gdb) x/s 0x402400
###        "Border relations with Canada have never been better."
```
# phase_2
```sh
(gdb) disas phase_2
# <read_six_numbers> -- update keepgo
## for ()
(gdb) b 82
(gdb) r keepgo
(gdb) si
(gdb) p $rsp
(gdb) p *0xaddr 
(gdb) p $eax
###        1 2 4 8 16 32
```
# phase_3
```sh
(gdb) disas phase_3
(gdb) b 89
(gdb) r keepgo
(gdb) si
__GI___isoc99_sscanf (s=0x603820 <input_strings+160> "test", format=0x4025cf "%d %d") at ./stdio-common/isoc99_sscanf.c:24
24      ./stdio-common/isoc99_sscanf.c: No such file or directory.
## %d %d -- update keepgo
(gdb) b *0x400f60
(gdb) b *0x400f6a
# -- update keepgo
(gdb) p $eax
(gdb) p *(int*)($rsp+0xc)
# -- update keepgo
## switch() // b *0x400fbe 
###        <0,207> <1,311> <2,707> <3,256> <4,389> <5,206> <6,682>
```
# phase_4
```sh
(gdb) disas phase_4
(gdb) b 95
(gdb) r keepgo
## same debug 
###        0 0
```
# phase_5
```sh
(gdb) disas phase_5
(gdb) b 95
(gdb) r keepgo
# call   40131b <string_length>
(gdb) b *0x4010d2
# movzbl (%rbx,%rax,1),%ecx
# mov    %cl,(%rsp)
# mov    (%rsp),%rdx
# and    $0xf,%edx
# movzbl 0x4024b0(%rdx),%edx
# mov    %dl,0x10(%rsp,%rax,1)
# mov    (%rsp),%rdx
(gdb) p *(char*)($rbx+offset)
(gdb) x/s $rbx
(gdb) b *0x401099
(gdb) x/s 0x4024b0
## 0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
(gdb) p $edx
## $1 = 97
## for (str[idx]) ?

(gdb) b *0x4010b8
(gdb) x/s 0x40245e
## 0x40245e:       "flyers"
(gdb) x/s $rdi
# 0x7fffffffde40: "aiuier" # test (adcbef)
##   9 15 14 5 6 7
## & 00001111
## 01001001 -> I
## 01001111 -> O
## 01001110 -> N
## 01000101 -> E
## 01000110 -> F
## 01000111 -> G
### IONEFG
```
# phase_6
```sh
(gdb) disas phase_6
(gdb) b 108
(gdb) r keepgo
(gdb) b *0x40111e
## for () for (jne)  -- update keepgo
(gdb) b *0x401153
(gdb) p *(int*)($rax)
# mov    %ecx,%edx   # edx = 7
# sub    (%rax),%edx # edx -= *rax
# mov    %edx,(%rax) # mov back
(gdb) p *$ecx
(gdb) p *0x6032d0
$6 = 332 # ?
(gdb) x/32d 0x6032d0
# 0x6032d0 <node1>:       332     1       6304480 0
# 0x6032e0 <node2>:       168     2       6304496 0
# 0x6032f0 <node3>:       924     3       6304512 0
# 0x603300 <node4>:       691     4       6304528 0
# 0x603310 <node5>:       477     5       6304544 0
# 0x603320 <node6>:       443     6       0       0
(gdb) x/12a 0x6032d0
# 0x6032d0 <node1>:       0x10000014c     0x6032e0 <node2>
# 0x6032e0 <node2>:       0x2000000a8     0x6032f0 <node3>
# 0x6032f0 <node3>:       0x30000039c     0x603300 <node4>
# 0x603300 <node4>:       0x4000002b3     0x603310 <node5>
# 0x603310 <node5>:       0x5000001dd     0x603320 <node6>
# 0x603320 <node6>:       0x6000001bb     0x0
## std::list && sizeof(node)==16
## { int val; int idx; node* next; }
(gdb) b *0x401176
(gdb) r keepgo
(gdb) p *$rdx
## std::reverse(list)
(gdb) b *0x4011bd
(gdb) p *(node*)($rax)
# mov    (%rax),%rdx
# mov    %rdx,0x8(%rcx)
## node[i].next = node[i-1]
(gdb) b *0x4011d2
(gdb) p /x $rdx
$1 = 0x6032d0 # node1
# mov    0x8(%rbx),%rax
# mov    (%rax),%eax
# cmp    %eax,(%rbx)
# jge    4011ee <phase_6+0xfa>
# call   40143a <explode_bomb>
## jge list
## -- update keepgo
##      1   2   3   4   5   6
##    332 168 924 691 477 443
##  =>  3   4   5   6   1   2
### =>  4   3   2   1   6   5
```