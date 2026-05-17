# Seafile 集成 ONLYOFFICE Docker Compose 排查与修复教程

## 1. 问题现象

在 Seafile 中打开 Office 文档时，ONLYOFFICE 报错：

- 文档安全令牌格式不正确
- Download failed / 下载失败
- ONLYOFFICE 日志出现：

```text
checkJwt error: JsonWebTokenError message = invalid signature
```

后续又出现：

```text
error downloadFile:url=http://192.168.51.201:30000/seafhttp/files/.../33.docx
code:ERR_BAD_REQUEST
AxiosError: Request failed with status code 404
```

最终确认：  
**JWT 问题解决后，真正导致下载失败的是 Seafile 端口映射错误。**

---

## 2. 最终根因

你当前 Seafile 的外部访问地址是：

```text
http://192.168.51.201:30000
```

但是 Docker 里实际配置成了：

```yaml
ports:
  - "30000:8000"
```

也就是：

```text
宿主机 30000 -> Seahub 8000
```

这会导致：

- Seafile 网页可以正常打开；
- 但是 ONLYOFFICE 下载文件时访问的 `/seafhttp/files/...` 会返回 `404`；
- 因为 `/seafhttp` 不是 Seahub 8000 处理的路径；
- `/seafhttp` 必须走 Seafile 的 HTTP 入口，也就是容器内的 `80` 端口。

正确方式应该是：

```yaml
ports:
  - "30000:80"
```

也就是：

```text
宿主机 30000 -> Seafile HTTP 入口 80
```

---

## 3. 正确的端口映射

进入 compose 目录：

```bash
cd /opt/seafile
```

查找当前端口配置：

```bash
grep -RInE '30000|8000|ports:' *.yml .env
```

如果看到类似：

```yaml
ports:
  - "30000:8000"
```

改成：

```yaml
ports:
  - "30000:80"
```

完整示例：

```yaml
services:
  seafile:
    image: seafileltd/seafile-mc:13.0-latest
    container_name: seafile
    restart: unless-stopped
    ports:
      - "30000:80"
    volumes:
      - /opt/seafile-data:/shared
    networks:
      - seafile-net
```

重点：

```text
不要使用 30000:8000
必须使用 30000:80
```

---

## 4. `.env` 配置

编辑：

```bash
nano /opt/seafile/.env
```

确认以下配置：

```env
SEAFILE_SERVER_HOSTNAME=192.168.51.201:30000
SEAFILE_SERVER_PROTOCOL=http

ONLYOFFICE_PORT=6233
ONLYOFFICE_JWT_SECRET=eaa3c5f72ff8ac7b97bfafce4522f54bff81dcfe
```

注意：

- `SEAFILE_SERVER_HOSTNAME` 要带端口 `30000`；
- 不要写成 `192.168.51.201:3000`；
- `ONLYOFFICE_JWT_SECRET` 要和 `seahub_settings.py` 里完全一致。

---

## 5. `seahub_settings.py` 配置

编辑：

```bash
nano /opt/seafile-data/seafile/conf/seahub_settings.py
```

推荐配置如下：

```python
# OnlyOffice
ENABLE_ONLYOFFICE = True
ONLYOFFICE_APIJS_URL = 'http://192.168.51.201:6233/web-apps/apps/api/documents/api.js'
ONLYOFFICE_JWT_SECRET = 'eaa3c5f72ff8ac7b97bfafce4522f54bff81dcfe'
OFFICE_PREVIEW_MAX_SIZE = 30 * 1024 * 1024

SERVICE_URL = "http://192.168.51.201:30000"
FILE_SERVER_ROOT = "http://192.168.51.201:30000/seafhttp"
```

重点：

```text
SERVICE_URL 必须是 30000
FILE_SERVER_ROOT 必须是 30000/seafhttp
不要残留 3000
```

检查是否还有残留 `3000`：

```bash
grep -RInE '192\.168\.51\.201:3000([^0-9]|$)|:3000([^0-9]|$)' \
  /opt/seafile /opt/seafile-data/seafile/conf 2>/dev/null
```

---

## 6. ONLYOFFICE 配置

编辑：

```bash
nano /opt/seafile/onlyoffice.yml
```

推荐配置：

```yaml
services:
  onlyoffice:
    image: onlyoffice/documentserver:8.1.0.1
    container_name: onlyoffice
    restart: unless-stopped
    ports:
      - "${ONLYOFFICE_PORT:-6233}:80"
    environment:
      JWT_ENABLED: "true"
      JWT_SECRET: "${ONLYOFFICE_JWT_SECRET}"
      JWT_HEADER: "Authorization"
      ALLOW_PRIVATE_IP_ADDRESS: "true"
    volumes:
      - /opt/onlyoffice/logs:/var/log/onlyoffice
      - /opt/onlyoffice/data:/var/www/onlyoffice/Data
      - /opt/onlyoffice/lib:/var/lib/onlyoffice
      - /opt/onlyoffice/db:/var/lib/postgresql
    networks:
      - seafile-net

networks:
  seafile-net:
    external: true
```

