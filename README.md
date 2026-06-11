# VPS 测速方法完整指南：Ping 延迟怎么测？带宽跑分怎么跑？路由追踪怎么看？一篇搞定，附 DMIT 三大机房实测参考数据

买了 VPS，第一件事是什么？

不是部署应用，不是搭服务——是测速。不测心里不踏实。拿到一台新机器，必须先跑几个数字出来，才知道自己买的到底是香饽饽还是坑。

**VPS 测速**，本质上是从三个维度去衡量一台服务器的网络质量：延迟（Ping 值）、带宽速率（上下行）、路由质量（丢包和绕路情况）。三者缺一，测出来的数据都是片面的。

这篇文章把测速的主流方法从头捋一遍，不管你是刚买 VPS 的新手，还是换了 DMIT 新机器想跑个基准测试，都能直接用。

---

## 一、搞清楚你在测什么：三个核心维度

很多人把测速理解成"测网速"，其实不准确。

**延迟（Ping/RTT）**是数据包从你本地发出、到服务器再返回的总时间，单位毫秒（ms）。这个数字越小越好。一般国内服务器几十毫秒，日本香港 20–80ms，美国通常 150–200ms 以上。

**带宽速率**是单位时间内传输的数据量，Mbps 或 Gbps。注意：1 Gbps 的带宽 ≠ 你每秒能跑到 1 GB，因为 1 字节 = 8 比特，Mbps 除以 8 才是 MB/s。

**路由质量**是三者里最容易被忽略的。路由决定数据包走哪条路。绕路多、丢包高，即便带宽够大，用起来也卡。这就是为什么同样标称 1 Gbps，CN2 GIA 和普通国际线路体验差那么多。

---

## 二、延迟测试：三种方法，从简单到专业

### 方法一：本地 ping 命令（最简单）

在你自己电脑上打开终端，直接测：

bash
# Windows 命令提示符
ping 你的VPS_IP

# Linux / macOS
ping -c 10 你的VPS_IP


重点看两个数字：平均 RTT（越低越好）和 Loss%（丢包率，0% 最理想）。

这个方法的局限是只测了你一个人的网络到 VPS 的延迟。换个地方、换个运营商，结果可能差很多。

### 方法二：ping.pe 多节点测试（推荐）

访问 `ping.pe`，输入你的 VPS IP，它会从全球几十个节点同时 ping 你的服务器，实时显示各地延迟和丢包，结果以图表呈现。

这才是真正意义上的"多地测速"。国内用户重点看电信、联通、移动三网的延迟数字，以及有没有丢包。晚高峰（晚上 8–10 点）测出来的数据才有参考价值，白天数据往往虚高。

### 方法三：Ping Chinaz（站长之家）

`ping.chinaz.com` 可以测国内不同城市、不同运营商到你 VPS 的 ping 值，对国内建站用户特别有用。

---

## 三、带宽速率测试：跑分脚本 + iperf3

### 方法一：一键跑分脚本

SSH 登录到你的 VPS，执行下面这条命令：

bash
wget -qO- bench.sh | bash


这是圈子里用了很多年的 bench.sh 脚本，输出结果包含：磁盘 IO、内存读写速度、以及从全球多个知名数据中心节点（伦敦、法兰克福、东京、新加坡、洛杉矶等）的下载速率。跑完大概 5 分钟，结果直接打在屏幕上，一目了然。

另一个更全面的是 YABS（Yet Another Benchmark Script）：

bash
curl -sL yabs.sh | bash


YABS 用 `fio` 测磁盘、`iperf3` 测网络、`Geekbench` 测 CPU，是海外论坛用户的标配测试工具。

还有专为国内三网测速设计的 Superspeed 脚本，专门测到电信、联通、移动各三个地区的速率：

bash
bash <(curl -sL bash.icu/speedtest)


### 方法二：iperf3 点对点测速

如果你想测的是**你本地到 VPS 的真实带宽**，而不是 VPS 到各节点的速率，iperf3 是最直接的工具。

原理很简单：VPS 端开服务器，你本地开客户端，两边直接打流量跑满带宽。

**第一步：在 VPS 上安装并启动服务端**

bash
# Ubuntu / Debian
apt install iperf3 -y
iperf3 -s -p 5201


**第二步：在本地安装 iperf3 并连接测试**

bash
# 测下载速度（从 VPS 到本地）
iperf3 -c 你的VPS_IP -p 5201 -R

# 测上传速度（从本地到 VPS）
iperf3 -c 你的VPS_IP -p 5201

# 多线程并发测试（更接近真实峰值）
iperf3 -c 你的VPS_IP -p 5201 -P 8 -R


