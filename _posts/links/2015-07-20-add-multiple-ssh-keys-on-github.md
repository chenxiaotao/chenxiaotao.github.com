---
layout: link
category : link
link: http://www.webmaster.me/uncategorized/add-multiple-ssh-keys-on-github.html
title: 在GitHub多个帐号上添加SSH公钥
---

1. 生成钥对

        ssh-keygen -t rsa -C 'sheldon.github.com'
        Generating public/private rsa key pair.
        Enter file in which to save the key (/home/sheldon/.ssh/id_rsa): /home/sheldon/.ssh/sheldon
        Enter passphrase (empty for no passphrase): 
        Enter same passphrase again: 
        Your identification has been saved in /home/sheldon/.ssh/sheldon.
        Your public key has been saved in /home/sheldon/.ssh/sheldon.pub.
        The key fingerprint is:
        ...略...

2. 把`/home/sheldon/.ssh/sheldon.pub` 放到 `https://github.com/settings/ssh`

3. 修改`~/.ssh/config` 添加:

        host sheldon
          user git 
          hostname github.com
          GSSAPIAuthentication no
          port 22
          identityfile ~/.ssh/sheldon

4. 修改(或者clone)

   * 修改: ` git remote set-url origin sheldon:sheldon/sheldon.github.com.git`

   * clone: ` git clone sheldon:sheldon/sheldon.github.com.git`
