**本代码 fork 自 [moreoronce/MosDNS-Config](https://github.com/moreoronce/MosDNS-Config) 在此感谢原作者。**

相比原代码，本代码适合 AdGuardHome -> MosDNS -> OpenClash(fake-ip) 的网络结构，具体配置请参考 [【分享】【教程】主路由中AdGuardHome+MosDNS+Open克拉什折腾记录](https://www.right.com.cn/forum/thread-8355510-1-1.html)   
本代码修改和使用的初衷是为了代替原 luci-app-mosdns `cn`走国内、`geolocation-!cn`走国外的简单粗暴方法（绕过大陆模式）。本代码的 DNS 查询相当于是 GFW 黑名单模式（PAC 模式），因此OpenClash 中的漏网之鱼都可以使用代理进行连接，减少一些本可以直连却使用代理的情况。对于本人这种机场流量拮据的来说合适不过了。但是缺点也很明显，访问国外的域名就必须走代理进行DNS查询得到结果后才能发起连接，当然 AdGuardHome 的缓存可以解决这一点；另外，就像 GFW 黑名单模式的缺点一样，如果 GFW 没有包含不可访问的域名，并且兜底也没进代理，客户端就只能直连而没法访问网站了，只能手动设置灰名单，强制这些域名进入代理。

代码做出修改如下：
- 注释缓存与去广告代码段，这部分功能将由 AdGuardHome 完成。
- 适配原 luci-app-mosdns 的自定义规则列表：白名单、黑名单、灰名单、DDNS 域名、Hosts、重定向、PTR 黑名单。（流媒体将由 OpenClash 处理）
- 对于 `geosite_location-!cn` 的域名查询做出修改如下：通过 OpenClash 代理访问国外 DNS 查询非大陆域名，`always_standby` 设置为 `false` 本意是为了在 OpenClash 代理的国外 DNS 无法返回结果时再使用国内 DNS 查询，等待时间为 5s，如果返回的都为国外 IP，则返回客户端，让客户端尝试直连。（GFW黑名单模式）
```yaml
  - tag: dns_nocn
    type: fallback
    args:
      primary: forward_remote
      secondary: forward_local
      threshold: 5000
      always_standby: false
```
- 兜底的漏网之鱼从原版的走国内查询变为走 OpenClash 代理。
- 对国内和国外的DNS进行合并，因为国内的腾讯和阿里都对 DOT 和 DOH 进行了限速，避免因为 MosDNS 查询过多而速度限制。

# 自用MosDNS配置

- 支持ECS
- 支持GEOIP
- 支持GEOSITE
- 支持自定义灰名单及白名单
- 支持广告过滤
- 支持数据导入Grafana
- 本层级DNS处理无泄漏

# 使用方法

配置文件共计**3**个，分别为`config_custom.yaml`, `dns.yaml`, `dat_exec.yaml` 。

各部分作用如下：

- `config_custom.yaml`: 主配置文件，负责DNS序列定义以及DNS序列执行。需要依赖`dns.yaml`和`dat_exec.yaml`运行。
- `dns.yaml`: dns定义配置文件，负责配置公共DNS服务器及远端解析DNS地址及端口。
- `dat_exec.yaml`: 规则配置文件，负责定义各规则tag及规则来源文件。

下载或克隆三个yaml文件，OpenWRT放到`/etc/mosdns`文件夹内。如果是luci-app-mosdns，需要选择使用自定义配置文件。其他系统可以通过`-c` 参数指定配置文件为`config_custom.yaml` 。

默认GeoSite和GeoIP的存放位置为`/var/mosdns/` ,请确保文件夹下含有`geoip_cn.txt`、`geosite_category-ads-all.txt`、`geosite_geolocation-!cn.txt`、`geosite_gfw.txt`、`geosite_cn.txt`以及`geoip_private.txt` ，OpenWRT用户可以通过luci-app-mosdns的GeoData Export功能自动下载解码生成。
同时，在/etc/mosdns/下需要建立rule文件夹，并新建whitelist.txt和greylist.txt文件，用于自定义白名单和污染域名名单。DDNS类域名可放到白名单中。

Openwrt 用户需要在 luci-app-mosdns 界面的 基本设置 -> GeoData导出 中设置：
  1. GeoSite 标签：`cn`、`gfw`、`geolocation-!cn`
  2. GeoIP 标签：`cn`

# DNS处理流程：

![image](https://github.com/user-attachments/assets/8b56d92c-c5ec-48dc-8b41-650324f46fad)


根据 [Jasper-1024/mosdns_docker](https://github.com/Jasper-1024/mosdns_docker/tree/master/mosdns_v5)  进行二次修改，在此基础上增加GFW域名远程解析规则，修改并发请求DNS连接数

教程及DNS处理队列详解：[自用MosDNS规则分享](https://deeprouter.org/article/mosdns-config-with-no-leak)
