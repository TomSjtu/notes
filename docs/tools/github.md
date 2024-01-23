# github

由于众所周知的原因，github在国内的访问不尽如人意，但是国外开源项目几乎都部署在github上，不得不用。如果是自己的虚拟机连接网速比较慢，建议开个梯子然后配置下流量代理：

```SHELL
git config --global http.proxy http://127.0.0.1:[端口号]
 
git config --global https.proxy http://127.0.0.1:[端口号]
```

这两行命令的意思是让http和https协议的流量全部走你梯子的代理，端口号根据梯子的配置自行输入。配置完以后`git clone`应该不会卡了。如果你要取消全局代理，输入以下命令：

```SHELL
git config --global --unset http.proxy
 
git config --global --unset https.proxy
```

一般来说，github的SSH默认通过端口22建立连接，除非你更改了配置文件强制使用HTTPS的443端口连接。

要测试通过HTTPS端口的SSH连接是否可行，输入以下命令：

```SHELL
ssh -T -p 443 git@ssh.github.com

> Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```
如果输出以上内容，则说明连接可行。你可以更改~/.ssh/config文件，强制与github的连接都通过该服务器和端口：

```SHELL
Host github.com
Hostname ssh.github.com
Port 443
User git
```

通过再次连接到github来测试是否有效，注意这时就不需要指定端口号了：

```SHELL
ssh -T git@github.com

> Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

> 注意：SSH通过公钥和私钥配对的方式来验证连接是否安全，当你第一次通过SSH连接时，会询问你是否信任该服务器，输入yes即可。

## 添加SSH密钥

添加SSH密钥的过程很简单，按照步骤一步步来即可。

1.首先输入以下命令，将邮件地址改为你注册github的邮箱：

```SHELL
ssh-keygen -t ed25519 -C "your_email@example.com"
```

系统会要求你输入一些信息，连续按回车跳过即可。

2.将SSH密钥添加到ssh-agent：

先启动ssh代理：

```SHELL
eval "$(ssh-agent -s)"
```

然后添加私钥：

```SHELL
ssh-add ~/.ssh/id_ed25519
```

3.向你的账户添加新的SSH密钥

现在，.ssh/目录下有两个文件，以pub结尾的就是公钥，需要你上传至github服务器。

```SHELL
cat ~/.ssh/id_ed25519.pub
```

复制打印的内容，然后在github个人资料面点击设置，找到`SSH and GPG keys`，点击`New SSH key`，最后粘贴复制的公钥即可。

## 管理远程仓库

要添加一个远程仓库链接，使用`git remote add` 命令，该命令需要两个参数：

- 远程仓库名，比如`origin`
- 远程仓库地址，比如`https://github.com/OWNER/REPOSITORY.git`

```SHELL
git remote add origin https://github.com/OWNER/REPOSITORY.git
```

在添加完远程仓库后，可以使用`git remote -v`命令可以查看你当前添加的远程仓库：

```SHELL
git remote -v
# Verify new remote
> origin  https://github.com/OWNER/REPOSITORY.git (fetch)
> origin  https://github.com/OWNER/REPOSITORY.git (push)
```

如果要更改连接远程仓库的方式，比如从HTTPS改为SSH，则可以使用`git remote set-url`命令：

```SHELL
git remote set-url origin git@github.com:OWNER/REPOSITORY.git
```

这时远程仓库的格式应该是：

```SHELL
origin  git@github.com:OWNER/REPOSITORY.git (fetch)
origin  git@github.com:OWNER/REPOSITORY.git (push)
```

## git常用命令

项目初始化：

```SHELL
git init
```

初始化个人信息：

```SHELL
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

将变动的文件添加到暂存区：

```SHELL
git add --all
```

提交代码：

```SHELL
git commit -m "提交信息"
```

推送代码：

```SHELL
git push origin main
```

> origin替换为远程仓库的名称，main替换为提交的分支名

拉取代码到本地：

```SHELL
git push origin main
```

查看当前git状态：

```SHELL
git status
```

查看提交日志：

```SHELL
git log
```

切换分支：

```SHELL
git check -b [新分支名]
```

合并代码：

```SHELL
git merge [另一个分支名]
```

