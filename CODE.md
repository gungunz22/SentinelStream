# 高配方案：数据帧解码完整流程

本文档详细描述“高配”版本数据帧的完整解码流程，包括字段解析、Hex 与 Base64 逆向，及最终数据恢复，并附带所有依赖的工具类实现。

---

## 一、数据帧格式

| 字段         | 字节数   | 内容说明                                 |
|------------|-------|--------------------------------------|
| 前缀         | 1     | 固定值 `0x10`，标识高配音频+文本消息               |
| text\_sHex | 4     | 文本长度字段（十六进制 Big-Endian），表示 UTF-8 字节数 |
| sizeHex    | 4     | 播放时长字段（十六进制 Big-Endian），单位：毫秒        |
| audioHex   | 剩余字节数 | 音频数据：每 16bit 一组，先 Hex 表示后 Base64 编码  |

**示例（Hex 串未 Base64 前）：**

```hex
10 0000000C 000003E8 <audioHex...>
```

上述示例表示 `text_s` 长度 0x0000000C（12 bytes），`size` 0x000003E8（1000 ms），后面是音频数据。

---

## 二、解码总流程

1. **读取并校验前缀**

    * 从数据帧第 1 个字节读取，验证是否为 `0x10`。

2. **提取并解析 `text_sHex`**

    * 取第 2–5 字节，按大端序拼成 4 字节 Hex 字符串。
    * 调用 `HexReverse.hexToLong` 获取文本长度 `textLen`。

3. **提取并解析 `sizeHex`**

    * 取第 6–9 字节，按大端序拼成 4 字节 Hex。
    * 调用 `HexReverse.hexToLong` 获取时长 `durationMs`（毫秒）。

4. **提取并还原 `text_s`**

    * 从第 10 字节开始，截取 `textLen` 字节，按 UTF-8 解码为字符串 `text_s`。

5. **提取并解码 `audioHex`**

    * 从第 (10 + textLen) 字节到结束，读取所有 Base64 字符串 `audioB64`。
    * 使用 `CodecUtils.fromBase64ToBytes(audioB64)` 解 Base64 得到原始二进制。
    * 将二进制转 Hex 字符串后，调用 `Hex16Reverse.hexToU16LongList` 还原为 PCM 数据列表。

---

# 模拟数据

```shell
高配 原始数据 audio [813, 5349, 1026, 7346]
高配 原始数据 audio 的长度4
高配 原始数据 size 240
高配 原始数据 text_s 6884
高配 中间数据 audioHex : 032D14E504021CB2
高配 中间数据 audioHex 的长度 : 16
高配 中间数据 audioHex b64: Ay0U5QQCHLI
高配 中间数据 audioHex b64 的长度 : 11
高配 中间数据 text_s : 1AE4
高配 中间数据 size : 00F0
高配 中间数据 text_s : 6884
高配 拼接的音频数据 ：  1AE400F0Ay0U5QQCHLI
高配 的头 ：  10
高配 整体报文： 101AE400F0Ay0U5QQCHLI
发送方手机号  is 17751927311 接受方手机号 is 18917805428 整体报文 is 101AE400F0Ay0U5QQCHLI
 内容21字节
```

## 三、Java 解码示例

```java 

public class HighConfigDecoder {
    public static DecodedData decodeFrame(String packet) {
//        if (packet == null || packet.length < 9 || packet[0] != 0x10) {
//            throw new IllegalArgumentException("非法数据帧");
//        }
        String textLenHex = packet.substring(2, 6);

        String durHex = packet.substring(6, 10);

        int durationMs = Integer.parseInt(durHex, 16);
        int text_S = Integer.parseInt(textLenHex, 16);
        String audioB64 = packet.substring(10);
        System.out.println(audioB64);
        byte[] audioHexBytes = CodecUtils.fromBase64ToBytes(audioB64);
        String audioHexStr = CodecUtils.bytesToHex(audioHexBytes);
        List<Long> audioData = Hex16Reverse.hexToU16LongList(audioHexStr);

        return new DecodedData(text_S + "", durationMs, audioData);
    }

    public static class DecodedData {
        public final String text;
        public final long durationMs;
        public final List<Long> audioSamples;

        public DecodedData(String t, long d, List<Long> a) {
            text = t;
            durationMs = d;
            audioSamples = a;
        }
    }
}
```

---

## 四、Kotlin 解码示例

```kotlin
fun decodeHighConfigFrame(packet: ByteArray): DecodedResult {
    require(packet.size >= 9 && packet[0] == 0x10.toByte()) { "非法数据帧" }

    val textLen = packet.sliceArray(1..4)
        .joinToString("") { "%02X".format(it) }
        .hexToLong()

    val durationMs = packet.sliceArray(5..8)
        .joinToString("") { "%02X".format(it) }
        .hexToLong()

    var idx = 9
    val textBytes = packet.copyOfRange(idx, idx + textLen.toInt())
    val text = textBytes.toString(Charsets.UTF_8)
    idx += textLen.toInt()

    val audioB64 = packet.copyOfRange(idx, packet.size).toString(Charsets.US_ASCII).trim()
    val audioHexBytes = CodecUtils.fromBase64ToBytes(audioB64)
    val audioHexStr = CodecUtils.bytesToHex(audioHexBytes)
    val audioList = Hex16Reverse.hexToU16LongList(audioHexStr)

    return DecodedResult(text, durationMs, audioList)
}

data class DecodedResult(
    val text: String,
    val durationMs: Long,
    val audioSamples: List<Long>
)
```

