# 验证网关规范

去中心化验证网关（验证网关）用于为所有联邦服务器提供基于 [Redirection Token](/PLAYER_IDENTITY.md) 的验证。此协议文档规定验证网关如何通过 Yggdrasil 对玩家进行鉴权、如何响应扩展安全协议以帮助客户端实现安全通信、联邦服务器如何跟验证网关通过 Profile Store 交互。

## 前置条件

 - 所有联邦服务器各自持有 32 字节的 ed25519 私钥，且所有从此私钥派生的公钥对验证网关均已知。
 - 验证网关自身持有 32 字节的 ed25519 私钥，用于签发 Redirection Token；其对应公钥对所有联邦服务器均已知且受信任（即作为 RT `issuer` 时能通过各服务器的 ODP 校验）。
 - 若要通过档案馆与验证网关交互，联邦服务器需要与档案馆之间有良好的网络连通性。
 
## 协议概述

验证网关是一种支持分布式部署的服务端软件，它通过实现 Minecraft: Java Edition 的网络协议并使用 [Yggdrasil](https://github.com/yushijinhun/authlib-injector/wiki/Yggdrasil-%E6%9C%8D%E5%8A%A1%E7%AB%AF%E6%8A%80%E6%9C%AF%E8%A7%84%E8%8C%83) 验证来验证玩家的身份，随后通过 RT 来允许玩家以特定身份加入联邦服务器。

本协议定义了验证网关为了实现验证系统需要实现的能力，并且提供了基于通过分享公钥列表的第三方 Mod 协议来修复在 RT 鉴权体系内 「客户端 -- 联邦服务器」之间流量可被窃听的问题。

## 验证网关

验证网关在 Login 阶段对玩家使用 Yggdrasil 进行验证，并在 Configuration 阶段颁发 Redirection Token 然后 Transfer 玩家到对应的服务器。此小节主要描述验证网关在各种条件下的行为。

"Handshake", "Login", "Configuration" 指代 Minecraft 协议流程中的不同阶段。请注意：不同阶段中的数据包不能混合发送，否则会出现不可预料的后果。

### 验证

1. (Handshake) 确认玩家的 Intent 为 `Login`，否则不进入以下流程。

2. (Login) 玩家的 ID 应该符合此正则表达式：`^[a-zA-Z0-9_]{3,16}$`

不符合此 ID 的玩家应该拒绝加入服务器。

3. (Login) 对玩家进行 Yggdrasil 验证

首先，并非所有尝试加入的 Minecraft 玩家都需要被 Yggdrasil 验证。通常。离线玩家的 UUID 总是内容为 `MD5("OfflinePlayer:"+name)` 的 3 型 UUID, 其中 `name` 是玩家的 ID。对于被提前辨别出的离线玩家，我们 _建议_ 实现者将他们 Transfer 到专用的游戏内账户密码登录服务器以进行身份验证。

Yggdrasil 验证允许验证网关使用多个 Yggdrasil 后端。但请注意， **验证网关以及联邦仅以 UUID 为唯一身份标识** ，尽管 UUID 对应的档案中玩家名或来源的 Yggdrasil 服务器可能发生重叠。 

对于 Yggdrasil 验证失败的玩家，实现者应拒绝他们加入服务器或将他们 Transfer 到回落服务器内。

4. (Configuration) 决定玩家的目的地

验证网关需要查询 [档案馆](#档案馆) 中的玩家信息以寻找上次加入的服务器（详见 [玩家网关档案](#玩家网关档案)）。除此之外，验证网关可以通过在 Handshake 阶段时给出的 Host 进行分流

如果不能确定目的地，请拒绝玩家加入或提供回落服务器。

5. (Configuration) 签名 Redirection Token 并通过 [Cookie 信道](./PLAYER_IDENTITY.md)下发。

6. (Configuration) 对玩家发送 Transfer 以发送到目标服务器。

7. (Configuration) 断开底层 TCP 连接（CLOSE）

### 转发

MCFP 的分布式特性允许玩家在服务器之间自由移动而不需要中央服务器。然而，对于某些需要保证状态一致的情景，玩家加入一个 "意料之外的" 服务器 (比如: 携带物品 redirect 出发时的服务器) 是不可接受的。因此，MCFP 提供了两种方案：转发 和 服务器之间自行实现追踪。

此小节即为 "转发"。在玩家由来源服务器前往目标服务器前，会经过验证网关以记录状态。

1. (Handshake) 确认玩家的 `Intent` 为 `Transfer`，否则不进入以下流程。

2. (Login) 玩家的 ID 应该符合此正则表达式：`^[a-zA-Z0-9_]{3,16}$`

不符合此 ID 的玩家应该拒绝加入服务器。

3. (Configuration) 通过 [Cookie 信道](./PLAYER_IDENTITY.md) 请求并检验玩家的 Redirection Token

与 [验证](#验证) 不同，在转发时验证网关不需要关心玩家是否能够通过 Yggdrasil 验证。

如果 Redirection Token 不能被检验，应该断开玩家的连接。

4. (Configuration) 更新玩家的 [玩家网关档案](#玩家网关档案) 中的相关条目。

5. (Configuration) 把玩家 Transfer 到 Redirection Token 中的目的地。

### 玩家网关档案

为了在验证网关之间追踪玩家所处服务器之状态，验证网关通过档案馆维护 "玩家网关档案" 以实现目的。

关于档案馆，详见 [档案馆](#档案馆)

玩家网关档案使用命名空间 `mcfp:gateway_profile`。其档案内数据定义如下：

| 名称 | 类型 | 是否可缺省 | 含义 |
| -- | -- | -- | -- |
| last_joined_at | number | ✔ | 上次加入服务器的时间 |
| server | string | ✔ | 最近一次连接的服务器 |

在需要保证状态一致的情况下，对于服务器之间直接进行的 Transfer, 服务器（与目标服务器）有必要及时更新玩家的网关档案。此外，也可以交给验证网关处理（转发）。

## 安全连接

玩家身份验证协议建立了去中心化的验证系统，从而也将原先以 Yggdrasil 作为权威的第三方就此抹除，这使得 [协议加密](https://minecraft.wiki/w/Java_Edition_protocol/Encryption) 不再能保证玩家与联邦服务器之间的通讯内容无法被窃听。为了恢复客户端与联邦服务器间通信的保密性，安全连接 对 Minecraft 的协议加密部分进行修改，从而在保持原版客户端兼容的同时使用 Mod 解决此问题（我们认为，Minecraft 的协议加密流程在短时间内不太可能有大幅度的变化，因此 安全连接 对于未来甚至较老版本的 Minecraft 也是适用的）。

需要注意的是，安全连接 协议仅支持 Minecraft 1.20.5 以及更新的版本。

### 对验证网关的修改

为了支持客户端对服务器公钥的验证，验证网关借助 Yggdrasil 作为权威验证与客户端连接的有效性，并发送联邦中所有服务器的公钥。

> [!IMPORTANT]
> 
> 一旦验证网关发送公钥，客户端 Mod 将总是检验 Transfer 过程中连接的服务器的公钥。因此，在使用前需要确保联邦中的所有服务器均已支持 「安全连接」 协议。

1. (Configuration) 对玩家发送所有已知联邦服务器的公钥

此步骤通过 [Clientbound Plugin Message (Configuration)](https://minecraft.wiki/w/Java_Edition_protocol/Packets#Clientbound_Plugin_Message_(configuration)) 实现。其中，我们使用 `mcfp:gateway_server_keys` 作为 Channel。其 Payload 为一个长度可被 32 整除的字节数组，解析方式如下：

```python
size = 32

def get_keys(raw_payload: bytes) -> list[bytes]:
    l = len(raw_payload)
    n = l % size
    if l == 0 or l % size != 0:
        return [] 
    return [raw_payload[i:i+size] for i in range(0, l, size)]
```

此步必须在 Transfer 玩家之前实施。

### 安全连接协议

此小节描述了安全连接实现者的行为。本小节中使用的封包以及字段类型均来自 https://minecraft.wiki/w/Java_Edition_protocol/Packets?oldid=3447129#Encryption_Request 

1. 联邦服务器向客户端发送 `Encryption Request (Login)`

Encryption Request 的结构如下：

| 名称 | 类型 | 备注 |
| -- | -- | -- |
| Server Id | String (20) | 总是为空, 我们不使用此字段。 |
| Public Key | Prefixed ByteArray | 服务器原先用于协议加密的 RSA 公钥（记作 `sk1` ）, 注意区分 |
| Verify Token | Prefixed ByteArray | 被安全连接用于证明，原先为随机数据 |
| Should Authenticate | Boolean | 总是为 false, 联邦服务器不需要关心 Yggdrasil |

其中，Verify Token 的结构如下：

| 魔数 | 联邦服务器公钥 | 签名数据 |
| -- | -- | -- |
| Int32 | Byte Array (32) | Prefixed ByteArray |
| 固定为 `18108736` | 连接目标联邦服务器的公钥 | 联邦服务器私钥对 `sk1` 的签名数据 |

2. 如果客户端 Mod 没有在经过验证网关时收到 `mcfp:gateway_server_keys`, 则就此结束。

否则，按照以下步骤严格检验所有收到的 Encryption Request, 并且拒绝在没有 Encryption Request 的情况下直接进入下一阶段（Configuration）：
 - 检查 `Verify Token` 中的魔数是否正确，否则断开连接。
 - 使用 `Verify Token` 中携带的「联邦服务器公钥」按 [RFC 8032 Section 5.1.7](https://www.rfc-editor.org/rfc/rfc8032.html#section-5.1.7) 验证「签名数据」确为对本次 `Encryption Request` 中 `Public Key`（`sk1`）的签名，否则断开连接。
 - 确认该「联邦服务器公钥」确实存在于此前收到的 `mcfp:gateway_server_keys` 列表中（即连接到的是联邦成员），否则断开连接。
 - 继续进行协议流程。

## 档案馆

验证网关作为去中心化的服务端软件，其本身并不存储状态。相反，它们从符合 Yggdrasil 规范的服务端中下载玩家的 [GameProfile](https://minecraft.wiki/w/Java_Edition_protocol/Data_types#Game_Profile)，再从档案馆中下载玩家 UUID 对应的附加数据。而附加数据通常是由联邦服务器直接添加/操作的。

本小节规定了所有对档案馆用户可用的 API 以及它们的行为，并且提供了对于使用档案馆的一般性建议。

### GET /api/player/:uuid

获取某一 UUID 对应的玩家在档案馆中的数据。

| 参数名 | 备注 |
| -- | -- |
| uuid | 玩家的 UUID, 格式类似 `4577e3ec-d66c-4c20-904c-18310d9fdec7`（有横杠） |

请求 Header:

| Header | 值 |
| -- | -- |
| Accept | 固定为 `application/json` |

响应头：

| Header | 值 |
| -- | -- |
| Content-Type | 固定为 `application/json; charset=utf-8`, 服务端也需要遵循此约定 |
| Last-Modified | 对应档案上次修改时间 |

响应体由多个命名空间-记录对构成。示例：

```json5
{
    "namespace": {
        "data": ".......",
        "signature": "......",
        "modified_at": 0
    }
}
```

记录对的定义如下：

| 字段 | 备注 |
| -- | -- |
| data | 由服务端上传的原始数据（通常为转义后的 JSON 文本）|
| signature | 服务端上传数据时提供的对此数据的签名 |
| modified_at | 此数据被修改时的 UNIX 时间 |

关于「命名空间」名称格式的规则，见下文。

API 响应状态码：

| 状态 | 情况 |
| -- | -- | 
| 200 | 正常获取数据 |

当不存在记录时，返回空 JSON Object `{}`。

### GET /api/player/:uuid/:namespace

获取某一 UUID 对应的玩家在档案馆中的对应命名空间的数据

| 参数名 | 备注 |
| -- | -- |
| uuid | 玩家的 UUID |
| namespace | 对应的命名空间 ID |

命名空间满足此正则表达式：`[a-z0-9._-]+:[a-z0-9._/]+`

UUID 的格式同上。

请求 Header:

| Header | 值 |
| -- | -- |
| Accept | 固定为 `application/json` |

响应头：

| Header | 值 |
| -- | -- |
| Content-Type | 固定为 `application/json; charset=utf-8`, 服务端也需要遵循此约定 |
| Last-Modified | 对应档案上次修改时间 |

响应体与 [`GET /api/player/:uuid`](#get-apiplayeruuid) 中的单个记录对格式一致，即返回该命名空间的 `data`、`signature`、`modified_at` 三元组，以便调用方在单独拉取某一命名空间时也能验证签名。示例：

```json5
{
    "data": ".......",
    "signature": "......",
    "modified_at": 0
}
```

尽管任意符合上述正则表达式的字符串都是合法的命名空间，但我们仍建议联邦服务器在使用档案馆服务时，遵循以下规范（命名空间规范）：

1. 命名空间应由两部分组成（`part1:part2`），我们建议 part1 使用关联于联邦服务器的唯一标识，并且避免使用保留命名空间 `federation` 与 `mcfp` 。
2. 使用人类可读的名称。若名称由程序生成，请添加必要的前缀或后缀以供识别。

API 响应状态码：

| 状态 | 情况 |
| -- | -- | 
| 200 | 正常获取数据 |

当不存在记录时，返回空 JSON Object `{}`。

### PUT /api/player/:uuid/:namespace

更新某一 UUID 对应的玩家在档案馆中对应命名空间的数据。

| 参数名 | 备注 |
| -- | -- |
| uuid | 玩家的 UUID |
| namespace | 对应的命名空间 ID |

UUID 与命名空间的约束如上。

请求 Header:

| Header | 值 |
| -- | -- |
| Accept | 固定为 `application/json` |
| X-Signature | 发布者对 body 的签名 |
| Content-Length | 请求体的总长度，为 `0` 时表示删除数据条目 |

此外，实现者需要实现 `If-Modified-Since` 和 `If-Unmodified-Since` 以允许简单事务性更新（详见 [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html#name-preconditions)）。

`X-Signature` 是对 POST 请求体中原始数据的签名（即字节流）。其计算方式如下：

```python
import time
from base64 import b64encode

def calc_sign(privateKey, publicKey, data: bytes) -> str:
    now = int(time.time())
    # now 以 8 字节大端序编码后与 data 拼接再签名
    message = now.to_bytes(8, "big") + data
    sub_sign = b64encode(ed25519_sign(privateKey, message)).decode()
    return f"{now}-{sub_sign}-{b64encode(publicKey).decode()}"
```

其中 `ed25519_sign(privateKey, message)` 是 [RFC 8032 Section 5.1.6](https://www.rfc-editor.org/rfc/rfc8032.html#section-5.1.6) 中描述的签名方法，被签名的 `message` 为 `now`（8 字节大端）与请求体原始字节 `data` 的拼接。`X-Signature` 由三段以 `-` 连接构成：`now` 的十进制文本、签名的 Base64、发布者公钥的 Base64（标准 Base64 字母表不含 `-`，故不产生歧义）。校验方须用第一段的 `now` 重建被签名的 `message` 并用第三段的公钥验签，同时应校验 `now` 与当前时间的偏差以防重放。

请求 Body:

任意合法 JSON。实现者应该确保至少能够上传 `32768` 字节 (32KiB) 的数据而不响应 `413 Entity Too Large`

API 响应状态码：

| 状态 | 情况 |
| -- | -- | 
| 413 | 上传的数据过大 |
| 403 | 无法校验或没有提供合法的 `X-Signature` |
| 200 | 上传成功 |
| 412 | Precondition Failed (参见 RFC 9100) |
