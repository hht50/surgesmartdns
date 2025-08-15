#  DNS enhanced (智能 DNS 增强配置)

本项目是一个为 [Surge](https://nssurge.com/) 等网络工具设计的智能 DNS 增强配置文件。其核心目标是通过精细化的域名分流策略，为不同的网络服务选择最高效、最准确的 DNS 服务器，从而优化网络请求速度、提高解析成功率并增强网络体验。

## 核心理念

本配置遵循三大原则：

1.  **域名按归属分流**：将不同国家或地区的互联网服务（如 Google, Apple, 阿里巴巴, 腾讯）的域名解析请求，分别发送至其各自或指定的 DNS 服务器。这确保了能够获取到针对用户所在区域最优化的 CDN 节点，大幅提升访问速度。
2.  **局域网地址本地解析**：所有路由器后台管理页面（如 `router.asus.com`, `tplogin.cn`）以及其他局域网设备的域名，均通过系统默认的 DNS (`syslib`) 进行解析。这保证了内网设备的访问永远正确。
3.  **特殊地址规范解析**：对一些特定的服务（如 Google API, Apple 服务）采用其官方推荐的 DNS 服务进行解析，保证服务的稳定性和可用性。

## 核心优势与升级

与基础配置相比，当前版本进行了以下关键升级，大幅提升了健壮性与安全性：

-   **DNSSEC 安全验证**：已在配置中启用 `dns-sec-validation = true`。DNSSEC (DNS Security Extensions) 能够对 DNS 响应进行加密验证，确保您收到的解析结果是真实、完整且未经篡改的，有效抵御 DNS 欺骗和缓存投毒等网络攻击。

-   **备用 DNS 与故障回退 (Fallback)**：为所有核心域名解析规则添加了备用 DNS 服务器和系统默认 DNS (`syslib`) 作为最终防线。这意味着，当首选的加密 DNS 服务器因网络问题暂时无法访问时，系统会自动切换到备用服务器，甚至最终回退到系统 DNS 进行解析，极大地提高了 DNS 解析的成功率和稳定性。

## 主要功能与实现细节

### 1. DNS 服务器预解析

在 `[Host]` 部分，配置文件预先将所有将要用到的加密 DNS 服务器域名（如 `dns.google`, `dns.alidns.com`）指向其固定的 IP 地址。这是一个关键的“自举”设计，避免了在解析一个域名时，又需要去请求另一个域名而产生的“鸡生蛋，蛋生鸡”的循环解析问题。

### 2. 精细化的域名分流规则

这是本配置文件的核心。通过 `域名 = server:DNS服务器` 的语法，实现了对海量域名的精细化管理。

-   **中国大陆服务**：
    -   **阿里巴巴、蚂蚁集团、淘宝、阿里云**等：使用 `quic://dns.alidns.com` (阿里公共 DNS) 解析，速度快、无污染。
    -   **腾讯、QQ、微信、腾讯云**等：使用 `https://doh.pub/dns-query` (DNSPod 公共 DNS) 解析，稳定可靠。
    -   **百度、百度云、爱奇艺**等：使用 `180.76.76.76` (百度公共 DNS) 解析。
    -   **字节跳动、抖音、今日头条**等：使用 `180.184.1.1` (火山引擎公共 DNS) 解析。
    -   **BiliBili**：根据其视频服务器所使用的不同 CDN 服务商（阿里云、腾讯云、百度云），智能地将解析请求分发给对应的 DNS 服务器，实现最优的视频加载速度。
    -   **政府 (`.gov.cn`) 和教育 (`.edu.cn`)**：使用 `1.2.4.8` (CNNIC SDNS) 进行权威解析。

-   **国际与地区服务**：
    -   **Apple 服务**：如 `icloud.com`，使用苹果官方的加密 DNS `https://doh.dns.apple.com/dns-query` 解析，保障服务稳定。
    -   **路由器管理**：将全球主流路由器品牌（华硕、网件、Linksys、小米、华为等）的管理域名全部指向 `server:syslib`，交由系统处理，确保内网访问无误。
    -   **区域化服务**：
        -   **全球通用**：对于日本、韩国、新加坡、欧洲、英国、俄罗斯等国家和地区的网站，现在采用了一套统一且强大的 DNS 解析方案。系统会并发地向 `dns.google`, `cloudflare-dns.com`, `dns.quad9.net`, `dns.adguard.com` 等多个全球顶级的公共 DNS 服务商发起查询，并采用最快返回的结果。这种策略极大地提升了国外网站的解析速度和成功率。

### 3. 加密 DNS (Encrypted DNS)

本配置大量采用了 `DoH (DNS over HTTPS)`、`DoQ (DNS over QUIC)` 等现代加密 DNS 协议。这可以有效防止 DNS 污染和劫持，保护用户的隐私和安全。

## 基础配置详解

这部分是整个 DNS 配置文件的基石，它定义了全局 DNS 行为和最基础的域名解析规则。

### `[General]` 部分：全局 DNS 设置

这部分定义了 Surge 的全局 DNS 行为。

-   `dns-sec-validation = true`
    *   **含义**：开启 DNSSEC (DNS Security Extensions) 验证。
    *   **作用**：这是一个安全特性。开启后，Surge 会对收到的 DNS 响应进行加密签名验证，确保解析结果来自权威的 DNS 服务器且在传输过程中未被篡改。这能有效抵御 DNS 欺骗和缓存投毒攻击。

-   `use-local-host-item-for-proxy = true`
    *   **含义**：允许代理请求使用本地的 DNS 映射结果。
    *   **作用**：当一个网络请求通过代理（Proxy）发出时，这条规则允许 Surge 直接使用 `[Host]` 部分定义的 IP 地址，而不需要再次通过代理服务器去查询 DNS。这对于确保加密 DNS 服务器（如 `dns.google`）自身的解析能被正确引导至预设 IP 地址至关重要。

-   `encrypted-dns-follow-outbound-mode = true`
    *   **含义**：加密 DNS 请求将遵循全局出站模式（Outbound Mode）。
    *   **作用**：这是针对中国大陆网络环境的关键优化。当设置为 `true` 时，发往 `dns.google`、`cloudflare-dns.com` 等国外加密 DNS 服务器的请求，将能够通过您配置的代理服务器（Proxy）进行连接。这解决了这些 DNS 服务因在大陆地区被屏蔽而无法直接访问的问题，确保了国外网站的解析功能可以正常使用。

### `[Host]` 部分：本地 DNS 映射

这部分相当于一个本地的 `hosts` 文件，用于定义域名到 IP 地址的静态映射，或者指定特定的 DNS 服务器。

-   **IPv6 静态定义**
    *   `ip6-localhost = ::1` 等规则为标准的 IPv6 地址（如本地回环、链路本地等）提供了固定的映射，确保了 IPv6 环境下的基础网络通信正常。

-   **加密 DNS 服务器预解析 (Bootstrap)**
    *   `dns.google = 8.8.8.8...`，`cloudflare-dns.com = 104.16.249.249...` 等一系列规则是整个配置文件的核心“自举”机制。
    *   **作用**：在后续的分流规则中，我们大量使用了 `server:https://dns.google/dns-query` 这样的域名形式来指定 DNS 服务器。但要访问 `dns.google` 这个域名，本身就需要一次 DNS 解析。为了避免“鸡生蛋、蛋生鸡”的死循环，我们在这里预先将这些 DNS 服务器的域名直接指向其固定的 IP 地址。这样，当 Surge 需要使用 `dns.google` 时，它会直接使用这里提供的 IP，而无需再次发起 DNS 请求。

-   **特定服务 IP 修改 (Modify Contents)**
    *   `talk.google.com = 108.177.125.188` 等规则，是针对特定服务（这里主要是 Google 的消息推送服务）进行的 IP 地址锁定。这可能是为了连接到特定区域的服务器或使用一个已知的、更稳定的服务 IP。

-   **特定服务 DNS 分流 (CUSTOM DNS)**
    *   `blog.google = server:119.29.29.29` 这类规则，是将特定域名的解析权交给指定的 DNS 服务器。例如，这里将 Google 博客的解析交给了 DNSPod 的 DNS。
    *   `stun.l.google.com = server:syslib` 中的 `syslib` 是一个特殊指令，意为“使用系统默认的 DNS 解析”。这通常用于解析局域网设备或需要通过运营商 DNS 才能正确解析的服务。

## 如何使用

1.  将本 `dns-enhanced.conf` 文件的内容添加或 `include` 到您的 Surge 配置文件中的 `[General]` 和 `[Host]` 等相应部分。

2.  **（关键步骤）** 在您的 **主配置文件** 的 `[Rule]` 部分，添加一条规则，用于将国外的加密 DNS 服务器域名指向您的代理策略。请将这条规则放置在其他分流规则的靠前位置：

    ```ini
    # [Rule]
    # ... 其他规则 ...

    # >> 国外加密 DNS 服务 -> 走代理
    # 匹配所有在 [Host] 部分预解析的国外 DoH/DoQ 服务器域名
    DOMAIN-KEYWORD,dns.google,PROXY
    DOMAIN-KEYWORD,cloudflare-dns,PROXY
    DOMAIN-KEYWORD,dns.quad9,PROXY
    DOMAIN-KEYWORD,dns.adguard,PROXY
    DOMAIN-KEYWORD,dns1.top,PROXY

    # ... 其他规则 ...
    # FINAL,DIRECT
    ```

    **说明**：
    *   请将上述规则中的 `PROXY` 替换为您自己的代理策略组名称（例如 `✈️ Proxy`, `MyProxyGroup` 等）。
    *   这一步是必需的。因为它确保了当 Surge 尝试连接这些国外的加密 DNS 服务器时，会通过代理进行连接，从而解决了它们在大陆地区无法直接访问的问题。
