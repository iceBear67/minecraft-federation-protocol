
# 服务器间数据传递

服务器间数据传递协议（传递协议）用于在联邦内服务器间传递数据并验证所传递数据的合法性，同时规定了数据传递的流程以及格式标准。

## 前置条件

本协议基于以下前提：

- 所有联邦服务器各自持有 32 位的 ed25519 私钥，且从此私钥派生的公钥对其他联邦服务器均已知。

## 协议概述

根据用途的不同，传递协议按有/无状态归类出两种变体：无状态数据传递协议 (stateLess Data Delivery Protocol, LDDP) 对每一个由源服务器发往目标服务器的数据进行签名封装，而有状态数据协议 (Stateful Data Delivery Protocol) 基于信道的连接状态实现可信信道。LDDP 用于服务器不直接相互通信的情况，而 SDDP 则是用于服务器之间直接进行通信的情况。

尽管 SDDP 只需要信道保证有序传输即可使用，为了设立 SDDP 实现的基准，本协议要求所有实现者必须支持基于 TCP / RFC 9293 传递数据。此外，本协议还提供了一个帮助 Minecraft 服务器复用游戏端口的协议扩展（STARTMFP）。我们建议实现者同时实现 STARTMFP，尽管这并不是必须的。

## 无状态数据传递协议

无状态数据传递协议 (stateLess Data Delivery Protocol, LDDP) 对每一个由源服务器发往目标服务器的数据 (封包) 进行签名封装。它适用于服务器间间接通信或一次性报文的情况。LDDP 仅规定了数据封装的格式而并非传输的办法。  

LDDP 可以用在所有用于需要鉴权或是传递实时性不高的数据的地方，比如玩家的票据，回执等。

### 数据封装

