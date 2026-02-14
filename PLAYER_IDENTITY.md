# 玩家身份验证协议

玩家身份验证协议 (Player Identity Protocol, PIP) 基于 LDDP 实现去中心化的玩家身份验证。在实施此协议的联邦服务器中，玩家的身份总是由其携带的 Redirecion Token 决定，而无需基于 Yggdrasil 或密码为所有联邦服务器设置重复的验证。

## 前置条件

本协议基于以下前提：

- 联邦服务器持有私钥，且此联邦服务器的公钥对其他联邦服务器均已知。
- 需要交互的联邦服务器之间具备良好的网络连通性。
- 需要交互的联邦服务器之间应该有近似同步的时间，误差应该在半分钟内。
- 目标服务器支持使用 1.20.5 引入的 transfer & cookies 特性

## 协议概览

Redirection Token (RT) 是一种用于鉴定玩家身份的凭据，它同时还提供了对 RT 之外一同携带的数据的扩展验证功能。PIP 规定了如何使用 LDDP 实现 RT 以及如何通过 Minecraft 客户端的 cookies 功能储存 RT 以及如何对附加数据实现验证。尽管 PIP 协议要求使用 cookies 功能，实现者也可以在其他类似设施的基础上（如：自定义 Mod ）实现 PIP。

此外，PIP 还规定了对于接收 RT 的回执机制。回执机制可以让 RT 的签发者回收/清理玩家对应的资源，以免发生状态脱同步。

## Redirection Token (RT)

