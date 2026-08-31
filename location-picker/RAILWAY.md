# Location Picker — 部署到 Railway

把 `server.js`（网页地图选点服务）跑在 Railway 上：**免 VPS、自带 HTTPS 域名**，比 Cloudflare Worker 更接近自托管（同一份 Node 代码，本地/Docker/Railway 完全一致）。

部署完成后你会得到两个地址：

| 用途 | 地址 |
|------|------|
| 手机上打开的**地图选点页** | `https://<你的服务>.up.railway.app/?token=<TOKEN>` |
| 模块里填的**远程配置 URL** | `https://<你的服务>.up.railway.app/loc.json?token=<TOKEN>` |

---

## 一、准备 TOKEN

先生成一个随机口令（**这就是唯一的访问控制，别用弱口令**）：

```bash
openssl rand -hex 24
```

复制输出备用。

---

## 二、网页后台部署（推荐）

1. 把本仓库 fork 到自己的 GitHub。
2. 打开 [railway.app](https://railway.app) → **New Project** → **Deploy from GitHub repo** → 选中你 fork 的仓库。
3. 进入服务 → **Settings**：
   - **Root Directory** 填 `location-picker`
     （必须填！否则 Railway 在仓库根目录找不到 Dockerfile。填了之后它会自动读取 `location-picker/railway.json`，用 Dockerfile 构建、`/health` 做健康检查。）
   - **Networking → Public Networking → Generate Domain**，得到 `xxx.up.railway.app`
4. **Variables** 标签页新增：

   | 变量 | 值 | 说明 |
   |------|-----|------|
   | `TOKEN` | 第一步生成的随机串 | **必填**，不填进程会直接退出 |
   | `DATA_FILE` | `/data/loc.json` | 配合下面的 Volume 持久化坐标（Dockerfile 已内置同样的默认值，保险起见显式填一次） |

   `PORT` **不用填**：Dockerfile 里默认 8080，Railway 也会注入自己的 `PORT`，`server.js` 两种都认。

5. 挂持久卷（**强烈建议**）：服务面板 → 右键 / **+ Create** → **Volume** → 挂载路径填 `/data`。
   不挂卷也能用，但每次重新部署或容器重启，坐标会退回默认的 Apple Park——`/health` 里的 `"persistent": false` 就是在提示这件事。
6. 触发一次 **Deploy**，等状态变绿。

### 验证

```bash
curl https://<你的服务>.up.railway.app/health
# {"ok":true,"persistent":true,"dataFile":"/data/loc.json"}

curl 'https://<你的服务>.up.railway.app/loc.json?token=<TOKEN>'
# {"enabled":true,"latitude":37.3349,...}
```

`persistent` 为 `false` → 卷没挂上或 `DATA_FILE` 指到了不可写的路径，回第 5 步检查。

---

## 三、CLI 部署（可选）

```bash
npm i -g @railway/cli
railway login
cd location-picker
railway init                       # 新建 project
railway up                         # 用当前目录的 Dockerfile 部署
railway variables --set TOKEN=<你的随机串>
railway domain                     # 生成公开域名
```

Volume 目前仍需在网页后台挂载（挂载路径 `/data`）。

---

## 四、把地址填进代理模块

### Shadowrocket / Surge / Stash

模块 `argument=` **末尾**追加（注意是 `&configUrl=`）：

```
&configUrl=https://<你的服务>.up.railway.app/loc.json?token=<TOKEN>
```

### Loon

设置 → 插件 → iOS Location Spoofer → **远程配置 URL** 填：

```
https://<你的服务>.up.railway.app/loc.json?token=<TOKEN>
```

### 然后

1. iPhone 浏览器打开 `https://<你的服务>.up.railway.app/?token=<TOKEN>`
2. 点地图选点（海拔自动获取）→ **保存定位**
3. 按主 README 的生效步骤刷新：iOS 15~18 关开定位服务；**iOS 26/27 必须重启设备**（`locationd` 缓存）

---

## 五、注意事项

- **口令即安全边界**：任何拿到 URL + TOKEN 的人都能读写你的定位。别把带 token 的链接贴进公开的 Issue / 截图。
- **免费额度**：Railway 的免费/试用额度用完后服务会停，`configUrl` 拉不到就会回落到模块 `argument=` 里写死的坐标（脚本本身不会因此失效）。
- **冷启动**：Railway 不会像 Serverless 那样休眠，但重新部署期间有几十秒不可用，期间脚本使用上一次缓存的远程配置。
- **改坐标不用重新部署**：坐标存在卷里，改完在网页上点保存即可。只有改 `server.js` 才需要重新部署。
- **换 TOKEN**：改 Variables 里的 `TOKEN` → 服务自动重启 → 记得同步更新模块里的 `configUrl` 和浏览器书签。
