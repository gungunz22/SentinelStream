 
---

## 完整使用示例（伪代码版）

本节提供从服务器收到消息，到 CRC 校验、AES 解密、字段提取的完整标准流程。

### 步骤总览

1. **收到原始Hex消息**
2. **拆分CRC校验码与数据体**
3. **执行CRC16-Modbus校验**
4. **提取命令ID与加密内容**
5. **AES解密加密内容**
6. **拼接完整解密消息**
7. **逐字段解析数据**
8. **输出结构化结果**

---

### 4.1 收到原始Hex消息

通常是从Socket、HTTP、MQTT等通道拿到的一段十六进制字符串。

```dart
String msg = "1300FE01FE01BA64......F8"; // 示例原始数据
```

---

### 4.2 拆分CRC校验码与数据体

- **最后4位**是**CRC校验码**
- **前面的部分**是**真正的加密数据体**

```dart
int length = msg.length;
String body = msg.substring(0, length - 4); // 主体
String crc = msg.substring(length - 4);     // CRC校验码
```

---

### 4.3 执行CRC16-Modbus校验

验证数据体的完整性。

```dart
if (CRC16Modbus.compute(body) != crc) {
  print("CRC校验失败");
  return;
}
print("CRC校验成功");
```

如果校验失败，则**中止处理**，保证数据安全。

---

### 4.4 提取命令ID与加密内容

- **前4位**：命令ID
- **后面**：加密的主体内容

```dart
String head = body.substring(0, 4);    // 命令ID
String content = body.substring(4);    // 加密内容
```

---

### 4.5 AES解密加密内容

使用 AES-192 + ECB 模式进行解密，默认跳过4字节的包头（0xFE01FE01）。

```dart
content = AESHelper.decryptHexString(content);
```

---

### 4.6 拼接完整解密消息

将命令ID与解密后的内容重新拼接成完整结构：

```dart
String msgFull = head + content;
```

---

### 4.7 逐字段解析数据

按协议解析每个字段：

| 起止位置 | 字段 | 说明 |
|:---|:---|:---|
| 0-4 | commend | 命令ID |
| 4-8 | text_s_Hex | 文本编号 |
| 8-12 | sizeHex | 音频大小（数据长度） |
| 12-22 | phoneHex | 电话号码 |
| 22-24 | typeHex | 音频类型 |
| 24之后 | audioHex | 音频内容 |

解析示例：

```dart
String commend = msgFull.substring(0, 4);
String text_s_Hex = msgFull.substring(4, 8);
String sizeHex = msgFull.substring(8, 12);
String phoneHex = msgFull.substring(12, 22);
String typeHex = msgFull.substring(22, 24);
String audioHex = msgFull.substring(24);
```

---

### 4.8 转成数字与数据列表

将解析出的16进制字符串转为对应的数值或字节列表：

```dart
int size = int.parse(sizeHex, radix: 16);        // 音频数据大小
int text_s = int.parse(text_s_Hex, radix: 16);    // 文本编号
int phoneNumber = int.parse(phoneHex, radix: 16); // 电话号码
int type = int.parse(typeHex, radix: 16);         // 音频类型
List<int> audioBytes = hexToBytes(audioHex);      // 音频数据
```

---

### 4.9 输出结构化结果

最终可以清晰地输出每一个重要字段：

```dart
print("=======解析结果=======");
print("命令ID: $commend");
print("文本编号: $text_s");
print("音频大小: $size");
print("电话号码: $phoneNumber");
print("音频类型: $type");
print("音频内容（list）: $audioBytes");
```

---
 