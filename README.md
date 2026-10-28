# 舞萌 DX 国服「机台二维码 → 水鱼成绩更新」完整方案（开源交付）

> 面向人群：想把「发二维码自动更新 b50」接入自己 QQ bot / 工具链的开发者。
> 本方案不依赖游戏本体、不依赖微信、不依赖任何付费网关，纯 HTTP + 少量密码学即可实现。
> 全部协议细节均为对官方服务器的黑盒逆向 + 开源项目（Ichino）交叉验证所得，供学习研究。

---

## 0. 一句话原理

机台登录二维码（SGWCMAID…）在官方 aimedb 服务上一次性兑换为 `(userId, token)`；
用该凭证登录华立业务服务器（42081），分页拉取逐曲成绩；
最后把成绩转成水鱼格式，凭 Import-Token 推送到 DivingFish。
三步各自独立，任何一步失败都容易定位。

```
SGWCMAID 二维码
   │ ① aimedb 登录（纯 HTTP）
   ▼
(userId, token)
   │ ② 42081 业务 API（手搓 TLS）
   ▼
全部逐曲成绩
   │ ③ 水鱼 update_records（普通 HTTPS）
   ▼
b50 更新完成
```

---

## 1. 第一步：aimedb 登录（QR → userId+token）

**端点**：`POST http://ai.sys-allnet.cn/wc_aime/api/get_data`（纯 HTTP :80 可用，实测）

**签名**：
```
key = SHA256( 完整Keychip + 北京时间(yymmddHHMMSS) + 盐 ).hex().upper()
完整Keychip 形如 A63E-01C2805XXXX（构成规则见下方「坑」）
盐 = "XcW5FW4cPArBXEk4vzKz3CIrMuA5EVVW"（源自 Ichino 开源项目）
```

**请求体**（注意：绝不能带 Content-Type 头，带了回 errorID=57）：
```json
{
  "chipID":   "A63E-01C2805XXXX",
  "openGameID": "MAID",
  "key":      "<上面的 SHA256>",
  "qrCode":   "<二维码第 21 位起的 64 位 hex>",
  "timestamp": "261027221032",
  "titlekey": "SDGB"
}
```
头：`User-Agent: WC_AIME_LIB`、`Connection: Keep-Alive`。

**响应**：`{"errorID":0,"userID":<userId>,"token":"<20位>"}`。
errorID 语义：`1`=码过期/已用，`57`=带了 Content-Type，`54`=老协议被拒。
**每个码一次性**，失败重试必须重新生成。

**坑**：时间戳必须 **UTC+8**（机器时区不是东八区时差一小时 = errorID 1）；
keychip 的完整形式 = 去掉连字符后的原始串前 4 位 + `-` + 其余位；
若原始串只有 11 位（如 `A63E01C2805`），则在末尾补上 placeId 凑成完整形式。

---

## 2. 第二步：42081 业务 API（登录 → 成绩）

### 2.1 传输层（最大门槛）

官方 42081 前面是 EdgeOne CDN，**按 TLS 指纹分流**：
- 普通 OpenSSL/httpx → 200 空体（被静默丢）
- Windows Schannel → 404 诱饵
- Dart/BoringSSL → 200 空体
- **只有与机台一致（Unity 2018 的 85 个密码套件、无 ALPN）的 ClientHello 才能到达真实业务**

解法 = **手搓 TLS 1.2**（约 250 行 Python）：从抓包里提取机台的精确 ClientHello 作模板，
自行实现 ECDHE-RSA-AES128-GCM-SHA256 握手（cryptography 库足够）：
- PRF(P_SHA256) 主密钥/密钥扩展（RFC 5246）
- GCM 记录的 AAD = seq+type+version+**明文长度**（不是 wire 长度，这里极易错）
- 握手哈希要包含服务器侧全部握手消息

> 模板方法：pktmon 抓 42081 明文 ClientHello → 替换随机数即可重放。
> 参考实现 `raw_tls_game.py` 已在本地与 OpenSSL 服务器对测逐字节一致，整理开源中。

### 2.2 路由与加密

```
URL  = https://maimai-gm.wahlap.com:42081/Maimai2Servlet/<md5>
md5  = MD5( apiName + "MaimaiChn" + ObfuscateParam )
ObfuscateParam = "8bF76dE9"（游戏 Net.Packet.Packet.ObfuscateParam 静态字段）
```

头（实测自游戏 .NET HttpWebRequest 反射）：
```
User-Agent: <md5>#<userId|clientId>
Content-Type: application/json
charset: UTF-8
Mai-Encoding: 1.55        ← 注意不是 1.53！
number: 0                  ← 登录后观察到 101
Content-Encoding: deflate
Cookie: JSESSIONID=...     ← 登录后从 Set-Cookie 提取
```

**Body = AES-256-CBC( zlib(JSON) )**：
```
key = "FKM2JX:VjZNK6hc:A0<JU:i5oR7LA]9W"（32字节）
iv  = "F>;24DjU9W6ZsRH["（16字节）
PKCS7 填充，zlib 压缩（等级随意）
```
响应同格式反向解密。注意 **mE2s3Jhd 是 movieServerUri，与 API 路径无关**（历史迷惑项）。

