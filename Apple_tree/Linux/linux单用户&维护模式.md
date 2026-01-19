# Linux 进入单用户模式

在开机引导界面选择内核版本然后按`e`，在linux16后面添加`rd.break console=tty0`，`ctrl+x`进入单用户模式。或者将`ro`修改为`rw`,把`rhgb quiet`修改为`init=/sysroot/bin/sh`  
需要重新挂载根目录为可读模式：`chroot /sysroot`,`mount -o remount,rw /`  
如果修改`root`密码之后需要重启，需检查`selinux`是否开启，如果开启需要`touch /.autorelabel`,然后`exec /sbin/init 6`重启。

如果有如下报错`pam_succeed_if(sshd:auth): requirement “uid 」= 1000“ not met by user “root“`  
检查`pam.d`配置文件
 `/etc/pam.d/login`, `/etc/pam.d/sshd` `/etc/pam.d/system-auth`中是否有 `auth required pam_succeed_if.so uid >= 1000`将这个配置注释或者修改。
