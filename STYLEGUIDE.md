# 样式指南

## 许可证头

所有源文件必须包含 SPDX 版权头。

```bash
# 添加/更新许可证头
mise run license:update

# 检查许可证头
mise run license:check
```

## 代码风格

- Rust：用 `rustfmt` 格式化，用 Clippy 检查。
- Python：用 `ruff` 格式化和检查，用 `ty` 做类型检查。

提交 PR 前的推荐流程：

```bash
mise run fmt
mise run lint
mise run ci
```

## CLI 输出风格

打印结构化 CLI 输出时：

- 标题后加一个空行。
- 键值字段缩进 2 个空格。
- 字段键使用暗色标签（例如 `"Id:".dimmed()`）。
- 主标题使用青色 + 粗体。

正确：

```text
Created sandbox:

  Id: cddeeb6d-a4d3-4158-a4d1-bd931f743700
  Name: sandbox-cddeeb6d
  Namespace: openshell
```

错误：

```text
Created sandbox:
  Id: cddeeb6d-a4d3-4158-a4d1-bd931f743700
  Name: sandbox-cddeeb6d
  Namespace: openshell
```