输出里的 `Bitrate` 就是实测带宽，比任何在线工具都准。

---

## 四、路由追踪：看懂数据包走了哪条路

知道延迟高，但不知道为什么高——这时候需要路由追踪。

### Windows：WinMTR

下载 WinMTR，输入 VPS IP，运行后会显示数据包经过的每个路由节点、每个节点的延迟和丢包率。哪个节点开始丢包，哪段路就有问题。

### Linux / macOS：mtr 命令

bash
mtr --report --report-cycles 20 你的VPS_IP


20 个来回跑完，打印出完整的路由路径和每跳的丢包率。

### 回程路由测试：BestTrace / AutoTrace

上面的追踪是从本地到 VPS 的去程。对于国内用户来说，**回程路由**（VPS 到国内三网）同样重要，甚至更重要。

在 VPS 上执行：

bash
curl https://raw.githubusercontent.com/ludashi2020/backtrace/main/install.sh -sSf | sh


或者更详细的 AutoTrace：

bash
wget -N --no-check-certificate https://raw.githubusercontent.com/Chennhaoo/Shell_Bash/master/AutoTrace.sh && bash AutoTrace.sh


结果里你会看到数据包经过的运营商节点名称，能直接判断走的是 CN2、CMIN2 还是普通国际线路。走 CN2 GIA 的回程会在路由里显示 `202.97.x.x` 等电信骨干网地址。

---

## 五、DMIT 各机房测速参考：测试 IP 汇总

在用上面工具测速之前，DMIT 官方提供了各机房的测试 IP，可以在购买前直接本地 ping，判断延迟是否符合预期。

| 机房 | 系列 | 测试 IP |
|---|---|---|
| 洛杉矶 LAX | EB / Pro | 154.17.226.2 |
| 洛杉矶 LAX | IPv6 | 2605:52c0:1:3:2c2a:59ff:fe05:65c2 |
| 香港 HKG | 全系 | 根据官网 Looking Glass 查询 |
| 东京 TYO | 全系 | 根据官网 Looking Glass 查询 |

实测数据参考（来自用户社区）：DMIT 洛杉矶到国内三网延迟通常在 150–180ms 之间，丢包率极低；香港机房延迟更短，通常 20–50ms，晚高峰依然稳定。

---

## 六、DMIT 套餐对比：按需选配

测完速，如果你正在考虑买 DMIT，下面是主要套餐的对比汇总（数据基于 2026 年 4 月官网信息，实际以官网为准）。

### 洛杉矶 LAX — Premium 系列（CN2 GIA 三网优化）

| 套餐 | CPU | 内存 | 硬盘 | 流量 | 带宽 | 价格 |
|---|---|---|---|---|---|---|
| TINY | 1核 | 2GB | 20GB | 1TB | 1Gbps | $88.88/年 |
| Pocket | 2核 | 2GB | 40GB | 1.5TB | 4Gbps | $159.98/年 |
| STARTER | 2核 | 2GB | 80GB | 3TB | 10Gbps | $29.90/月 |
| MINI | 4核 | 4GB | 80GB | 5TB | 10Gbps | $58.88/月 |

