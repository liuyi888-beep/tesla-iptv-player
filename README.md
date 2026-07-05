# Tesla IPTV Player

特斯拉车机专用 IPTV 播放器，支持行驶期间乘客观看。频道来自 `index.m3u`。

支持的节目源：

```text
http://
https://
rtsp://
rtmp://
```

## 文件清单

```text
tesla-iptv.tar       Docker 镜像
docker-compose.yml   Docker Compose 启动文件
index.m3u            示例播放列表，后续维护频道只改这个文件
README.md            说明文档
```

镜像导入后的标签是：

```text
tesla-iptv-player:local
```

## 环境要求

- Docker
- Docker Compose v2

检查命令：

```bash
docker -v
docker compose version
```

如果系统只支持旧版 `docker-compose` 命令，也可以把下文中的 `docker compose` 替换为 `docker-compose`。

## 快速部署

进入本目录：

```bash
cd tesla-iptv-player
```

导入镜像：

```bash
docker load -i tesla-iptv.tar
```

正常会看到类似输出：

```text
Loaded image: tesla-iptv-player:local
```

启动服务：

```bash
docker compose up -d
```

查看日志：

```bash
docker compose logs -f
```

访问地址：

```text
http://服务器IP:5188/
```

如果是在服务器本机测试：

```text
http://127.0.0.1:5188/
```

## 维护播放列表

频道都写在当前目录的 `index.m3u`。

 `index.m3u` 有一个可直接播放的公开测试源，用来验证服务是否正常。实际使用时，把它替换成自己的频道列表即可。

修改 `index.m3u` 后，一般不需要重启容器。服务端每次请求频道列表都会重新读取这个文件。车机端刷新页面或重新打开播放器即可看到新频道。

如果没有立即生效，可以重启容器：

```bash
docker compose restart
```

## 播放列表示例

最小可用示例：

```m3u
#EXTM3U
#EXTINF:-1 tvg-name="Tears of Steel Demo",Tears of Steel Demo
#EXTVLCOPT:tesla-direct=1
https://demo.unified-streaming.com/k8s/features/stable/video/tears-of-steel/tears-of-steel.ism/.m3u8
```

HTTP/HTTPS、RTSP、RTMP 可以混合：

```m3u
#EXTM3U
#EXTINF:-1 tvg-name="CCTV-1",CCTV-1
http://192.168.10.12:5140/CCTV-1
#EXTINF:-1 tvg-name="RTSP-Test",RTSP-Test
rtsp://example.com:8554/live
#EXTINF:-1 tvg-name="RTMP-Test",RTMP-Test
rtmp://example.com/live/stream
```

HTTP/HTTPS 源默认走服务器代理。如果某个 HTTP/HLS 源支持 CORS，并且车机可以直接跨域访问，可以在频道 URL 前加：

```m3u
#EXTVLCOPT:tesla-direct=1
```

## 按频道转码

如果服务器带宽较小，比如只有 5M，高清源中转会卡。可以给单个频道开启转码：

```m3u
#EXTINF:-1 tvg-name="CCTV-1HD",CCTV-1HD
#EXTVLCOPT:tesla-low=720
http://192.168.10.12:5140/CCTV-1HD
```

这会把视频转码到最高 720P，音频保持 copy。

如果有视频没声音，或者音频编码不兼容，可以同时转音频：

```m3u
#EXTINF:-1 tvg-name="CCTV-1HD",CCTV-1HD
#EXTVLCOPT:tesla-low=720&aac
http://192.168.10.12:5140/CCTV-1HD
```

频道列表会显示“转码”。如果同一个 HTTP 频道同时配置 `tesla-direct=1` 和 `tesla-low=720`，会以转码优先，自动走服务器代理。

## 按频道软解

播放器默认使用硬解模式。某个频道如果硬解不兼容，可以在频道 URL 前加：

```m3u
#EXTVLCOPT:tesla-decode=software
```

完整示例：

```m3u
#EXTINF:-1 tvg-name="Software-Test",Software-Test
#EXTVLCOPT:tesla-decode=software
http://192.168.10.12:5140/CCTV-1
```

这个频道会改用软解，频道列表会显示“软解”。当前版本不做自动回退，是否软解由播放列表控制。

## 常用命令

查看运行状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f
```

重启：

```bash
docker compose restart
```

停止：

```bash
docker compose down
```

重新启动：

```bash
docker compose up -d
```

查看 ffmpeg 进程：

```bash
docker exec -it tesla-iptv-player sh
ps -ef | grep ffmpeg
```

正常情况下，切换频道或关闭播放页后，旧的 ffmpeg 进程会自动退出。

## 局域网源注意事项

如果节目源在局域网，例如：

```text
http://192.168.10.12:5140/CCTV-1
rtsp://192.168.10.12:8554/live
```

请确保部署 Docker 的服务器可以访问这些地址。

如果节目源运行在 Docker 宿主机自己的 `127.0.0.1` 上，容器内的 `127.0.0.1` 指的是容器本身，不能直接访问宿主机服务。请改用宿主机局域网 IP，或在 Docker Desktop 环境中使用 `host.docker.internal`。

## 对外发布提醒

这个播放器默认没有账号密码，也没有鉴权。公网部署时，建议至少做一项保护：

- 只开放给自己的局域网或 VPN。
- 用 Nginx/Caddy 反向代理并加访问控制。
- 在服务器防火墙里限制访问 IP。

不要把内网敏感直播源、摄像头源或带鉴权信息的 URL 直接公开发布。

## 更新镜像

如果以后提供了新的 `tesla-iptv.tar`，在同目录替换旧文件后执行：

```bash
docker compose down
docker load -i tesla-iptv.tar
docker compose up -d
```

如果镜像标签仍然是 `tesla-iptv-player:local`，`docker-compose.yml` 不需要修改。
