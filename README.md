# nezha-build

给面板重新构建 [Nezha](https://github.com/nezhahq/nezha) 的 dashboard 二进制，**只换国家库，不魔改任何代码**。

## 为什么需要它

面板上的国旗不是探针上报的，是面板拿探针报的 IP 去查一份**编译进二进制**的国家库（`go:embed`）。
官方 v2.2.10 是 2026-07 构建的，库就冻在那天，之后 ipinfo/MaxMind 改了归属，面板不会知道。

官方地址 `nezhahq/nezha` 的 release 资产我们改不了，所以自己出一份同样版本、只换库的二进制。

## 这个构建做了什么

1. clone 官方 `nezhahq/nezha` 的 `v2.2.10`（**代码一行没改**）
2. 换掉 `pkg/geoip/geoip.db`：用 [P3TERX 镜像的 GeoLite2-Country](https://github.com/P3TERX/GeoLite.mmdb)（每日更新，不需要 token）
3. 替换 `pkg/geoip/geoip.go`：让它同时能读 MaxMind 的嵌套结构（上游只认 ipinfo 的扁平结构），并支持一个可选的 `GEOIP_OVERRIDES` 覆盖表
4. `script/fetch-frontends.sh` + `swag init` + `go build`（CGO 关闭，静态二进制）
5. 把 `dashboard-linux-amd64.zip` 发到本仓库 release `v2.2.10`

构建时会跑一次 `go test ./pkg/geoip`，把几个固定 IP 的查询结果打进日志（真实输出，不是推断）。

## 怎么用

面板容器每次启动都会下载这个 zip：

```
https://github.com/aoi675145/nezha-build/releases/download/v2.2.10/dashboard-linux-amd64.zip
```

**刷新国家库**：Actions → `Build dashboard (fresh GeoIP)` → Run workflow → 完成后再重启面板实例。

**升级 Nezha 版本**：把 `.github/workflows/build.yml` 里的 `DASHBOARD_VERSION` 改成新 tag，重跑；release 用了同一个 tag，下载地址里的版本号也要跟着改（在面板镜像的 `/app/start.sh` 里）。

## GEOIP_OVERRIDES（可选）

某个网段的归属明显不对、又等不到上游修的时候，给面板容器加环境变量即可，不用重建：

```
GEOIP_OVERRIDES=13.140.248.0/24=JP,2.2.2.0/24=US
```

优先级：覆盖表 > 数据库。格式不对的条目会被忽略（不会让面板起不来）。

## 排查

| 现象 | 怎么办 |
|---|---|
| 面板起不来 | 看实例日志 `3/6 下载 dashboard`：sha256 校验失败会直接退出；通常是 release 资产正在被覆盖，重跑构建 |
| 国旗没变 | 先确认实例日志下载成功；再 `curl -s https://ipinfo.io/<ip>/json` 看上游数据到底改没改 |
| 构建失败 | 多半是 `swag` / `yq` 下载问题，重跑一次 |
