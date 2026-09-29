# 闲鱼店铺对标库采集

> WorkBuddy Skill · 屋里涛说

给定闲鱼店铺分享链接或 userid，批量抓取店铺全部在售商品的标题/文案/想要数/浏览数/主图，输出每店 Excel 对标表 + Top20 商品复刻 PDF。

## 技能清单

| 技能 | 说明 |
|---|---|
| **xianyu-store-benchmark** · 对标库采集 | 一店一表 + Top20 爆款复刻文档，用于选题与标题复刻。 |


## 安装

把 `skills/` 下的技能目录拷贝到 WorkBuddy 的技能目录：

```bash
cp -r skills/* ~/.workbuddy/skills/
```

Windows PowerShell：

```powershell
Copy-Item .\skills\* "$env:USERPROFILE\.workbuddy\skills\" -Recurse -Force
```

重启 WorkBuddy 后，技能列表即可看到。

## 使用要点

- 触发词：参考这几个闲鱼店铺、建对标库、爬闲鱼店铺商品、闲鱼对标分析、复刻闲鱼爆款标题。

## 环境依赖

- Python 3.13
- 网络访问

## 目录规范

```
xianyu-store-benchmark/
└── skills/
    ├── xianyu-store-benchmark/
```

每个技能遵循统一结构：`SKILL.md`（必需，含 name/description frontmatter）+ `scripts/`（可选）+ `references/`（可选）。

---

## License

MIT — 随意取用、修改、二次分发。
