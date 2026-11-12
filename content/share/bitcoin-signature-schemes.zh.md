---
title: "比特币签名机制拆解：ECDSA、Schnorr、Taproot 与 SIGHASH 详解"
slug: "bitcoin-signature-schemes"
date: 2026-11-12T17:30:00+08:00
draft: false
author: "Jackie"
tags: ["比特币", "ECDSA", "Schnorr", "Taproot", "SIGHASH", "密码学", "签名方式", "加密货币", "区块链"]
categories: ["每日分享"]
summary: "把比特币签名拆成三个相对独立的层次：签名算法（ECDSA vs Schnorr）、锁定脚本（P2PK、P2PKH、P2SH、SegWit、Taproot）、SIGHASH 授权范围，逐层对比安全、费用与隐私，再比较三种多签实现，最后给出选型建议与安全清单。"
showtoc: true
cover:
  image: "images/bitcoin-signature-schemes-flow.svg"
  alt: "比特币签名机制三层结构图：签名算法（ECDSA vs Schnorr）、锁定脚本（P2PK → P2PKH → P2SH → P2WPKH/P2WSH → P2TR）、SIGHASH（ALL / NONE / SINGLE + ANYONECANPAY; DEFAULT）"
  relative: false
images:
  - "images/bitcoin-signature-schemes-og.png"
---

聊“比特币的签名方式”时，许多文章把三个不同的问题混在一起——**用什么数学算法签名**、**满足什么花费条件**、**签名承诺交易的哪些部分**。P2TR 不是签名算法，MuSig2 也不是脚本类型——它们分属不同层。拆开看才清楚：

| 层次 | 回答的问题 | 选项 |
| --- | --- | --- |
| 签名算法 | 用什么数学算法签名？ | ECDSA (secp256k1) vs Schnorr (BIP340) |
| 锁定脚本 | 满足什么花费条件？ | P2PK → P2PKH → P2SH → P2WPKH/P2WSH → P2TR |
| SIGHASH | 签名承诺交易的哪些部分？ | ALL / NONE / SINGLE + ANYONECANPAY；DEFAULT |

这套分层覆盖常见的比特币签名方案，但各层并非完全正交：例如 `SIGHASH_DEFAULT` 只用于 Taproot，特定脚本也只接受特定签名算法。下面逐层展开，最后给出选型建议。

## 第一层：签名算法

### ECDSA：2009–2021 年间唯一的链上签名算法

比特币诞生前后，Schnorr 签名专利才到期（主流说法为 2008 年 2 月；若按申请后 20 年计算，则为 2010 年 2 月），而 ECDSA 已有标准化实现，OpenSSL 也提供支持。于是，在 Taproot 激活前，`secp256k1` 上的 ECDSA 是比特币唯一的链上签名算法。签名采用 DER 编码，通常为 70–72 字节（不含追加的 1 字节 SIGHASH 标志）：

```text
s = k⁻¹ · (H(m) + r·x) mod n
```

这个方程带出三类关键问题：

