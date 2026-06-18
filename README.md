# 搬瓦工改端口教程：SSH 端口被封怎么办？如何检查端口是否被墙？改端口会不会断线？完整操作步骤一篇搞定

刚买搬瓦工，登进去部署了点东西，隔天一看，SSH 死活连不上。

先用 ping 测了一下 IP，通的。那就奇怪了——IP 没被封，但就是连不上。后来查了一圈，才搞明白：是**SSH 端口被封了**，不是 IP 的问题。

这种情况比 IP 被封还容易被忽视，但处理起来并不难。这篇文章把搬瓦工改端口的完整流程写清楚——从怎么判断端口被封，到怎么在 KiwiVM 里改，再到防火墙放行，一步都不省。

---

## 搬瓦工端口这件事，先说几个容易搞混的点

搬瓦工（BandwagonHost）的 SSH 端口**不是默认的 22**。每个账号购买后，系统会随机分配一个五位数端口，比如 27790、8719 这种。这个设计本身是出于安全考虑，随机端口让自动化扫描工具直接跑过去。

但问题是，这个随机端口也可能被封。

所以一旦遇到 SSH 连不上、IP 没问题的情况，第一步先排查端口。

---

## 第一步：确认端口是否被封

打开你本地的终端（Windows 用 PowerShell 或 cmd，Mac/Linux 直接用系统终端），输入：


telnet 你的IP地址 你的端口号


比如你的 IP 是 45.78.32.221，端口是 27790，那就输：


telnet 45.78.32.221 27790


**能连上的结果**：屏幕出现类似 `SSH-2.0-OpenSSH_X.X` 的字样，说明端口正常。

**被封的结果**：连接超时，或者直接毫无反应，光标一直在转。

确认端口被封之后，就进入下一步——改端口。

---

## 第二步：通过 KiwiVM 后台进入系统

端口被封意味着你正常的 SSH 客户端（XShell、Termius、系统终端）全进不去了。这时候只能走搬瓦工的 KiwiVM 后台面板里的 **Root shell – interactive** 功能——这个是网页版终端，不走 SSH 端口，所以端口被封照样能用。

**操作路径**：

1. 登录搬瓦工账号，进入 KiwiVM 管理面板
2. 左侧菜单找到「Root shell – interactive」
3. 点击「Launch」，弹出网页终端
4. 输入用户名 `root` 和你的 root 密码

进来之后，就是个正常的 Linux 终端，接下来的命令跟普通 VPS 完全一样。

---

## 第三步：修改 SSH 配置文件

在终端里输入：


vim /etc/ssh/sshd_config


如果提示 vim 不存在，先装一下：

- Ubuntu / Debian：`apt install -y vim`
- CentOS：`yum -y install vim`

打开文件后，往下翻找到 `Port` 这一行。

**vim 操作**：按 `i` 进入编辑模式，把端口号改成你想要的数字，然后按 `Esc`，输入 `:wq` 回车保存。

端口选什么好？建议 10000–65535 之间随便选一个不常用的数，比如 54321、39999。80、443、8080 这些别用，跟 Web 服务冲突。

---

## 第四步：防火墙放行新端口

这一步是很多教程没写清楚、最容易翻车的地方。

改完配置文件还没完——如果你的系统开了防火墙，新端口必须手动放行，不然改了也连不上。

**检查用的是哪个防火墙**：


systemctl status firewalld
systemctl status iptables


哪个有输出用哪个。

**firewalld 放行**：


firewall-cmd --zone=public --add-port=54321/tcp --permanent
firewall-cmd --reload


**iptables 放行**：


iptables -A INPUT -p tcp --dport 54321 -j ACCEPT
service iptables save


把 `54321` 换成你实际设置的端口号。

---

## 第五步：重启 SSH 服务

防火墙放行之后，重启 SSH 让新配置生效：


systemctl restart sshd


如果是老系统用不了 systemctl，试这个：


/etc/init.d/sshd restart


或者直接重启整台 VPS 也行：


reboot


---

## 第六步：用新端口验证能否登录

**先别关掉 KiwiVM 的网页终端**。

打开你的 SSH 客户端，用新端口试着连一下。能连上，说明整个流程走通了，之后就用新端口登录。

连不上，回 KiwiVM 终端继续排查：检查防火墙规则有没有生效，检查 SELinux 有没有拦截（CentOS 7 经常这个问题）。

---

## CentOS 7 特别注意：SELinux 可能拦截新端口

讲真，这个坑踩过才知道多烦。

CentOS 7 的 SELinux 默认对 SSH 端口有限制，改了配置文件、放了防火墙，还是连不上，可能就是它在搞鬼。

处理方法有两种，选一个就行：

**方法 A：用 semanage 给 SELinux 添加端口规则（推荐）**


semanage port -a -t ssh_port_t -p tcp 54321


如果 semanage 命令不存在，先装：


yum install -y policycoreutils-python


**方法 B：直接关掉 SELinux（懒人方案，但不推荐生产环境）**

打开 `/etc/selinux/config`，把 `SELINUX=enforcing` 改成 `SELINUX=disabled`，保存后重启。

---

## 改端口还是建议保留随机大数字

搬瓦工默认给你分配的五位数随机端口，安全性其实挺好——自动化扫描工具一般只扫 22、2222 这些常见端口，随机五位数基本不在扫描列表里。

端口被封之后改端口当然是必须的。但改完之后，用 10000 以上的大数字比改回 22 要好。改回 22 相当于把门重新开在大家都知道的地方。

