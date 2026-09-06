# mcp-webcam 📸

基于 [evalstate/mcp-webcam](https://github.com/evalstate/mcp-webcam) 的修改版本，增加了 HTTPS 支持、手机摄像头接入和连接稳定性优化。

## 新增功能

- **HTTPS 支持**：通过自签名证书让手机浏览器在局域网内访问摄像头
- **手机摄像头接入**：手机通过局域网 HTTPS 连接，配合画中画保持前台运行
- **SSE 自动重连**：断线后 3 秒自动重连，提升稳定性
- **Ping 优化**：默认关闭对 Claude 无效的 ping 心跳，避免终端刷屏

## 使用场景

| 场景 | 模式 | 说明 |
|------|------|------|
| Claude Desktop / Claude Code | stdio | 本地直连，开箱即用 |
| Claude App / 远程客户端 | streaming HTTP + Cloudflare Tunnel | 通过公网连接 |
| 手机摄像头 | streaming HTTP + HTTPS | 局域网直连，需要生成证书 |

## 快速开始

### 1. 安装依赖

```bash
npm install
```

### 2. 选择你的使用模式

#### 模式 A：Claude Desktop / Claude Code（stdio）

无需额外配置，按原项目说明在 Claude Desktop 中添加 MCP server 即可。

#### 模式 B：Claude App 远程连接（streaming HTTP）

**启动服务：**

```bash
npm run build
npx . --streaming
```

**启动 Cloudflare Tunnel（让 Claude App 能连接到本地服务）：**

```bash
cloudflared tunnel --url https://localhost:3333 --no-tls-verify --protocol http2
```

> 需要先安装 [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)

**在 Claude App 中添加 MCP 连接：**

URL 填写：`https://你的tunnel地址/mcp?user=mobile`

**打开电脑浏览器：**

访问 `https://localhost:3333/?user=mobile`，选择摄像头，开始使用。

#### 模式 C：手机摄像头接入

在模式 B 的基础上，额外需要：

**生成 HTTPS 自签名证书：**

```bash
npm install node-forge
```

```bash
node -e "const forge=require('node-forge'),pki=forge.pki,keys=pki.rsa.generateKeyPair(2048),cert=pki.createCertificate();cert.publicKey=keys.publicKey;cert.serialNumber='01';cert.validity.notBefore=new Date();cert.validity.notAfter=new Date();cert.validity.notAfter.setFullYear(cert.validity.notBefore.getFullYear()+1);const a=[{name:'commonName',value:'你的电脑局域网IP'}];cert.setSubject(a);cert.setIssuer(a);cert.setExtensions([{name:'subjectAltName',altNames:[{type:7,ip:'你的电脑局域网IP'}]}]);cert.sign(keys.privateKey);require('fs').writeFileSync('cert.pem',pki.certificateToPem(cert));require('fs').writeFileSync('key.pem',pki.privateKeyToPem(keys.privateKey));console.log('Done!');"
```

> 将 `你的电脑局域网IP` 替换为实际 IP（如 `192.168.1.107`），可通过 `ipconfig` 查看 WLAN 适配器的 IPv4 地址。建议在路由器或系统设置中将 IP 固定，避免重启后变化。

**手机端访问：**

手机浏览器（推荐 Chrome）打开 `https://你的电脑局域网IP:3333/?user=mobile`

- 首次访问会提示不安全，选择继续访问即可
- **iOS 用户**：推荐使用 Safari，配合 [Web PiP Extension & Browser](https://apps.apple.com/app/id6450466249)（App Store 免费下载）开启画中画，将摄像头视频分屏保持在前台，避免切后台导致 SSE 断连
- **Android 用户**：推荐使用 Chrome，大部分安卓手机自带分屏/小窗功能，无需额外安装插件
- 确保手机和电脑在同一 WiFi 下

**切换摄像头：**

电脑和手机使用同一个 `user` 参数，同一时间只开一边的网页即可切换：
- 想用电脑摄像头 → 电脑打开网页，关掉手机网页
- 想用手机摄像头 → 手机打开网页，关掉电脑网页

## 注意事项

- `cert.pem` 和 `key.pem` 是本地证书文件，不要上传到 Git
- Cloudflare Tunnel 每次重启会生成新的 URL，需要更新 Claude 的 MCP 配置
- 如需固定 Tunnel URL，可在 Cloudflare 绑定自定义域名
- 手机端 SSE 连接在切后台时会断开，回到前台后会自动重连（3 秒）
- 可根据自身需求在代码中调整摄像头分辨率，分辨率越高画质越好但 token 消耗也越大

## 致谢

原项目：[evalstate/mcp-webcam](https://github.com/evalstate/mcp-webcam)
