## Linux自动复制最新文件脚本

#### 1. 场景

某个文件更新时会上传到Linux的某路径下。Linux上其它程序可能要复制这个文件到别的路径下, 但要在文件有更新的时候再复制。平时手动将更新程序上传到指定环境之后进行调试的过程经常会需要手动cp文件, 此脚本将cp过程自动化。

#### 2. 特性

如果源路径文件不存在什么也不做, 如果源路径文件的时间戳比目标路径文件的大则将文件从源路径复制到目标路径下一次

#### 3. 代码

auto_cp_newer.sh

```shell
#!/bin/bash

# 自动复制时间更新的文件: 参数1: 被复制的源路径文件, 参数2: 复制到的目标路径文件
function auto_cp_newer(){
    src=$1
    dst=$2

    # 源路径文件不存在分支
    if [ ! -f $src ]; then return 0; fi

    # 别名: 去掉默认的-i选项, 并复制时保留文件特征: 权限、属主、时间戳
    alias cp='cp -p'

    # 截取源文件名称
    sFName=${src##*/}

    # 如果目标是一个路径参数分支
    if [ -d $dst ]; then
        # 拼接目标文件完整路径
        dst=$dst/$sFName

        # 如果目标文件不存在直接复制过去
        if [ ! -f $dst ]; then cp $src $dst; return 0; fi
    fi

    # 对比源文件和目标文件的时间戳, 如果源文件的时间更新则复制过去
    sTime=`stat -c %Y $src`
    dTime=`stat -c %Y $dst`
    if [ $sTime -gt $dTime ]; then cp $src $dst; fi

    return 0
}

```

sed_replace.sh

```shell
#!/bin/bash

fSrc=/home/user/file.txt
fDst=.

# 加载脚本, 将使用其中的auto_cp_newer函数
. auto_cp_newer.sh

# 调用函数
auto_cp_newer $fSrc $fDst

sed -i 's/AAA/BBB/g' ./file.txt

```