### 2.3 会话流

```
GetUserPreviewApi {userId, segaIdAuthKey:"", token, clientId}
   → 响应含 userName / playerRating / isLogin / lastLoginDate
GetUserMusicApi  {userId, nextIndex, maxCount:50}  分页直到 nextIndex=0
```

**关键**：Preview 响应的 `isLogin=true` 表示会话仍挂在机上，此时**必须跳过
UserLoginApi**（再调会 returnCode=100）；只有 isLogin=false 才走 UserLoginApi：
```
UserLoginApi {userId, accessCode:"", regionId:<regionId>, placeId:<placeId>, clientId,
              dateTime:now-600, loginDateTime:now, isContinue:false,
              genericFlag:0, token}
   → returnCode=1 + Set-Cookie(JSESSIONID)
```
（regionId/placeId 取自机台 `config_client.json`，不参与校验也能过。）

成绩字段：`userMusicList[].userMusicDetailList[]` →
`{musicId, level, achievement, comboStatus, syncStatus, deluxscoreMax, ...}`
（achievement 为万分制，level 0-4 = 基本/进阶/专家/大师/红谱，>4 与 musicId≥100000 为宴谱，推送水鱼时跳过）。

---

## 3. 第三步：推送水鱼

**曲名映射**：CN musicId → 官方原名。
映射表 1378 条（可由机台 Music.xml 生成，或直接复用 Ichino 的 `data/id_mapping/music.json`）。

```
POST https://www.diving-fish.com/api/maimaidxprober/player/update_records
Header: Import-Token: <成员的水鱼导入密钥>
Body: [{
  "title": "<歌名>",
  "level_index": 0..4,
  "achievements": 101.0,      // 万分制/10000
  "fc": ""|"fc"|"fcp"|"ap"|"app",
  "fs": ""|"fs"|"fsp"|"fsd"|"fsdp",
  "dxScore": 0,
  "type": "SD"|"DX"
}]
```
fc/fs 由 comboStatus/syncStatus（0-4）映射。
取水鱼用户名（用于绑定展示）：`GET .../player/records` + `Import-Token` 头（**GET，不是 POST**）。

---

## 4. 平台兼容性

| 组件 | Windows | Linux |
|---|---|---|
| aimedb 登录 | 纯 HTTP:80 | ✅ 同左（无需 WinHTTP） |
| 42081 TLS | 手搓 TLS | ✅ 同左（纯 Python） |
| 曲名映射 | JSON | ✅ |
| 水鱼推送 | httpx | ✅ |

实测香港 VPS（Ubuntu）全链路原生可跑，42081 无地域限制。

---

## 5. 建议的代码结构（可直接开源）

```
maimai_cn_login.py   # aimedb QR→token（~40行）
raw_tls_game.py      # 手搓 TLS1.2 + 机台指纹模板（~250行）
maimai_cn_api.py     # md5路由 + AES(zlib) + 会话流（~120行）
music_map.json       # musicId→歌名
push_divingfish.py   # 成绩→update_records
```

---

## 附录 A：如何从游戏 HDD 提取「开罗尔材料」与哈希值（详细）

> 「开罗尔材料」= 让协议跑起来所需的全部常量与指纹素材。
> 国服 `Assembly-CSharp.dll` 的方法体在磁盘上是**清零的**（运行时才解密），
> 所以纯静态扫描拿不到关键逻辑——**字段与 IL 必须在运行时提取**。
> 下面按「静态可读 → 运行时提取 → 抓包取证」三层给出完整方法。

### A1. 静态层（HDD 上直接可读）

| 材料 | 位置 | 用途 |
|---|---|---|
| keychip 完整 ID | `segatools.ini` 的 `id=` | 登录签名输入 |
| gameId / region | `segatools.ini` 的 `gameId`/`region` | 请求常量 |
| lite_auth_key | `config_common.json` | 本地 ALL.Net 层（本方案用不到） |
| 曲名映射 | `music/Music.xml`（每首歌的 id/name） | 生成 `music_map.json` |
| 机台参数 | `config_client.json` 等 | placeId / regionId |

曲名映射的生成逻辑可参考 Ichino 的 `tools/gen_id_mapping.dart`：
遍历 `Music.xml` → `{musicId: name}`，只收 `id < 100000` 的常规曲（10 万以上为宴谱）。

### A2. 运行时提取（关键！字段 + 解密后的 IL）

磁盘上的 `Assembly-CSharp.dll` 方法体被清零（`GetMethodBody()` 返回空），
但 Unity 运行时内存里是解密后的完整 IL。方法 = 注入一个 MelonLoader mod，
在游戏进程内做两件事：**①读静态字段值 ②dump 解密后的 IL**。

最小化 mod 核心代码（C#，MelonLoader 0.6.x，net35）：

