# 用户信息清单 + 剪贴板复制模式（通用参考）

适用场景：
当一本书蒸馏出来的 Skill 在运行时，需要用户补充上下文，尤其是高时效、高准确性、强结构化的信息时，推荐默认采用本模式。

目标：
1. 先给用户一张结构化 Markdown 表格
2. 再自动复制同一份表格到剪贴板
3. 同时给出模板文件路径
4. 若复制失败，明确说明失败原因，不假装成功

## 推荐目录结构

在目标 Skill 目录中默认补齐：

```text
<skill-name>/
  SKILL.md
  README.md
  templates/
    input-checklist.md
  scripts/
    copy_input_checklist.sh
    show_and_copy_input_checklist.sh
```

## SKILL.md 建议写法

在需要用户补充信息的章节中，明确写出：

- 该 Skill 依赖用户提供哪些上下文
- 这些信息为什么需要
- 若信息缺失，只能输出低置信度预分析或待验证清单
- 显示问题表后，默认立刻执行 show-and-copy 脚本
- 若复制成功，明确告诉用户“问题表已复制到剪贴板”
- 若复制失败，明确告诉用户失败原因，并给出模板文件路径

建议固定写法：

```md
显示完问题列表后，默认立刻执行展示并复制脚本，而不是只停留在口头说明。

默认动作：
- 先展示同一份 Markdown 问题表
- 再立刻执行剪贴板复制动作
- 同时给出模板文件路径

优先使用模板文件：`templates/input-checklist.md`
默认执行脚本：`scripts/show_and_copy_input_checklist.sh`
复制子脚本：`scripts/copy_input_checklist.sh`
```

## README.md 建议写法

至少写明：
- 模板文件路径
- 默认展示并复制脚本
- 复制子脚本
- 适用场景

## 模板文件建议

模板文件建议命名为：

`templates/input-checklist.md`

模板内容建议：
- 标题
- 使用说明
- 多个分组表格
- 最小必需包
- 缺失信息时的后果说明

## 通用 copy 脚本模板

文件建议：
`scripts/copy_input_checklist.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SKILL_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"
TEMPLATE="$SKILL_DIR/templates/input-checklist.md"

if [[ ! -f "$TEMPLATE" ]]; then
  echo "Template not found: $TEMPLATE" >&2
  exit 1
fi

copy_with() {
  local cmd="$1"
  case "$cmd" in
    pbcopy) pbcopy < "$TEMPLATE" ;;
    wl-copy) wl-copy < "$TEMPLATE" ;;
    xclip) xclip -selection clipboard < "$TEMPLATE" ;;
    xsel) xsel --clipboard --input < "$TEMPLATE" ;;
    *) return 1 ;;
  esac
}

for candidate in pbcopy wl-copy xclip xsel; do
  if command -v "$candidate" >/dev/null 2>&1; then
    copy_with "$candidate"
    echo "Copied checklist to clipboard via $candidate"
    echo "Source: $TEMPLATE"
    exit 0
  fi
done

echo "No clipboard command found. Checklist file is at: $TEMPLATE" >&2
exit 2
```

## 通用 show-and-copy 脚本模板

文件建议：
`scripts/show_and_copy_input_checklist.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SKILL_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"
TEMPLATE="$SKILL_DIR/templates/input-checklist.md"
COPY_SCRIPT="$SCRIPT_DIR/copy_input_checklist.sh"

if [[ ! -f "$TEMPLATE" ]]; then
  echo "Template not found: $TEMPLATE" >&2
  exit 1
fi

copy_status=0
copy_output=""
if [[ -x "$COPY_SCRIPT" ]]; then
  if copy_output="$($COPY_SCRIPT 2>&1)"; then
    copy_status=0
  else
    copy_status=$?
  fi
else
  copy_output="Copy script not executable: $COPY_SCRIPT"
  copy_status=3
fi

printf '=== COPY STATUS ===\n'
if [[ $copy_status -eq 0 ]]; then
  printf '%s\n' "$copy_output"
else
  printf 'Copy failed (exit %s): %s\n' "$copy_status" "$copy_output"
fi

printf '\n=== TEMPLATE PATH ===\n%s\n' "$TEMPLATE"
printf '\n=== TEMPLATE CONTENT START ===\n'
cat "$TEMPLATE"
printf '\n=== TEMPLATE CONTENT END ===\n'

exit 0
```

## 最佳实践

- 模板必须是 Markdown，方便复制到聊天系统、表单系统、笔记系统
- 表格字段应按 Skill 的分析步骤来设计，而不是泛泛罗列信息
- 高时效信息必须由用户提供或确认主口径
- 脚本失败时，必须清楚报错，不允许静默失败
- 输出给用户时，要同时说清：是否已复制、模板路径是什么、如果未复制成功该怎么做

## 不该做的事

- 不要只显示表格，却不执行复制动作
- 不要只说“你可以复制一下”而不真正执行
- 不要在复制失败时假装成功
- 不要用 agent 默认搜索替代用户提供的高时效主口径
