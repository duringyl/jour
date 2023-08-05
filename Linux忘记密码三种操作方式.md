### 一、拥有sudo权限的其它用户

1. 直接sudo passwd root
2. 修改/etc/shadow文件

##### 原理

> 拥有sudo权限的用户居然可以直接改/etc/shadow这个文件的内容

</br>

### 二、进入BIOS启动特殊SHELL

0. ubuntu情况下有时想要进入启动选项界面需要按SHIFT+ESC
1. 按e进入编辑启动选项
2. 在`linux /vmlinuz-xxx ...`一行的后面添加`init /bin/sh`
3. 编辑完成按ctrl-x启动, 进入一个特殊的shell
4. `mount -o remount,rw /`将根目录重新挂载为读写
5. `passwd root`修改root密码

##### 原理

> 不知道是什么原理能进入这个shell，也不知道这个shell支持什么样的命令和权限，但确实能修改任何用户的密码

</br>

### 三、运行在U盘的系统(Live CD | PE盘)(终极方案)

1. 启动系统(一般系统所在的U盘插在具体的主机上)
2. 挂载主机的磁盘
3. 进入磁盘中原系统的目录
4. 直接修改/etc/shadow文件

##### 原理

> 这种其实等价于把你的电脑拆开来取下硬盘，然后把硬盘插另一台电脑上，当然可以读写硬盘的内容。
> 除非你把硬盘里的文件/etc/shadow加密，但问题来了，你需要另一个密码来解密这个文件，那另一个密码存在哪里呢？其所在的文件需要加密吗？

</br>

### NOTE

0. 所有都基于/etc/shadow这个文件, 加盐字符串
1. 有时候/etc/shadow添加了`i`或`a`的attr，导致此文件任何人都修改不了包括root
2. 查看文件的attr: `lsattr /etc/shadow`
3. 修改文件的attr: `chattr -i /etc/shadow`
