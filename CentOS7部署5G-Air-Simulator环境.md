### 1. 5G-air-simulator版本

Release 20.02


### 2. 环境部署概述

1. 依赖libarmadillo，需要安装libarmadillo 9版本,这个是必须的否则会编译报错，而其中libarmadillo又依赖很多其它的包
2. yum源上的gcc版本和cmake版本就已经够用，当然安装更新版本的这两个工具没有问题(cmake为了编译安装armadillo)

### 3. armadillo

是目前使用比较广的C++矩阵运算库之一，相当于Matlab的C++替代库

### 4. 直接yum安装的依赖
```shell
yum install -y vim unzip gcc gcc-c++ lapack-devel libstdc++-devel atlas-devel cmake

# 在epel的一些包
yum install epel-release
yum install -y --downloadonly --downloaddir=/root/epel/ \
    hdf5-devel openblas-devel arpack-devel SuperLU-devel
cd /root/epel/
rpm -ivh *.rpm --force
```

### 5. 不可选: 安装armadillo(版本不对)

**通过pre-built rpm包安装**: 

  [1. armadillo](https://kojipkgs.fedoraproject.org//packages/armadillo/10.8.2/1.el7/x86_64/armadillo-10.8.2-1.el7.x86_64.rpm)
  
  [2. armadillo-devel](https://kojipkgs.fedoraproject.org//packages/armadillo/10.8.2/1.el7/x86_64/armadillo-devel-10.8.2-1.el7.x86_64.rpm)


```shell
# 如果没有yum安装上面的各种包会出现下面的依赖错误
rpm -ivh armadillo-10.8.2-1.el7.x86_64.rpm armadillo-devel-10.8.2-1.el7.x86_64.rpm
# 错误：依赖检测失败：
#         SuperLU-devel 被 armadillo-devel-10.8.2-1.el7.x86_64 需要
#         arpack-devel 被 armadillo-devel-10.8.2-1.el7.x86_64 需要
#         atlas-devel 被 armadillo-devel-10.8.2-1.el7.x86_64 需要
#         hdf5-devel 被 armadillo-devel-10.8.2-1.el7.x86_64 需要
#         lapack-devel 被 armadillo-devel-10.8.2-1.el7.x86_64 需要
#         libarmadillo.so.10()(64bit) 被 armadillo-devel-10.8.2-1.el7.x86_64 需要
#         libstdc++-devel 被 armadillo-devel-10.8.2-1.el7.x86_64 需要
#         openblas-devel 被 armadillo-devel-10.8.2-1.el7.x86_64 需要

# ====>>>> 实际上面这个版本的armadillo太新，导致编译报错

```

**编译错误**: 
```
src/phy/ue-phy.cpp: In member function ‘virtual void UePhy::StartRx(std::shared_ptr<PacketBurst>, ReceivedSignal*)’:
src/phy/ue-phy.cpp:365:49: error: no matching function for call to ‘inv(arma::cx_mat&, bool)’
  365 |                       arma::cx_mat W = arma::inv( HHTN, true ) * sqrt(avgPower) * precodedH0;
      |                                        ~~~~~~~~~^~~~~~~~~~~~~~
In file included from /usr/include/armadillo:475,
                 from src/phy/ue-phy.cpp:25:
```

**另外的尝试，手动编译安装12.x版本**:

armadillo源码在sourceforge: 
[armadillo源码浏览器下载](https://sourceforge.net/projects/arma/files/armadillo-12.8.1.tar.xz)

armadillo代码库在gitlab，README能看更多的信息: 
[armadillo代码库在gitlab](https://gitlab.com/conradsnicta/armadillo-code)

```shell

# 如果进行了通过pre-built包安装的版本，先卸载，不卸载也行
rpm -qa | grep armadillo
    # armadillo-devel-10.8.2-1.el7.x86_64
    # armadillo-10.8.2-1.el7.x86_64

rpm -e armadillo-devel-10.8.2-1.el7.x86_64 armadillo-10.8.2-1.el7.x86_64


tar -Jxf armadillo-12.8.1.tar.xz
cd armadillo-12.8.1/
cmake .   # 或者./configure，里面实际也是运行: cmake .

make -j`nproc`
make install

```

### 6. 必选: 安装正确版本的armadillo

下载旧版本的armadillo，这个版本下载量很高: 
[armadillo v9.900](https://gitlab.com/conradsnicta/armadillo-code/-/archive/9.900.x/armadillo-code-9.900.x.tar.gz)

```shell
tar -zxf armadillo-code-9.900.x.tar.gz
cd armadillo-code-9.900.x/
cmake .
make -j`nproc`
make install
```

### 7. 编译5G-air-simulator

```shell
unzip -qq 5G-air-simulator-master.zip
cd 5G-air-simulator-master
make -j`nproc`
```