重点：

```text
JWT_ENABLED 必须是 true
JWT_SECRET 必须使用和 Seafile 一样的密钥
ALLOW_PRIVATE_IP_ADDRESS 建议开启，因为你用的是内网 IP
```

不要配置成：

```yaml
JWT_ENABLED: "false"
```

---

## 7. 重建容器

修改环境变量、端口映射后，不建议只 restart。  
建议完整重建：

```bash
cd /opt/seafile

docker compose down
docker compose up -d
```

或者只重建 ONLYOFFICE：

```bash
cd /opt/seafile

docker compose stop onlyoffice
docker compose rm -f onlyoffice
docker compose up -d onlyoffice
docker compose restart seafile
```

---

## 8. 验证端口映射

执行：

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep -Ei 'seafile|onlyoffice|caddy'
```

正确结果应该看到类似：

```text
0.0.0.0:30000->80/tcp
0.0.0.0:6233->80/tcp
```

错误结果是：

```text
0.0.0.0:30000->8000/tcp
```

如果还是 `30000->8000`，说明 compose 没改对，或者改的不是当前正在使用的 compose 文件。

---

## 9. 验证 JWT 是否一致

执行：

```bash
grep ONLYOFFICE_JWT_SECRET /opt/seafile/.env
grep ONLYOFFICE_JWT_SECRET /opt/seafile-data/seafile/conf/seahub_settings.py
docker exec onlyoffice printenv | grep -E 'JWT_ENABLED|JWT_SECRET|JWT_HEADER'
```

正确结果应该类似：

```text
ONLYOFFICE_JWT_SECRET=eaa3c5f72ff8ac7b97bfafce4522f54bff81dcfe
ONLYOFFICE_JWT_SECRET = 'eaa3c5f72ff8ac7b97bfafce4522f54bff81dcfe'
JWT_ENABLED=true
JWT_SECRET=eaa3c5f72ff8ac7b97bfafce4522f54bff81dcfe
JWT_HEADER=Authorization
```

如果 ONLYOFFICE 容器里的 `JWT_SECRET` 不一样，就会出现：

```text
invalid signature
```

---

## 10. 验证 ONLYOFFICE 能访问 Seafile

在 ONLYOFFICE 容器里测试：

```bash
docker exec onlyoffice bash -lc "curl -I http://192.168.51.201:30000"
docker exec onlyoffice bash -lc "curl -I http://192.168.51.201:30000/seafhttp/"
```

判断：

- `Connection refused`：端口映射或服务没起来；
- `No route to host`：网络不通；
- `timeout`：防火墙或路由问题；
- `404`：普通 `/seafhttp/` 没带文件 token 时可能正常；
- 关键是打开文档时 `/seafhttp/files/...` 不能再返回 404。

---

## 11. 重新打开文档并查看日志

打开文档时执行：

```bash
docker logs -f onlyoffice
```

重点看是否还有：

```text
invalid signature
Download failed
status code 404
private IP address
Connection refused
```

如果修复成功：

- 不再出现 `invalid signature`；
- 不再出现 `/seafhttp/files/... status code 404`；
- 文档可以正常加载和编辑。

---

## 12. 最终推荐配置总结

### Seafile 访问地址

```text
http://192.168.51.201:30000
```

### Seafile 端口映射

```yaml
ports:
  - "30000:80"
```

### ONLYOFFICE 访问地址

```text
http://192.168.51.201:6233
```

### ONLYOFFICE API 地址

```python
ONLYOFFICE_APIJS_URL = 'http://192.168.51.201:6233/web-apps/apps/api/documents/api.js'
```

### Seafile 文件服务地址

```python
FILE_SERVER_ROOT = "http://192.168.51.201:30000/seafhttp"
```

### Seafile 服务地址

```python
SERVICE_URL = "http://192.168.51.201:30000"
```

### JWT 密钥

```text
eaa3c5f72ff8ac7b97bfafce4522f54bff81dcfe
```

---

## 13. 一句话结论

这次问题的核心不是 ONLYOFFICE 本身，而是：

```text
30000 映射到了 Seahub 8000，导致 /seafhttp/files/... 返回 404。
```

修复方法：

```text
把 30000:8000 改成 30000:80。
```

然后统一：

```text
SEAFILE_SERVER_HOSTNAME=192.168.51.201:30000
SERVICE_URL=http://192.168.51.201:30000
FILE_SERVER_ROOT=http://192.168.51.201:30000/seafhttp
```
