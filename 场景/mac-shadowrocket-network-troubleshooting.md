# Mac + Shadowrocket 断网排查文档

适用场景：关闭 Shadowrocket 后，微信、百度、浏览器、终端、git、npm 等出现没网、外网打不开、国内 App 异常。

## 0. 先判断影响范围

先不要乱改设置，先判断是哪一类问题。

| 现象 | 优先检查 |
| --- | --- |
| 微信、百度、浏览器都没网 | macOS 系统代理 / VPN / DNS |
| 只有终端、git、npm 没网 | `~/.proxyrc`、环境变量、git/npm 代理 |
| 国内能上，外网不能上 | Shadowrocket 是否开启、节点是否可用、规则是否正确 |
| 外网能上，微信/百度异常 | Shadowrocket 规则里国内/腾讯是否走 `DIRECT` |
| 关掉 Shadowrocket 后全系统没网 | 系统代理还指向 `127.0.0.1:端口` |

## 1. 核心原理

Shadowrocket 开启系统代理时，通常会这样工作：

```text
微信/浏览器/系统 App
  -> macOS 系统代理
  -> 127.0.0.1:7897
  -> Shadowrocket
  -> 网络
```

如果 Shadowrocket 关闭了，但 macOS 系统代理没有恢复，就会变成：

```text
微信/浏览器/系统 App
  -> 127.0.0.1:7897
  -> 没有程序监听
  -> 断网
```

所以这类问题不是 Wi-Fi 坏了，而是系统还在把流量送到已经关闭的本地代理端口。

## 2. 查看网络服务名称

先看你的 Mac 网络服务叫什么。通常是 `Wi-Fi`。

```bash
networksetup -listallnetworkservices
```

如果你的服务不是 `Wi-Fi`，后面的命令要把 `Wi-Fi` 替换成实际名字。

## 3. 检查 macOS 系统代理

查看全局代理状态：

```bash
scutil --proxy
```

查看 Wi-Fi 上的代理状态：

```bash
networksetup -getwebproxy Wi-Fi
networksetup -getsecurewebproxy Wi-Fi
networksetup -getsocksfirewallproxy Wi-Fi
networksetup -getautoproxyurl Wi-Fi
networksetup -getproxyautodiscovery Wi-Fi
```

如果看到类似下面的内容，说明系统代理还开着：

```text
Enabled: Yes
Server: 127.0.0.1
Port: 7897
```

这时要继续检查 Shadowrocket 的本地端口是否还活着。

## 4. 检查本地代理端口是否有人监听

假设端口是 `7897`：

```bash
lsof -nP -iTCP:7897 -sTCP:LISTEN
```

结果判断：

| 结果 | 含义 |
| --- | --- |
| 有输出，进程是 Shadowrocket | 本地代理还活着 |
| 没有输出 | 系统代理指向了一个不存在的端口，必然断网 |
| 有输出，但不是 Shadowrocket | 可能是别的代理软件占用了端口 |

## 5. 一键恢复系统直连

如果微信、百度、浏览器都没网，先执行这一组：

```bash
networksetup -setwebproxystate Wi-Fi off
networksetup -setsecurewebproxystate Wi-Fi off
networksetup -setsocksfirewallproxystate Wi-Fi off
networksetup -setautoproxystate Wi-Fi off
networksetup -setproxyautodiscovery Wi-Fi off
```

然后断开并重新连接 Wi-Fi。

也可以用图形界面操作：

```text
系统设置
  -> 网络
  -> Wi-Fi
  -> 详细信息
  -> 代理
```

把下面这些全部关掉：

- 网页代理 HTTP
- 安全网页代理 HTTPS
- SOCKS 代理
- 自动代理配置
- 自动代理发现

## 6. 检查 VPN / 过滤器

如果系统代理关了还不行，检查：

```text
系统设置
  -> 网络
  -> VPN 与过滤器
```

如果看到 Shadowrocket、VPN、Filter、Proxy 相关项目：

- 先断开
- 不需要的可以删除
- 然后重新连接 Wi-Fi

## 7. 检查 DNS

如果代理关了，但仍然打不开网站，可能是 DNS 异常。

