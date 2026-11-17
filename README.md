# LiteSpeed主机怎么选：共享主机自带的那份许可证，和 VPS 上要另买的那份差在哪？cPanel 从 Apache 换过去会踩哪些坑？（附套餐对比表）

买 LiteSpeed主机，钱花在两个地方：机器本身，和 LiteSpeed 的许可证。cPanel 共享主机一般两样都替你办好了，方案特性里直接写 LiteSpeed Web Server，你登录面板开缓存就行；VPS 和独立服务器只给硬件，许可证要自己去 LiteSpeed 买，装完还得手动启用 LSCache 才真正生效。

下面把三件事讲清楚：RackNerd 现在在卖的共享主机套餐是什么配置、LiteSpeed 几种许可证的门槛分别卡在哪、以及在一台 cPanel 服务器上从 Apache 切到 LiteSpeed 的完整顺序和几个容易翻车的点。

> 想省事：个人博客、企业展示站、要用 cPanel 又不想碰命令行，选 RackNerd 的共享主机，30 GB NVMe 那档年付 **$10.49** 起，方案里含 cPanel、LiteSpeed Web Server、免费 SSL、免费迁移和 JetBackup 每日异地备份，机房可选美国、欧洲、亚洲。要 root 权限、要自己配环境，再去看 KVM VPS，那类机器不附 LiteSpeed 许可证。
> 👉 [查看 RackNerd 共享主机当前套餐与价格](https://bit.ly/RacKnerd)

## RackNerd 共享主机套餐对比

| 套餐 | 核心配置 | 价格 | 计费周期 | 购买 |
| --- | --- | --- | --- | --- |
| SHARED 30 GB | 30 GB NVMe、3 TB 月流量、可托管 2 个域名、不限数据库 | **$10.49/年** | 年付 | [查看 30 GB 套餐](https://bit.ly/RacKnerd) |
| SHARED 85 GB | 85 GB NVMe、10 TB 月流量、可托管 10 个域名、不限数据库 | 见官方页面 | 年付 | [查看 85 GB 套餐](https://bit.ly/RacKnerd) |
| SHARED 200 GB | 200 GB NVMe、30 TB 月流量、不限域名、不限数据库 | 见官方页面 | 年付 | [查看 200 GB 套餐](https://bit.ly/RacKnerd) |

三档共有的东西：cPanel、CloudLinux、KernelCare、Softaculous 一键装脚本、SiteJet AI 建站、免费迁移、免费 SSL、JetBackup 每日异地备份，以及 LiteSpeed Web Server。85 GB 和 200 GB 的价格随促销调整，表里不写死，点进去看当天的报价。

RackNerd 的 KVM VPS 是另一条产品线，配置从入门到高配都有，价格和库存以官网为准。要留意的是 VPS 默认不带 LiteSpeed 许可证，那部分成本要单独算。

## 先分清你买的是哪一层：主机套餐 ≠ LiteSpeed 许可证

LiteSpeed 官方产品页把 LSWS 许可证分成三档：Free Starter、Site Owner、Web Host。免费档的门槛写在 cPanel 快速上手文档里，两个条件必须同时满足：**只托管一个域名，且是一台物理内存小于 2 GB 的物理服务器**。内存超过 2 GB、又要放多个域名，走的是试用许可证，官方另有一份试用说明。长期用就买生产许可证，按序列号发放，**一个序列号对应一台服务器的安装**，序列号存在安装目录下的 `serial.no` 里，注册服务器凭它取回 license key。安装程序只认有效试用许可证或有效 Enterprise 许可证，两样都没有，装到一半就会停。

买主机的人会被这套规则分成三种情况：

- 买共享主机：主机商已经买好许可证并装好，你付的是主机费，看不到也用不着序列号。
- 买 VPS 或独立服务器：自己买。官方免费 Starter 那条路只适合单域名加小内存的物理机，多数 VPS 不满足。
- 想先试：试用许可证按服务器 IP 追踪，同一台机器领过就不能自动再领，符合条件的可以申请人工重置。

所以「LiteSpeed 主机」之间的价差，有一部分差在许可证那一层，不全是 CPU 和内存。

👉 [对比 RackNerd 三档共享主机：方案特性里都写着 LiteSpeed Web Server](https://bit.ly/RacKnerd)

## cPanel 上从 Apache 换到 LiteSpeed，按这个顺序来

顺序错了会白折腾一轮，下面每一条都附上原因。

1. **先关掉 Apache 的 `mod_ruid2`**。这个模块会把某个域名的 HTTP 请求权限切换成域名所有者的权限，LSWS 不允许进程中途换用户，两者不兼容。切换前在 WHM 设置里禁用。
2. **防火墙放行 TCP 80、TCP 443、UDP 443 和 TCP 7080**。UDP 443 是 QUIC 用的，官方自己说 QUIC 是 LiteSpeed 最受欢迎的特性之一，也是最常被漏掉的一条。7080 是 WebAdmin 控制台，只对管理 IP 开放，登录需要用户名和密码。
3. **Apache port offset 先设成非 0**（默认值 1000 就行）。这样 Apache 继续占 80/443，LiteSpeed 在 1080 之类的位置并行跑，你可以先测。确认站点正常后再改成 0，让 LiteSpeed 直接接管 80/443。改成 0 的同时必须打开「Switch to LiteSpeed Automatically」，否则装完它不会自己启动。
4. **LiteSpeed 进程的 username / groupname 设成和 Apache 一样的值**。用默认的 `nobody:nobody` 容易出文件权限问题。
5. **确认服务器能连上 LiteSpeed 的许可证验证服务器**，连不上安装会卡在验证环节。
6. **安装过程会显示一个随机生成的 WebAdmin 密码**，当场存好，之后可以改。
7. **切换在 WHM 的 Plugins > LiteSpeed Web Server 里做**，页面上有一键 Switch to LiteSpeed / Switch to Apache，能回退。不要用 `service httpd stop` 再 `service lsws start` 这种手动切法，官方明确不推荐，中间还有别的步骤。
8. **出问题先看安装目录下的 `logs/error.log`**。端口冲突用 `netstat -an | grep 端口号` 查；swap 目录默认 `/tmp/lshttpd/swap`，写在 `conf/httpd_config.xml` 里，不可写就给运行用户授权或换目录。

装完的默认端口：普通站点 8088，WebAdmin 界面 7080。LSWS 在 cPanel 上是 Apache 的完整替代，会自动接管服务器上已经托管的所有域名，新加的域名也不用额外配置。升级时重跑安装脚本选 Upgrade，现有配置保留；官方安装文档当前给出的手动包是 **LSWS Enterprise v6.3.4**。

## LSCache：许可证里包含，但默认不生效

LiteSpeed 官方把全页缓存列为自己最受关注的功能，WordPress、Magento 这类 CMS 上尤其明显。有个细节常被误解：**缓存能力包含在每一份 LiteSpeed 许可证里，但默认不生效**，得在服务器上手动启用 LSCache 并设置 cache root。

cPanel 上的做法：

- 安装器里有选项可以自动部署 LSCache。勾上之后会在服务器级和虚拟主机级建好默认 cache root，并扫描服务器上所有 WordPress 站点自动装 LSCache 插件。
- 扫描完成后点 Mass Enable/Disable Cache → Start 批量开启，耗时看站点数量。
- 扫描结束时底部可能刷出一堆报错，多数是无效的 WordPress 安装（数据库没配好、WordPress 版本低于 4.0），LiteSpeed 会自动标记并跳过，可以忽略。
- 已经在用其他 WordPress 缓存插件的站点不会被装 LSCache，会显示失败，这部分要人工处理。
- 其他报错查 `lsws/log/webcachemgr.log`。
- 之后新增的 WordPress 站点，可以用 `/usr/local/lsws/admin/misc/lscmctl` 这个工具自动扫描并开启缓存。

LSCache 插件覆盖 WordPress、Drupal、Joomla、PrestaShop、Magento、OpenCart 等。

👉 [查看 RackNerd 共享主机方案包含的完整特性列表](https://bit.ly/RacKnerd)

## 官方给的性能数字，先看清是谁说的

LiteSpeed 官方产品页的原话：事件驱动架构能把现有 Apache 服务器的最大容量翻倍；典型情况下服务器负载减半、TTFB 提升 3 倍；替换 Apache 15 分钟内完成、零停机。这些是厂商自述口径，没有第三方实测数据可以对照，看到「3 倍」这种数字先记住出处。

机制上能确认的差异更值得看。LSWS 直接读 Apache 配置文件，支持 `.htaccess` 和 mod_rewrite，所以为 Apache 写的控制面板（cPanel、Plesk、DirectAdmin、CyberPanel）都能直接用。nginx 同样是事件驱动，但不读 Apache 配置，这是两者在迁移成本上的主要区别。另外支持 HTTP/2（需 5.0 及以上版本）、QUIC、HTTP/3；兼容 ModSecurity 规则集，Liquid Web 的帮助文档把 ModSecurity 支持标注为 Enterprise Edition 功能；PHP 侧用 LSPHP 配 suEXEC。

## 这几种情况，别急着买 LiteSpeed 主机

- **只有一个域名，服务器物理内存不到 2 GB**：LiteSpeed 官方的 Free Starter 许可证就够用，不必为许可证掏钱。这条是 LiteSpeed 的政策，跟具体主机商无关。
- **不跑 cPanel**：安装流程会不一样。InterWorx 上 port offset 既不能设 0（和 Apache 抢端口），也不能设 2000（和面板冲突），踩坑概率高。
- **指着「LiteSpeed」三个字解决所有速度问题**：共享主机上你调不了服务器级缓存参数，能做的就是把 LSCache 插件装好、把缓存规则配对。站点本来访问量不大，先压缩图片、减掉冗余脚本，效果可能比换服务器更直接。
- **需要退款兜底**：按 RackNerd 现行服务条款，付款后默认不退款，年付特价也一样。先买最小的那档试，比事后申请退款现实。

## 常见问题

**安装时提示 `lvectl command not found`，要处理吗？**
不用。这条只说明你的 cPanel/WHM 不在 CloudLinux 上。如果服务器确实是 CloudLinux 还报这个错，才需要开工单。

**试用许可证领过一次，还能再领吗？**
LiteSpeed 按服务器 IP 追踪试用。同一台机器领过就不能自动再领，符合条件的服务器可以联系官方人工重置。

**升级 LiteSpeed 会不会把现有配置冲掉？**
重跑安装脚本时选 Upgrade，现有配置保留。从 2.0 版本起还能在 WebAdmin 的 Actions - Version Management 里管理版本。

**换服务器，许可证怎么迁？**
在旧服务器上跑 `./lshttpd -m`，它会联系注册服务器解锁序列号，生成一个 **3 天有效**的临时密钥，给你留出切换窗口。或者把旧安装目录整体复制过去，在新机器上跑 `./lshttpd -r` 取回新密钥。两种都失败，就到在线商店的 client area 提交工单。

**共享主机还要自己买 LiteSpeed 许可证吗？**
不用。RackNerd 的共享主机方案特性里写明用 LiteSpeed Web Server，许可证由主机商持有。你要做的是在 cPanel 里把 LSCache 插件装上并开启缓存。

## 怎么选

一个站、想省事、按年付预算，用 RackNerd 共享主机 30 GB 那档，**$10.49/年**起，2 个域名够用，LiteSpeed、cPanel、SSL、异地备份都在里面。多个站就往上走：85 GB 给 10 个域名，200 GB 不限域名，价格点进去看当天报价。要 root、要装自定义环境，走 VPS，但把 LiteSpeed 许可证的成本和安装时间算进去。只有一个域名加小内存物理机，先看 LiteSpeed 官方的 Free Starter。

下单前确认两件事：机房位置（美国、欧洲、亚洲都有，延迟差别不小，按访客在哪儿挑），以及计费周期和退款条款。年付便宜，但 RackNerd 付款后默认不退款，先想清楚再付。

👉 [以 $10.49/年 起开通 RackNerd 30 GB 共享主机](https://bit.ly/RacKnerd)