```csharp
public class KaleidoDump : MelonMod
{
    static string[] KeyWords = {"sign","secret","token","aes","kaleido","allnet",
                                "encrypt","decrypt","cipher","auth","aime","serial",
                                "key","session","credential"};

    public override void OnInitializeMelon()
    {
        var asm = typeof(Game.Process.Entry).Assembly;   // 示例：任取游戏命名空间下的一个类定位 Assembly-CSharp
        foreach (var t in asm.GetTypes())                // 注意 try/catch ReflectionTypeLoadException
        {
            // ① 静态字段值（密钥/token/混淆串都在这里）
            foreach (var f in t.GetFields(BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic))
            {
                if (KeyWords.Any(k => t.FullName.Contains(k) || f.Name.Contains(k)))
                {
                    var v = f.GetValue(null);
                    if (v is string || v is byte[])
                        Log($"STATIC {t.FullName}.{f.Name} = {v}");
                }
            }
            // ② 解密后的方法 IL（磁盘上本来为零）
            foreach (var m in t.GetMethods(BindingFlags.Instance | BindingFlags.Static |
                                           BindingFlags.Public | BindingFlags.NonPublic |
                                           BindingFlags.DeclaredOnly))
            {
                if (!KeyWords.Any(k => t.FullName.Contains(k) || m.Name.Contains(k))) continue;
                var mb = m.GetMethodBody();
                var il = mb?.GetILAsByteArray();
                if (il != null && il.Length > 0) Log($"METHOD {t.FullName}.{m.Name} ilLen={il.Length}\n{Hex(il)}");
            }
        }
    }
}
```

**这一招能拿到的关键常量（本方案实测）**：

| 常量 | 字段 | 值 |
|---|---|---|
| 路由混淆串 | `Net.Packet.Packet.ObfuscateParam` | `8bF76dE9` |
| AES 密钥 | `Net.CipherAES.AesKey` | 32 字节（见第 2.2 节） |
| AES IV | `Net.CipherAES.AesIV` | 16 字节 |
| clientId | `Net.NetConfig.<ClientId>k__BackingField` | 11 位机器编号 |
| 请求头常量 | `Net.NetHttpClient.Post/ContentTypeJson/...` | 头名字典 |

**实例字段（静态 dump 拿不到，要反射活实例）**：`Net.NetHttpClient` 的
`HttpHeaderEncryption` / `DefaultHttpHeaderEncryption` 两个实例字段 =
`"Mai-Encoding"` / `"1.55"`——这就是「加密头」的真身：`Headers.Add("Mai-Encoding", "1.55")`。
获取方法：在 `Request` 方法上挂 Harmony 前缀钩子，反射 `__instance` 全部字段。

### A3. 哈希值的两个来源（算出来的 + 抓出来的）

**① 路由哈希（可计算）**：
```
URL 末段 = MD5( apiName + "MaimaiChn" + ObfuscateParam )
```
拿到 A2 的 ObfuscateParam 后即可离线生成全部 API 的路径。
验证：抓包看到游戏请求 URL/UA 里的 md5 与你算的一致即闭环。

**② 签名哈希（可计算）**：见第 1 节（SHA256 + UTC+8 + 盐）。

**③ TLS 指纹模板（必须抓包，算不出来）**：
EdgeOne 按 ClientHello 逐字节分流，所以要**原样抓取**机台的 ClientHello：
- Windows：`pktmon filter add -p 42081 -t TCP` → 启动游戏 → `pktmon stop` →
  `pktmon etl2pcap` 转 pcapng
- 从 pcap 里找发往 42081 的首个 TLS 记录（type 0x16）→ 提取完整 ClientHello
  字节存成模板（record 头 + handshake 头 + version + random + 密码套件…）
- 复刻时**只换随机数与 session id**，其余字节原样重放；85 个套件顺序、
  无 ALPN、曲线列表、点格式都不能动

**④ 会话头语义（抓包+反射双重确认）**：
- `number: 0`（未登录）/ 登录后观察到 `101`——从 AddHeader 调用反射得到
- UA = `<md5>#<userId|clientId>`；登录后 Cookie 带 JSESSIONID
- `isLogin=true` 时必须跳过 UserLoginApi（见第 2.3 节）

### A4. 素材清单（一份 HDD + 一次抓包全齐）

```
静态：segatools.ini、Music.xml、config_*.json
运行时：静态字段 5 个 + 实例字段 2 个 + 任意方法的解密 IL
抓包：ClientHello 模板 + 请求头集合 + Set-Cookie 时机
外部：登录签名盐（Ichino 开源，见致谢）
```

---

## 6. 致谢与许可

- 协议骨架源自开源项目 **[WinDolls/Ichino](https://github.com/WinDolls/Ichino)**（MIT，原 AirFoundation/Ichino，已迁移）——签名盐、字段名、传输思路
- 水鱼生态：DivingFish 查分器 + maimai-py 的 update_records 格式
- 本文档与其派生实现建议以 MIT 发布，注明上述来源
- 我们的 1.55 服务器实测勘误（8 点）已以 [PR #1](https://github.com/WinDolls/Ichino/pull/1) 回馈 Ichino，全文见本仓库 `SUPPLEMENT_FOR_ICHINO.md`

> 注：接口为未公开能力，仅作技术研究与存档用途；实际接入前自行评估风险。
