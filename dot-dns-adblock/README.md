# DoT DNS 广告拦截方案

> 自建 **DoT（DNS over TLS）** 加密 DNS 服务器，在域名解析层屏蔽 App 广告。
> 不改包、不越狱、不用 root。Wi-Fi 与移动数据全生效。

[![AdGuard Home](https://img.shields.io/badge/AdGuard%20Home-v0.107.71-67B279?logo=adguard)](https://github.com/AdguardTeam/AdGuardHome)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Android%20%7C%20iOS-blue)](#手机端配置)

---

## 目录

- [方案原理](#方案原理)
- [为什么必须用 DoT](#为什么必须用-dot)
- [服务端部署](#服务端部署)
- [手机端配置](#手机端配置)
- [规则集](#规则集)
- [核心技术点](#核心技术点)
- [如何定位广告域名](#如何定位广告域名)
- [实测效果](#实测效果)
- [已知局限](#已知局限)
- [注意事项](#注意事项)
- [常见问题](#常见问题)
- [免责声明](#免责声明)

---

## 方案原理

在 **DNS 解析层**把广告域名解析到 `0.0.0.0`，广告 SDK 的连接建立不起来，素材自然拉取失败。

```
手机 ──DoT(TLS 1.3 :853)──> 自建服务器 ──> 上游 DNS
                              │
                              ├─ AdGuard Home
                              │    ├─ 域名规则匹配
                              │    ├─ 本地缓存
                              │    └─ 查询日志（定位广告域名的唯一依据）
                              │
                              └─ Let's Encrypt 证书
```

整个链路加密，运营商与本地路由器都无法干预解析结果。

---

## 为什么必须用 DoT

**这是方案能跑通与否的关键。**

| 方式 | 实测结果 |
|---|---|
| 修改 Wi-Fi 静态 DNS | ❌ 手机上不生效，路由器/运营商接管了 DNS |
| 系统「私人 DNS」(DoT) | ✅ **Wi-Fi 和移动数据全生效**，运营商无法干预 |

很多教程教你改 Wi-Fi 的 DNS，但在国内多数家庭宽带（尤其是双层 NAT 环境）下会被上游覆盖，改完毫无效果。

**私人 DNS 是系统级设置，走加密通道，绕过所有中间环节。**

---

## 服务端部署

### 前置条件

| 需要 | 说明 |
|---|---|
| 公网 IP 服务器 | 24 小时运行，1 核 1G 足够 |
| 已解析的域名 | 用于 DoT，必须有有效证书 |
| 80 端口可用 | 用于 Let's Encrypt HTTP-01 验证 |

### 第一步：安装 AdGuard Home

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

安装后访问 `http://服务器IP:3000` 完成初始化向导。

### 第二步：配置文件

编辑 `/opt/AdGuardHome/AdGuardHome.yaml`：

```yaml
http:
  address: 127.0.0.1:3000        # 管理后台只绑本机，绝不暴露公网

dns:
  bind_hosts:
    - 0.0.0.0
  port: 53
  blocking_ipv4: 0.0.0.0         # 关键：不设的话规则匹配但不生效
  blocking_ipv6: "::"
  upstream_dns:
    - 223.5.5.5
    - 119.29.29.29
  protection_enabled: true
  filtering_enabled: true
  querylog_enabled: true          # 查广告域名必须开

tls:
  enabled: true
  server_name: dns.example.com    # 换成你的域名
  port_dns_over_tls: 853
  certificate_path: /opt/AdGuardHome/certs/cert.crt
  private_key_path: /opt/AdGuardHome/certs/key.pem
```

> ⚠️ `blocking_ipv4` 必须显式写成 `0.0.0.0`。默认空值时规则会加载但返回真实解析结果，看起来像"没生效"，很难排查。

### 第三步：申请证书

DoT 必须使用受信任证书，**自签证书手机会拒绝连接**。

```bash
mkdir -p /var/www/letsencrypt
certbot certonly --webroot -w /var/www/letsencrypt -d dns.example.com

mkdir -p /opt/AdGuardHome/certs
cp -L /etc/letsencrypt/live/dns.example.com/fullchain.pem /opt/AdGuardHome/certs/cert.crt
cp -L /etc/letsencrypt/live/dns.example.com/privkey.pem   /opt/AdGuardHome/certs/key.pem
chown -R adguard:adguard /opt/AdGuardHome/certs
chmod 600 /opt/AdGuardHome/certs/key.pem
```

验证私钥与证书是否匹配（两行 MD5 必须一致）：

```bash
openssl x509 -noout -modulus -in cert.crt | md5sum
openssl rsa  -noout -modulus -in key.pem  | md5sum
```

### 第四步：放行端口

```bash
ufw allow 53/tcp && ufw allow 53/udp
ufw allow 853/tcp && ufw allow 853/udp
```

> ⚠️ **云服务器还必须到控制台安全组里放行 53 和 853**，只改 ufw 不够。这是最常卡住的一步。

### 第五步：验证

```bash
# TLS 握手
openssl s_client -connect 127.0.0.1:853 -servername dns.example.com </dev/null

# 实际查询（Debian: apt install knot-dnsutils / macOS: brew install knot）
kdig +tls +tls-hostname=dns.example.com @你的域名 广告域名.com A
```

被拦截的域名应返回 `0.0.0.0`。

---

## 手机端配置

### Android

设置 → 网络和互联网 → 私人 DNS → **指定提供商主机名** → 填入 DoT 域名

### iOS

设置 → 通用 → VPN 与设备管理 → DNS → 填写 DoT 域名

> **填域名，不要填 IP。** DoT 靠 SNI 匹配证书，填 IP 会因证书域名不符而失败。

---

## 规则集

### 红果短剧

`rules/hongguo.txt`

```
||hongguoapp.com^
||hongguosdk.com^
||pangolin-sdk-toutiao.com^
||pglstatp-toutiao.com^
||ad.pangle.cn^
||ad.pangolin.cn^
||tnc.snssdk.com^
||mon.zijieapi.com^
||mcs.zijieapi.com^
||sf3-fe-tos.pglstatp-toutiao.com^
||sf6-fe-tos.pglstatp-toutiao.com^

@@||hongguo.com^
@@||hongguo.douyin.com^
@@||pstatp.com^
@@||byteimg.com^$denyallow=p3-ad-sign.byteimg.com,p3-orange-sign.byteimg.com
```

### 喜马拉雅

`rules/ximalaya.txt`

```
||adse.ximalaya.com^
||adbehavior.ximalaya.com^
||ad-incentive.ximalaya.com^
||pgdt.ugdtimg.com^
```

### 通用广告追踪

`rules/general-ads.txt`

```
||trackingprd.hwwt8.com^
||fp.hwwt8.com^
||p3-ad-sign.byteimg.com^
||scene-sdk.clickwifi.net^
||sdktt.data.comfylink.com^
```

### 订阅使用

AdGuard Home 支持直接从 URL 订阅规则：

```
https://raw.githubusercontent.com/P50091/dot-dns-adblock/main/rules/hongguo.txt
```

在 `过滤器 → DNS 黑名单 → 添加过滤器` 中填入即可。

---

## 核心技术点

### 1. 白名单优先级高于黑名单

AdGuard 中 `@@` 规则的优先级**高于** `||` 规则。

**实测踩坑**：只写 `@@||byteimg.com^`，导致广告域名 `p3-ad-sign.byteimg.com` 被一并放行，游戏推广广告照样弹出。

### 2. `$denyallow` 修正白名单

`byteimg.com` 是字节的图片 CDN，**正常内容图和广告素材混在同一父域下**：

| 子域 | 用途 |
|---|---|
| `p6-novel.byteimg.com` | 小说内容图 |
| `p9-novel-sign.byteimg.com` | 小说内容图 |
| `p3-ad-sign.byteimg.com` | **广告素材签名** |

正确写法：

```
@@||byteimg.com^$denyallow=p3-ad-sign.byteimg.com,p3-orange-sign.byteimg.com
```

语义：`byteimg.com` 全部放行，**唯独排除**这两个广告子域。

这样既保留白名单的广度（未来新子域自动放行），又精确排除了广告域。**这是本方案最有价值的技术点。**

### 3. 锚定语法

```
||domain.com^     只匹配 domain.com 及其子域，不会误伤 notdomain.com
@@||domain.com^   白名单放行
```

---

## 如何定位广告域名

**这是可复用的方法论，适用于任何 App。**

### 关键词筛选

打开 AdGuard 查询日志，访问 App 时观察请求，重点看：

```
ad / mkt / track / sign / union / ssp / dsp / banner / click / promo / incentive
```

### 实战案例

| 现象 | 抓到的域名 | 判断依据 |
|---|---|---|
| 肯德基营销弹窗 | `mkt.kfc.com.cn` | `mkt` = marketing |
| 肯德基弹窗追踪 | `trackingprd.hwwt8.com` | `tracking` |
| 传奇游戏推广 | `p3-ad-sign.byteimg.com` | `ad-sign` = 广告签名 |
| 喜马拉雅激励广告 | `ad-incentive.ximalaya.com` | `ad-incentive` |

### 标准流程

1. 打开 App，制造广告（看视频 / 开屏）
2. 到 AdGuard 查询日志筛可疑域名
3. 加规则试拦
4. **测 App 核心功能是否正常**（最怕误伤内容）
5. 白名单冲突时用 `$denyallow` 精确排除

---

## 实测效果

### 红果短剧

| 项目 | 结果 |
|---|---|
| 开屏广告 | ✅ 消失 |
| 插屏广告 | ✅ 消失 |
| 第三方营销弹窗（KFC 等） | ✅ 消失 |
| 游戏推广弹窗 | ✅ 消失 |
| 短剧封面 / 播放 / 搜索 | ✅ 正常 |

### 喜马拉雅

| 项目 | 结果 |
|---|---|
| 开屏 / 弹窗广告 | ✅ 消失 |
| 播放功能 | ✅ 正常 |

**附带效果**：启用 AdGuard 默认过滤器后，还顺带拦截了腾讯优量汇、快手联盟、OPPO 广告等通用广告域（`v3.gdt.qq.com`、`open.e.kuaishou.com`、`adx.ads.heytapmobi.com`）。

---

## 已知局限

| 限制 | 说明 |
|---|---|
| **HTTPDNS 绕不过** | 部分 App（如字节系）用 HTTPDNS，`*.zijieapi.com` 返回 CNAME 直达 IP，不走常规解析 |
| **同源广告拦不住** | 音频 App 插播的广告与内容走同一 CDN |
| **服务端下发的数据拦不住** | 若广告内容已包含在 API 响应中，DNS 层无能为力 |
| **只作用于解析层** | 无法修改 App 内部逻辑，广告位可能变成空白而非完全消失 |
| **证书需定期续期** | Let's Encrypt 90 天，建议配 DNS-01 自动化 |

---

## 注意事项

1. **不要整域拦截** `qq.com`、`baidu.com` 这类大域，会连带瘫痪正常功能
2. **每加一条规则都要测 App 核心功能**，防止误伤内容
3. **管理后台只绑 `127.0.0.1`**，通过 SSH 隧道访问，绝不暴露公网
4. **务必开启查询日志**，这是定位广告域名的唯一依据
5. **云服务器安全组必须放行 53/853**，只改 ufw 不够
6. **证书续期要提前处理**，否则 90 天后 DoT 会突然失效

---

## 常见问题

### Q：手机填了私人 DNS 后显示"无法连接"

按顺序排查：

1. 云安全组是否放行 **853**
2. 证书是否正确加载（`openssl s_client` 测试）
3. 填的是**域名**还是 IP（必须填域名）
4. 域名解析是否指向服务器

### Q：规则加了但广告还在

查看查询日志中该域名是否命中：

- `FilteredBlackList` → 已拦截，广告可能来自别的域名
- `FilteredWhiteList` → **被白名单覆盖了**，需要用 `$denyallow` 排除
- `NotFilteredNotFound` → 规则未匹配，检查语法或缓存

### Q：改完 Wi-Fi 的 DNS 完全没反应

说明路由器或运营商接管了 DNS。**放弃改 Wi-Fi，直接用私人 DNS（DoT）。**

### Q：服务器自身无法解析域名了

AdGuard Home 接管了 53 端口，如果它停止运行，服务器会失去 DNS 解析能力。

**保留一个备用：**

```bash
echo "nameserver 223.5.5.5" | sudo tee /etc/resolv.conf
```

---

## 免责声明

本仓库仅用于**个人学习与技术研究**。

拦截广告可能违反部分 App 的用户协议，并影响内容提供方的收入。**请仅在自己拥有合法使用权的设备上操作，不要分发修改后的客户端。**

本方案不修改任何 App 的代码，不提供任何破解产物，仅描述通用的 DNS 过滤技术。

使用者应自行承担因使用本方案产生的全部风险。

---

## License

[MIT](./LICENSE)