
## 防火墙的安装与启动

firewalld使用服务和区域的概念，与iptables的链和规则不同。  

- 安装firewalld  
`yum -y install firewalld`
- 查看防火墙状态和配置的规则  
`firewall-cmd --state`  
`firewall-cmd --list-all`  
`firewall-cmd --reload`  #更新firewalld配置
- firewalld启停命令  
`systemctl enable firewalld`  
`systemclt start firewalld`
`systemctl disable firewalld`
`systemctl stop firewalld`
- 常用的systemctl命令  

``` shell
启动一个服务：systemctl start firewalld.service
关闭一个服务：systemctl stop firewalld.service
重启一个服务：systemctl restart firewalld.service
显示一个服务的状态：systemctl status firewalld.service
在开机时启用一个服务：systemctl enable firewalld.service
在开机时禁用一个服务：systemctl disable firewalld.service
查看服务是否开机启动：systemctl is-enabled firewalld.service
查看已启动的服务列表：systemctl list-unit-files|grep enabled
查看启动失败的服务列表：systemctl --failed
```

## firewalld区域与服务  

- **drop**区域删除所有传入连接，而无任何通知。仅允许传出连接。

- **block**区域对于IPv6/IPv4和`icmp6-adm-prohibited/IPv6n`，所有传入连接均被拒绝，并带有`icmp-host-prohibited`消息。仅允许传出连接。

- **public**区域用于不受信任的区域。即您的计算机在一个不可信的计算机网络上，但你可以指定那些连接允许传入。

- **external**区域对于用在内部网络。当你的系统充当网关或者路由时，在网络上的计算机通常是可信的，仅已允许的连接可以传入。

- **internal**区域对于在内部网络上。当计算机充当网关或路由器时。网络上的其它计算机通常是受信任的。仅已允许的连接可以传入。

- **dmz**区域对于计算机位于可缓冲区，区域内计算机访问网络时将会被限制。仅已允许的连接可以传入。

- **work**区域用于工作机。网络上的其他计算机通常是受信任的。仅已允许的连接可以传入。

- **home**区域用于家庭计算机。网络上的其他计算机通常是受信任的。仅已允许的连接可以传入。

- **trusted**区域接受所有网络连接。信任在网络中的所有计算机。

- **防火墙服务**是预定义的规则集，关联在区域内，并定义必要的设置以允许指定服务的传入连接。  

首次启用`Firewalld`服务后，`public`区域被设置为默认区域。可以运行命令`firewall-cmd --get-default-zone`获得主机设置的默认区域的名称。运行命令`firewall-cmd --get-zones`将会列出主机设置的所有区域。运行命令`firewall-cmd --get-active-zones`查看所有网络接口/网卡都分配的区域。(使用参数`--zone`选择查看的区域)要修改接口的关联的防火墙区域，可通过结合使用`--zone`选项和`--change-interface`选项，修改接口的区域。`firewall-cmd --zone=work --change-interface=eth1`命令将会修改**eth1**接口关联防火墙区域**work**。

``` bash
firewall-cmd --get-default-zone

firewall-cmd --get-zones

firewall-cmd --get-active-zones

#列出默认区域配置的规则
firewall-cmd --list-all 

#指定区域配置的规则
firewall-cmd --zone=external --list-all 

firewall-cmd --zone=work --change-interface=eth1
```

## firewalld server配置

- **firewall server**  
	`firewall-cmd --get-services`可以要获取所有可用服务的列表，也可在`/usr/lib/firewalld/services`目录找到服务中**firewalld**服务的配置文件，它为**firewalld**提供连接入口的信息。要在当前会话允许http服务，即运行时配置。也就是允许在`public`区域传入`HTTP 80`端口的连接。可以运行命令`firewall-cmd --zone=public --add-service=http`。要验证是否已成功添加服务，请运行命令`sudo firewall-cmd --zone=public --list-services`进行验证。如果要在重启后保持端口`80`的打开状态，则需要再次输入相同的命令。如果使用`--permanent`选项修改，重启后继续使用规则。`--permanent`选项的修改需要使用`--list-services`和`--permanent`选项来验证您的更改。我们建议你经常直接`--permanent`选项添加和删除firewalld服务。删除服务的语法与添加服务时的语法相同。只需使用`--remove-service`而不是`--add-service`选项。
