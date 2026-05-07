# idealab + claude code + cc-viewer 协同实现

> cc-viewer github：[https://github.com/weiesky/cc-viewer.git](https://github.com/weiesky/cc-viewer.git)

## 问题现象

启动 ccv（Claude Code 的可视化包装器）时遇到两个交替出现的错误：

:::
不配置 env 时：Not logged in · Please run /login

按某些教程配置 ANTHROPIC\_BASE\_URL=http://127.0.0.1:7009 后：There's an issue with the selected model  (claude-opus-4-7\[1m\]). It may not exist or you may not have access to it.
:::

## 根因分析（三个被混淆的概念）

排查过程中发现三个被普遍误解的点：

### 误解 1：把 ccv 当成了 LLM 代理

*   真相：ccv = cc-viewer，是 Claude Code 的可视化工具+流量记录器，不是 API 代理
    
*   端口 7009 是它的 Web UI（浏览器看 transcripts），不是 Anthropic 协议端点
    
*   验证方法：curl http://127.0.0.1:7009/v1/messages → 404 Not Found
    
*   因此 ANTHROPIC\_BASE\_URL=127.0.0.1:7009 是完全错误的配置
    

### 误解 2：以为 Idealab 直接支持 Claude Code

*   真相：Idealab 只暴露 OpenAI 协议端点（/api/openai/v1/chat/completions），用 from openai import OpenAI 调用
    
*   Claude Code 只发 Anthropic 协议请求（POST /v1/messages），两者协议不兼容
    
*   必须中间架一层"协议翻译器"才能打通
    

### 误解 3：模型 ID 后缀 \[1m\] 在第三方都可用

*   真相：opus\[1m\]（1M 上下文变体）是 Anthropic 官方的特殊标识，第三方代理（包括 Idealab）几乎都不支持
    
*   必须改成纯净的模型 ID 如 claude-opus-4-7
    

## 解决方案：本地起协议翻译层

**Claude Code (ccv)**   ↓ Anthropic 协议 POST /v1/messages   **claude-code-router (本地 127.0.0.1:3456)**   ↓ 翻译为 OpenAI 协议   **Idealab (https://idealab.alibaba-inc.com/api/openai/v1)**   ↓   **Claude 模型**

## 实施步骤

### Step 1：探测 Idealab 上可用的 Claude 模型

Idealab 只支持部分官方模型，且没有 haiku 系列、没有 1m 后缀。用 curl 探测确认：

```shell
for m in "claude-opus-4-7" "claude-opus-4-6" "claude-opus-4-5" "claude-sonnet-4-6" "claude37_sonnet"; do
resp=$(curl -sS -m 8 -X POST "https://idealab.alibaba-inc.com/api/openai/v1/chat/completions" -H "Authorization: Bearer <你的_AI_STUDIO_TOKEN>" \ 
-H "Content-Type: application/json" \ 
-d "{"model":"$m","messages":[{"role":"user","content":"hi"}],"max_tokens":3}") 
echo "$resp" | grep -q '"success":false' && echo "[FAIL] $m" || echo "[OK ] $m" 
done
```
> 实测确认可用：claude-opus-4-7、claude-opus-4-6、claude-opus-4-5、claude-sonnet-4-6、claude37\_sonnet。

### Step 2：安装 claude-code-router

```shell
npm install -g @musistudio/claude-code-router --registry=https://registry.npmmirror.com 

ccr --version # 确认安装成功
```

### Step 3：写 router 配置 `**~/.claude-code-router/config.json**`

```json
{ 
   "LOG": false, 
   "APIKEY": "local-router-secret", 
   "HOST": "127.0.0.1", 
   "PORT": 3456, 
   "Providers": [ 
     { 
       "name": "idealab", 
       "api_base_url": "https://idealab.alibaba-inc.com/api/openai/v1/chat/completions", 
       "api_key": "<你的_AI_STUDIO_TOKEN>", 
       "models": [ 
         "claude-opus-4-7", 
         "claude-opus-4-6", 
         "claude-opus-4-5", 
         "claude-sonnet-4-6", 
         "claude37_sonnet" 
         ]
     } 
   ], 
   "Router": {
     "default": "idealab,claude-opus-4-7",
     "background": "idealab,claude37_sonnet", 
     "think": "idealab,claude-opus-4-7", 
     "longContext": "idealab,claude-opus-4-7", 
     "longContextThreshold": 60000, 
     "webSearch": "idealab,claude-opus-4-7" 
     }
 }
```

关键点：

:::
api\_base\_url 必须带完整路径 /chat/completions，不能只写到 /v1

Router.background 给小任务（标题生成、token 计数、AskUserQuestion）用便宜模型，不要全用 opus

longContextThreshold 是 token 数，超过自动切 longContext 角色
:::

### Step 4：改 `**~/.claude/settings.json**`

新增顶层 env 块、把 model 字段改成纯净 ID：

```json
{ 
   "env": { 
     "ANTHROPIC_BASE_URL": "http://127.0.0.1:3456",
     "ANTHROPIC_AUTH_TOKEN": "local-router-secret", 
     "ANTHROPIC_MODEL": "claude-opus-4-7", 
     "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-7", 
     "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6", 
     "ANTHROPIC_SMALL_FAST_MODEL": "claude37_sonnet" 
   }, 
   "model": "claude-opus-4-7",
 }
```

关键点：

:::
ANTHROPIC\_AUTH\_TOKEN 的值要与 router config 里的 APIKEY 一致

去掉 \[1m\] 后缀——这是踩坑率最高的点

没有 haiku 时，SMALL\_FAST\_MODEL 用 claude37\_sonnet 兜底（如果你的环境能走 bedrock haiku，那是另一条路）
:::

### Step 5：启动并验证

```shell
ccr start # 起后台 router 服务 
ccr status # 确认 Running，Port 3456
```

### Step 6：端到端联通验证

```shell
curl -sS -X POST "http://127.0.0.1:3456/v1/messages" \ 
 -H "x-api-key: local-router-secret" \ 
 -H "anthropic-version: 2023-06-01" \ 
 -H "Content-Type: application/json" \ 
 -d '{"model":"claude-opus-4-7","max_tokens":30, 
 "messages":[{"role":"user","content":"Reply with: ROUTER_OK"}]}'
```

期望返回包含 "ROUTER\_OK" 的 Anthropic 格式响应

### Step 7：通了之后正常用： 

```shell
ccv # 等同于 claude，但带 cc-viewer 可视化
```

## 日常使用清单

| **时机** | **命令** |
| --- | --- |
| 每次重启电脑后 | ccr start（起 router） |
| 改完 router 配置 | ccr restart |
| 看 router 状态 | ccr status |
| 启动 Claude Code | ccv（router 必须先在跑） |

## 复用时的环境差异检查清单

这套方案复用时，请确认

:::
有自己的 Idealab AI\_STUDIO\_TOKEN

node 版本 ≥ 22（node --version）

端口 3456 没被占用（lsof -iTCP:3456）；冲突就改 router config 的 PORT + settings.json 的 BASE\_URL

用 Step 1 的脚本自己跑一遍模型探测——Idealab 上线/下线模型不通知，model 列表会变

如果在公司外网，需要 vipserver IP 替换域名（参考 Idealab 文档的 vipserver 章节，把  https://idealab.alibaba-inc.com 改成 http://{vipserver\_ip}）

如果原本就装了 ccv 并改过 settings.json，保留已有的 hooks/permissions 块，只增加 env 和改 model 字段
:::

## 避坑清单

:::
不要把 ANTHROPIC\_BASE\_URL 设成 ccv 的端口（7009）—— ccv 不是代理

不要把 ANTHROPIC\_BASE\_URL 直接设成 Idealab 的 OpenAI 端点 —— 协议不兼容

不要用带后缀的模型 ID（opus\[1m\]、\*-1m 等）—— 第三方不支持

不要忘记 router config 里 api\_base\_url 的 /chat/completions 后缀

不要让 background/small-fast 角色也用 opus —— 标题生成都用 opus 会很贵很慢
:::