- **nonce `k` 是命门**。同一个私钥用同一个 `k` 签两条消息，私钥可以直接解出。2013 年 Android 的 SecureRandom 缺陷导致一批移动钱包私钥泄露，就是这个原因。后来的缓解是 [RFC 6979](https://datatracker.ietf.org/doc/html/rfc6979)：`k` 由私钥与消息确定性导出，不再依赖随机数发生器的质量。
- **可塑性（malleability）**。ECDSA 的数学性质允许把 `s` 换成 `n−s`，签名仍然有效；早期对非 DER 编码的宽松解析和可变的 `scriptSig` 也提供其他修改途径。对非 SegWit 交易，第三方可能因此改变 txid。Mt. Gox 2014 年暂停提币时曾将问题归咎于交易可塑性，但真实亏损不能简单归因于此。[BIP66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki) 在共识层严格 DER 编码，low-S 等规则则主要通过标准交易策略落地；[BIP62](https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki) 提出了更完整的修复方向，但最终未能推进，状态已关闭（Closed）。SegWit 将签名移出 txid 的计算，消除了 **SegWit 输入**中第三方修改签名所导致的 txid 可塑性，但并不消除所有交易可塑性。
- **缺少 Schnorr 式的线性聚合**。这不等于 ECDSA 完全不能批量验证或做多方签名，而是无法获得 Schnorr 那样简洁的原生密钥与签名聚合；门限 ECDSA 通常需要更复杂的协议。

### Schnorr（[BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)）：随 Taproot 激活

2021 年 11 月（高度 709,632）随 [Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) 软分叉上线。核心签名关系如下（略去 x-only 编码要求的条件取负）：

```text
s = k + e·x mod n,   e = tagged_hash("BIP0340/challenge", R || P || m)
```

签名固定 64 字节（`r` 32 字节 + `s` 32 字节），编码严格；公钥编码为 32 字节（x-only，验证时取偶数 y）。验证关系 `s·G = R + e·P` 不涉及 ECDSA 验证中的模逆，也不存在将 `s` 替换为 `n−s` 的等价签名。

| | ECDSA | Schnorr（BIP340） |
| --- | --- | --- |
| 签名大小 | 约 70–72 B（DER，不含 SIGHASH 字节） | 64 B 固定 |
| 公钥 | 33 B（压缩） | 32 B（x-only） |
| 可塑性 | 存在等价 high-S 形式；可用标准交易规则限制 | 严格编码与签名构造避免 |
| 单次验证 | 验证关系含模逆 | 无 ECDSA 式模逆；实际性能取决于实现 |
| 批量验证 | 标准形式无 Schnorr 这类原生高效批验 | 支持随机系数批量验证 |
| 签名/密钥聚合 | 无同等简洁的原生协议；可用更复杂的多方方案 | [MuSig2（BIP327）](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki)，n-of-n |

**线性是关键，但不能把验证方程不加区分地直接相加**：批量验证需要随机系数来防止无效签名相互抵消，再借助多标量乘法提速。多方则可借助 MuSig2 等协议聚合密钥，生成外观上与普通 BIP340 单签相同的签名。ECDSA 并非在数学上绝对无法批量验证或实现门限签名，但缺少这种简洁的原生路径。

## 第二层：锁定脚本

算法决定“怎么签”，脚本决定“满足什么条件后签名有效”。演进主线可粗略概括为：**把条件从明文藏进哈希，再把更多可选条件藏进 [Merkle 树](https://en.wikipedia.org/wiki/Merkle_tree)**。

- **P2PK**（2009）：公钥直接写进输出，早期常见；一般认为中本聪早期挖出的币大多使用这种输出。未花费时公钥已在链上公开，因此也是量子威胁讨论中常被点名的类型。不过在经典安全模型下，公开公钥并不等同于泄露私钥。
- **P2PKH**（2009）：只放 `hash160(pubkey)`，花费时才揭示公钥与签名。输出脚本 25 字节，压缩公钥输入典型约 148 vB，主网地址以 `1` 开头。
- **P2MS**（裸多签，[BIP11](https://github.com/bitcoin/bips/blob/master/bip-0011.mediawiki)）：m-of-n 直接写进输出，n 个公钥全部公开；`OP_CHECKMULTISIG` 还有多弹出一个元素的 off-by-one 历史缺陷，需要额外的占位元素。
- **P2SH**（[BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki)，2012）：输出脚本只承诺一个 20 字节脚本哈希，多签、时间锁等复杂条件移到花费时才揭示完整脚本，主网地址以 `3` 开头。2017 年为兼容老钱包派生出嵌套的 P2SH-P2WPKH（输入约 91 vB），如今主要有过渡价值。
- **P2WPKH / P2WSH**（SegWit，[BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) / [BIP143](https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki)，2017-08，高度 481,824）：签名移入 witness，witness 数据每字节计 1 weight unit，非 witness 数据每字节计 4，P2WPKH 输入降到约 68 vB。txid 不再覆盖 witness，消除了第三方修改签名造成的 txid 可塑性，为 [Lightning Network](https://lightning.network/) 提供了关键基础。BIP143 顺带修了两个老问题：签名消息纳入本输入金额，避免主机虚报金额却仍能生成有效签名；可复用的哈希让签名哈希计算从 O(n²) 降为 O(n)。
- **P2TR**（[BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) / [BIP342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)，2021-11）：输出脚本为 `OP_1 <32 字节输出公钥 Q>`，共 34 字节，主网地址以 `bc1p` 开头：

```text
Q = P + t·G,   t = tagged_hash("TapTweak", P || merkle_root)
```

没有脚本树时，`merkle_root` 省略，而不是按零值参与哈希。两种花费路径如下：

- **Key path**：一个 Schnorr 签名即可；使用 `SIGHASH_DEFAULT` 时签名 64 字节，典型输入约 57.5 vB，在常见标准单签输入中最小。未花费时，P2TR 输出不会暴露内部公钥或脚本树；以 key path 花费时，也不会揭示脚本树或内部公钥 `P`。因此，就输出脚本和见证结构而言，单签与带隐藏脚本的 key path 花费无法区分。
- **Script path**：揭示实际执行的脚本分支和控制块（含叶版本与输出公钥奇偶位、内部公钥、Merkle 路径）。Tapscript 用 `OP_CHECKSIGADD` 累计成功签名数替代 `OP_CHECKMULTISIG`，没有占位元素缺陷，也便于批量验证；本次花费不会揭示其他脚本分支。

### 输出脚本与典型输入大小对比

| 类型 | 输出脚本大小 | 输入（典型） | 签名算法 | 备注 |
| --- | --- | --- | --- | --- |
| P2PK | 35 B（压缩公钥）/ 67 B（未压缩公钥） | 约 114 vB | ECDSA | 公钥长期暴露 |
| P2PKH | 25 B | 约 148 vB | ECDSA | 假设使用压缩公钥 |
| P2SH-P2WPKH | 23 B | 约 91 vB | ECDSA | 嵌套过渡 |
| P2WPKH | 22 B | 约 68 vB | ECDSA | 常见单签类型 |
| P2WSH 2-of-3 | 34 B | 约 104.5–105 vB（随 DER 长度略变） | ECDSA ×2 | witness 脚本全公开 |
| **P2TR key path** | 34 B | **约 57.5 vB** | **Schnorr** | 使用 DEFAULT；不暴露脚本树 |

这里的输出大小仅指 `scriptPubKey`，不含 8 字节金额等字段；输入统一用 vB 表示，ECDSA 行按 low-R（71 字节 DER）估算，实际大小还会随 DER 长度、SIGHASH、脚本和见证结构变化。

## 第三层：SIGHASH

同一个算法，配不同 SIGHASH，签出来的“合同”可以完全不同——它决定这份签名承诺交易的哪些输入、哪些输出：

| Flag | 值 | 承诺输入 | 承诺输出 | 典型用途与注意事项 |
| --- | --- | --- | --- | --- |
| ALL | 0x01 | 全部 | 全部 | legacy / SegWit v0 的默认选择，最常见 |
| NONE | 0x02 | 全部 | 无 | 输出可被替换；仅用于明确的协议设计 |
| SINGLE | 0x03 | 全部 | 与输入同索引的输出 | 联合交易；须处理同索引输出不存在的情况 |
| +ANYONECANPAY | 与低位标志组合，叠加 0x80 | 仅本输入 | 由低位标志决定 | 众筹、追加输入等场景 |
| DEFAULT | 0x00 | 全部 | 全部 | Taproot 专用；承诺范围同 ALL，签名可省 1 字节 |

表格按“承诺哪些 prevout、哪些输出”的粒度做了简化。更细地说：legacy 与 BIP143 下，NONE / SINGLE 会把其他输入的 `nSequence` 哈希置零，因此并不承诺这些字段；Taproot（BIP341）则相反，在 NONE / SINGLE 下仍承诺全部输入的 `nSequence`（设置 ANYONECANPAY 时除外）。这也是 BIP341 相对 BIP143 的变更之一。

三个值得记住的历史坑：

- **SINGLE 越界 bug**：legacy 中，当输入索引没有同索引输出时，`SignatureHash` 返回整数 1 的 32 字节表示，而不是正常交易摘要（内部 little-endian 表示为 `0x01` 后接 31 个 `0x00`）。这不会让任意签名自动有效，但会使针对该常数生成的签名不再承诺正常交易内容，并可能在其他满足相同条件的交易中复用。BIP143 的 v0 witness 路径避免了这一退化分支；Taproot（BIP341）则直接将缺少对应输出的 `SIGHASH_SINGLE` 判定为无效。
- **不承诺金额**：legacy 签名消息不含输入金额，是硬件钱包金额欺骗漏洞的根源；BIP143 将本输入金额纳入签名消息修复。
- **Taproot 的签名消息**使用 `TapSighash` tagged hash，避免 legacy / BIP143 的双重 SHA256，并承诺被花费输出的金额和 `scriptPubKey` 等上下文；script path 还承诺 tapleaf hash、密钥版本和 `OP_CODESEPARATOR` 位置。具体承诺字段仍受 SIGHASH 标志影响。

**实践建议**：legacy / SegWit v0 通常用 ALL，Taproot 通常用 DEFAULT。只有协议明确需要且理解授权范围时，才使用 NONE、SINGLE 或 ANYONECANPAY。

## 多方签名：三种实现的正面对比

多签是“签名方式”最能拉开差距的场景：

| | P2WSH m-of-n | Taproot script path m-of-n | **MuSig2（BIP327）key path** |
| --- | --- | --- | --- |
| 链上形态 | 输出为脚本哈希；花费时揭示 n 个公钥与门限 | 输出为 tweaked key；花费时揭示执行分支 | 单个聚合公钥，可再作 Taproot tweak |
| 签名数 | m 个 ECDSA | m 个 Schnorr | **1 个聚合 Schnorr** |
| 2-of-3 典型输入 | 约 104.5–105 vB | 约 108–116.5 vB（含控制块） | **不直接支持；3-of-3 key path 才约 57.5 vB** |
| 花费时揭示 | 完整脚本 + 全部公钥 | 执行分支 + 控制块 | **无额外脚本信息** |
| 门限 | 任意 m-of-n | 任意（单脚本或按分支） | 仅 n-of-n |
| 未花费时可见信息 | 仅知是 P2WSH，不知具体门限与公钥 | 与其他 P2TR 输出同形 | **与其他 P2TR 输出同形** |

Taproot 一栏的约 108–116.5 vB，对应单个 2-of-3 `OP_CHECKSIGADD` 叶或三个 2-of-2 组合叶的常见结构，按 `SIGHASH_DEFAULT` 与相应树深估算；树深、脚本和 SIGHASH 不同都会改变结果。

几个关键点：

- **Rogue-key 攻击**：设受害者公钥为 `P₁`，攻击者可选择 `P₂ = x₂·G − P₁`，使朴素相加得到 `P₁ + P₂ = x₂·G`，从而独占聚合键。BIP327 先对公钥列表计算 `L`，再用 `a_i = int(tagged_hash("KeyAgg coefficient", L || P_i)) mod n` 派生聚合系数；其中第二个不同公钥及与其相同的公钥按 MuSig2* 优化取系数 1，规范也定义了重复公钥的处理。
- MuSig1 需要三轮交互；MuSig2 减到两轮。每个签名者的公开 nonce 由两个曲线点组成，实现必须安全保存会话状态，且不得重复使用 `secnonce`。BIP327 已提供互操作标准，但选用前仍需逐项确认钱包、协调器和硬件固件的端到端支持。
- MuSig2 只有 n-of-n。真正的 t-of-n 门限可以使用 [FROST](https://eprint.iacr.org/2020/852)，也可以在 Taproot 中用 `OP_CHECKSIGADD` 写门限脚本，或把各种签名者组合拆成脚本分支。
- 2-of-3 的 Taproot script path 未必比 P2WSH 便宜，在上述假设下约多 4–12 vB。它的主要收益在于：常用协作路径可走 key path，应急分支平时隐藏，且未花费时与其他 P2TR 输出同形。
- **隐私边界**：n 方通过 MuSig2 以 key path 花费时，就输出脚本与见证结构而言，与普通 P2TR 单签花费无法区分；这不等于金额、地址重用和钱包指纹等元数据也匿名。

## 怎么选

1. **个人自托管**：工具链支持时可选 P2TR key path；钱包或收款方不支持就退到 P2WPKH。
2. **全员共管**（如团队冷钱包 n-of-n）：MuSig2，链上占用最小，并与普通 P2TR 单签同形。
3. **条件支出**（继承、超时、双因素、逃生舱）：Taproot 脚本树，让最常用的协作路径通过 key path 完成，其余条件放进脚本树。
4. **兼容性优先**（老交易所白名单、旧硬件钱包）：P2WPKH / P2WSH 多签仍是稳妥选项；更老旧的环境可能还需要 P2PKH。

[PSBT（BIP174）](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) 与 [输出描述符](https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md) 让上述方案可以跨设备、跨厂商协作，但前提是各环节支持相同的脚本与签名协议，选型时先确认完整工具链。

## 安全的命门没有变

无论算法层怎么演进，**nonce 泄露或复用都可能暴露对应私钥**：

- ECDSA 使用 RFC 6979 确定性 nonce；BIP340 按自身规范生成 nonce；MuSig2 不能简单照搬纯确定性的会话 nonce，必须遵循 BIP327 的高质量随机数与一次性 `secnonce` 规则。
- 支持 anti-klepto 的硬件钱包可让主机注入额外熵，并用 [DLEQ 证明](https://github.com/bitcoin/bips/blob/master/bip-0374.mediawiki) 等方式证明 nonce 按约定生成，降低通过 nonce 研磨外泄私钥的风险；这不是所有钱包默认具备的能力。
- MuSig2 中，某方重复使用自己的 `secnonce`，可能暴露**该方**私钥，而不是自动泄露其他参与者的私钥。每个端点仍都必须审计 nonce 的生成、存储、使用与清除，避免自己成为授权体系中的薄弱环节。

往后看：[BIP118（ANYPREVOUT）](https://github.com/bitcoin/bips/blob/master/bip-0118.mediawiki) 允许签名不承诺具体 UTXO，为 eltoo 式 Lightning 更新机制铺路；跨输入签名聚合（多个输入的签名合成一个）有望降低 CoinJoin 等交易的成本。前者依托 Schnorr / Taproot 的可扩展签名结构，后者更直接利用 Schnorr 的线性。两者都不是现行共识规则：BIP118 目前仍是 Draft，跨输入聚合也尚未部署。

## 速查清单

**算法层**

1. ECDSA 是否使用 RFC 6979 或同等可靠的 nonce 方案？Schnorr 与 MuSig2 是否分别遵循 BIP340、BIP327？
2. nonce 纪律是否可审计？硬件钱包是否支持 anti-klepto？MuSig2 的 `secnonce` 是否严格只用一次？

**脚本层**

3. 公钥是否长期暴露？P2PK 与 P2TR 输出会直接公开公钥；P2PKH / P2WPKH 在花费后公开公钥；地址重用还会强化交易关联。
4. 是否需要隐藏花费条件？P2TR key path 可隐藏内部公钥与脚本树，但输出公钥 `Q` 始终在链上；脚本隐私与公钥暴露不能混为一谈。

**多方与 SIGHASH**

5. 多签究竟是 n-of-n 还是 t-of-n？MuSig2 只直接支持前者；交互轮数与实现审计是否明确？
6. SIGHASH 是否需要非默认模式？legacy / SegWit v0 默认 ALL，Taproot 默认 DEFAULT；没有明确需求就别碰 NONE / SINGLE / ANYONECANPAY。
7. 多方合约是否验证了 Taproot 内部公钥、聚合系数、Merkle 根与 tweak？简单相加公钥或信任协调器都可能引入 rogue-key 或替换风险。

---

签名方式的演进不是简单替换而是叠加：新方案更多以 Schnorr 为基础，脚本层在向“按花费路径最小披露”演进，SIGHASH 则持续修补并扩展承诺范围。真正的分化在多方协作协议——那里既是费用与隐私的杠杆，也是新的攻击面入口。
