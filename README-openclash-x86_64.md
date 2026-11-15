# 只带 OpenClash 的 x86_64 软路由固件（Lean's LEDE + GitHub Actions）

**只用一个文件**：把 `build-openclash-x86_64.yml` 放到仓库的
`.github/workflows/` 下即可，配置、旁路脚本全部内嵌在 workflow 里，不再需要
`configs/`、`files/` 等其它文件。

## 固件内容

- LuCI + 中文 + design 主题 + ttyd 网页终端
- **OpenClash**（上游 <https://github.com/vernesong/OpenClash>，当前 0.48.002）+ ruby 运行时
- dnsmasq-full(nftset)、kmod-nft-tproxy、kmod-tun、kmod-inet-diag
- 不含 ssr-plus、passwall、turboacc、UPnP、DDNS、Docker、Samba 等

默认网络（首次开机自动写入）：

- LAN `192.168.1.2/24`，网关 / DNS `192.168.1.1`，**DHCP 已关闭**
- 所有物理网口桥接进 `br-lan`，插哪个口都能进后台；WAN 关闭
- 后台 <http://192.168.1.2>，`root` / `password`（**请第一时间修改**）
- 同时产出 BIOS（`combined.img.gz`）和 UEFI（`combined-efi.img.gz`）

## 安装

1. 仓库 → `.github/workflows/` → **Add file → Create new file**
   文件名填 `build-openclash-x86_64.yml`，把本目录同名文件的内容粘进去并提交。
2. 如果仓库里还有旧的 `uild-openclash-x86_64.yml`（少了一个 b）或
   `openclash-x86_64.tar.gz`，删掉，避免混淆。
3. 仓库 **Actions** 页面首次使用需点一下启用（fork 默认禁用 workflow）。
4. 手动 **Run workflow**（可填 `openclash_ref` 固定 OpenClash 分支/标签，默认 `master`）；
   或推送 `v*` 标签自动编译并发 Release。
5. 编译约 **2～3 小时**（4 核 Runner）。产物在 Actions → Artifacts：
   `openclash-x86_64-firmware`（镜像）、`openclash-x86_64-packages`（ipk）。

## 刷机 & 旁路拓扑

```
        光猫
         │
      主路由 192.168.1.1（拨号 / DHCP / 无线）
         │ LAN 口
      交换机 ──── 电脑 / AP
         │
   旁路设备 192.168.1.2（本固件，关 DHCP）
```

1. `.img.gz` 解压得到 `.img`，用 Rufus(DD 模式) / balenaEtcher / `dd` 写盘；
   BIOS 用 `*-combined.img.gz`，UEFI 用 `*-combined-efi.img.gz`。
2. 旁路设备**任意一个**网口接主路由 LAN（只接一根线，两个口接同一台交换机会环路）。
3. 电脑从主路由拿 `192.168.1.x`，打开 <http://192.168.1.2> 登录。
4. 主路由 DHCP 的**网关和 DNS 都指向 192.168.1.2**；或给个别设备手动指定。
5. 主路由网段不是 192.168.1.0/24 时：改 workflow 里 `LAN_IP` / `LAN_GW`
   （job 的 `env`）后重编，或进后台在 LuCI 里改 LAN 和网关。

**不要关防火墙**：OpenClash 的 nft 规则挂在 `inet fw4` 表上，关掉代理就失效。

## 配置 OpenClash

「服务 → OpenClash → 配置文件」填订阅 → 更新 → 「插件设置 → 版本更新」下载
Clash Meta 内核（国内可能慢，可手动放到 `/etc/openclash/core/clash_meta` 并
`chmod +x`）→ 回「运行状态」启用。旁路推荐 Fake-IP 模式，DNS 保持默认。

## 说明 / 排错

- OpenClash 通过 feeds 引入上游 <https://github.com/vernesong/OpenClash>：
  workflow 会把 `src-git openclash https://github.com/vernesong/OpenClash.git;master`
  插到 `feeds.conf.default` 的**最前面**（feeds 顺序决定同名包谁生效），并删掉
  其它 feed 里可能残留的同名链接，保证编进去的是上游源码而不是 Lean 的镜像副本。
- OpenClash 打包需要 luci-base 的 host 工具 `po2lmo`，workflow 会给它的
  `DEPENDS` 补上 `luci-base` 以保证编译顺序。
- ruby 的 YJIT/ZJIT 会拖入 `rust/host`（上游标记 BROKEN），已显式关闭。
- 编译失败先看 `build-log` artifact 的最后几十行；`make download` 的网络抖动
  直接重跑即可。
