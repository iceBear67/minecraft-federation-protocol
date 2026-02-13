# Minecraft: Java Edition Federation Protocol (MCFP)

此仓库存放有关 MCFP 相关的协议标准。MCFP 是一个用于描述特定条件下的 Minecraft: Java Edition 服务器之间（联邦）如何去中心化的传递数据，验证身份以及玩家如何与服务器交换信息的协议。

当前状态：草案

文档列表：
 - [服务器间数据传递协议](./INTERCONNECTION.md)
 - [玩家身份认证协议 (WIP)](./PLAYER_IDENTITY.md)
 - [威胁模型（WIP）](./THREAT_MODEL.md)

# 定义

## 联邦

联邦是在各服务器管理员的协作下将若干 Minecraft: Java Edition 服务器组合而成的整体，通常也可指代所有管理员形成的组织。本仓库中的所有文档所代指的联邦均为服务器组合体，同时使用管理员指代某个或特定服务器的管理员（依据上下文）。

## 服务器

服务器特指 Minecraft: Java Edition 服务器。本文档中 Minecraft: Java Edition 服务器以联邦中的所有玩家是否允许且可以连接该端口作为评判标准，即能否完成 TCP 握手。

此处为一些常见情况：

 - 通过 Velocity / BungeeCord 等代理服务器将服务器聚集起来暴露成整体的，算作一个服务器。
    - 若被代理服务器并未明确单独提供服务，则不算作一个服务器。
 - 通过 transfer 等技术实现跳转的，每一个可跳转的目标均算作一个服务器。
 - 直接提供服务的 Notchian（即基于 Vanilla 的) 服务器也算作一个服务器。

## 联邦服务器

指联邦内服务器。

