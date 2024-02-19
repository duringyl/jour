## CentOS7部署OAI编译调试环境(支持离线)

主页[https://gitlab.eurecom.fr/oai/openairinterface5g/](https://gitlab.eurecom.fr/oai/openairinterface5g/)

下载[https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.tar.gz](https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.tar.gz)


### 1. OAI的各种值得看的文件

- `README.md`：其中的`How to build`链接到`doc/BUILD.md`，`How to run the modems`链接到`doc/RUNMODEM.md`
- `CMakeLists.txt`：里面是TOP层的一些环境检查和编译参数决定脚本
- `docker/Dockerfile.gNB.rhel9:76`：查看最小化需要哪些额外依赖
- `cmake_targets/tools/build_helper:578`：查看CentOS7自动安装依赖的语句：安装哪些包
- `cmake_targets/build_oai`：里面对编译参数的处理也值得参考：清理、自动解决依赖等功能

### 2. 编译和运行

```shell
# oai主目录
cd oai/

# 编译主目录
cd cmake_targets

# 编译NR终端和基站 # 添加-c选项所有重新编译
./build_oai --nrUE --gNB

# 编译输出路径
cd ran_build/build

# 运行NR基站
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim --sa

# 运行NR终端
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 --rfsim --sa
```

### 3. CentOS7上部署oai环境的实践过程

**额外的依赖**

- [asn1c](https://github.com/mouse07410/asn1c/archive/refs/heads/vlm_master.zip)
- [cmake3](https://github.com/Kitware/CMake/releases/download/v3.28.3/cmake-3.28.3.tar.gz)
- [SIMDe](https://github.com/simd-everywhere/simde/archive/refs/tags/v0.7.6.tar.gz)

**过程**

```shell
# 安装各种依赖
yum install -y flex bison gcc gcc-c++ lapack lapack-devel libtool \
    libconfig-devel autoconf automake blas blas-devel atlas-devel \
	lksctp-tools lksctp-tools-devel openssl openssl-devel

# 看yacc -V有没有，没有的话把bison软链接成yacc命令
cd /usr/bin;ln -s bison yacc;cd -

# 编译和安装asn1c(需要oai他们指定的版本，在BUILD.md中找链接，去Github下载)
test -f configure || autoreconf -iv		# 必要时手动删除configure文件
./configure
make -j`nproc`
make install	# /usr/local/bin/asn1c

# 编译安装cmake3
./configure
gmake -j`nproc`
gmake install
# 后续可能该编译脚本里面的命令名字好点
cd /usr/local/bin; ln -s cmake cmake3; cd -

# gcc11: >>>>>每次退出登录后需要切换<<<<<，除非设置登录时自动切，否则还是系统原版本
yum install --downloadonly --downloaddir=/root/scl/ centos-release-scl
yum install --downloadonly --downloaddir=/root/scl/ devtoolset-11-gcc \
    devtoolset-11-gcc-c++ devtoolset-11-binutils
cd /root/scl/
rpm -ivh *.rpm
# 切换语句
scl enable devtoolset-11 bash
# ./build_oai -c	# !!!换了gcc版本需要清理以前的编译缓存!!

# SIMDe从Github下载
tar -zxf simde-0.7.6.tar.gz
cp -a simde-0.7.6/simde /usr/include/

# 修改编译文件
cd ${oai}
grep "pkg_check_modules" -n CMakeLists.txt
    486:pkg_check_modules(libconfig REQUIRED libconfig)
    503:  pkg_check_modules(LIBDPDK_T2 REQUIRED libdpdk=20.11.7)
    677:pkg_check_modules(OpenSSL openssl REQUIRED)
    1149:pkg_check_modules(blas REQUIRED blas)				# 将这一行的REQUIRED去掉
    1150:pkg_check_modules(lapacke REQUIRED lapacke)		# 将这一行的REQUIRED去掉
    1156:pkg_check_modules(cblas cblas)

```