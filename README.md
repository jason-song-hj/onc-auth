# onc-auth Skill — 使用说明

> 锐捷ONC网络控制器认证流程的 Claude Skill；当需要对ONC系统进行API调用、获取token、或者需要了解ONC登录流程时使用此skill。适用于所有涉及ONC REST API对接、curl测试、Python脚本调用ONC接口的场景。

---

## 这个 Skill 是什么？

`onc-auth` 是一个存储在 Claude Skill 系统中的知识文件，记录了锐捷 ONC（网络控制器）的完整认证流程——包括踩坑记录、正确路径规律、Python 示例代码。

当你在 Claude 对话中输入 `/onc-auth`，Claude 会**主动加载这份文件**，之后的所有 ONC 相关问题都能直接给出正确答案，不需要你重复解释背景。

---

## 怎么在 Claude 或者其他支持Skills的平台里用？

把SKILL.md下载后传入Claude, 让Claude添加Skills即可；

→ 其他平台比如openclaw或者Hermes需要转成对应的平台skills格式才能使用，如有需要可自行转换下；

### 方式一：触发词激活（推荐）

在对话开头输入：

```
/onc-auth 帮我写一个获取所有设备列表的Python脚本
```

Claude 会先读取 skill 内容，然后直接输出包含正确认证流程的完整代码，**无需你再解释什么是 RSA 加密、什么是 client_auth**。

---

### 方式二：描述关键词自动触发

只要你的问题涉及以下关键词，Claude 也会自动加载这个 skill：

- ONC / 锐捷ONC / 网络控制器
- ONC token / ONC登录 / ONC API
- `curl` 测ONC接口
- Python 调用 ONC

示例提问（无需加 `/onc-auth`）：

```
帮我用Python登录ONC，然后查一下所有在线交换机
```

```
我想用curl测试一下ONC的拓扑接口，怎么写？
```

---

## Skill 帮你记住了什么？

不用再反复解释这些背景知识：

| 知识点 | 内容 |
|--------|------|
| **端口** | 外部统一走 HTTPS 443，不是 18080 |
| **路径前缀** | 所有业务 API 加 `/api` 前缀 |
| **SSL** | 自签证书，curl 加 `-k`，Python 加 `verify=False` |
| **密码加密** | 明文密码会报错，必须用 ONC 返回的 RSA 公钥加密 |
| **client_auth** | 固定值 `Basic d2ViX2FwcDpyZ3Nkbg==`（web_app:rgsdbn） |
| **常见报错** | `invalid_grant`、`invalid_client`、`invalid_scope` 的原因和解法 |

---

## 典型使用场景举例

### 场景 1：写一个新的 API 调用脚本

```
/onc-auth 帮我写个脚本，查询 ONC 里所有端口状态为 down 的接口
```

→ Claude 会直接输出包含三步认证 + 正确路径前缀的完整 Python 代码

---

### 场景 2：调试 curl 请求

```
/onc-auth 我想用curl测一下 /ne-mgr/v1/devices，怎么写完整命令？
```

→ Claude 会输出正确的 curl 命令，含 `-k`、`Authorization: Bearer` 头、完整路径

---

### 场景 3：排查认证报错

```
/onc-auth 登录时一直报 invalid_grant，已经传了用户名密码了
```

→ Claude 直接定位：密码没有 RSA 加密，输出加密代码片段

---

### 场景 4：集成到新工具

```
/onc-auth 需要加一个新的 ONC 光模块查询模块，帮我写 OncClient 的基础类
```

→ Claude 输出带正确 `_get` / `_post` 封装的 client 类骨架

---

## 客户现场快速参考

（Skill 内置，Claude 可直接引用，注意改写成你真实的IP和用户名）

| 字段 | 值 |
|------|----|
| ONC 地址 | `https://10.1.7.1` |
| 用户名 | `admin` |
| 密码 | `Ruijie@123` |
| RSA 公钥接口 | `GET /api/uaa/api/v1/rsa/public` |
| 登录接口 | `POST /api/auth/token` |
| client_auth（固定） | `Basic d2ViX2FwcDpyZ3Nkbg==` |

---

## Skill 文件位置

```
/mnt/skills/user/onc-auth/SKILL.md
```

如需更新（比如换了项目、新增踩坑记录），直接编辑这个文件即可。Claude 每次触发时实时读取，改完立刻生效。
