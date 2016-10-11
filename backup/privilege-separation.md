# 特权分离1：

##Zoobar特权分离实验

本实验学习如何在`zookws`和`zoobar`上应用特权分离，使得其中的bug不会令攻击者来转移zoobar到其账号。

###实验准备

`zoobar`应用在用户之间转移信贷，该功能通过`transfer.py`实现。下面启动并访问该服务：

``` sh
$ sudo make fix-flask
$ make
$ sudo make setup
$ sudo ./zookld zook.conf
```
- 用浏览器访问`zoobar`网站，创建两个账号
- 登录到一个用户，向另一个用户转账
- 注册用户可以更新用户profile，向其他用户转账，查看用户余额，profile，较易记录
- 理解`transfer.py`如何实现转账，阅读以下代码
`templates/transfer.html`, `__init__.py`, `transfer.py`, `bank.py`

- 类似OKWS，用`/jail`目录来建立`chroot` jail
- 编辑`chroot-setup.sh`来改变文件和目录权限，重新执行`sudo make setup`
- 挑战：
	- 需要拆分应用
	- 确保每个部分在最小特权下运行

每个Python脚本中都导入了一个debug库。`debug.py`提供单个函数`log(msg)`，该函数打印消息`msg`和栈踪迹到`stderr`（`zookld`所在终端）。

配置文件`zook.conf`中说明每个服务该如何运行：

```
[zookd]
```
`zook.conf`中只包含一个HTTP服务`zookfs_svc`。通过`zookfs`程序实现。`chroot`到`/jail`中，其中包含可执行程序（除了`zookld`），支撑库，`zoobar`站点。详见`zook.conf`和`zookfs.c`。

###利用Unix用户和权限实现特权分离

####练习1：支持chroot和非特权用户
启动精灵进程`zookld`读取`zook.conf`，设定所有服务在`root`下运行，绑定到特权端口80。

在缺省配置中，`zookd`和有漏洞的服务**不正确地**在`root`下运行，`zookld`未jail相关进程。

为了修复此问题，应该在`root`之外的非特权用户下运行服务。更改`zookld.c`和`zook.conf`来设定用户ID和组ID，并`chroot`。

- 更改`zookld.c`中`launch_svc()`来支持`chroot`（使用系统调用`chroots`）
- 更改`zookld.c`中`launch_svc()`来支持`root`之外的用户ID和组ID，(使用系统调用`setresuid`，`setresgid`，`setgroups`）
- 需要更改`chroot-setup.sh`来确保硬盘上的文件只能被正确进程读取，（`chmod`和`chown`命令，或`set_perms`函数，例如`set_perms 1234:5678 755 /path/to/file`设定文件拥有者为1234，组为5678，权限755（rwxr-xr-x））

更改`zook.conf`来使用上述功能。`sudo make check`来验证更改的配置是否通过基本测试

###练习2：拆分服务

改造`zookfs_svc`并更改`zook.conf`，将`zookfs_svc`分离成两个在不同用户下运行的服务：`static_svc`提供静态文件和`dynamic_svc`执行动态内容。

用`chroot-setup.sh`来设定文件和目录权限，确保静态服务不能读取动态服务中的数据库，动态服务不能更改静态文件，动态服务不能执行其他脚本。

分离需要`zookd`来确定哪一个服务应该处理哪一个请求。通过`zookwd`不更改应用或URL来过滤URL。URL过滤器在`zook.conf`中说明，支持正则表达式。例如，`url = .*`来匹配所有请求，而`url = /zoobar/(abc|def)\.html`来匹配`/zoobar/abc.html`和`/zoobar/def.html`。

不要为了安全而依赖URL过滤器；极难正确实现。

文件系统中有许多可执行文件，不能将它们都标记为不可执行。例如`zookfs`服务需要执行`/jail/usr/bin/python`来运行zoobar站点。已经在`zookfs`添加一个功能，通过设定拥有者和组来只运行可信程序。为使用这一功能，在服务配置文件中加入一行`args = UID GID`，例如`args = 123 456`说明服务只执行用户123和组456的文件。

用`sudo make check`来验证更改的配置是否通过基本测试。

###RPC库

学习RPC库实现进程间通过Unix套接字通信。作为一个演示程序，在`zoobar/echo-server.py`中实现了一个简单的"echo"服务，该服务由`zookld`启动，服务配置在`zook.conf`中。

`echo-server.py`通过定义一个类`EchoRpcServer`实现，该类继承了`zoobar/rpclib.py`中的`RpcServer`类。`echo-server.py`通过调用`run_sockpath_fork(sockpath)`来启动服务器，该函数监听UNIX域套接字`/echosvc/sock`。

包含了一个简单的客户端。访问`/zoobar/index.cgi/echo?s=hello`，请求被路由到`zoobar/echo.py`，该程序连接`/echosvc/sock`并启动echo操作。一旦从echo服务接收到应答，将返回一个包含应答的网页。这个客户端通过`rpclib`中`RpcClient`类的`call`方法实现。

###分离login服务

目前攻击者利用漏洞可以从`person`数据库中获得所有用户口令。数据库在`zoobar/db/`中，所有Python代码都能够访问。

创建一个新服务来处理用户口令和cookie。目前，所有用户相关信息存储在`Person`表中（见`zoodb.py`）。将认证信息从`Person`表中移动到一个`Cred`表中（Credential，委任状）。将访问该认证信息的代码（`auth.py`）移动到一个分离的服务。

- 确定认证服务的接口。查看`login.py`和`auth.py`，决定哪些在认证服务上运行，哪些在客户端运行。在`zoobar/auth_client.py`中提供了客户端的初始RPC代码。
- 创建用于用户认证的`auth_svc`服务，参考`echo-server.py`。提供了初始文件`zoobar/auth-server.py`,服务使用`auth.py`中函数。
- 更开`zook.conf`来启动`auth-server`（用一个不同的UID）。
- 从`Person`数据库中分离用户委任状（即口令和token）到`Cred`数据库，存储在`/zoobar/db/cred`。
- 更改`chroot-setup.sh`来在`cred`数据库上设定权限，创建认证服务的socket。
- 更改`login.py`中登录代码来唤起认证服务，而不是直接调用`auth.py`。

使用加盐哈希函数来保存密码，哈希函数选择Python的[PBKDF2](https://www.dlitz.net/software/python-pbkdf2/)模块。大致上，用`pbkdf2.PBKDF2(passward, salt).hexread(32)`来焊锡口令。`pbkdf2.py`在`zoobar`目录。用`os.urandom`替代`random.random`来产生salt。

###分离bank服务

将`zoobar`账户信息分离到一个`Bank`数据库中，建立`bank_svc`服务，在新的`Bank`和`Transfer`数据库上实现。

- 分理处`bank_svc`服务，以及认证服务
- 实现`transfer`和`balance`功能，目前在`bank.py`中实现
- 将`zoobar`余额信息放入`Bank`数据库（在`zoodb.py`中）
- 更改`bank-server.py`；将`bank_svc`服务加入到`zook.conf`；
- 更改`chroot-setup.sh`来创建新的`Bank`数据库和服务套接字，设定`Bank`和`Transfer`数据库；
- 创建客户端RPC stub来调用bank服务；
- 更改其余的代码来启动RPC stub，而不是调用`bank.py`的函数；
- 处理账号创建，新用户初始获得10个zoobar。


为了认证`transfer`操作调用者，需要一个额外的`token`参数。
在`transfer`RPC中加入认证。当前用户token是`g.user.token`

使用`sudo make check `来验证。


