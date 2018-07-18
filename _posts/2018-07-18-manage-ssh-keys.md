---
title: Git下管理多个ssh key
key: 20180719
tags: Git
---


&nbsp;&nbsp;&nbsp;&nbsp;对于程序员来说，可能由于私有库限制，网站访问速度及备份等原因将代码放在不同的代码托管平台，比如程序员交友网站[Github](https://github.com/),[Gitlab](https://gitlab.com/)以及国内的[码云](https://gitee.com)等，那么我们上传代码时就需要对不同的平台的ssh key进行管理，否则一个ssh的私钥是不可能在多个平台通用的，针对这种情况，解决方法如下：

<!--more-->

1. 生成两个key：
<br>
    使用命令`ssh-keygen -t rsa -C “Your Email Address” -f 'Your Name'`，-f后面指定生成key的文件名，如果没有指定新的名字，那么每次ssh-keygen生成的名字相同，就会发生覆盖同名文件的情况的发生。
<br>

2. 添加生成的公钥到对应服务器的ssh kyes管理设置中：
<br>

3. 修改配置文件：
<br>
    
&nbsp;&nbsp;&nbsp;&nbsp;在~/.ssh目录下新建一个config文件并添加以下内容

    # oschina
    Host git.oschina.net
    HostName git.oschina.net
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/oschina
    # github
    Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa
<br>

&nbsp;&nbsp;&nbsp;&nbsp;其中要注意Host及HostName填对应的托管网址，IdentityFile及我们之前生成的私钥，要与我们在网站上添加的公钥一一对应，切勿搞混！！！
&nbsp;&nbsp;&nbsp;&nbsp;最后就是测试是否成功，测试方法如下:

&nbsp;&nbsp;&nbsp;&nbsp;oschina的测试方法:
 `ssh -T git@gitee.com`显示`“Hi,‘Your Name’! ...”`代表设置成功
<br>
&nbsp;&nbsp;&nbsp;&nbsp;github的测试方法:
 `ssh -T git@github.com` 显示`“Hi,‘Your Name’! ...”`代表设置成功

            
        
