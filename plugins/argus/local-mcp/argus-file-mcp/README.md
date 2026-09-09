# argus-file-mcp（bundle）

本目录 `index.js` 是 [Argus_Local_MCP/argus-file-mcp](https://github.com/Lugia123/Argus_Local_MCP)
仓库 esbuild bundle 后的产物（含所有 npm 运行时依赖，单文件自包含）。
**不要直接编辑这个文件**——它是构建产物，下次构建会被整个覆盖。

改代码请去 Argus_Local_MCP：

```bash
cd Argus_Local_MCP/argus-file-mcp
npm run build:bundle      # esbuild → bundle/index.js
cp bundle/index.js <本目录>/index.js
```

## 为什么产物要提交进 plugin 仓库

早先 plugin.json 走 `npx -y @bestfunc-com/argus-file-mcp@latest`，
结果是「代码进了 git、npm 上还是旧版本，谁都拿不到新工具」，而且每改一次本地 MCP
就得发一次 npm 包。产物随 plugin 一起发就没有这层延迟，也少一个发布渠道。

## 为什么是 CJS 而不是 ESM

产物是个孤零零的 `index.js`，旁边没有 `package.json`。Node 22 会按语法自动判定成
ESM，所以 ESM 产物在新版 Node 上跑得通；但 **Node 18 / 20 没有这个探测**，`.js` 一律
当 CJS，里面的 `import` 语句直接 `SyntaxError: Cannot use import statement outside a module`。

所以这里刻意输出 CJS，Node 18 / 22 都实测能起。改构建配置时别顺手改回 `format: 'esm'`。

## plugin.json 启动语法

```json
"argus-files": {
  "type": "stdio",
  "command": "node",
  "args": ["${CLAUDE_PLUGIN_ROOT}/local-mcp/argus-file-mcp/index.js"],
  "env": { "ARGUS_ENDPOINT": "https://argus.bestfunc.com" }
}
```

## 提供的工具

`upload_file` / `upload_to_sandbox` / `download_file` / `upload_agent_version`

都走 HTTP 直连 Argus Server，文件流不过 MCP 通道，AI context 与文件大小解耦。
