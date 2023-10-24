### 一、Linux上创建git仓库

```sh
groupadd git                        # 创建git用户组
useradd -g git git                  # 创建git用户
git init --bare projName.git        # git创建一个新的空项目
chown -R git:git projName.git       # 创建后目录下生成projName.git目录, 把属主改为git
```

### 二、Git设置本地操作账户(身份)

```sh
git config --global user.name duringyl          # 设置操作git命令的用户的名称
git config --global user.email duringyl@163.com # 设置操作git命令的用户的邮箱

git config user.name                # 查看当前用户名
git config user.email               # 查看当前用户邮箱

git config --global core.quotepath false        # 解决git status中文乱码问题
```

### 三、clone项目(第一次拉取代码)

```sh
git clone git@xx.xx.xx.xx:/git/projName.git     # 类似于scp命令, 仓库所在的IP和目录所在绝对路径
git push origin master                          # clone后如果是空的，可能需要第一次提交
```

### 四、Git一般命令 

```sh
git status                          # 查看本地的改动状态
git add xxx                         # 添加(新)文件或目录改动
git commit -m "xxx"                 # 提交到本地
git push                            # 所有本地提交上传到远程仓库
git pull                            # 从远程仓库更新代码

git mv oldPath newPath              # git(记录)文件重命名, 如果直接改名可能导致改动丢失
git rm xxxfile                      # 跟mv类似, 记录删除, 使用-r删除文件夹
git diff xxxfile                    # 查看文件的改动情况, 在尚未add时
```

### 五、Git分支

```sh
git branch                          # 查看本地分支
git branch -a                       # 查看所有分支(远程)

git branch -d xxx                   # 删除本地分支, -D强制删除
git push origin --delete xxx        # 删除远程分支(origin为远程名)
```

### 六、checkout

```sh
# 想要switch的时候可以用switch是切换分支，如果本地没有会新建分支
# clone 是获取整个代码库
# pull 是把当前代码更新到最新
# checkout是从服务器获取分支并切换到该分支

# 合并代码
git checkout $dBranch               # 切换到待合并的目标分支
git merge $sBranch                  # 将分支名所在分支的改动合并到本分支
```

### 七、远程库

```sh
git remote -v                       # 查看所有配置的远程库
git fetch $remoteName               # 切换本地使用的远程库(空则为当前默认)
git remote remove $remoteName       # 删除本地使用的候选远程库

# 远程覆盖本地(丢弃本地修改)
git fetch --all
git reset --hard origin.master

# 修改远程库url地址
git remote set-url origin git@xx.xx.xx.xx:/xx/xxx.git
```

### 八、把已有项目添加到gitee

```sh
# git remote add $远程名 $远程仓库链接 e.g.:
git remote add gitee https://gitee.com/duringyl/projName.git
git push gitee --all                # 本地所有记录push到$远程(提示输入密码)
```
























