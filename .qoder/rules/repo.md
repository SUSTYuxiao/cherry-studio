---
trigger: always_on
alwaysApply: true
---

## 代码修改工作流规则

当你需要修改现有文件时，必须遵循以下shadow文件夹策略：

### 核心规则：
1. **影子文件夹原则**: 绝不直接修改原文件
2. **命名规范**: 创建同名文件夹 + "_shadow" 后缀
3. **修改位置**: 所有变更都在shadow文件夹内进行
4. **同步责任**: 手动或定期将shadow变更与原文件对比合并

### 操作流程：
1. 识别要修改的原始文件/文件夹
2. 创建对应的shadow目录（如果不存在）
3. 在shadow目录中创建和修改文件
4. 提供手动合并指导（如需要）

### 示例：
- 修改: `/src/components/Button.js`
- 创建: `/src/components_shadow/Button.js`
- 说明: "我在shadow目录中创建了修改版本，请后续手动对比合并"

请始终遵循这个工作流，除非我明确要求直接修改原文件。


## 功能开发Shadow规则

当开发新功能时：
1. 在shadow目录中完整实现功能
2. 保持shadow目录结构清晰
3. 提供功能测试和使用说明
4. 列出需要同步的文件清单

示例：
"我将在 /src/utils_shadow/ 中实现新工具函数：
- 新文件: stringUtils.js
- 修改: index.js（添加导出）
- 测试: stringUtils.test.js
- 合并步骤: 1. 对比原utils目录 2. 逐个文件合并"

## Bug修复Shadow规则

修复bug时的shadow工作流：
1. 在shadow目录中创建修复版本
2. 详细说明修复逻辑
3. 提供回归测试建议
4. 标记影响范围

响应模板：
"🔧 Bug修复完成
- 位置如: /components_shadow/Form.js
- 问题: [bug描述]
- 解决方案: [修复方案]
- 影响范围: [可能受影响的模块]
- 合并检查项: [需要验证的点]"

## 代码重构Shadow规则

重构时遵循：
1. 在shadow目录创建重构版本
2. 保持原API兼容性说明
3. 提供迁移指南
4. 性能对比数据（如果适用）

输出格式：
"🔄 重构完成
- 重构范围如: /src/services_shadow/
- 新架构: [架构说明]
- 兼容性: [API变更说明]
- 迁移步骤: [如何从原版本迁移]"

## 智能合并助手

当shadow版本创建后，自动提供：
```diff
@@ -原文件 vs shadow文件差异 @@
- 删除的代码
+ 新增的代码
~ 修改的代码

💡 合并建议：
1. [第一步操作]
2. [第二步操作]
3. [验证步骤]

## Shadow同步状态

维护一个shadow状态文件，如：
```json
// .shadow_status.json
{
  "last_sync": "2024-01-20",
  "shadow_directories": ["src_shadow", "components_shadow"],
  "pending_merges": [
    {"file": "Button.js", "status": "ready", "priority": "high"}
  ],
  "conflicts": []
}
