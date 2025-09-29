目标

- 说明在局域网内通过 UDP 进行认证，获取设备会话 AES 密钥的协议要点与数据格式（基于本仓库实现）。

通信与加密参数

- 传输: UDP 单播到设备 :80。
- 初始密钥: INIT_KEY = 097628343fe99e23765c1513accf8b02（16 字节，HEX）。
- 初始 IV: INIT_IV  = 562e17996d093d28ddb3ba695a2e6f58（16 字节，HEX）。
- 算法: AES-CBC；认证前使用初始密钥与固定 IV；认证成功后改用设备返回的会话 AES Key（IV 不变）。

外层数据包（UDP 载荷）

- 头部魔数: packet[0x00:0x08] = 5aa5aa555aa5aa55。
- packet[0x20:0x22]: 整包校验和（小端）。计算方法见“校验”。
- packet[0x22:0x24]: 响应错误码（小端，0 表示成功）。
- packet[0x24:0x26]: 设备类型 devtype（小端，来自发现或已知）。
- packet[0x26:0x28]: 指令类型 packet_type（小端）。认证为 0x0065。
- packet[0x28:0x2A]: 计数器 count（小端），发送前按 count = ((count + 1) | 0x8000) & 0xFFFF 递增。
- packet[0x2A:0x30]: 设备 MAC（倒序字节）。
- packet[0x30:0x34]: 设备 ID（小端）。认证前置 0，认证成功后由设备下发并沿用。
- packet[0x34:0x36]: 明文负载校验和（小端）。计算方法见“校验”。
- packet[0x38:]: 加密后的负载（16 字节对齐，AES-CBC）。

内层负载（认证请求，packet_type=0x65）

- 长度: 0x50 字节。
- payload[0x04:0x14] = 0x31 重复 16 次。
- payload[0x1E] = 0x01
- payload[0x2D] = 0x01
- payload[0x30:0x36] = "Test 1"（ASCII）
- 其余字节填 0。
- 发送前按 16 字节对齐后，用初始 AES/IV 加密。

认证响应（解密 response[0x38:] 后的负载）
- resp_payload[0x00:0x04]: 设备分配的 id（小端）。后续请求写入外层 packet[0x30:0x34]。
- resp_payload[0x04:0x14]: 会话 AES Key（16 字节）。收到后更新本地 AES（IV 不变）。
- 后续所有业务指令（packet_type=0x6A 等）均用该会话密钥加解密。

校验

- 负载校验 p_checksum（写入 packet[0x34:0x36]）:
    - 对“明文负载（未加密、含 padding 前还是后？实现为未加密未填校验位的实际 payload）”逐字节求和后加 0xBEAF，再
& 0xFFFF。
- 整包校验 checksum（写入 packet[0x20:0x22]）:
    - 组装好整包（含加密后的负载）后逐字节求和，加 0xBEAF，再 & 0xFFFF。
- 响应校验:
    - 取 resp[0x20:0x22] 与对响应数据同法计算的值比对；不一致视为校验错误。

交互流程（摘要）

1. 准备 AES=INIT_KEY/INIT_IV，id=0。
2. 构造认证负载（见上）并计算 p_checksum。
3. 组装外层包头（填 devtype、MAC 倒序、count、id=0、packet_type=0x65），负载 AES-CBC 加密后附加。
4. 计算整包校验并发送 UDP→设备:80。
5. 收到响应后校验→解密 resp[0x38:]→提取 id 与会话 AES Key。
6. 更新本地 AES Key；后续业务指令使用 packet_type=0x6A 与新的 id/会话密钥通信。

注意事项

- 开发与抓包时避免泄露会话密钥与 id。
- 认证失败时检查响应错误码（response[0x22:0x24]）与校验范围。
- 负载与整包校验均基于简单求和+0xBEAF，大小端按字段说明写入。