还有一个隐患：旧端口改掉之后，确认新端口能用之前，别急着把旧端口从配置里删掉。可以先在 sshd_config 里并行配置两个端口（比如 `Port 27790` 和 `Port 54321` 各一行），两个都能登上去再把旧的删掉，这样操作更安全。

---

## 搬瓦工在售套餐汇总（含 AFF 直达）

👉 [查看搬瓦工当前所有套餐与最新优惠](https://bit.ly/BanWaGon)

| 套餐系列 | 内存 | CPU | 硬盘 | 月流量 | 带宽 | 适合场景 | 价格 | 购买 |
|---|---|---|---|---|---|---|---|---|
| KVM 入门款 | 1GB | 2核 | 20GB SSD | 1TB | 1Gbps | 学 Linux / 轻量建站 | $49.99/年 | [ 选择此方案](https://bwh81.net/aff.php?aff=80238&pid=44) |
| KVM 进阶款 | 2GB | 3核 | 40GB SSD | 2TB | 1Gbps | 建站/跑脚本 | $99.99/年 | [ 选择此方案](https://bwh81.net/aff.php?aff=80238&pid=45) |
| CN2 GIA-E 基础款 | 1GB | 2核 | 20GB SSD | 1TB | 2.5Gbps | 国内访问首选 / 日常使用 | $49.99/季 · $169.99/年 | [ 立即选购](https://bwh81.net/aff.php?aff=80238&pid=87) |
| CN2 GIA-E 大内存款 | 2GB | 3核 | 40GB SSD | 2TB | 2.5Gbps | AI 部署 / 多用户 | $89.99/季 · $299.99/年 | [ 立即选购](https://bwh81.net/aff.php?aff=80238&pid=88) |
| SLA 保障款（1GB） | 1GB | 独享2核 | 20GB SSD | 1TB | 2.5Gbps | 外贸建站 / 99.99% SLA | $65.89/季 · $239.99/年 | [ 立即选购](https://bwh81.net/aff.php?aff=80238&pid=164) |
| SLA 保障款（2GB） | 2GB | 独享3核 | 40GB SSD | 2TB | 2.5Gbps | 高稳定外贸/企业应用 | $116.99/季 · $399.99/年 | [ 立即选购](https://bwh81.net/aff.php?aff=80238&pid=165) |
| 香港 CN2 GIA（2GB） | 2GB | 2核 | 40GB SSD | 0.5TB | 1Gbps | 极低延迟 / 国内用户 | $89.99/月 · $899.99/年 | [ 查看香港套餐](https://bwh81.net/aff.php?aff=80238&pid=95) |
| 香港 CN2 GIA（4GB） | 4GB | 4核 | 80GB SSD | 1TB | 1Gbps | 高端建站 / 企业级 | $155.99/月 · $1559.99/年 | [ 查看香港套餐](https://bwh81.net/aff.php?aff=80238&pid=96) |
| 大阪 CN2 GIA（2GB） | 2GB | 2核 | 40GB SSD | 0.5TB | 1.5Gbps | 高端性价比 / 日本线路 | $49.99/月 · $499.99/年 | [ 查看大阪套餐](https://bwh81.net/aff.php?aff=80238&pid=134) |
| 大阪 CN2 GIA（4GB） | 4GB | 4核 | 80GB SSD | 1TB | 1.5Gbps | 大流量 / 高配 | $86.99/月 · $869.99/年 | [ 查看大阪套餐](https://bwh81.net/aff.php?aff=80238&pid=135) |

支持支付宝、银联、PayPal、信用卡付款。新账号 30 天内不满意可申请全额退款，后台直接提交，自动处理。

---

## 常见问题（FAQ）

**Q：搬瓦工的 SSH 端口为什么不是 22？**

搬瓦工给每个用户随机分配 SSH 端口，一般是五位数。这是官方默认的安全策略。你在 KiwiVM 面板首页就能看到分配给你的端口号，连接 SSH 时要把这个端口填进去，不能直接用 22。

**Q：改完端口之后，之前设置的东西（比如 bbr、代理配置）还在吗？**

在的。改端口只改了 SSH 的连接入口，服务器上的其他软件、配置、文件全部不动。相当于换了个门，屋里的东西原封不动。

**Q：CentOS 改完 sshd_config 但新端口还是连不上，怎么排查？**

按顺序检查三个地方：1）防火墙有没有放行新端口；2）SELinux 有没有拦截（这是 CentOS 7 最常见的坑，用 `semanage port -a -t ssh_port_t -p tcp 端口号` 解决）；3）sshd_config 里的端口号有没有保存成功（可以用 `cat /etc/ssh/sshd_config | grep Port` 确认）。

**Q：改端口后，KiwiVM 里显示的端口号会自动更新吗？**

不会自动更新。KiwiVM 面板里显示的端口是系统记录的，改了 sshd_config 之后面板不会同步变。你以后连 SSH 要用自己改的新端口，不要再看 KiwiVM 里的那个旧数字。

**Q：没有 VPS 还想试试搬瓦工，入门买哪个？**

KVM 年付 $49.99 那款，够学习用，搭个个人网站也够，出了问题 30 天内可以退。对国内访问速度有要求的，直接看 CN2 GIA-E，算下来每个月不到 15 美元，线路质量差很多。

👉 [对比全部套餐，选最适合你的方案](https://bit.ly/BanWaGon)
