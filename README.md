# alibaba-java-manual

Agent skill generated from *《Java开发手册》华山版 v1.5.0 + 泰山版 v1.6.0* and *《Java开发手册（泰山版）灵魂13问》* by 阿里巴巴与 Java 社区开发者 / Hollis with [book-to-skill](https://github.com/virgiliojr94/book-to-skill).

写 Java 代码或做 Code Review 时，按手册的【强制】/【推荐】/【参考】规约给出建议，覆盖编程规约、异常日志、单元测试、安全规约、MySQL、工程结构、设计规约七个维度。

## 安装

```bash
npx skills add https://github.com/xiaoleige0611/alibaba-java-manual --skill alibaba-java-manual
```

安装后重启 agent 会话。在 Claude Code 中可以用 `/alibaba-java-manual` 调用，也可以直接提问，例如“按阿里手册 review 这段代码”。

## 文件

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 核心规约、章节索引、主题索引 |
| `chapters/ch01–ch12` | 分章规约速查表、正反例、实战推演（ch11 日期时间、ch12 错误码为泰山版新增） |
| `chapters/ch13` | 《灵魂13问》：13 条规约背后的原理 |
| `glossary.md` | 术语表 |
| `patterns.md` | 可复用技法 |
| `cheatsheet.md` | 决策表、阈值与默认值 |

## 说明

内容是对手册规约的提炼与综合，不是原书全文。本 skill 基于华山版 v1.5.0（2019.06.19）生成，并已并入泰山版 v1.6.0（2020.04.22）与《灵魂13问》；后续的嵩山版、黄山版有新增和修改，需要以最新版为准时请对照[官方仓库](https://github.com/alibaba/p3c)。

## 来源与许可

- 原著：《Java开发手册》，阿里巴巴与 Java 社区开发者，依 [Apache License 2.0](https://github.com/alibaba/p3c) 发布。
- 本仓库是上述手册的衍生整理，同样以 [Apache License 2.0](LICENSE) 发布。使用时请保留本声明。
