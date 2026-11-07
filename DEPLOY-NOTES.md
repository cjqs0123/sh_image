# 部署记录 / Deployment Notes

## 项目信息

| 项 | 值 |
| --- | --- |
| 上游来源 | https://github.com/rodneyzhang4marine-oss/product-image-studio.git（原始提交 `8e77390`） |
| 本仓库 | 重新初始化，历史从 `9a3a0f5` 开始，不保留上游提交历史 |
| 部署目录 | `E:\Ai_Pject\sh_image` |
| Gitee 远程 | `origin` → https://gitee.com/sooa233/sh_image.git（SSH） |
| GitHub 远程 | `github` → https://github.com/cjqs0123/sh_image.git（HTTPS） |
| 主分支 | `main` |
| Node / npm | v22.23.2 / 10.9.8 |
| 运行端口 | 5177（可用环境变量 `PORT` 覆盖） |
| 访问地址 | http://localhost:5177 |
| sharp 版本 | 0.34（libvips 8.17.3，Windows 原生二进制正常） |

## 首次推送记录

| 远程 | 协议 | 认证方式 | 结果 |
| --- | --- | --- | --- |
| Gitee | SSH 22（`git@gitee.com:sooa233/sh_image.git`） | 本机 `~/.ssh/id_ed25519`，已验证 `Hi 雷梦(@sooa233)` | 成功，新建 `main` |
| GitHub | HTTPS（`https://github.com/cjqs0123/sh_image.git`） | Git Credential Manager 浏览器授权（`cjqs0123` 账号） | 成功，新建 `main` |

两者 `refs/heads/main` 均指向 `9a3a0f517c55a4b831149c18cebc8e42d0f2fcc6`。

## 部署步骤

```powershell
cd E:\Ai_Pject\sh_image
npm install
npm run build          # 产出 dist/
npm run preview        # 生产模式启动（= node server/index.mjs --production）
```

开发模式用 `npm run dev`（Vite 中间件模式，同样监听 5177）。

## 部署中遇到的问题及处理

### 1. clone 报 SSL 证书错误

现象：`fatal: unable to access ... SSL certificate problem: unable to get local issuer certificate`。
原因：本机 Git（OpenSSL 后端，2.47.0）无法校验到 GitHub 证书链，系统未配置代理，属本机证书信任问题。
处理：仅对本次克隆改用 Windows 证书库后端，未改动全局配置：

```powershell
git -c http.sslBackend=schannel clone https://github.com/rodneyzhang4marine-oss/product-image-studio.git .
```

后续所有对 GitHub 的 HTTPS 操作（push / ls-remote）同样需要带 `-c http.sslBackend=schannel`。

### 2. 生产模式启动崩溃（Express 5 路由不兼容）

现象：`npm run preview` 立刻退出，报
`PathError [TypeError]: Missing parameter name at index 1: *`，`originalPath: '*'`。
原因：`package-lock.json` 锁定的 express 为 5.2.1（内部 path-to-regexp 8.4.2），Express 5 不再接受裸 `*` 通配路由，仓库里的 `app.get("*", ...)` 已失效。
处理：在 `server/index.mjs` 的生产分支把通配路由改成等价的中间件兜底（SPA 回退行为不变，仅排除非 GET/HEAD）：

```js
app.use(express.static(path.join(projectRoot, "dist")));
app.use((request, response, next) => {
  if (request.method !== "GET" && request.method !== "HEAD") {
    next();
    return;
  }
  response.sendFile(path.join(projectRoot, "dist", "index.html"));
});
```

未降级 express 版本，也未改动其他业务逻辑。

### 3. 本机网络下 GitHub 的 SSH 与 curl 都不可用

- `ssh -T git@github.com`（22 端口）与 `ssh -T -p 443 git@ssh.github.com` 均返回 `Connection refused`，GitHub 的 SSH 通道在本机网络不通。
- `curl https://github.com` 报 SSL 连接错误（curl 自带证书链问题），因此 GitHub 侧只能用 **git + schannel** 走 HTTPS。
- Gitee 的 SSH 与 HTTPS 均正常。

结论：GitHub 统一用 `git -c http.sslBackend=schannel <命令>` + HTTPS；Gitee 用 SSH。

## 已验证项

| 检查 | 结果 |
| --- | --- |
| `sharp` 原生模块加载 | 通过（libvips 8.17.3） |
| `npm run build` | 通过，1574 模块，dist 约 281 KB JS + 16 KB CSS |
| `GET /` | 200，482 B |
| `GET /api/defaults` | 200，返回 `{"sourceRoot":"","outputRoot":"E:\\Ai_Pject\\sh_image\\output\\product-ready"}` |
| `GET /some/spa/route`（SPA 回退） | 200 |
| `GET /assets/index-*.js` | 200，281000 B |
| `GET /assets/index-*.css` | 200，15924 B |
| 5177 端口监听 | 正常 |
| 首次提交文件数 | 14 个（`node_modules/`、`dist/`、`output/` 已由 `.gitignore` 排除） |

## 日常推送

```powershell
cd E:\Ai_Pject\sh_image
git add -A
git commit -m "你的改动说明"
git push origin main                                    # Gitee
git -c http.sslBackend=schannel push github main        # GitHub
```

一次命令推两边：

```powershell
git push origin main; git -c http.sslBackend=schannel push github main
```

如果不想每次都写 `-c http.sslBackend=schannel`，可只对 GitHub 域名设置证书后端：

```powershell
git config --global http.https://github.com.sslBackend schannel
```

## 可选：批量图片接口

浏览器编辑器不需要任何配置即可使用。若要用本地批量扫描/处理接口，设置环境变量后再启动：

```powershell
$env:PRODUCT_IMAGE_SOURCE = "D:\path\to\source\images"
$env:PRODUCT_IMAGE_OUTPUT = "D:\path\to\output"
npm run preview
```

默认输出目录为 `E:\Ai_Pject\sh_image\output\product-ready`。

## 启动 / 停止

```powershell
# 启动（生产模式）
cd E:\Ai_Pject\sh_image; npm run preview

# 停掉占用 5177 的进程
$pid5177 = (Get-NetTCPConnection -LocalPort 5177 -State Listen).OwningProcess
Stop-Process -Id $pid5177
```
