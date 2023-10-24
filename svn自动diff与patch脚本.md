## svn自动diff与patch脚本

场景: 代码机与编译机在不同的机器上, 使用svn进行版本控制, 分成两个脚本, 分别部署在两台机器上

非上述的其他场景亦适用

## 代码机(diff端)

```sh
#!/bin/bash

fname=$1
fpath='../diff_path'

if [[ $# -lt 1 ]]; then
    aa=`pwd` && bb=${aa%/*} && fdir=${bb##*/}
    fname=$fdir"_`date +%Y%m%d`.diff"
fi

# 先更新最新代码
svn update
if [ $? -ne 0 ]; then
    echo "svn update failed"
    exit 1
fi

# 最终文件名
file=$fpath/$fname

# svn diff 将代码更改保存到指定文件
svn diff --force > $file

# 文件可能出现不合规的换行, 将其删除, 其中^M是该符号的显示, 输入方式ctrl-v加回车
sed -i "s/^M//g" $file

# 自动发送, 对端配置参数
dst_user='duringyl'
dst_host='192.168.1.2'
dst_path='~/work/'
dst_pass='123456'

# 检测本机支持expect则尝试自动发送，否则scp命令需手动输入密码
type expect &>/dev/null && if [ $? -eq 0 ] && [ "$dst_pass" != "" ]; then
    expect << EOF
spwn scp "$file" $dst_user@$dst_host:$dst_path
set timeout 1
expect "yes/no" {send "yes\n"}
set timeout 4
expect "*password:" {send "$dst_pass\n"}
set timeout -1
expect eof
EOF
else
    echo send: scp "$file" $dst_user@$dst_host:$dst_path
    scp "$file" $dst_user@$dst_host:$dst_path
fi

```

## 编译机(patch端)

```sh
#!/bin/bash

# 第一个参数作为diff文件
fname=$1

# 不提供diff文件, 则自动决定, 根据*.diff文件的时间选最新的
if [ $# -ne 1 ]; then
    fname=`ls -lt *.diff | awk 'NR==1{print $NF}'`
    echo "will patch $fname"
fi

# 根据diff文件决定代码(svn)版本
code_ver=`grep "(revision .*)" $fname | awk 'NR==1{print $NF}' | tr -d ')'`
echo "code version is $code_ver"

# 收集对于版本库修改了的文件, 其中^M是正常字符
aa=`svn status | grep -E "^M" | tr -s ' ' | cut -d ' ' -f 2`
echo "file is $aa"

# 收集对于版本库新增或删除了的文件
bb=`svn status | grep -E "^A|^!" | tr -s ' ' | cut -d ' ' -f 2`

if [ $? -eq 0 ]; then
    # 将修改过的文件删除
    rm -f $aa
    if [ "$bb" != "" ]; then
        echo "$bb"
        # 将在版本库新增或删除文件的记录删除
        svn del --force $bb
    fi

    # 将版本更新到指定版本
    svn update -r $code_ver

    # 执行patch
    svn patch $fname
fi

```
