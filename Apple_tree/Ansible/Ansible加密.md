# 简介

> 众所周知，ansible是很火的一个自动化部署工具，在ansible控制节点内，存放着当前环境服务的所有服务的配置信息，其中自然也包括一些敏感的信息，例如明文密码、IP地址等等。
>
> 从安全角度来讲，这些敏感数据的文件不应该以明文的形式存在。此时就用到了ansible加密的特性。
>
> ansible通过命令行「ansible-vault」给你目标文件/字符串进行加密。在执行playbook时，通过指定相应参数来给目标文件解密，从而实现ansible vault的功能。

**ansible可以加密任何部署相关的文件数据，例如：**

- 主机/组变量等所有的变量文件
- tasks、hanlders等所有的playbook文件
- 命令行导入的文件（eg : -e @file.yaml  ,-e @file.json）
- copy，template的模块里src参数所使用的文件，甚至是二进制文件。
- playbook里用到的某个字符串参数也可以加密（Ansible>=2.3）

## 加密文件

**创建加密文件**  
`ansible-vault create foo.yml`  
执行该命令后，交互式输入两次相同的密码，则创建加密文件成功。 

**给现有文件加密**  
`ansible-vault encrypt foo.yml bar.yml baz.yml`  
如示例所示，该命令可以为多个文件同时加密，同样需要输入两次相同的密码，则加密成功。  

很多情况下，我们需要给主机（组）部分变量文件进行加密，而不是全加密。这里介绍下加密和非加密需要同时存在时推荐的用法。以创建组变量为例：

``` bash
cat hosts.yaml
---
# inventory/hosts
nodes:
  hosts:
    node1:
    node2:
    node3:

#组变量文件我们这样创建，目录结构如下：
tree group_vars
group_vars
└── nodes
    ├── vars.yaml
    └── vault.yaml

#文件「group_vars/nodes/vault.yaml」内容如下：
---
vault_db_name: test
vault_db_host: 127.0.0.1
```

上面示例中，我们为「nodes」主机组创建了一个「group_vars/nodes」目录用于存放该主机组的变量文件，其中「vars.yaml」和「vault.yaml」文件分别存放不加密和加密变量，通过这种拆分变量的方法，实现了加密和非加密同时存在的功能。

另外，为了便于识别，「vault.yaml」文件里的所有key值建议使用「vault_」开头，这样再playbook调用这些变量时，可以有很高的辨识度。

## 编辑加密文件

对加密过的文件进行编辑也是我们常遇到的，命令为:  
`ansible-vault edit foo.yml`  
这个命令会将该文件解密，并放到一个临时文件，编辑完后再保存到原文件。  

## 查看加密文件

有时我们不是想编辑文件，而是简单查看下文件内容，命令为：  
`ansible-vault view foo.yml bar.yml baz.yml`  
从示例中我们可以看出，一行命令可以查看多个加密文件。  

## 更改密码

给加密文件更改密码，命令为：  
`ansible-vault rekey foo.yml bar.yml baz.yml`  
先交互式输入旧的密码，然后再交互式输入两次新密码，即更改密码成功。  

## 取消加密

取消加密的命令为：  
`ansible-vault decrypt foo.yml bar.yml baz.yml`  
交互式输入密码，即取消加密成功。  

## encypt_string创建加密变量

Vault ID和多密码  
ansible2.4版本以后，新增了Vault ID和多密码的特性。Vault ID可以理解为为一个密码设置一个标签，用于管理员识别使用的是哪个密码，例如dev，prod，cloud等等；多密码指执行一次playbook时可以指定多个密码文件。  
**注：vault_id不会影响加密文件的解密，只是为了便于管理员识别是哪个密码。**  
encypt_string创建加密变量  
在plabook文件中，我们也可以为某个字符串加密，从而达到加密变量嵌套在未加密YAML文件内的效果。  
「ansible-vault encrypt_string」命令用于给某个字符串加密。  
不设置vault_id，示例如下：  

``` bash
ansible-vault encrypt_string --vault-id playbooks/test_vault.yaml  'maurice' --name 'the_name'
the_name: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
          ...

#设置vault_id为「dev」（给密码打标签）（加密参数也会有「dev」的标识）：
ansible-vault encrypt_string --vault-id dev@playbooks/test_vault.yaml  'maurice' --name 'the_name'
the_name: !vault |
          $ANSIBLE_VAULT;1.2;AES256;dev
          ...
          ...

#从stdin读取密码：
echo -n 'maurice' | ansible-vault encrypt_string --vault-id dev@playbooks/test_vault.yaml --stdin-name 'the_name'
Reading plaintext input from stdin. (ctrl-d to end input)
the_name: !vault |
          $ANSIBLE_VAULT;1.2;AES256;dev
          ...
          ...

#交互式设置密码，设置完密码后按Ctrl+d：
ansible-vault encrypt_string --vault-id dev@playbooks/test_vault.yaml --stdin-name 'the_name'
Reading plaintext input from stdin. (ctrl-d to end input)
maurice
the_name: !vault |
          $ANSIBLE_VAULT;1.2;AES256;dev
          ...
          ...
```

## 使用加密文件

上面我们主要介绍了密码文件/变量的创建方式，接下来介绍下如何使用。  
ansible执行playbook时，可以通过交互式或指定密码文件的方式来解密文件。  
交互式  
执行playbook时在终端以交互式的形式输入密码，示例：  
ansible版本<=2.4，该方式只支持一个密码，不能指定vault_id。  
`ansible-playbook --ask-vault-pass site.yml`  
ansible2.4版本后，对应命令行如下，该方式只支持一个密码，能指定vault_id：  
`ansible-playbook --vault-id  dev@prompt  playbooks/test_vault.yaml`  
指定密码文件  
另外一种使用方式，是将密码放在某个文件内，执行playbook时，通过指定该密码文件进行解密。  
ansible版本<=2.4，可以使用「--vault-password」参数：  
`ansible-playbook --vault-password-file dev-password site.yml`  
ansible2.4版本以后，又增加了「--vault-id」的参数，用来指定密码文件：  
`ansible-playbook --vault-id /path/to/my/vault-password-file site.yml`  
从可执行脚本获取密码：  
`ansible-playbook --vault-id my-vault-password.py`  
指定密码文件同时指定vault_id:  
`ansible-playbook --vault-id dev@prompt site.yml`
指定多vault_id+多密码文件：  
`ansible-playbook --vault-id dev@dev-password --vault-id prod@prompt site.yml`