Redirection Token 是基于 LDDP 实现的票据。除 LDDP 中提到的各项字段外，RT 额外添加以下内容（字段类型由 [MessagePack Spec](https://github.com/msgpack/msgpack/blob/master/spec.md#bin-format-family) 规定）：

| 名称 | 类型 | 可缺省 | 备注 |
| -- | -- | -- | -- |
| action | str 8 | | LDDP 中的 action 必须固定为 `redirect` |
| reuse | bool | | 固定为 `false` |
| subject | bin 8 | | LDDP 中的 subject 必须为该 RT 对应玩家的 UUID |
| data | PlayerProfile | | 见下文 |

其中， `PlayerProfile` 包含以下字段：

| 名称 | 类型 | 可缺省 | 备注 |
| -- | -- | -- | -- |
| name | str 8 | | 玩家的游戏内 ID（aka），注意与 uuid 进行区分，此处指没有任何附加元素的游戏名 |
| texture | bin 32 | ✔ | 玩家的皮肤数据, _建议_ 长度不应超过 524288 (0.5MB) |
| cape | bin 32 | ✔ | 玩家的披风数据，_建议_ 长度不应超过 524288 (0.5MB) |
| extsign | map 16 | ✔ | 指示扩展字段，见下文 |

考虑到 RT 存放在客户端的 cookie 槽，而 cookie 本身只能储存约 5KiB 的数据。因此，为了减少 RT 本身的大小并且避免不必要的解析开销，实现者可以将体积较大的数据从 RT 中剥离出来，再使用 `extsign` 对他们进行验证。

`extsign` 的 key 为 `str 8`, key 用于表示扩展的 id. value 为 `ExtensionSignature`, 结构如下：

| 名称 | 类型 | 可缺省 | 备注 |
| -- | -- | -- | -- |
| issuer | bin 8 | | 对此扩展数据的担保人 (公钥) |
| sign | bin 8 | | issuer 对扩展数据的签名 |
| size | uint 32 | 扩展数据的长度 |

考虑扩展数据 D, `issuer` 对应的私钥为 `a`, 则签名算法如下：`S = Sign_a(SHA3-224(D))`

### 校验

实现在使用 RT 之前，应按照以下步骤对 RT 进行检查：

1. 按照 LDDP 中提到的办法检查。
2. 检查 `target` 是否是实现自己的公钥以防挪用。

若实现需要使用 RT 携带的附加数据，对于每一个被用到的扩展都需要按照以下步骤检查 `ExtensionSignature`:

1. 检查 `issuer` 是否已知
2. 读取附加数据 `D` 然后使用 SHA3-224 计算哈希值，再使用 [RFC 8032 Section 5.1.7](https://www.rfc-editor.org/rfc/rfc8032.html#section-5.1.7) 中的办法检验该签名的确来自 `issuer` 派生自的私钥

## Cookies 信道

为了通过玩家在服务器之间传递 LDDP，我们使用客户端的 [Cookie Store/Request](https://minecraft.wiki/w/Java_Edition_protocol/Packets?oldid=3410741#Cookie_Request_(configuration) 功能来存放数据。然而，Cookie 的大小上限仅有 5 KiB, 这使得我们需要规定切割数据的办法以使用 Cookie，本信道协议即用于解决此问题。

### Cookie 格式规范

封装格式（上为名称，下为类型，从左到右依次写入最终 Cookie 数据）：

| 魔数 | 数据 |
| -- | -- |
| ByteArray (2) | ByteArray |

其中 `ByteArray` 的定义详见 [传递协议中的数据定义](/INTERCONNECTION.md)。数据的大小可以从收到的 Cookie 数据长度中减去两字节魔数得出。

魔数的定义如下：

 - `CA FE`: 没有后续分段
 - `CA AC`: 有后续分段

所有欲设置的 Cookie 都必须对应一个独一无二的 `id`，id 满足模式 `namespace:value`, 其正则表达式为 `[a-z0-9.-_]+:[a-z0-9._/]+` (注意 `value` 中的减号被预留)。

考虑玩家即将携带的 Cookie 数据 `C` 以及其 Id `i`，应该将 `C` 按照如下规律进行封装:

1. 如果 `C` 的总长度小于或等于 5118，则设置魔数为 `CAFE` 并不再对数据做分段处理。
2. 否则，将数据按长度 5118 进行切割并从 0 开始编号（记作 `i`）。从 i = 1 开始的分段写入名为 `$id-$i` (如: `pip:inventory-1`) 的 Cookie 中，除了最后一项的魔数使用 `CA FE` 其他项均使用 `CA AC`。

切割的最后一项往往长度不足 5118。对于这种情况，实现需要在尾端填充 0 以确保所有分段按 5KiB 对齐。这是为了避免客户端在发送数据时的不确定性从而导致计算签名不一致，以及避免可能存在的拒绝服务攻击。

对端接受时，对于需要的数据 (id 为 `id`) 应按照以下步骤下载完整 Cookie:

1. 发送 `Cookie Request` 查询对应 `id` 的 Cookie 是否存在。
2. 检查客户端返回的 Cookie 的魔数是否为 `CA FE`
    - 如果是，直接读取魔数后的 5118 字节作为 `M` 并结束读取。
3. 检查客户端返回的 Cookie 的魔数是否为 `CA AC`
    - 如果是，则使用 `$id-$i` 继续读取每个数据包中除去魔数之外的前 `5118` 字节，直到魔数为 CA FE 为止。其中 `i` 是从 1 开始的自增整数。  

实现在读取分段时，应注意已读取的数据总量（除去魔数，包含填充）一定等于从上下文中获得的数据总量大小（如 `ExtensionSignature` 中规定的 `size`）。若没有上下文规定大小，则上限为 524288 Bytes (0.5M)。

我们建议实现者在 Configuration Phase 进行 Cookie 的查询以及运行相关逻辑。这是因为 Cookie 的核验与后文提到的回执系统实施较为繁琐，可能会花费一定时间。而客户端在进入 Configuration Phase 时已经可以接受心跳包防止掉线，且在 Configuration Phase 下玩家尚未正式进入服务器，可以给可能的数据错误/源服务器错误预留出空间。

## 回执

RT 的签发者可以目标服务器获得回执以确认 RT 被核销/玩家到达目标服务器。回执使用 SDDP 实现。

### 协议流程

 - 目标服务器接收到并校验了有效 RT
 - 目标服务器根据 RT 中规定的来源向来源服务器发起 `SDDP` 连接。
 - 目标服务器发送 `PlayerRedirectionArrived` 表示玩家已经抵达。
 - 目标服务器等待对应的 `PeerAcknowledge (0x04)`, 确定没有错误信息。
 - 目标服务器发送相同事务 Id 的 `PeerAcknowledge (0x04)` 表示已经收到回复，随后放行玩家。
 - 来源服务器若等待不到相同事务 Id 的 `PeerAcknowledge (0x04)` 则放弃冻结数据。

 
等待不应该超过 30 秒。如果网络连接状况不佳或无法建立 TCP 连接，应至少重试 3 次。
 
实现者应该考虑回执无法发送到源服务器的情况，并将结果提供给实现者的下游代码。下游有权根据回执情况来决定是否放行。

此外，对于需要保证数据一致性的情况，我们建议积极采用冻结而非删除数据的方式来避免状态不同步：

1. 如果玩家在获得 RT 后又进入了源服务器，那么撤销 RT 的有效性（不回答有效 `PeerAcknowledge`)
2. 在第一次 `PeerAcknowledge` 之后立即冻结玩家的数据，并且禁止加入服务器。
3. 如果来源服务器未能等待到第二个 `PeerAcknowledge`, 取消冻结数据。
 
### 封包参考

#### PlayerRedirectionArrived

`PlayerRedirectionArrived` (PRA) 是基于 `PeerPayload (0x03)` 传递的子协议。其参数如下：

| action | subject | 事务 ID | 数据 |
| -- | -- | -- | -- |
| `arrive` | 目标玩家的 UUID | 一个随机数 | RT 作为 LDDP 的 `signature` (64字节) |

## 建议

- 对于数据敏感的操作，总是请求来源服务器的 `PeerAcknowledge (0x04)`

如果 RT 被用于传输安全敏感的数据（如：玩家的背包），则协议应该总是要求来源服务器回应无错误的 `PeerAcknowledge (0x04)`，否则应该拒绝玩家加入服务器。


