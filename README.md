# Keepora

给图标、Logo 去掉纯色或近似纯色背景，下载真正带 Alpha 通道的透明 PNG。
所有像素处理都在打开页面的设备上完成：**不上传图片、不加载模型、不需要网络、账号或后端 Worker**。

**在线使用：https://keepora.pages.dev/**

> 本项目源自 [OpenBitFun](https://github.com/openBitFun/OpenBitFun) 的 `MiniApp/Demo/background-remover` 示例，
> 在其基础上更名为 Keepora、改造成可独立部署的静态站点并发布到 Cloudflare Pages。
> 上游原始说明保留在 [`MiniApp/Demo/background-remover/README.md`](MiniApp/Demo/background-remover/README.md)。

## 使用

1. 点击上传、拖入或粘贴一张 PNG、JPG、WebP 图片。
2. 自动读取图片边缘的主要背景颜色，并显示抠图结果。棋盘格代表真实透明。
3. 背景未去干净时，在原图上点击背景取色，或调整颜色容差和边缘柔化。
4. 点击「下载透明 PNG」，保留原图像素尺寸。对比底色只改变预览，不写入文件。

默认只移除与图片四边连通的背景，保留主体内部的同色区域；需要去掉字母、镂空处的背景时取消该选项。对已有透明边缘的图片，自动模式先保留原图，手动取色后可继续处理。

## 算法原理

纯前端实现，核心是一条「先判定背景、再按连通性扩散」的流程：

1. **自动背景识别**：采样图片四条边上的像素，取出主背景色，不依赖任何模型。
2. **边缘连通扩散**：默认以四边为种子做连通填充，只清除「与边缘连通且在容差内」的像素，因此 Logo 内部的同色区域会被保留；关闭该选项则对全图做颜色匹配，可去掉字母孔洞里的背景。
3. **容差与边缘柔化**：容差决定颜色接近程度的阈值；柔化在主体边缘产生部分透明（partial alpha），去色边（decontamination）减轻主体边缘残留的背景色光晕。
4. **保留既有透明度**：对已有 alpha 的像素做乘法而不是覆盖，避免重复处理破坏原图。
5. **导出**：`canvas.toBlob` 生成 PNG；在 OpenBitFun 宿主内走 `app.fs.writeFile`，独立网页版回退为 `<a download>` 浏览器下载。

由于全部计算都在 `Canvas` + `ImageData` 上完成，页面运行时不会发出任何图片数据。

## 限制

- 最大输入 30 MiB、1600 万像素，单边不超过 8192 像素；超限明确提示，不自动缩图。
- 适合纯色/近似纯色背景。复杂照片、已经烘焙进图片的棋盘格、与背景相连的同色主体无法可靠分离。
- 容差过大可能清空图片，此时禁用导出并提示调整设置。

## 目录结构

| 路径 | 说明 |
| --- | --- |
| `MiniApp/Demo/background-remover/source/` | MiniApp 源码（`index.html`、`style.css`、`ui/*.js`、构建与打包脚本） |
| `MiniApp/Demo/background-remover/source/ui.js` | 由 `ui/*.js` 拼接生成的入口，勿手改 |
| `MiniApp/Demo/background-remover/tests/` | 算法与交付产物的自测 |
| `shared/default_appearance_style.html` | 从上游 vendor 的外观令牌（构建时内联，保证渲染一致） |
| `output/miniapps/background-remover.html` | 构建产物：自包含单文件，可离线打开 |
| `deploy/` | Cloudflare Pages 部署配置与发布目录 |
| `deploy/pages/_headers` | CSP 与安全响应头（`connect-src 'none'` 强制禁止外发请求） |

## 本地构建与测试

需要 Node.js 18 及以上（使用 `node:test`）。

```sh
node MiniApp/Demo/background-remover/source/build.mjs
node --test MiniApp/Demo/background-remover/tests/*.test.mjs
```

构建后直接双击 `output/miniapps/background-remover.html` 即可离线使用，无需启动服务器。

测试覆盖背景识别、边缘连通、封闭区域、既有透明度、柔化与去色边、取消处理、超限与参数校验，以及产物的自包含性与中英文文案键覆盖。

## 打包为 OpenBitFun 小应用

```sh
python3 MiniApp/Demo/background-remover/source/package.py
```

生成 `output/miniapps/background-remover.bfminiapp`，在 OpenBitFun 的「导入小应用」中导入。
也可直接「从文件夹导入」`MiniApp/Demo/background-remover`（含 `meta.json` 的目录，不是 `source` 子目录）。

仅申请 `{user-selected}` 写权限，用于保存 PNG；图片输入来自页面文件选择器、拖拽或粘贴，不读取工作区或任意目录。

## 部署到 Cloudflare Pages

发布目录是 `deploy/pages`（`index.html` + `_headers` + `robots.txt`）。

```sh
cd deploy
npx wrangler login
npx wrangler pages deploy pages --project-name keepora --branch production
```

注意必须带 `--branch production`：项目的生产分支是 `production`，用其他分支名只会生成预览部署，生产域名不会更新。

`_headers` 中除了 CSP、HSTS、`X-Content-Type-Options`、`Referrer-Policy` 外，还把 `connect-src` 设为 `'none'` —— 浏览器层面禁止一切 `fetch`/XHR/WebSocket，使「不上传图片」成为可验证的约束而不是口头承诺。

## 来源与许可

上游为 [openBitFun/OpenBitFun](https://github.com/openBitFun/OpenBitFun)。本仓库保留上游来源说明；使用与再分发请遵循上游仓库的许可条款。