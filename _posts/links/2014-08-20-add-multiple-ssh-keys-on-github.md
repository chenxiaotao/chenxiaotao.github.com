---
layout: link
category : link
link: http://www.webmaster.me/uncategorized/add-multiple-ssh-keys-on-github.html
title: 在GitHub多个帐号上添加SSH公钥
---

1. 生成钥对

        ssh-keygen -t rsa -C 'zhonghua800.github.com'
        Generating public/private rsa key pair.
        Enter file in which to save the key (/home/zhonghua/.ssh/id_rsa): /home/zhonghua/.ssh/zhonghua800
        Enter passphrase (empty for no passphrase): 
        Enter same passphrase again: 
        Your identification has been saved in /home/zhonghua/.ssh/zhonghua800.
        Your public key has been saved in /home/zhonghua/.ssh/zhonghua800.pub.
        The key fingerprint is:
        ...略...

2. 把`/home/zhonghua/.ssh/zhonghua800.pub` 放到 `https://github.com/settings/ssh`

3. 修改`~/.ssh/config` 添加:

        host zhonghua800
          user git 
          hostname github.com
          GSSAPIAuthentication no
          port 22
          identityfile ~/.ssh/zhonghua800

4. 修改(或者clone)

   * 修改: ` git remote set-url origin zhonghua800:zhonghua800/zhonghua800.github.com.git`

   * clone: ` git clone zhonghua800:zhonghua800/zhonghua800.github.com.git`
