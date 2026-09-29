---
name: xianyu-store-benchmark
description: 闲鱼店铺对标库采集——给定闲鱼店铺分享链接或 userid，批量抓取店铺全部在售商品的标题/文案/想要数/浏览数/主图，输出每店 Excel 对标表 + Top20 商品复刻 PDF。当用户说「参考这几个闲鱼店铺」「建对标库」「爬闲鱼店铺商品」「闲鱼对标分析」「复刻闲鱼爆款标题」时使用。
agent_created: true
---

# 闲鱼店铺对标库采集

从闲鱼店铺批量采集商品数据并产出对标库（Excel + PDF）。

## 前置依赖

- Node 22+（`node:sqlite`、全局 `WebSocket`、`fetch` 可用）
- Python 3 + `openpyxl`、`Pillow`、`pypdf`（PDF 合并）、`PyMuPDF`（可选，核验）
- Chrome（路径可用 `CHROME_EXE` 环境变量覆盖）
- 一次**扫码登录**（详情接口必须登录态，见下）

## 核心接口

| 用途 | 接口 | 是否需登录 |
|---|---|---|
| 店铺商品列表 | `mtop.idle.web.xyh.item.list` | 否 |
| 商品详情（文案+浏览数） | `mtop.taobao.idle.pc.detail` | **是** |

**列表参数**：`{needGroupInfo:false, pageNumber, userId, pageSize:20, defaultGroup:true}`

**鉴权链路**：`_m_h5_tk` 从公告接口 `mtop.gaia.nodejs.gaia.idle.data.gw.v2.index.get` 获取；
`sign = md5(token & t & appKey & json(data))`，`appKey=34839810`，Cookie 域需为 `.goofish.com`。

**从短链取 userid**：请求 `https://m.tb.cn/h.xxxx` 的 HTML，正则提取 `userid` 或跳转后的 `goofish.com` 链接。

## 字段来源

- 列表：标题、售价、原价（藏在标签 `¥xx`）、**想要数**（标签 `N人想要`）、热销排名（`热销第N名`）、主图
- 详情：`itemDO.desc`（文案）、`itemDO.browseCnt`（浏览数）、`itemDO.wantCnt`、`imageInfos`

## 关键坑（务必先读）

1. **详情接口有 IP+会话级限流**：单窗口约 100~150 次请求后返回 `RGV587_ERROR::SM::哎哟喂,被挤爆啦`。
   该错误的 `data.url` 是登录跳转地址 —— 匿名调用时它同时意味着「需要登录」。
2. **严禁并发**：两个抓取进程并行会立刻互相触发限流。
3. **退避变量要在成功时归零**，否则一次限流后进程永久变慢（约 3 条/分钟）。
4. **失败记录不要写成"已完成"**，否则断点续传会把它们当已完成跳过。
5. **headless Chrome 会被识破**（页面返回"非法访问"），登录必须 headful。
6. **Edge 的 Cookie 拿不到**（v20 App-Bound 加密 + 文件被占用）。正解：自己起常驻 Chrome 扫码登录。
7. `nohup ... &` 在受限沙箱里会随命令结束被回收；长任务用前台分段跑（脚本内置断点续传）。
8. **Chrome headless `Page.printToPDF` 遇到 `<img>` 会死锁**（多页文档必挂，单页带图正常）。
   方案：**逐件单页打印 + `pypdf` 合并**。
9. 打印用图先降采样（手机原图 2516×3660 → 宽 1000px JPEG），否则栅格化极慢。

## 工作流

### 1. 起常驻 Chrome 扫码登录
```
node xy_login.js          # 打开 passport.goofish.com 登录页，headful，profile 持久化（端口 9336）
```
用户用闲鱼 App 扫码。用 `xy_check.js` 确认（页面出现昵称/「快速进入」即成功）。

### 2. 抓列表
```
node xy_list_only.js      # 输出 _data/<uid>_list.json
```

### 3. 抓详情（登录态）
```
node xy_detail_login.js 2,3,1     # 参数为店铺索引，可逗号分隔；支持断点续传
XY_DELAY=500 ...                  # 降低间隔提速（默认 900ms）
```
从常驻 Chrome 的 CDP `Network.getAllCookies` 取 Cookie 回灌 Node 客户端。
**注意**：登录后即使 `_m_h5_tk` 为空，请求仍可 SUCCESS。

限流时等冷却（≥5 分钟），用 `xy_fix.js` 定向补抓缺口（只补列表里有、详情缺文案的）。

### 4. 下载主图
```
node xy_images.js         # 存到 <店>/images/，文件名 <商品ID>_<序号>.jpg（实际为 webp）
```

### 5. 生成 Excel
```
python build_tables.py    # 每店一份 + 00_四店对标库总览.xlsx；排序：想要数↓，前20行高亮
```

### 6. 生成 PDF
```
python prep_pdf_images.py # Top20 主图降采样为 _pdf_src/img/<itemId>.jpg
node xy_pdf.js            # 逐件单页打印 -> 单件PDF/，再 pypdf 合并成 <店名>_Top20商品复刻.pdf
```

### 7. 总览页（可选）
```
node build_report.js      # 对标库总览.html
```

## 排序口径

用户口径以「想要人数」为主序、浏览数为次序（`build_tables.py` / `xy_pdf.js` 一致）。
若用户明确要求按浏览数，则只改排序键。

## 目录约定

```
<工作目录>/
  00_四店对标库总览.xlsx
  对标库总览.html
  0N_<店名>_<userid>/
    <店名>_商品对标库.xlsx
    <店名>_Top20商品复刻.pdf
    单件PDF/  （20 份单件）
    images/   （主图）
    _pdf_src/ （打印用 HTML 与降采样图）
  _scripts/   （全部脚本 + _data/ 中间数据）
```
