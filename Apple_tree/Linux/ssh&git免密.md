## 创建密钥
使用命令`ssh-keygen -b 4096 -t rsa`（敲三次回车，-b参数指定密钥长度，-t参数指定密钥类型）创建私钥id_rsa和公钥id_rsa_pub

## 放置公钥
将公钥`id_rsa_pub`保存到免密目标主机的`/home/$user/.ssh/authorized_keys`文件中
`md5sum file`检查文件md5值可以校验文件内容是否一致。
