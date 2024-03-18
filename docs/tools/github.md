# github

github支持HTTPS和SSH连接，但无论是哪种方式，都需要验证你的身份。

## HTTPS连接

过去，HTTPS连接要求你输入用户名和密码，但是这种方式由于不安全已经被github废弃。现在，要想通过HTTPS连接，你必须使用另外一种叫[Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager/blob/main/README.md)的方式。请直接查看官网文档。

## SSH连接

SSH连接默认采用22端口，但是有时候防火墙会阻止这种行为。如果遇到这种情况，你可以采用上面的HTTPS连接。如果HTTPS连接也不行，你可以尝试采用HTTPS端口的SSH连接。

要测试通过HTTPS端口的SSH连接是否可行，输入以下命令：

```SHELL
ssh -T -p 443 git@ssh.github.com

> Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```
如果输出以上内容，则说明连接可行。你可以更改~/.ssh/config文件，强制与github的连接都通过SSH的443端口。

```SHELL
Host github.com
Hostname ssh.github.com
Port 443
User git
```

更改配置后，再次测试SSH连接是否可行：

```SHELL
ssh -T git@github.com

> Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

!!! note ""

    注意：SSH通过公钥和私钥配对的方式来验证连接是否安全，当你第一次通过SSH连接时，会询问你是否信任该服务器，输入yes即可。github的SSH公钥存放在[Github's SSH key fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)。你可以添加到~/.ssh/known_hosts来避免验证。

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

## 网络配置

github支持流量代理：

```SHELL
git config --global http.proxy http://127.0.0.1:[端口号]
 
git config --global https.proxy http://127.0.0.1:[端口号]
```

这两行命令的意思是让http和https协议的流量全部走你梯子的代理，端口号根据梯子的配置自行输入。配置完以后`git clone`应该不会卡了。如果你要取消全局代理，输入以下命令：

```SHELL
git config --global --unset http.proxy
 
git config --global --unset https.proxy
```

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
git pull origin main
```

查看当前git状态：

```SHELL
git status
```

查看提交日志：

```SHELL
git log --oneline
```

切换分支：

```SHELL
git check -b [新分支名]
```

合并代码：

```SHELL
git merge [另一个分支名]
```

## 版本控制

放弃工作目录中的修改(还未提交到工作区)：

```SHELL
git checkout -- [file]  //放弃某个文件的修改
git checkout -- .       //放弃所有修改
```

回退到之前某个版本，且丢弃该版本之后的所有修改：

```SHELL
git reset --hard [版本号]
```

撤销之前某个版本，但保留该版本后面的版本：

```SHELL
git revert -n [版本号]  //-n表示手动解决冲突
```

暂存工作区：

```SHELL
git stash
```




