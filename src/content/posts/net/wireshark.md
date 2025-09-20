---
title: WireShark
published: 2025-09-01
tags: [net]
category: net
draft: false
---

```sh
sudo apt install wireshark tshark
# sudo dpkg-reconfigure wireshark-common # Yes
sudo usermod -aG wireshark $USER

	
sudo chgrp wireshark /usr/bin/dumpcap
sudo chmod 750 /usr/bin/dumpcap
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap

tshark -D # 列出可用网络接口
	
tshark -i eth0 -w output.pcap
tshark -r output.pcap         # 读取和分析已保存的抓包文件
tshark -r output.pcap -Y "ip.src == 192.168.1.100 and ip.dst == 192.168.1.102 and tcp.port == 80"

tshark -i eth0 -f "tcp port 80 and (src host 192.168.1.100 or src host 192.168.1.102)"
tshark -i eth0 -f "tcp port 80 and src net 192.168.1.0/24"
tshark -i eth0 -Y "http"
```
