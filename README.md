# onc-auth

锐捷 ONC 网络控制器认证流程的 Claude Code Skill。

涵盖 RSA 公钥获取、密码加密、JWT Token 获取的完整三步流程，以及常见错误排查。

---

## 安装（Claude Code）

### 方法一：命令行一键安装

```bash
cd ~/.claude/skills && curl -L https://github.com/jason-song-hj/onc-auth/archive/refs/heads/master.zip -o onc-auth.zip && unzip -o onc-auth.zip && mv onc-auth-master onc-auth && rm onc-auth.zip
```

### 方法二：手动安装

1. 点击右上角 **Code → Download ZIP**，下载后解压
2. 将解压出的文件夹重命名为 `onc-auth`
3. 移动到 Claude Code 的 skills 目录：

```bash
mv onc-auth ~/.claude/skills/
```

### 验证安装

```bash
ls ~/.claude/skills/onc-auth/SKILL.md
```

---

## 使用方法

重启 Claude Code 后，在对话中直接描述需求即可触发，例如：

- "帮我写一个调用 ONC API 的 Python 脚本"
- "ONC 怎么获取 token？"
- "用 curl 测试一下 ONC 接口"
- "对接锐捷 ONC 的认证流程是什么"

Claude 会自动读取此 skill，按照正确的三步流程（获取公钥 → RSA 加密密码 → 登录获取 token）生成代码或解答。

---

## Skill 内容摘要

| 知识点 | 说明 |
|--------|------|
| 端口 | 统一走 HTTPS 443，不是 18080 |
| API 前缀 | 所有路径需加 `/api` 前缀 |
| 证书 | 自签证书，需跳过验证（`-k` / `verify=False`）|
| 密码加密 | PKCS1_v1_5 RSA 加密，依赖 `pycryptodome` |
| OAuth Client | 固定值 `web_app:rgsdbn`，与账号无关 |
