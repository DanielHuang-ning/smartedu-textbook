# 平台接口实测记录

实测日期：2026-07-30。所有结论均来自真实请求，非文档推断。
平台如有变更，以实际请求为准。

---

## 1. 资源鉴权：两条通道

| 资源 | 路径 | 状态 |
|---|---|---|
| 整本 PDF | `https://r{1,2,3}-ndr.ykt.cbern.com.cn/edu_product/esp/assets/{contentId}.pkg/pdf.pdf` | **403 AccessDenied**，需登录 Token |
| 单页图片 | `https://r{1,2,3}-ndr.ykt.cbern.com.cn/edu_product/esp/assets/{contentId}.t/zh-CN/{timestamp}/transcode/image/{page}.jpg` | **200，公开** |

`r1` / `r2` / `r3` 为等价镜像，内容一致，可用于失败轮换。

本工具只使用图片通道，不实现 PDF 通道（那需要用户账号凭据）。

## 2. 三个元数据来源的对比

### 2.1 老目录接口 —— 已失效（路径错误）

```
https://s-file-1.ykt.cbern.com.cn/zxx/ndrs/tch_material/version/data_version.json
→ 403
```

网络上大量教程使用此路径，导致普遍认为"平台接口已关闭"。
**实际是路径少了 `resources/`。**

### 2.2 单本详情接口 —— 可用但数据过期

```
https://s-file-1.ykt.cbern.com.cn/zxx/ndrs/resources/tch_material/details/{contentId}.json
→ 200，约 13.7 KB
```

问题有二：

1. **坐标过期**。返回的 timestamp 指向旧版本资源。
2. **出版社字段错误**。`provider_list[0].name` 返回的是平台方"智慧中小学"，
   而非真实出版社。

**结论：不要用这个接口。**

### 2.3 目录分片 —— 权威源（本工具使用）

```
https://s-file-1.ykt.cbern.com.cn/zxx/ndrs/resources/tch_material/version/data_version.json
→ 200
{
  "module": "tch_material",
  "module_version": 1919079415,
  "urls": "https://…/part_100.json,https://…/part_101.json,https://…/part_102.json,https://…/part_103.json"
}
```

4 个分片合计约 **33 MB**，共 **3637 条**记录。
`module_version` 可用于判断本地缓存是否过期。

分片规模实测：

| 分片 | 大小 |
|---|---|
| part_100.json | 10.2 MB |
| part_101.json | 10.2 MB |
| part_102.json | 7.2 MB |
| part_103.json | 5.9 MB |

学段分布：小学 1282、初中 871、高中 516。

## 3. 目录条目的可用字段

```jsonc
{
  "id": "c0cc18a9-e27d-40ab-96dd-b9236e19080c",
  "title": "（根据2022年版课程标准修订）义务教育教科书•数学三年级上册",
  "tag_list": [
    {"tag_name": "三年级"}, {"tag_name": "电子教材"}, {"tag_name": "苏教版"},
    {"tag_name": "数学"},   {"tag_name": "上册"},     {"tag_name": "小学"}
  ],
  "provider_list": [{"name": "江苏凤凰教育出版社"}],   // ← 详情接口拿不到
  "custom_properties": {
    "preview": {
      "Slide1": "https://r3-ndr…/assets/{cid}.t/zh-CN/{ts}/transcode/image/1.jpg",
      "Slide2": "…"
    }
  },
  "ti_items": []       // ← 目录分片里此字段为空，坐标只能从 preview 取
}
```

**坐标提取**：从 `custom_properties.preview` 任取一条 URL，正则解析出
`contentId` 与 `timestamp`。

**可用率实测：84.8%（3085 / 3637）**。其余条目无 preview，需回退到用户提供 URL。

### 3.1 ⚠️ subject / volume 字段污染（2026-09 / 2026-11 复测）

`tag_list` 与解析出的 `subject` / `volume` **并不总是可靠的科目/册次**，分三种情况：

| 学段 | subject 实际内容 | volume | 后果 |
|---|---|---|---|
| 小学 / 初中（多数科目） | 正常科目名 | 正常册次 | 结构化参数可用 |
| 小学 / 初中 · 英语类 | **年度**（如 `2026年度`） | 正常册次 | `--subject 英语` 搜不到译林版/人教 PEP 等主流版本，`subject=英语` 只剩接力社、重庆大学等冷门版本 |
| **高中（全部科目）** | **册次**（如 `必修 第一册`） | **空** | 科目名只出现在 `title` 里，结构化参数无法表达科目 |

同一本教材在目录里还可能存在多条**年度记录**（书名完全相同，仅 timestamp 不同，
如译林版英语八年级上册同时有 2024 / 2026 两条），`timestamp` 最大者为最新版。

**实践结论**：

1. 检索一律优先走 `--search` 的标题匹配（query 分词后在 title/subject/version/
   grade/volume/stage/publisher 上做 AND 包含匹配）；
2. 高中教材搜索词用「教科书·{科目}必修」整段子串，可精确排除「选择性必修」；
3. 需要锁定多条同名记录中的最新版时，把年度当 subject 传
   （如 `--subject 2026年度`），或直接读 `catalog.json` 比 `timestamp`；
4. 副作用：归档路径由 `subject` 拼出，脏字段会导致目录名变成年度/册次
   （高中甚至没有科目层），需下载后手工整理并在 `info.json` 里修正。

## 4. ⚠️ 版本错配陷阱

同一本《苏教版数学三年级上册》，两个来源给出不同 timestamp：

| 来源 | timestamp | 第 1 页 | 第 50 页 | 第 132 页 |
|---|---|---|---|---|
| 详情接口 | 1688374778076（≈2023-07） | 200 | 200 | **403** |
| 目录分片 | 1754968748479（≈2025-08） | 200 | 200 | **200** |

两者指向**同一本书的不同版本**。旧坐标下载**不会报错**——文件能下、PDF 能合成、
页数看着也正常，只有翻到版权页才发现是旧版教材。

2024–2025 正是义务教育教材按 2022 年版新课标换版的窗口期，新旧版单元结构差异很大。

**本工具的防护**：只从目录分片取坐标；`info.json` 存档完整坐标便于追溯；
下载完成后提示用户核对版权页。

## 5. 其它实测细节

- **HEAD 请求不被支持**，探测必须用 GET（可加 `Range: bytes=0-1023` 只取头部）
- **不存在的页返回 403 而非 404**；而限流同样返回 403，两者无法从状态码区分。
  因此判定末页前需二次确认（间隔 1.5 s，换镜像），否则会把限流误判为"已下完"
- 请求需带常规浏览器 `User-Agent`；带 `Referer: https://basic.smartedu.cn/` 更稳
- 教材总页数无统一字段，只能探测。实测范围 100–200 页

## 6. 与 tchMaterial-parser 的差异

[happycola233/tchMaterial-parser](https://github.com/happycola233/tchMaterial-parser)
是同类工具中最知名的（v3.3.4，2025-07 仍在维护）。主要差异：

| | tchMaterial-parser | 本工具 |
|---|---|---|
| 下载通道 | 整本 PDF | 单页图片合成 |
| 是否需要账号 | **需要**（v3.0 起引入 Access Token） | 不需要 |
| 归档 | 手动指定路径 | 按年级/科目/版本自动归档 |
| 形态 | GUI + CLI | CLI + Claude Code Skill |

需要批量下载大量册次时，tchMaterial-parser 更省事（前提是愿意登录）。
