# 部署记录 / Deployment Notes

## 项目信息

| 项 | 值 |
| --- | --- |
| 仓库 | https://github.com/rodneyzhang4marine-oss/product-image-studio.git |
| 部署目录 | `E:\Ai_Pject` |
| 当前提交 | `8e77390 Improve README SEO and channel links` |
| Node / npm | v22.23.2 / 10.9.8 |
| 运行端口 | 5177（可用环境变量 `PORT` 覆盖） |
| 访问地址 | http://localhost:5177 |
| sharp 版本 | 0.34（libvips 8.17.3，Windows 原生二进制正常） |

## 部署步骤

```powershell
cd E:\Ai_Pject
git -c http.sslBackend=schannel clone https://github.com/rodneyzhang4marine-oss/product-image-studio.git .
npm install
npm run build          # 产出 dist/
npm run preview        # 生产模式启动（= node server/index.mjs --production）
```

开发模式用 `npm run dev`（Vite 中间件模式，同样监听 5177）。

## 部署中遇到的两个问题及处理

### 1. clone 报 SSL 证书错误

现象：`fatal: unable to access ... SSL certificate problem: unable to get local issuer certificate`。
原因：本机 Git（OpenSSL 后端，2.47.0）无法校验到 GitHub 证书链，系统未配置代理，属本机证书信任问题。
处理：仅对本次克隆改用 Windows 证书库后端，未改动全局配置：

```powershell
git -c http.sslBackend=schannel clone https://github.com/rodneyzhang4marine-oss/product-image-studio.git .
```

若后续 `git pull` 仍报同类错误，可对单条命令继续带 `-c http.sslBackend=schannel`。

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

## 已验证项

| 检查 | 结果 |
| --- | --- |
| `sharp` 原生模块加载 | 通过（libvips 8.17.3） |
| `npm run build` | 通过，1574 模块，dist 约 281 KB JS + 16 KB CSS |
| `GET /` | 200，482 B |
| `GET /api/defaults` | 200，返回 `{"sourceRoot":"","outputRoot":"E:\\Ai_Pject\\output\\product-ready"}` |
| `GET /some/spa/route`（SPA 回退） | 200 |
| `GET /assets/index-CoWkEaGu.js` | 200，281000 B |
| `GET /assets/index-KFDvZ7iC.css` | 200，15924 B |
| 5177 端口监听 | 正常（进程 PID 31684） |

## 本地改动清单（相对上游）

- `server/index.mjs`：生产模式 SPA 兜底路由改为中间件写法（修复 Express 5 崩溃）。
- `package-lock.json`：`npm install` 后由 npm 自动规范化，无手工改动。

以上两处未提交 git，保持工作区改动状态，方便与上游 diff。

## 可选：批量图片接口

浏览器编辑器不需要任何配置即可使用。若要用本地批量扫描/处理接口，设置环境变量后再启动：

```powershell
$env:PRODUCT_IMAGE_SOURCE = "D:\path\to\source\images"
$env:PRODUCT_IMAGE_OUTPUT = "D:\path\to\output"
npm run preview
```

默认输出目录为 `E:\Ai_Pject\output\product-ready`。

## 启动 / 停止

```powershell
# 启动（生产模式）
cd E:\Ai_Pject; npm run preview

# 停掉占用 5177 的进程
$pid5177 = (Get-NetTCPConnection -LocalPort 5177 -State Listen).OwningProcess
Stop-Process -Id $pid5177
```
