# Lab 1: Booting a PC

# 1.1 Software Setup

## 在ubuntu20.04配置环境时遇到问题（最后还是妥协用16.04来配置）:

#### 在配置qemu时出现python not found

```
./configure --disable-kvm --target-list="i386-softmmu x86_64-softmmu" --python=python2.7
```

Ubuntu 20.04 Focal 默认自带python3.8，但是根据指引，所有引用python的包必须显示指定python3或其他python版本。

Ubuntu从20.04开始不再将python加入PATH环境变量，在编译安装一些软件会提示无法运行并提示找不到python，然而python3已安装，需要额外重定向。(安装其他版本类似)

```
sudo ln -s /usr/bin/python3 /usr/bin/python
```



**将ubuntu连接vpn时需要将network中的手动修改ip修改为clash的WLAN ip地址 (clash中的ip地址可能会变化)**

**有时候不能 安装应用程序时应当先检查是否update upgrade 然后不行再更换国内源**



#### 在安装make之前提高权限

```
sudo make
```

然后在qemu目录下配置完成后安装make时

```
make && make install（也可以分开 sudo make \n sudo make install）
```

出现报错

```
make: *** [Makefile:288：qemu-ga] Error 1
```

解决方法:

在qga/commands-posix.c文件中加上头文件<sys/sysmacros.h>。

gcc可能版本过高 需要降低版本

```
make: *** [kern/Makefrag:71: obj/kern/kernel] Error 1
```

## 最终在ubuntu16.04中配置完成环境

**按照官方文档进行 遇到不会的步骤可以google大佬们的配置方法**

1. `$ mkdir ~/6.828`

2. `$ cd ~/6.828`

3. `$ sudo apt install git`

4. `$ git init` 初始化仓库

5. `$ git clone https://github.com/fatsheep9146/6.828mit` 由于官方代码拉取时总是报错 所以从github上拉取一份

6. `$ cd ~/6.828/6.828mit/lab`

   ***JOS源码已经拉取到本地 接下来需要安装qemu***

1. `$ sudo apt update` 首先更新索引 避免接下来的操作中出现报错

2. 首先安装一些QEMU需要的包，以下命令按顺序输入。
   `sudo apt-get install libsdl1.2-dev`
   `sudo apt-get install libglib2.0-dev`
   `sudo apt-get install libz-dev`
   `sudo apt-get install libpixman-1-dev`
   `sudo apt-get install libtool*`

3. `$ git clone https://github.com/mit-pdos/6.828-qemu.git qemu` 将官方qemu源码拉取到本地

4. `$ cd qemu` 

5. `$ su root` 给予最高权限

6. `$ ./configure --disable-kvm --disable-werror --prefix=$HMOE --target-list="i386-softmmu x86_64-softmmu"` 配置QEMU

7. $ 在~/6.828/6.828mit/lab/qemu/qga/commands-posix.c中的头文件添加

   `\#include <sys/sysmacros.h>`

8. `$ sudo apt-get install gcc-multilib`安装开发环境需要的32位gcc，系统自带的是64位的

9. `$ make && make install`最后一步，编译并安装，关闭root。

10. `$ qemu-system-i386`，出现了QEMU仿真器的界面成功。(我是直接通过apt install qemu_x86安装)

    ![image-20230910225946737](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230910225946737.png)

11. `$ cd ..` 返回到lab中

12. `$ make` 出现一个kernel的镜像文件 说明成功

    ![image-20230910230108986](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230910230108986.png)

13. `$ make qemu` 

    ![image-20230910230312361](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230910230312361.png)

    

    #### 成功!!（遇到问题最好的解决方法就是上google查找 实在不行就将整个文件删除重新按步骤开始操作）

    