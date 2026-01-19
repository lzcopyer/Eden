# icmp信息泄露修复

1为禁用icmp，0为开启icmp
`echo 1 >  /proc/sys/net/ipv4/icmp_echo_ignore_all`  
`sysctl -w net.ipv4.icmp_echo_ignore_all=1`  
`sysctl -p`

## tcp/ip时间戳信息泄露

``` bash
#设置为0禁用
sysctl -w net.ipv4.tcp_timestamps=0
sysctl -p
#查看修改是否成功
sysctl -a |grep net.ipv4.tcp_timestamps
```
