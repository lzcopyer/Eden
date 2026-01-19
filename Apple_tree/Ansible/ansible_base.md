# ansible 基础用法

## Ansible程序目录

- 配置文件目录: /etc/
- 执行文件目录： /usr/bin/
- Lib库依赖目录: /usr/lib/pythonX.X/site-packages/ansible/
- 功能模块目录: /usr/local/ansible/{monitoring, network, notification, packaging, system…}
- Help文档目录: /usr/share/doc/ansible-X.X.X/
- Man文档目录: /usr/share/man/man1/

## Ansible命令功能

- ansible-galaxy  
  ansible下载官方playbook的模块
- ansible-pull  
  适用于没有网络连接的主机使用ansible
- ansible-doc  
  ansible模块文档，可以查看模块的说明
- ansible-playbook  
  ansible的script
- ansible-vault  
  ansible加密模块

## 管理Inventory file

Inventory file 即定义ansible管理主机的配置文件. Ansible为方便批量管理主机,便捷使用其中的部分主机,我们可以在Inventory file中按需对主机进行group分组.默认的inventory file为/etc/ansible/hosts .
Inventory file可以多个,通过 –i 或 –inventory-file 指定读取,同时可动态生成 inventory file, 如 host、node等。

``` yaml
#Inventory file支持的参数列表
ansible_ssh_host
ansible_ssh_port
ansible_ssh_user
ansible_ssh_pass
ansible_sudo_pass
ansible_connection
ansible_ssh_private_key_file
ansible_sftp_extra_args
ansible_scp_extra_args
ansible_ssh_extra_args
ansible_ssh_pipelining
ansible_shell_type
ansible_python_interpreter
```

## 主机和组

可以自定义Inventory file，储存远程主机的信息，可以使用组保存主机ip或域名。在组的信息中存放用户和密码。

``` yaml
[node]
node1
node2

[node:vars]
ansible_ssh_user=
ansible_ssh_pass=
ansible_become=true
ansible_become_method=sudo
ansible_become_user=root
ansible_become_pass=
```

## Ansible Patterns

Patterns 功能类似正则匹配,对于主机管理灵活性有着极大帮助。  
patterns 语法  
`ansible <pattern_goes_here> -m <module_name> -a <arguments>`  
实例  
`ansible node -i host -m ping`  
all 技巧  
all 等同于* 即Inventory file中的所有组  
`ansible all -i host -m ping`  
or 技巧  
同时执行多个主机或主机组, 相互之间用 ： 分隔  
`ansible node1:node2 -i host -m ping`  
逻辑非技巧  
筛选出在node1中同时不在node2中的主机  
`ansible node1:!node2 -i host -m ping`  
逻辑与技巧  
筛选出所有同时在node1和node2中的主机  
`ansible node1:&node2 -i host -m ping`  
ansible还支持域切割，模糊匹配和正则匹配。
