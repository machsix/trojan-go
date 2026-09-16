---
title: "高级TLS设置：指纹、ALPN和ECH"
draft: false
weight: 11
---

本页介绍影响trojan-go**客户端**与远端通信的TLS层选项：伪造的Client Hello指纹、ALPN列表和ECH（加密客户端问候）模式。对应配置字段位于```ssl```块；完整配置结构见[完整的配置文件](../../basic/full-config)。

这里描述的行为由```tunnel/tls```回归测试覆盖；尤其```TestDefaultFingerprintIsChrome```锁定了默认指纹，因此它不会在版本间悄然改变。

### 默认指纹

trojan-go使用[utls](https://github.com/refraction-networking/utls)伪造类似浏览器的TLS Client Hello。指纹控制密码套件列表、扩展顺序、支持的曲线组等可被审查者用于识别非浏览器TLS客户端的字节特征。

| ```fingerprint```值 | utls ClientHelloID | 说明 |
|---|---|---|
| ```""```（默认） | ```HelloChrome_Auto``` | 解析为```chrome```；自2024年起保持稳定。 |
| ```"chrome"``` | ```HelloChrome_Auto``` | 默认值。 |
| ```"firefox"``` | ```HelloFirefox_Auto``` | |
| ```"ios"``` | ```HelloIOS_Auto``` | |
| ```"edge"``` | ```HelloEdge_Auto``` | |
| ```"safari"``` | ```HelloSafari_Auto``` | |
| ```"360browser"``` | ```Hello360_Auto``` | |
| ```"qqbrowser"``` | ```HelloQQ_Auto``` | |

名称不区分大小写。任何其他值都会在启动时以明确错误拒绝。

选择指纹后，```cipher```、```curves```、```alpn```和```session_ticket```将被该指纹的规范值覆盖，不应混用。如果需要自行覆盖其中一项，请保持```fingerprint```为空，并手动设置全部相关字段；这样得到的握手可能被识别为非浏览器流量。

### ECH GREASE与完整ECH

```ech```启用加密客户端问候，但两种模式的威胁模型完全不同：

* **GREASE ECH**（```ech: true```、```ech_config: ""```）：trojan-go向Client Hello注入语法有效但密码学上无意义的ECH扩展。真实SNI**不会**被加密，仍以明文位于外层Client Hello。其作用是使握手外形接近已启用ECH的Chrome，避免“缺少ECH扩展”成为指纹信号。适用于不发布ECH的普通HTTPS目标。

* **完整ECH**（```ech: true```、```ech_config: "<base64 ECHConfigList>"```）：trojan-go执行真实ECH握手。带有真实SNI的内层Client Hello会使用ECHConfigList中的公钥加密，并嵌入外层Client Hello；外层SNI为ECHConfigList的公开名称。仅在目标明确发布ECHConfigList（通常通过HTTPS DNS记录）时使用。

如果设置```ech_config```但```ech```为```false```，该配置会被忽略并在启动时记录```WARN```；这是运维错误，而非特性。

#### CDN和ALPN注意事项

* 通常CDN并不支持ECH。经多数CDN部署时，完整ECH会导致握手失败或落到错误的虚拟主机；通过CDN前置时应使用GREASE ECH或不使用ECH。
* ```alpn```列表是所选TLS指纹的一部分。普通传输使用指纹的规范ALPN；启用WebSocket后，trojan-go仅覆盖该指纹扩展并通告```http/1.1```，因为WebSocket实现使用HTTP/1.1 Upgrade，无法在CDN协商出的HTTP/2连接上完成握手。其余指纹字段保持不变。
* GREASE ECH只改变Client Hello外形；通过普通TLS终止器时它不会提高SNI保密性。如果威胁模型要求SNI保密，请对支持ECH的目标使用完整ECH，或使用网络内汇合方案（超出本文范围）。

### 单一事实来源

支持的指纹名称列表位于```tunnel/tls/client.go```中的```resolveFingerprint```。添加或删除值时，同时更新：

1. ```resolveFingerprint```的```switch```。
2. ```resolveFingerprint```对未知值返回的错误信息。
3. 本文中的表格。
4. [完整的配置文件](../../basic/full-config)中```fingerprint```下的列表。
5. ```tunnel/tls/fingerprint_default_test.go```中的```TestResolveFingerprintKnownNames```测试。
