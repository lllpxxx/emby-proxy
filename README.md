# emby-proxy（go-emby2openlist 定制构建）

`lllpx579/emby-proxy` 是 [go-emby2openlist](https://github.com/ambitiousjun/go-emby2openlist) v2.9.0 的个人定制镜像。在上游基础上修掉两个实际踩到的坑，其余行为与上游一致；完整功能文档见 [README.upstream.md](README.upstream.md)。

## 与上游的差异

**1. 本地实体媒体不再被误判为必须转码** — `internal/service/emby/playbackinfo.go`

ge2o 处理 `PlaybackInfo` 时会把请求体换成自己的通用 payload 再回源，Emby 因此拿不到客户端的 `DeviceProfile`，直连本地文件也会被判成必须 HLS 转码。补丁在识别出「所有媒体源都是本地实体文件」时，用客户端**原始请求体重放一次**并原样返回，缓存 12 小时。

**2. 反向代理端口与 `emby.host` 不一致时，不再错误 307 直连** — `internal/service/emby/redirect.go`

上游只用 hostname 判断是否「同一台服务器」。客户端经 `https://域名:25121` 进来、而 `emby.host` 写的是 `10.0.0.2:8097` 时，hostname 相同但端口不同，上游仍会 307 把客户端甩去它根本连不上的端口。补丁改成严格比较 `host:port`（带 scheme 推导与默认端口补全），不一致就走代理回源。

**3. 构建提速** — Dockerfile 增加 `NPM_REGISTRY` / `GOPROXY` 构建参数，默认值面向国内网络，Actions 构建时覆盖为官方源。

## 镜像

```bash
docker pull lllpx579/emby-proxy:latest
```

标签：`latest`、`v2.9.0-local`、`sha-*`。

## 运行

```bash
docker run -d \
  --name emby-proxy \
  --restart unless-stopped \
  -p 8096:8095 \
  -p 8094:8094 \
  -v /mnt/user/appdata/go-emby2openlist:/app \
  -e TZ=Asia/Shanghai \
  lllpx579/emby-proxy:latest
```

容器内监听 8095（Emby 代理入口）和 8094。

## config.yml

挂载目录里的 `config.yml` 是唯一配置入口：

```yaml
emby:
  host: http://10.0.0.2:8097          # Emby 源站，必须带端口
  mount-path: /media
  proxy-error-strategy: origin
  local-media-roots:                  # 本地实体目录：命中后回源，不走网盘
    - /media/PT整理

openlist:
  host: http://10.0.0.2:5678
  token: "换成你自己的 OpenList token"

path:
  map:
    - /media/xiaoya整理:/电视剧        # 本地路径:OpenList 路径

cache:
  enable: true
  expired: 1d
```

`local-media-roots` 下的文件会有 `本地媒体: ... 代理回源` 日志；没列进去的目录会被当成网盘资源，连本地实体文件也会去 OpenList 轮询。

## 从源码构建

```bash
# 国内网络
docker build \
  --build-arg NPM_REGISTRY=https://registry.npmmirror.com \
  --build-arg GOPROXY=https://goproxy.cn \
  -t emby-proxy .

# 或直接让 CI 构建：推 main 分支 / 打 tag 即触发
```

CI 需要仓库 Secrets：`DOCKERHUB_USERNAME`、`DOCKERHUB_TOKEN`（Docker Hub → Account Settings → Personal access tokens，权限 Read & Write）。

## 许可

AGPL-3.0，见 [LICENSE](LICENSE)。本项目是上游的衍生作品，分发与二次修改请同样遵守上游协议并保留来源声明。

---

Personal fork of [ambitiousjun/go-emby2openlist](https://github.com/ambitiousjun/go-emby2openlist) v2.9.0. Two patches: original `PlaybackInfo` request body is replayed for local media to avoid false transcoding decisions, and the 307 direct-play check now compares `host:port` instead of hostname only, so reverse-proxied clients on a different port fall back to proxying instead of being redirected to an unreachable port. Licensed AGPL-3.0.
