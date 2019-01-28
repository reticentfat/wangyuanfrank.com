---
title: "id_rsa_are_too_open解决方法"
author: "wangyuan"
date: '2019-01-28'
slug: id_rsa_are_too_open解决方法
categories: ["technology"]
tags:  ["技术", "记录", "问题解决"]
---
# 2019-01-28_id_rsa_are_too_open解决方法

今天遇到一个ssh的问题，如下：

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/root/id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "/root/id_rsa": bad Permission denied (publickey,gssapi-keyex,gssapi-with-mic).

```

- 问题思考 :

看到664，感觉是文件权限问题，在so上搜索确认

- 贴下在stackoverflow答案 :

地址：[http://stackoverflow.com/questions/1556119/ssh-private-key-permissions-using-Git-gui-or-ssh-keygen-are-too-open](https://link.jianshu.com?t=http://stackoverflow.com/questions/1556119/ssh-private-key-permissions-using-Git-gui-or-ssh-keygen-are-too-open)

> You changed the permissions on the whole directory, which I agree with Splash is a bad idea. If you can remember what the original permissions for the directory are, I would try to set them back to that and then do the following

```
cd ~/.ssh
chmod 700 id_rsa
```

> inside the .ssh folder. That will set the id_rsa file to rwx (read, write, execute) for the owner (you) only, and zero access for everyone else.
>  If you can't remember what the original settings are, add a new user and create a set of SSH keys for that user, thus creating a new .ssh folder which will have default permissions. You can use that new .ssh folder as the reference for permissions to reset your .ssh folder and files to.
>  If that doesn't work, I would try doing an uninstall of msysgit, deleting ALL .ssh folders on the computer (just for safe measure), then reinstalling msysgit with your desired settings and try starting over completely (though I think you told me you tried this already).
>  Edited: Also just found this link via Google -- [Fixing "WARNING: UNPROTECTED PRIVATE KEY FILE!" on Linux](https://link.jianshu.com?t=http://www.howtogeek.com/wiki/Fixing_%22WARNING:_UNPROTECTED_PRIVATE_KEY_FILE!%22_on_Linux) While it's targeted at linux, it might help since we're talking liunx permissions and such

- 解决方法：

  增加执行权限

  ```
  chmod 700 id_rsa
  ```

- 修改后文件权限如下：

  ```
  [root@hadoop0 ~]# ll -h id_rsa*
  -rwx------ 1 root root 1.7K 1月  28 08:56 id_rsa
  -rw-r--r-- 1 root root  418 1月  28 09:35 id_rsa.pub
  ```
