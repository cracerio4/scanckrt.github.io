图形桌面共享系统（称为虚拟网络计算 (VNC)）可让您使用键盘和鼠标远程操作另一台计算机。Microsoft
远程桌面协议 (RDP) 的免费开源替代品现已推出。

本文介绍了如何在 [Ubuntu](https://xiny.phpnet.us/tag/ubuntu/) 20.04 上安装和配置 VNC 服务器。
此外，我们还将演示如何与 VNC 服务器建立安全的 SSH 隧道连接。

安装桌面环境
Ubuntu 服务器没有预装桌面环境，而是通过命令行进行操作。
如果你在桌面上使用 Ubuntu，则可以跳过此步骤。

Ubuntu 存储库包含各种桌面环境。
安装 Gnome（Ubuntu 20.04 中的默认桌面环境）是一种选择。
安装 Xfce 是另一种选择。
它是在远程服务器上使用的最佳桌面环境，因为它快速、可靠且轻量级。

本指南介绍了如何安装 Xfce。
具有 sudo 权限的用户应输入以下命令：

sudo apt update
sudo apt install xfce4 xfce4-goodies

下载并安装 Xfce 包可能需要一些时间，具体取决于您的机器。

安装 VNC 服务器
Ubuntu 存储库中有多种 VNC 服务器，包括 TightVNC、TigerVNC 和 x11[vnc](https://xiny.phpnet.us/tag/vnc/)。
每种 VNC 服务器都有独特的性能和安全优缺点。

sudo apt install tigervnc-standalone-server

配置 VNC 访问
安装 VNC 服务器后，即可完成初始用户配置和密码设置。

vncpasswd 命令可用于更改用户密码。
无需使用 sudo，运行以下命令：

vncpasswd

系统将要求您输入密码、确认密码，并决定是否要将其设为仅供查看的密码。
如果您选择设置仅供查看的密码，用户将无法使用鼠标或键盘与 VNC 实例通信：

Password:
Verify:
Would you like to enter a view-only password (y/n)? n
 
密码文件存储在~/.vnc 目录中，如果不存在则创建该目录。

接下来，我们需要配置 TigerVNC 以使用 Xfce。为此，请创建以下文件：

nano ~/.vnc/xstartup

#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4 
 
保存并关闭文件。每当您启动或重新启动 TigerVNC 服务器时，上述命令都会自动执行。

~/.vnc/xstartup 文件也需要有执行权限，使用 chmod 命令设置文件权限：

chmod u+x ~/.vnc/xstartup

创建一个名为 config 的文件，如果您需要为 VNC 服务器提供[更多选择，](https://xiny.phpnet.us/?golink=aHR0cHM6Ly90aWdlcnZuYy5vcmcv)则每行添加一个选项。
以下是一个例子：编辑 ~/.vnc/config

geometry=1920x1080
dpi=96
 
您现在可以使用 vncserver 命令启动 VNC 服务器：

vncserver

Output:
New 'server2.linuxize.com:1 (linuxize)' desktop at :1 on machine server2.linuxize.com
Starting applications specified in /home/linuxize/.vnc/xstartup
Log file is /home/linuxize/.vnc/server2.linuxize.com:1.log
Use xtigervncviewer -SecurityTypes VncAuth -passwd /home/linuxize/.vnc/passwd :1 to connect to the VNC server.
 
在上面的输出中，请注意主机名后面的 :1。
这将显示 vnc 服务器当前正在运行的显示端口号。
此实例中的服务器在 TCP 端口 5901（5900+1）上运行。
服务器当前在端口 5902（5900+2）上运行，因为如果您使用 vncserver 启动第二个实例，它将在下一个可用端口上运行，即

使用 VNC 服务器时，务必记住 :X 代表 5900+X 的显示端口。

您可以通过输入以下内容来获取当前处于活动状态的所有 VNC 会话的列表：

vncserver -list

Output:
TigerVNC server sessions:
X DISPLAY # RFB PORT #  PROCESS ID
:1            5901          5710
 
在继续下一步之前，请使用带有 -kill 选项和服务器编号作为参数的 vncserver 命令停止 VNC 实例。在此示例中，服务器在端口 5901 (:1) 上运行，因此我们将使用以下命令停止它：

vncserver -kill :1

Output:
Killing Xtigervnc process ID 5710... success!
 
创建 Systemd 单元文件
让我们编写一个 systemd 单元文件来自动启动、停止和重新启动 VNC 服务，而不是手动开始会话。

打开文本编辑器后，将以下配置复制并粘贴到其中。
请确保第 7 行的用户名与您的用户名相对应。

sudo nano /etc/systemd/system/vncserver@.service

[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target
[Service]
Type=simple
User=linux
PAMName=login
PIDFile=/home/%u/.vnc/%H%i.pid
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill :%i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/vncserver :%i -geometry 1440x900 -alwaysshared -fg
ExecStop=/usr/bin/vncserver -kill :%i
[Install]
WantedBy=multi-user.target
 
保存并关闭文件。
通知 systemd 新的单元文件已创建：

sudo systemctl daemon-reload

启用服务在启动时启动：

sudo systemctl enable vncserver@1.service

@ 符号后面的数字 1 定义 VNC 服务将在其上运行的显示端口。这意味着 VNC 服务器将监听端口 5901，正如我们在上一节中讨论的那样。

通过执行以下操作启动 VNC 服务：

sudo systemctl start vncserver@1.service

使用以下命令验证服务是否已成功启动：

sudo systemctl status vncserver@1.service

Output:
● vncserver@1.service - Remote desktop service (VNC)
     Loaded: loaded (/etc/systemd/system/vncserver@.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2021-03-26 20:00:59 UTC; 3s ago
...
 
连接到 VNC 服务器
由于 VNC 不是加密协议，因此可以进行数据包嗅探。
建议的操作是创建 SSH 隧道并安全地将流量从端口 5901 上的本地工作站转发到服务器。

在 Linux 和 macOS 上设置 SSH 隧道
如果您使用 Linux、macOS 或任何其他基于 Unix 的操作系统，则可以使用以下命令在您的计算机上快速构建 SSH 隧道：

ssh -L 5901:127.0.0.1:5901 -N -f -l vagrant 192.168.33.10

请求时必须输入用户密码。

确保分别用您的用户名和服务器的 IP 地址替换用户名和服务器的 IP 地址。

在 Windows 上设置 SSH 隧道 [PuTTY SSH 客户端](https://xiny.phpnet.us/?golink=aHR0cHM6Ly93d3cucHV0dHkub3JnLw==)
可用于在 Windows 系统上配置 SSH 隧道。

启动 Putty 并在主机名或 IP 地址字段中输入服务器的 IP 地址。

图片描述

在连接菜单框下，展开 SSH，然后选择隧道。在源端口字段中输入 VNC 服务器端口 (5901)，在目标字段中输入 server_ip_address:5901，然后单击添加按钮，如下图所示：

图片描述

返回 Session 页面保存设置，这样就不需要每次都输入了。在远程服务器上，选择已保存的会话并单击 Open 按钮。

使用 Vncviewer 连接
现在是时候打开您的 VNC 查看器并连接到 localhost:5901 的 VNC 服务器了，因为 SSH 隧道已经形成。

任何 VNC 查看器均可以接受，包括 Vinagre、TigerVNC、TightVNC、RealVNC、UltraVNC 和适用于 Google Chrome 的 VNC 查看器。

这里使用 TigerVNC，打开查看器，输入 localhost:5901 后点击 Connect 按钮。

图片描述

出现提示时输入您的用户密码，您将看到默认的 Xfce 桌面。它看起来会像这样：

图片描述

您可以从本地计算机使用键盘和鼠标开始与远程 XFCE 桌面进行交互。

结论
在 Ubuntu 20.04 上，我们演示了如何设置和配置 VNC 服务器。

使用 vncpasswd 命令设置基本设置和密码，以便配置 VNC 服务器以启动多个用户的显示。
此外，必须创建具有不同端口的新服务文件。

如果您有任何疑问，请随时发表评论。