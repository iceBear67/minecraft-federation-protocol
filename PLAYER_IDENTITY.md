# 玩家身份验证协议

玩家身份验证协议用于去中心化的身份验证。此协议基于 1.20.5 引入的 transfer 及 cookies 特性实现。在实施此协议的联邦服务器中，玩家的身份总是由其携带的 Redirecion Token 决定，而无需基于 Yggdrasil 或密码为所有联邦服务器设置重复的验证。

## 前置条件

本协议基于以下前提：

- 联邦服务器持有私钥，且此联邦服务器的公钥对其他联邦服务器均已知。
- 需要交互的联邦服务器之间具备良好的网络连通性。
- 需要交互的联邦服务器之间应该有近似同步的时间，误差应该在半分钟内。

## Token

Token 由 Payload 及 Signature 构成，通常用于验证玩家的身份。Token 可被联邦中的任意服务器颁发

其结构为：
| Length of Payload | Payload | Length of Signature | Signature |
| -- | -- | -- | -- |
| VarInt | ByteArray | VarInt | ByteArray |

其中，Payload 使用 CBOR 编码，其包含：

| 名称 | 类型 | 说明 |
| -- | -- | -- | 
| issuer | byte[] | 该 token 颁发者的 ed25519 公钥 |
| target | byte[] | 目标服务器的 ed25519 公钥 |
| time | long | 签发时间 |
| uuid | byte[] | 玩家的 UUID |
| name | string | 玩家的名字 |
| subject | string | 该 Token 的用处 |
| ext | Extension[] | Redirection Token 的扩展信息 |


Signature 为 issuer 私钥对 SHA3-224(Payload) 的签名。