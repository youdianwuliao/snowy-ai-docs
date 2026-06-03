# Snowy 框架 AI 开发规范

> 让 Codex / Claude Code / Cursor 秒懂 Snowy 框架，不再给出不符合框架规范的代码。

## 用法

把 `CLAUDE.md` 放到 Snowy 项目根目录。AI 工具自动读取后：

| 你说 | AI 生成 |
|------|---------|
| "用 Snowy 方式建一个商品管理" | Entity + Mapper + Service + Controller + 前端页面 |
| "加一个用户角色接口" | 按 Snowy 规范的 Param → Service → Controller |
| "新增业务模块" | 完整的前后端 CRUD |

## 内容

| 章节 | 说明 |
|------|------|
| 后端规范 | Entity / Service / Controller / Param 模式 |
| 前端规范 | API / 列表页 / 表单弹窗 / 路由 模式 |
| CRUD 步骤 | 后端 6 步 + 前端 4 步 |
| Add.import | AI 写代码时自动使用的那些 import |

## 关于 Snowy

[Gitee: xiaonuobase/snowy](https://gitee.com/xiaonuobase/snowy) — 国产前后分离开发平台

- Vue 3 + Antd Vue 4 + Vite 5
- Spring Boot 3 + MyBatis-Plus
- Sa-Token 鉴权
- Apache 2.0 开源免费商用