👉 [查看 DMIT 洛杉矶 Premium 全部套餐与最新价格](https://www.dmit.io/aff.php?aff=18446&pid=237)

### 洛杉矶 LAX — Eyeball 系列（AS9929 + CMIN2 优化，性价比首选）

| 套餐 | CPU | 内存 | 硬盘 | 流量 | 带宽 | 价格 |
|---|---|---|---|---|---|---|
| TINY | 1核 | 2GB | 20GB | 1.5TB | 2Gbps | $88.88/年 |
| Pocket | 2核 | 2GB | 40GB | 3TB | 4Gbps | $159.98/年 |
| STARTER | 2核 | 2GB | 80GB | 5TB | 10Gbps | $29.90/月 |
| MINI | 4核 | 4GB | 80GB | 10TB | 10Gbps | $58.88/月 |

👉 [以 $88.88/年 入手 DMIT LAX Eyeball TINY](https://www.dmit.io/aff.php?aff=18446&pid=245)

### 香港 HKG — Premium 系列（CN2 GIA 直连，延迟 20–50ms）

| 套餐 | 线路 | 特点 | 购买 |
|---|---|---|---|
| HKG.Pro 全系 | 电信CN2 GIA + 联通AS9929 + 移动CMI | 延迟极低，大陆访问优秀 | [👉 查看香港 Premium 套餐](https://www.dmit.io/aff.php?aff=18446) |
| HKG.EB 全系 | CMI 三网 | 中等优化，价格低于 Pro | [👉 查看香港 Eyeball 套餐](https://www.dmit.io/aff.php?aff=18446) |
| HKG.T1 全系 | 国际线路 | 无中国优化，最经济 | [👉 查看香港 Tier 1 套餐](https://www.dmit.io/aff.php?aff=18446) |

### 东京 TYO — 全系套餐

| 系列 | 线路 | 适合场景 | 购买 |
|---|---|---|---|
| TYO.Pro | CN2 GIA + AS9929 + CMI | 亚太业务首选 | [👉 查看东京 Premium 套餐](https://www.dmit.io/aff.php?aff=18446) |
| TYO.EB | CMI 三网优化 | 性价比不错，抢手易缺货 | [👉 查看东京 Eyeball 套餐](https://www.dmit.io/aff.php?aff=18446) |
| TYO.T1 | 国际 Tier 1 | $36.9/年起，适合落地中转 | [👉 查看东京 Tier 1 套餐](https://www.dmit.io/aff.php?aff=18446) |

---

## 七、当前有效优惠码汇总

使用优惠码可以在季付或年付时获得循环折扣，在结账页面的 "Apply Promo Code" 框里输入即可。

| 优惠码 | 适用范围 | 折扣 |
|---|---|---|
| `LAX-EB-LAUNCH-NON-MONTHLY-RECURRING-20OFF` | 洛杉矶 EB 系列，季付及以上 | 永久 8 折 |
| `HKG-Lite-LAUNCH-NON-MONTHLY-RECURRING-20OFF` | 香港 Lite 系列，季付及以上 | 永久 8 折 |
| `2025-TYO-T1-HI-GSL-NON-MONTHLY-30OFF` | 东京 T1 系列，季付及以上 | 永久 7 折 |
| `HKG-T1-ANNUALLY-45OFF-RECUR` | 香港 T1，年付 | 永久 5.5 折 + 规格升级 |

注意：优惠码有使用限制，建议下单前在官网验证是否仍然有效。月付套餐通常不参与折扣活动。

👉 [前往 DMIT 官网选购套餐并应用优惠码](https://www.dmit.io/aff.php?aff=18446)

---

## 八、VPS 测速常见问题解答

**Q：测速应该在什么时间跑才准？**

晚上 8–10 点的高峰期测出来的数据最有参考价值。很多 VPS 白天跑分好看，一到晚高峰就原地爆炸。如果你买的服务器主要是夜间使用，必须在这个时段验证一遍。

**Q：ping 值低，为什么网速还是感觉慢？**

延迟和带宽是两个维度。ping 值低说明响应快，但如果带宽被限速，传输大文件或跑视频依然会卡。另外丢包率高也会导致 TCP 重传，实际体验很差，而 ping 值不一定能反映丢包问题——建议同时看路由追踪里的 Loss%。

**Q：bench.sh 测出来的下载速度和我实际感受差很多，正常吗？**

正常。bench.sh 测的是 VPS 到各个测速节点的速率，不等于你本地到 VPS 的速率。影响你实际体验的是你本地网络 → 你的运营商 → 骨干网 → VPS 这整条链路的质量。想测真实体验，用 iperf3 从本地直测更准。

**Q：DMIT 的 IP 被墙了怎么办？**

DMIT 的 Premium 和 Eyeball 系列支持免费换 IP，条件是 IP 确认被墙，每 15 天可免费更换一次。其他情况每次收费 5 美元。如果选购时勾选 IP Guarantee+ 服务，购买后发现 IP 被墙可发工单免费重新分配。

**Q：T1 系列和 Premium 系列测速结果差多少？**

T1 走国际 Tier 1 线路，不做中国大陆专项优化，延迟比 Premium 高，适合落地中转或面向国际用户的业务。Premium（CN2 GIA）是 DMIT 的旗舰线路，对国内三网做了深度优化，但价格相应也高出不少。如果你的用户主要在大陆，首选 Premium 或 Eyeball。

**Q：DMIT 流量用完了会停机吗？**

不会直接停机。DMIT 的策略是流量超额后降速，T1 系列超额后限速至 50Mbps–1Gbps 不等，不中断服务，适合流量需求大但对速率要求不那么严格的场景。

---

总结一句：**VPS 测速方法**，从简单到专业，本地 ping → 多节点在线测试 → 一键跑分脚本 → iperf3 点对点 → 路由追踪，五步走完，心里有数。测速是手段，选到合适的机器才是目的。

DMIT 如果你还没测过，拿着上面的测试 IP 先 ping 一下，再决定要不要买。

👉 [查看 DMIT 所有机房套餐与当前优惠活动](https://www.dmit.io/aff.php?aff=18446)