LDDP 中传递数据的基本单位为封包。LDDP 完全使用 [MessagePack](https://msgpack.org/) 封装数据。MessagePack (MsgPack) 是一种相比 JSON 更紧凑且无需预先约定格式的二进制编码格式。考虑到 LDDP 的用途，无需编码的特点有助于 LDDP 在未来继续扩展，而紧凑的字节序列化也允许 LDDP 高效地存储从 Minecraft 中导出的 NBT 数据。

### 约定字段

LDDP 规定，封包可能包含以下字段（字段类型由 [MessagePack Spec](https://github.com/msgpack/msgpack/blob/master/spec.md#bin-format-family) 规定）：

| 名称 | 类型 | 可缺省 | 备注 |
| -- | -- | -- | -- |
| version | uint8 |  | 常数 1，用于标记版本号 |
| issuer | bin 8 |  | 签发者的公钥 |
| target | bin 8 |  | 此封包发送目标的公钥 |
| time | uint 64 |  | 此封包的签发时间（UNIX时间戳，秒） |
| until | uint 64 |  | 此封包的过期时间 （UNIX时间戳，秒）|
| reuse | bool |  | 此封包是否可以被重复核验 |
| action | str 8 |  | 此封包的用途 |
| subject | bin 8 | ✔ | 此签名所属的对象。通常为玩家的 UUID |
| data | bin 32 | ✔ | action 所对应的数据 |
| signature | bin 8 |  | 此封包的签名 |

`signature` 使用 [RFC 8032 Section 5.1.6](https://www.rfc-editor.org/rfc/rfc8032.html#section-5.1.6) 中描述的签名办法计算，其中参数为：发送者的 ed25519 私钥 以及 签名数据 `M = version || target || time || until || reuse || action || subject || SHA3-224(data)`

`||` 表示字节数组的拼接。签名数据中的序列化规则如下：

 - 数字类型均采用大端序。
 - `uint8` 使用一个字节。
 - 布尔类型使用 0 表示 `false`, 1 表示 `true`。
 - 对于没有出现在封包里的可缺省数据，计算签名数据时跳过即可。
 - 对于字符串 (action) 的序列方式，依照上文数据定义中在序列化前加入长度前缀进行处理以避免歧义。

### 数据校验

在使用封包中的数据前，LDDP 的实现者必须依照以下规则检查其合法性：

1. `version` 为 1
2. 检查 `time < until` 且现在的时间戳 `NOW` 满足 `time < NOW < until`。
3. 所有不可缺省字段均存在
4. 确定 `issuer` 是已知公钥列表中的一员，即可以根据 `issuer` 关联到对应被授权联邦服务器。
5. 计算签名数据 `M` 并使用封包中的公钥 `issuer` 检验该签名的合法性。
6. 若 `reuse` 为 false 则检查 `signature` 是否已经被使用过。若没有被使用过，则记录该 signature；否则封包无效。

如果有任意一条不满足，拒绝承认其有效性。对于被记录的 `signature` ，记录的销毁时间至少应该在 `until` 之后，且此记录必须能够存活在服务器重启之间存活（持久化）。

## 有状态数据传递协议

有状态数据传递协议在信道上进行身份验证，随后将通信对端的身份与信道连接绑定来避免频繁签名的开销。它适用于服务器之间通信条件良好或对实时性要求较高的情况。

此子协议在 「前置条件」 下额外附加几条条件：

- 需要交互的联邦服务器之间具备良好的网络连通性。
- 需要交互的联邦服务器之间应该有近似同步的时间，误差应该在半分钟内。

### 封包

本协议中发送/接受数据的基本单位是封包。封包是一种按照约定的顺序（表格左到右）依次序列化进字节数组以待发送的数据。

在下文中所指的封包格式如下（表头为名称第一行为数据类型，第二行为备注）：

| 长度 | 封包 Id | 数据 |
| -- | -- | -- |
| VarInt | VarInt | ByteArray |
| 除长度本身之外，剩余部分的长度 | 封包的类型 ID | 封包的实际数据 |

所有封包均使用此格式进行封装。其中 `数据` 又根据 `明文封包` 和 `密文封包` 而有区分，详见下文。  
根据网络情况的不同，封包可能会被信道进行拆分重组甚至丢包重传。因此，实现者应根据 `长度` 从数据流中分割出封包，避免可能存在的数据损坏问题。为了安全考虑，单个封包有效数据（除去 `长度` ）的长度不应超过 5,242,880 个字节 (5MiB)。对于可能过长的封包，请积极在协议设计中采用分段上传的办法。

### 数据类型

此小节包含一系列可能在此协议规范中出现的数据类型的名称以及其编解码方式。  

需要注意的是，除了 VarInt 和 VarInt64 之外的所有数据均使用**大端序（Big-Endian）**，即数据的发送顺序总是从最高有效字节开始到最低有效字节结束。

#### VarInt 和 VarInt64

VarInt/VarInt64 为可变长(长)整数的缩写。作为 Minecraft 内部使用的一种数据类型，它们类似于 [Protocol Buffer](https://protobuf.dev/programming-guides/encoding/#varints) 中的 Base 128 Varints。本协议中描述的 VarInt 为 Prrotocol Buffer 中的 `int32` / `int64` (for VarInt64) 变种，并非使用 ZigZag 编码的 `sint32` / `sint64`.

每个 VarInt 的长度不低于 1 个且不超过 5 个字节，并且 VarInt 使用小端序排列。其每个字节的最高有效位为 1 时表示 VarInt 没有结束，应往后读一个字节。VarInt64 则是长度不低于 1 个，不超过 10 个字节，其他规则与 VarInt 一致。

具体的编码细节可以查看 [Minecrat Wiki 上的对应章节](https://minecraft.wiki/w/Java_Edition_protocol/Packets?oldid=3410741#VarInt_and_VarInt64)

#### ByteArray

ByteArray 指字节数组，它的长度只能从上下文中获取。

在其后标注数字表示此为固定长度的字节数组，如 `ByteArray (8)` 表示这是一个 8 字节的字节数组。

#### Prefixed ByteArray

Prefixed ByteArray 指包含长度前缀的 ByteArray.

| 长度 | 数据 |
| -- | -- |
| VarInt | ByteArray |

#### Unsigned Short

Unsigned Short 指无符号的 16 位整数

#### String
https://minecraft.wiki/w/Java_Edition_protocol/Packets?oldid=3410741#VarInt_and_VarInt64  
储存标准 UTF-8（不使用 Java 变种） 格式字符串的 Prefixed ByteArray。

| 长度 | 数据 (UTF-8) |
| -- | -- |
| VarInt | ByteArray |

#### Int32/Int64

指有符号（补码）的 32/64 位整数

#### Bool

指布尔值。布尔值使用单字节 0 表示 false, 1 表示 true.

### 协议流程

为了实现不可抵赖性以及在不安全信道上可靠地传递数据，SDDP 强制使用加密。尽管数据传递协议并没有明确的客户端-服务端角色，我们仍需要使用客户端 C 与服务端 S 来分别指代信道中的发起连接和接受连接的两端。在实践中，二者并无实际区分。

若欲通信的两方之间已经从任意一方开始建立了可信信道，则无需再次建立信道，直接使用已建立的信道通信即可。否则，依照如下的步骤进行：

1. C -> S: 建立信道连接。
2. C -> S: C 生成本地临时 x25519 私钥 `k_C` 并导出公钥 `K_C`，随后发送 `PeerAuthHello`，其中包含 C 的被签名临时公钥 `K_C`。
3. S -> C: S 生成本地临时 x25519 私钥 `k_S` 并导出公钥 `K_S`，随后发送 `PeerAuthHello`，其中包含 S 的被签名临时公钥 `K_S`。

在双方的临时公钥均已送达后，即可通过 ECDH 计算共享秘密 `S = X25519(k_C/S, K_S/C)` (参见 [RFC 7748 Section 6.1](https://www.rfc-editor.org/rfc/rfc7748.html#section-6.1) )。随后，使用 `S` 计算出最终加密参数：

> PRK = HKDF-Extract(0, shared_secret)  
> client_AES_key, client_IV = HKDF-Expand(PRK, b"sddp client")  
> server_AES_key, server_IV = HKDF-Expand(PRK, b"sddp server")  

HKDF-Extract 和 HKDF-Expand 详见 [RFC 5869 Section 2.2](https://datatracker.ietf.org/doc/html/rfc5869#section-2.2)

随后，双方可以开始发送密文封包（见下文）。

4. C -> S: 发送 `PeerPing` 
5. S -> C: 发送 `PeerPong`，握手协议结束。

在握手协议结束后，信道两端使用 `PeerPayload (0x03)` 进行数据传递。关于具体的传输细节，请参考下文中对应封包的说明。

### 封包类型参考

有状态数据传递协议规定了一组封包，大体分类如下：

- 明文封包：封包的 `数据` 段没有额外封装
- 密文封包：密文封包只在加密后发送，并且遵循如下的封装规定：

| 包序号 | AEAD Tag | 密文 |
| -- | -- | -- |
| Int32 | Prefixed ByteArray | ByteArray |

包序号从 0 开始，每发一个数据包则递增，一直到 `2^31 - 1` 之后回归 1。协议对端应该记录最近一次接收到的合法封包的包序号 `R` 。

实现者每收到一个密文封包，都应该按以下规则进行检查：

1. 若接收到的封包包序号 `N` 不满足 `N == (R+1) % (2^31 - 1)` 则拒绝封包并断开连接 (`PeerDisconnect (0x05)`)。  
2. 尝试解密密文 `M`, `M = AES_GCM_Decrypt(AES_key, IV, 密文, AEAD Tag, 包序号)`。若解密或 AEAD 校验失败，则断开连接 (`PeerDisconnect (0x05)`)。

其中 `AES_key`, `IV` 的计算方式详见协议流程。SDDP 为了防止 IV 碰撞导出了两套 AES 的参数。因此为了解密来自服务端的流量，客户端应该使用 `server_IV` 和 `server_AES_key`，服务端同理。至于封包的加密方式也不再赘述。

#### PeerAuthHello (0x00)

此封包设计用于有状态数据传递协议，且必须依照明文封包格式封装。

| 本地公钥 | 目标公钥 | 本地临时公钥 | 当前 UNIX 时间 | 签名 |
| - | - | - | - | - |
| ByteArray (32) | ByteArray (32) | ByteArray(32) | Int64 | ByteArray (64) |
| | | | 必须是正整数，否则断开连接 (`PeerDisconnect (0x05)`) |

SDDP 使用 Curve25519 进行密钥协商（ECDH）。因此在发送 PeerAuthHello 前，发送者应该在本地生成 32 位随机字节 (称作: `a`) 作为 x25519 使用的私钥，并通过 [RFC 7748 中提到的方法](https://www.rfc-editor.org/rfc/rfc7748.html#section-5) 计算公钥 `A=X25519(a, 9)`。

`签名` 使用 [RFC 8032 Section 5.1.6](https://www.rfc-editor.org/rfc/rfc8032.html#section-5.1.6) 中描述的签名办法计算，

其中参数为：发送者的 ed25519 私钥 以及 签名数据 `M = 目标公钥 || 本地临时公钥 || 当前 UNIX 时间`

任意一端接收到 `PeerAuthHello` 后，应依据以下办法对签名进行检查：

1. 检查封包中 UNIX 时间的有效性，时间应该在当前主机 UNIX 时间的 +/- 30 秒内（**UNIX 时间标准以秒为基本单位，而非毫秒**）
2. 按照上文中给出的办法计算签名数据，并使用 [RFC 8032 Section 5.1.7](https://www.rfc-editor.org/rfc/rfc8032.html#section-5.1.7) 中的办法检验该签名确实属于来源提供的 "本地公钥"

否则拒绝承认封包的有效性并断开连接 (`PeerDisconnect (0x05)`)。

#### PeerPing (0x01)

此封包设计用于有状态数据传递协议，且必须依照密文封包格式封装。

没有数据。  

在收到 `PeerPing` 时，实现必须往对端发送 `PeerPong` 表示响应。

#### PeerPong (0x02)

此封包设计用于有状态数据传递协议，且必须依照密文封包格式封装。

没有数据。  

#### PeerPayload (0x03)

此封包设计用于有状态数据传递协议，且必须依照密文封包格式封装。

| action | subject | 事务 ID | 数据 |
| -- | -- | -- | -- |
| String | Prefixed ByteArray | VarInt | Prefixed ByteArray |
| 动作 | 此动作关联的主体，如玩家 UUID | |

`action` 和 `subject` 意义与 LDDP 中的一致。实现端应该对 LDDP 和 SDDP 中的 `action` 有感知，不建议混淆 LDDP 和 SDDP 的处理（除非设计使然）。

如果此封包 `事务 ID` 不为 0, 则在封包处理完毕后必须回应 `PeerAcknowledge (0x04)`。

#### PeerAcknowledge (0x04)

此封包设计用于有状态数据传递协议，且必须依照密文封包格式封装。

| 事务 ID | 错误信息 |
| -- | -- |
| VarInt | String |

用于表示某一封包已被处理。其中，事务 ID 来自 `PeerPayload (0x03)`, 而错误信息在没有发生错误时应该留空。

#### PeerDisconnect (0x05)

此封包设计用于有状态数据传递协议，且必须依照密文封包格式封装。

| reason | message |
| -- | -- |
| VarInt | String |

此包发送后应该立即断开连接。其中，当 `reason` 为 1 时表示重新协商密钥，此时被连接的对端（服务端）需等待客户端重新连接即可。

为了确保安全期间，我们建议实现者在包序号溢出（即恢复到 1 时）重新创建连接。

## SDDP 协议扩展：STARTMFP

为了贴合 Minecraft 服务器的情景，本扩展协议提供了基于 【Minecraft: Java Edition 协议](https://minecraft.wiki/w/Java_Edition_protocol/Packets) 的端口复用方案。

在 SDDP 开始之前，向目标端口发送 Intent 为 127 [ServerboundHandshake (0x00)](https://minecraft.wiki/w/Java_Edition_protocol/Packets?oldid=3410741#Handshake) 后即可直接开始 SDDP 流程。

实现者应该提供选项以供在 STARTMFP 和直接 SDDP 之间选择。此外，STARTMFP 基于 `ServerboundHandshake` 的封包格式，并不利于 Minecraft 以外的第三方程序使用。
