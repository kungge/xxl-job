# xxl-job 项目长期记忆

## 目录约定
- `kun-doc/`：本项目 troubleshooting / 分析文档沉淀目录，命名格式 `YYYY-MM-DD_主题.md`，文档结构统一为 `> 日期：` `> 场景：` 头部 + 编号小节（现象/排查/根因/解决方案/踩坑点/Checklist）+ 表格 + code blocks

## 环境要点
- 本地为 macOS（arm64），`/data` 受 SIP 保护只读，任何写死 `/data` 绝对路径的配置在本地跑不动，需改为 `/Users/wankun/data/...` 或相对路径
- 本地日志统一基路径：`/Users/wankun/data/applogs/`（logback.xml 中 `${LOG_HOME:-/Users/wankun/data/applogs}`）
- executor 的 `xxl.job.executor.logpath` 不支持 Spring `${var:-default}` 占位符语法，只能给确定路径
