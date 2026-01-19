>`ansible` 需要不少的依赖，这对于一些生产环境而言不太友好。这里推荐使用`python`虚拟环境进行安装，这可以提高`ansible`的可移植性。

# 准备`ansible`安装包

``` Bash
#创建一个虚拟环境
/usr/bin/python -m venv /usr/local/ansible
source /usr/local/ansible/bin/activate
cd /usr/local/ansible

#安装ansible
pip install ansible cryptography paramiko PyYAML Jinja2 httplib2 six argcomplete

#打包ansible
cd ..
tar zcf ansible.tar.gz ansible
```

# 在其他环境上安装

``` Bash
#安装
tar zxf ansible.tar.gz -C /usr/local/
chmod -R 755 /usr/local/ansible

#在.bash_profile中配置环境变量
echo 'export ANSIBLE_HOST_KEY_CHECKING=False' >> ~/.bash_profile
#使用paramiko作为默认的ssh工具，不过这个参数在高版本被弃用了，推荐在Inventory文件中配置
echo 'export ANSIBLE_TRANSPORT=paramiko' >> ~/.bash_profile

#或者配置在ansible.cfg中
mkdir ~/.ansible
cat <<EOF > ~/.ansible/ansible.cfg
[defaults]
host_key_checking = false
tarnsport = paramiko
stdout_callback = actionable
EOF

#加载虚拟环境就可以使用了
source /usr/local/ansible/bin/activate
```

# `Inventory`配置

``` TOML
#推荐使用paramiko连接方式
[node]
node1
node2

[node:vars]
ansible_connection=paramiko
ansible_user=
ansible_password=
ansible_paramiko_host_key_auto_add=True
ansible_paramiko_look_for_keys=False
ansible_become=true
ansible_become_method=sudo
ansible_become_user=root
ansible_become_pass=

#当然也可以使用经典的sshpass工具
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