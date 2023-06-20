# 漏洞简介

## 0x01 基本情况

sudo中的sudoedit对处理用户提供的环境变量（如SUDO_EDITOR、VISUAL和EDITOR）中传递的额外参数存在缺陷。当用户指定的编辑器包含绕过sudoers策略的“--”参数时，拥有sudoedit访问权限的本地攻击者可通过将任意条目附加到要处理的文件列表中，最终在目标系统上实现权限提升（由普通用户到超级用户，即"root"）

## 0x02 影响范围

sudo 1.8.0 ~ sudo 1.9.12p1



# 漏洞环境构建

## 0x01 漏洞相关软件安装

```sh
wget https://www.sudo.ws/dist/sudo-1.9.12p1.tar.gz
tar -zxvf ./sudo-1.9.12p1.tar.gz
cd sudo-1.9.12p1
./configure
make -j8
make install
```

确认目标版本安装成功

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/1.png)

## 0x02 其他软件安装

```sh
#vim
sudo apt install vim

#pwndbg
git clonehttps://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh
```



# 漏洞调试分析

## 0x01 漏洞触发条件设置

**创建普通用户（test/test）**

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/3.png)



**赋予普通用户使用sudoedit编辑指定文件的权限**

创建/etc/test文件，文件内容随意

使用root账户编辑/etc/sudoers文件：

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/4.png)

前面的test是用户名，后面的/etc/test是任意文件

漏洞触发条件设置完毕



## 0x02 漏洞根本原因

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/5.png)

如上图，在函数resolve中，当sudoedit要编辑文件时，会在待编辑文件的前面加上“--”用以标记，若我们在传入参数时预先传入“--”加上敏感文件，就可以实现编辑任意敏感文件，从而实现提权



## 0x03 动态调试验证

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/6.png)

调试开始，指定EDITOR为“vim -- /etc/sudoers”用于后面绕过验证编辑sudo权限敏感文件，断点下在main函数，“r /etc/test”用于匹配参数进行运行

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/7.png)

下两个断点准备跟进程序

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/8.png)

在policy_check中，使用虚表，实际上是调用了sudoers_policy_check函数，进入查看

后续再加三个断点直至进入resolve_editor函数

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/9.png)

执行进入resolve_editor函数，调用栈如下

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/10.png)

检查传入参数

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/11.png)

发现EDITOR确实是被我们篡改的指定字符串

在漏洞处监测加断点

![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/12.png)



![](https://github.com/Chan9Yan9/CVE-2023-22809/blob/main/picture/13.png)

执行过漏洞语句后检查nargv数组，发现在“/etc/sudoers”后添加了“--”用以标定后续传入的/etc/test文件，结合前文注入的“-- /etc/sudoers”，sudoedit便会编辑sudoers文件，绕过本来应有的权限，触发漏洞。



# sudo相关参考资料

https://blog.csdn.net/weixin_42445065/article/details/125606166

https://blog.csdn.net/albertsh/article/details/130894098

https://blog.csdn.net/zhaoyangwyyx/article/details/107228431

