---
name: onc-auth
description: "锐捷ONC网络控制器的认证流程。当需要对ONC系统进行API调用、获取token、或者需要了解ONC登录流程时使用此skill。适用于所有涉及ONC REST API对接、curl测试、Python脚本调用ONC接口的场景。"
---

# ONC 认证流程

锐捷ONC（网络控制器）使用 HTTPS + RSA加密密码 的三步认证流程获取JWT token。

## 背景信息

- ONC前端统一走 **HTTPS 443端口**，不是18080（18080是后端内部端口，外部不可达）
- **所有API路径必须加 /api 前缀**，无一例外，文档里写的路径全部要在前面加 /api
- **所有API路径必须加  前缀**，无一例外，包括认证接口
- 使用自签证书，所有请求需要跳过证书验证（curl加`-k`，Python加`verify=False`）
- 密码必须用ONC返回的RSA公钥加密后才能登录，明文密码会报错
- OAuth client固定是 `web_app:rgsdbn`（Base64: `d2ViX2FwcDpyZ3Nkbg==`），这是系统内置值，与管理员账号无关

---

## 三步认证流程

### 第一步：获取RSA公钥

```bash
curl -k https://<ONC_IP>/uaa/api/v1/rsa/public
```

返回：
```json
{
  "publicKey": "MIIBojANBg..."
}
```

### 第二步：RSA加密密码（Python）

纯curl无法做RSA加密，必须用Python：

```python
import base64
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5

def rsa_encrypt(public_key_b64: str, plaintext: str) -> str:
    pem = (
        "-----BEGIN PUBLIC KEY-----\n"
        + public_key_b64
        + "\n-----END PUBLIC KEY-----"
    )
    key = RSA.import_key(pem)
    cipher = PKCS1_v1_5.new(key)
    encrypted = cipher.encrypt(plaintext.encode("utf-8"))
    return base64.b64encode(encrypted).decode("utf-8")
```

依赖：`pip install pycryptodome`

### 第三步：登录获取token

```bash
curl -k -X POST "https://<ONC_IP>/auth/token" \
  -H "Authorization: Basic d2ViX2FwcDpyZ3Nkbg==" \
  -F 'username="admin"' \
  -F 'password="<RSA加密后的密码>"' \
  -F 'grant_type="password"' \
  -F 'encrypt="true"'
```

返回：
```json
{
  "access_token": "eyJhbGci...",
  "token_type": "bearer",
  "expires_in": 604800,
  "scope": "openid"
}
```

### 后续API请求

所有API请求在Header携带token：
```
Authorization: Bearer <access_token>
```

---

## 路径前缀规律

所有接口统一走 HTTPS 443，**业务API统一在路径前加 `/api`**：

| 类型 | 路径 |
|------|------|
| RSA公钥 | `/api/uaa/api/v1/rsa/public` |
| 登录 | `/api/auth/token` |
| 拓扑 | `/api/topo-srv/...` |
| 设备管理 | `/api/ne-mgr/...` |
| 监控指标 | `/api/ne-monitor/...` |
| 告警/Syslog | `/api/sys-monitor/...` |

Python代码推荐在 `_get` / `_post` 方法统一加前缀：
```python
def _get(self, path: str, params=None):
    resp = session.get(f"{BASE_URL}/api{path}", ...)

def _post(self, path: str, body):
    resp = session.post(f"{BASE_URL}/api{path}", ...)
```

认证接口单独处理（已含 /api）：
```python
f"{BASE_URL}/api/uaa/api/v1/rsa/public"  # 公钥
f"{BASE_URL}/api/auth/token"              # 登录
```

---

## 完整Python示例

```python
import time, base64, requests, urllib3
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5

urllib3.disable_warnings()

ONC_HOST     = "10.1.7.1"   # 客户现场IP
ONC_USERNAME = "admin"
ONC_PASSWORD = "Ruijie@password"
BASE_URL     = f"https://{ONC_HOST}"
CLIENT_AUTH  = "Basic d2ViX2FwcDpyZ3Nkbg=="   # web_app:rgsdbn 固定值

session = requests.Session()
session.verify = False

def get_public_key():
    r = session.get(f"{BASE_URL}/uaa/api/v1/rsa/public")
    r.raise_for_status()
    return r.json()["publicKey"]

def rsa_encrypt(pub_key_b64, plaintext):
    pem = "-----BEGIN PUBLIC KEY-----\n" + pub_key_b64 + "\n-----END PUBLIC KEY-----"
    key = RSA.import_key(pem)
    encrypted = PKCS1_v1_5.new(key).encrypt(plaintext.encode())
    return base64.b64encode(encrypted).decode()

def get_token():
    pub_key = get_public_key()
    encrypted_pw = rsa_encrypt(pub_key, ONC_PASSWORD)
    r = session.post(
        f"{BASE_URL}/auth/token",
        headers={"Authorization": CLIENT_AUTH},
        data={
            "username": ONC_USERNAME,
            "password": encrypted_pw,
            "grant_type": "password",
            "encrypt": "true",
        }
    )
    r.raise_for_status()
    return r.json()["access_token"]
```

---

## 客户现场信息（xx项目）

| 项目 | 值 |
|------|----|
| ONC地址 | https://10.1.7.1|
| 用户名 | admin |
| 密码 | Ruijie@password |
| RSA公钥接口 | /uaa/api/v1/rsa/public |
| 登录接口 | /auth/token |
| client_auth（固定） | Basic d2ViX2FwcDpyZ3Nkbg== |

---

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `curl: (7) Failed to connect port 18080` | 18080是内部端口，外部不通 | 改用443，去掉端口号 |
| `SSL certificate problem` | 自签证书 | curl加`-k`，Python加`verify=False` |
| `invalid_scope: web-app` | scope名称错误 | 去掉scope参数或用`openid` |
| `invalid_grant` | 密码未加密 | 必须RSA加密后传入 |
| `invalid_client` | client_auth错误 | 用固定值`d2ViX2FwcDpyZ3Nkbg==` |