---

## 五、工具类实现

### 1. HexReverse.java

```java
public final class HexReverse {
    private HexReverse() {
    }

    private static String normalizeHex(String s) {
        if (s == null) throw new IllegalArgumentException("null 不是有效的 16 进制字符串");
        String hex = s.trim();
        if (hex.startsWith("0x") || hex.startsWith("0X")) hex = hex.substring(2);
        hex = hex.replaceAll("\\s+", "");
        if (hex.isEmpty()) throw new IllegalArgumentException("空的 16 进制字符串");
        if (!hex.matches("[0-9a-fA-F]+")) throw new IllegalArgumentException("非法 16 进制: " + s);
        if ((hex.length() & 1) == 1) hex = "0" + hex;
        return hex;
    }

    public static String hexToDecimalString(String hexInput) {
        String hex = normalizeHex(hexInput);
        long value = Long.parseUnsignedLong(hex, 16);
        return Long.toString(value);
    }

    public static long hexToLong(String hexInput) {
        String hex = normalizeHex(hexInput);
        return Long.parseUnsignedLong(hex, 16);
    }
}
```

### 2. Hex16Reverse.java

```java
public final class Hex16Reverse {
    private Hex16Reverse() {
    }

    private static String normalizeAndValidate(String s) {
        if (s == null) throw new IllegalArgumentException("null 不是有效的十六进制字符串");
        String hex = s.replaceAll("\\s+", "");
        if (hex.isEmpty()) throw new IllegalArgumentException("空的十六进制字符串");
        if (!hex.matches("[0-9a-fA-F]+"))
            throw new IllegalArgumentException("非法 16 进制字符: " + s);
        if (hex.length() % 4 != 0)
            throw new IllegalArgumentException("长度必须是4的倍数，当前长度=" + hex.length());
        return hex;
    }

    public static List<Long> hexToU16LongList(String hexInput) {
        String hex = normalizeAndValidate(hexInput);
        int n = hex.length() / 4;
        List<Long> out = new ArrayList<>(n);
        for (int i = 0; i < hex.length(); i += 4) {
            String word = hex.substring(i, i + 4);
            long v = Integer.parseUnsignedInt(word, 16) & 0xFFFFL;
            out.add(v);
        }
        return out;
    }
}
```

### 3. CodecUtils.java

```java  

import java.util.Base64;
import java.util.Locale;

public final class CodecUtils {
    private static final char[] HEX_UPPER = "0123456789ABCDEF".toCharArray();

    private CodecUtils() {
    }

    public static byte[] hexToBytes(String hex) {
        if (hex == null) throw new IllegalArgumentException("hex 不能为 null");
        String s = hex.replaceAll("\\s+", "").toUpperCase(Locale.ROOT);
        if (s.length() % 2 != 0) throw new IllegalArgumentException("hex 长度必须为偶数");
        int len = s.length();
        byte[] out = new byte[len / 2];
        for (int i = 0, j = 0; i < len; i += 2, j++) {
            int hi = fromHexNibble(s.charAt(i));
            int lo = fromHexNibble(s.charAt(i + 1));
            out[j] = (byte) ((hi << 4) | lo);
        }
        return out;
    }

    public static String bytesToHex(byte[] bytes) {
        if (bytes == null) return "";
        char[] out = new char[bytes.length * 2];
        int k = 0;
        for (byte b : bytes) {
            int v = b & 0xFF;
            out[k++] = HEX_UPPER[v >>> 4];
            out[k++] = HEX_UPPER[v & 0x0F];
        }
        return new String(out);
    }

    private static int fromHexNibble(char c) {
        if (c >= '0' && c <= '9') return c - '0';
        if (c >= 'A' && c <= 'F') return 10 + (c - 'A');
        if (c >= 'a' && c <= 'f') return 10 + (c - 'a');
        throw new IllegalArgumentException("非法 hex 字符: " + c);
    }

    public static String hexToBase64(String hex, boolean noPadding, boolean urlSafe) {
        return toBase64(hexToBytes(hex), noPadding, urlSafe);
    }

    public static byte[] fromBase64ToBytes(String b64) {
        if (b64 == null) {
            throw new IllegalArgumentException("Input base64 string must not be null");
        }
        // 去掉所有空白字符
        String s = b64.trim().replaceAll("\\s+", "");
        try {
            // 先尝试 URL-safe 解码
            return Base64.getUrlDecoder().decode(s);
        } catch (IllegalArgumentException e) {
            // 再尝试标准 Base64 解码
            return Base64.getDecoder().decode(s);
        }
    }

    /**
     * 将字节数组编码为 Base64 字符串，可选无填充 & URL-safe 模式。
     * @param data 要编码的字节数组
     * @param noPadding 是否去掉末尾的 '=' 填充字符
     * @param urlSafe 是否使用 URL-safe 模式（替换 '+'/'/' 为 '-'/'_'）
     * @return Base64 编码后的字符串（不包含换行）
     */
    public static String toBase64(byte[] data, boolean noPadding, boolean urlSafe) {
        if (data == null) {
            return "";
        }
        Base64.Encoder encoder = urlSafe
                ? Base64.getUrlEncoder()
                : Base64.getEncoder();
        if (noPadding) {
            encoder = encoder.withoutPadding();
        }
        // 默认不会输出换行
        return encoder.encodeToString(data);
    }
}
```