查看 DNS：

```bash
scutil --dns
```

图形界面恢复 DNS：

```text
系统设置
  -> 网络
  -> Wi-Fi
  -> 详细信息
  -> DNS
```

建议先改成自动，或者临时用常见 DNS：

```text
223.5.5.5
119.29.29.29
8.8.8.8
1.1.1.1
```

国内网络优先试 `223.5.5.5` 或 `119.29.29.29`。

## 8. 检查终端 / git / npm 代理

如果只有终端、git、npm 没网，检查这些：

```bash
env | grep -i proxy
git config --global --list | grep proxy
npm config get proxy
npm config get https-proxy
```

清理终端当前会话代理：

```bash
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
```

清理 git 代理：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
git config --global --unset "http.https://github.com.proxy"
git config --global --unset "https.https://github.com.proxy"
```

清理 npm 代理：

```bash
npm config delete proxy
npm config delete https-proxy
```

如果你有 `~/.proxyrc`，也可以执行你自己的：

```bash
proxy-all-off
```

## 9. 你想要的长期正确配置

目标：

```text
国外网站 / GitHub / Google / ChatGPT -> PROXY
微信 / 百度 / 国内网站 / 局域网      -> DIRECT
```

Shadowrocket 不要使用全局模式，使用规则模式。

规则思路：

```text
微信、腾讯域名 -> DIRECT
中国大陆 IP/域名 -> DIRECT
最后兜底 -> PROXY
```

常见规则示例：

```ini
DOMAIN-SUFFIX,wechat.com,DIRECT
DOMAIN-SUFFIX,weixin.qq.com,DIRECT
DOMAIN-SUFFIX,qq.com,DIRECT
DOMAIN-SUFFIX,tencent.com,DIRECT
DOMAIN-SUFFIX,gtimg.com,DIRECT
DOMAIN-SUFFIX,qpic.cn,DIRECT
DOMAIN-SUFFIX,wechatpay.cn,DIRECT
DOMAIN-SUFFIX,tenpay.com,DIRECT
GEOIP,CN,DIRECT
FINAL,PROXY
```

规则顺序很重要：

```text
DIRECT 规则必须放在 FINAL,PROXY / MATCH,PROXY 前面。
```

## 10. 下次断网时的最短流程

按顺序执行：

```bash
scutil --proxy
networksetup -getwebproxy Wi-Fi
networksetup -getsecurewebproxy Wi-Fi
networksetup -getsocksfirewallproxy Wi-Fi
```

如果看到 `127.0.0.1:7897` 且 Shadowrocket 已关闭：

```bash
networksetup -setwebproxystate Wi-Fi off
networksetup -setsecurewebproxystate Wi-Fi off
networksetup -setsocksfirewallproxystate Wi-Fi off
networksetup -setautoproxystate Wi-Fi off
networksetup -setproxyautodiscovery Wi-Fi off
```

然后重新连接 Wi-Fi。

如果只有终端/git/npm 异常：

```bash
env | grep -i proxy
git config --global --list | grep proxy
npm config get proxy
npm config get https-proxy
```

再按需清理终端、git、npm 的代理。

## 11. 记忆口诀

```text
全系统没网，先查系统代理。
只有终端没网，查环境变量。
git/npm 没网，查 git/npm 代理。
Shadowrocket 关后没网，查 127.0.0.1 端口是否残留。
想外网代理国内直连，用规则模式，不用全局模式。
```


$$
\zeta(s) =
\frac{1}{1-2^{1-s}}
\sum_{n=1}^{\infty}
\frac{(-1)^{n-1}}{n^s}
$$

$$
\begin{aligned}
\mathcal{L}_{\mathrm{SM}} =\;&
-\frac{1}{4} G_{\mu\nu}^a G^{a\mu\nu}
-\frac{1}{4} W_{\mu\nu}^i W^{i\mu\nu}
-\frac{1}{4} B_{\mu\nu} B^{\mu\nu} \\
&+ \sum_{\text{fermions}} \bar{\psi}_i
i\gamma^\mu D_\mu \psi_i
\end{aligned}
$$



