# 开源项目
## 一、ui-ux-pro-max-skill
git:https://github.com/nextlevelbuilder/ui-ux-pro-max-skill


## 二、impeccable
git:https://github.com/pbakaus/impeccable

# 流程
```
需求
 ↓
UI/UX Pro Max
 ↓
Design System
 ↓
页面设计
 ↓
写代码
 ↓
Impeccable
 ↓
Critique
 ↓
Audit
 ↓
Polish
 ↓
最终页面
```
UI/UX Pro Max 会根据产品类型、行业等生成设计系统，并且可以持久化成 `design-system/MASTER.md` 作为项目的设计真相源。

Impeccable 则明确提供 `critique`、`audit`、`polish` 等命令来做后续评估和修正。

## 第一步 让 UI/UX Pro Max 建立“财务系统设计宪法”
如果你的 Skill 支持 `/ui-ux-pro-max`，可以直接使用；
## 第二步 一定要把 Design System 固化下来
UI/UX Pro Max 的 `--persist` 就是为了这种场景：生成 `design-system/MASTER.md` 作为全局 Source of Truth。
## 第三步 再让 Cursor 做页面
例如第一张页面，我建议你做：
 General Ledger
不要一上来 Dashboard。
## 第四步 现在才轮到 Impeccable
页面完成后，**不要让 UI/UX Pro Max 再重新设计一遍。**
而是让 Impeccable 来当：

> **Design Reviewer**

先：
/impeccable critique General Ledger
它主要检查：
- 信息层级
- 页面结构
- Cognitive Load
- UX clarity
- 操作路径
- 视觉重点
Impeccable 的 `critique` 就是针对 UX 层面的 review。
## 第五步 然后 Audit
接下来：
/impeccable audit General Ledger
这个阶段重点不是“好不好看”。
而是：
	Accessibility
	Responsive
	Performance
	Interaction
	Loading
	Error
	Focus
	Contrast
	Touch target
	Typography
Impeccable 的 audit 会寻找系统性问题，并可以把发现映射到后续修复命令。

## 第六步 然后才 Polish
最后：
	/impeccable polish General Ledger
这个阶段才是：
 	**把已经正确的东西打磨到专业。**
原来
	Debit       Credit
	1234567     0
	0           823456

可能会进一步变成：
	Debit          Credit
	1,234,567.00   —
	—              823,456.00