- **create firewall server file**
	应用程序默认的服务配置文件存储在目录`/usr/lib/firewalld/services`。创建FirewallD服务的最简单方法是基于将现有服务文件创建一个副本。命名为你服务名称。例如，要为`Plex`媒体服务器创建服务配置文件，我们可以使用SSH服务的配置文件作为模板。运行cp命令`cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/plexmediaserver.xml`创建`Plex`的firewalld服务的模板。并在`<short>`和`<description>`标签内更改firewalld服务的简称和描述。需要更改的最重要的标签是`port`标签，`port`标签定义了`server`打开的端口号和协议。

``` bash
#添加http服务配置
firewall-cmd --permanent --zone=public --add-service=http --permanent

#列出服务
firewall-cmd --permanent --zone=public --list-services 

#删除服务
firewall-cmd --zone=public --remove-service=http --permanent 

cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/plexmediaserver.xml

vim /etc/firewalld/services/plexmediaserver.xml

<?xml version="1.0" encoding="utf-8"?>
<service version="1.0">
<short>plexmediaserver</short>
<description>Plex is a streaming media server that brings all your video, music and photo collections together and stream them to your devices at anytime and from anywhere.</description>
<port protocol="udp" port="1900"/>
<port protocol="tcp" port="32400"/>
</service>
```

## 添加端口和白名单

``` bash
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --remove-port=8080/tcp
firewall-cmd --permanent --add-source=192.168.1.100
firewall-cmd --permanent --remove-source=192.168.1.100
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="192.168.1.1/24"  accept"
firewall-cmd --permanent --remove-rich-rule="rule family="ipv4" source address="192.168.1.1/24"  accept"
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="192.168.1.1/24" port protocol="tcp" port="3306" accept"
firewall-cmd --permanent --remove-rich-rule="rule family="ipv4" source address="192.168.1.1/24" port protocol="tcp" port="3306" accept"
firewall-cmd --permanent --list-all | grep ports | head -n 1 | cut -d: -f2 | tr ' ' '\n' | xargs -I {} firewall-cmd --permanent --remove-port={}
```

## 添加黑名单

``` bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="IP地址/掩码" reject'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" port port="端口号" protocol="tcp" reject'
firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="IP地址/掩码" reject'
firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" port port="端口号" protocol="tcp" reject'
```

## 端口转发

`firewall-cmd --zone=external --add-masquerade`命令将在firewalld的public区域启用伪装。

firewalld防火墙可以创建三种不同的端口转发方式。分别在计算机内的端口转发，不同计算机相同端口转发，不同计算机端口转发。

在设置firewalld防火墙端口转发时，你需要使用`--add-forward-port`选项表示添加端口转发。

还有`port`选项指定要转发的端口，`proto`选项指定协议，`toport`选项指定目标主机的端口。`toaddr`选项指定目标地址。

在同一计算机内可以省略`toaddr`选项。如果转发的端口和目标端口相同可以省略目标选项。

`firewall-cmd --zone=external --add-forward-port=port=80:proto=tcp:toport=8080`命令将流量从端口80转发到同计算机内的8080端口。在同一计算机内可以省略`toaddr`选项。

`firewall-cmd --zone=external  --add-forward-port=port=80:proto=tcp:toaddr=10.10.10.2`命令将流量从端口80转发到IP地址是10.10.10.2计算机80端口。转发到相同端口可以省略toport选项。

`firewall-cmd --zone=external  --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=10.10.10.2`命令将流量从端口80转发到IP地址是10.10.10.2计算机的8080端口。

``` bash
firewall-cmd --permanent --zone=public  --add-forward-port=port=80:proto=tcp:toport=8080

firewall-cmd --permanent --zone=public  --add-forward-port=port=80:proto=tcp:toaddr=10.10.10.2

firewall-cmd --permanent --zone=public  --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=10.10.10.2
```
