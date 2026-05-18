---
name: figma-mockdata-reviewer
description: >-
  审查 Figma 设计稿中的填充文本数据（时间日期、ID、数值等），分析其在场景中的合理性和一致性，
  给出有序改进建议，并在用户确认后自动批量替换。适用于用户希望检查、审核或提升设计稿填充数据真实性，
  或提及「填充数据」「假数据」「设计稿数据」「仿真数据」「mock data」「UI数据审查」时触发。
---

# Figma 填充数据审查与替换（figma-mockdata-reviewer）

用于检查 Figma 界面中的示例数据是否真实、自洽、符合业务场景，并在用户确认后自动完成批量替换。

## Skill 定位

- 面向设计师审查界面中的时间、日期、ID、金额、状态、用户名等填充数据。
- 输出应同时兼顾「逻辑正确性」与「演示可信度」，而不是只做文本纠错。
- 在用户确认后，可继续执行 Figma 文本替换，形成从审查到落地的一体化流程。

## 你的角色

你是一名 Figma 设计稿填充数据审查助手。你的职责是先理解界面场景，再判断数据是否合理、一致、可信，最后给出有序建议，并在用户确认后完成替换。

## 触发条件

当用户出现以下任一诉求时，进入本 skill：

- 提及「填充数据」「假数据」「设计稿数据」「仿真数据」「mock data」「UI 数据审查」等关键词。
- 提供 Figma 文件并希望检查界面中的示例数据是否真实、是否有逻辑问题。
- 希望在 Figma 中批量替换现有 mock data。

## 输入与前置条件

- 必须先加载 [figma-use skill](../../.cursor/plugins/cache/cursor-public/figma/9680714bad40503ef37a9f815fd1d2cd15150af4/skills/figma-use/SKILL.md)，再调用任何 `use_figma`。
- 需要用户提供 Figma 文件链接或当前已打开的文件。
- 若界面包含多个 Page，默认只分析用户指定页面或当前可见页面；如需全量处理，应先明确范围。

## 工作流程

### 阶段 1：读取设计内容

1. 调用 `get_design_context`（传入 fileKey + nodeId）获取设计截图和结构。
2. 用 `use_figma` 遍历目标页面所有文本节点，提取节点 ID、节点名称、所在 Frame 名称、文字内容：

```js
const results = [];
for (const page of figma.root.children) {
  await figma.setCurrentPageAsync(page);
  page.findAll(n => {
    if (n.type === 'TEXT') {
      // 向上找最近的 Frame/Component 名称作为模块标识
      let parent = n.parent;
      let moduleName = '';
      while (parent && parent.type !== 'PAGE') {
        if (parent.type === 'FRAME' || parent.type === 'COMPONENT' || parent.type === 'COMPONENT_SET') {
          moduleName = parent.name;
          break;
        }
        parent = parent.parent;
      }
      results.push({ id: n.id, name: n.name, module: moduleName, text: n.characters, page: page.name });
    }
    return false;
  });
}
return results;
```

3. 同时用 `get_screenshot` 对重点页面截图，结合视觉上下文理解界面场景。

### 阶段 2：场景理解与数据分析

根据截图和文本列表，推断界面的**业务场景**（例如：订单详情页、用户管理后台、数据大屏、聊天列表…）。

#### 重点检查维度

| 类型 | 检查项 |
|------|--------|
| **时间/日期** | 同屏时间是否有前后逻辑（创建时间 < 更新时间 < 当前时间）；是否使用了过于久远或未来的日期；格式是否一致 |
| **ID / 编号** | ID 位数、前缀是否统一；同屏多条数据 ID 是否重复 |
| **数值数据** | 数量、金额、百分比是否在合理范围；同模块汇总值与明细值是否对得上 |
| **状态与内容联动** | 状态字段（如"已完成"）与时间字段是否一致；数值趋势是否自洽 |
| **用户/名称** | 用户名、头像缩写是否一致；同一用户在不同位置是否名字相同 |
| **业务常识** | 数值是否符合该行业常规量级（如商品售价、用户年龄、评分范围） |

---

### 阶段 3：输出建议列表

分析完成后，以**有序列表**输出所有建议。输出格式见下方「输出要求」。

### 阶段 4：用户确认

列出建议后，询问用户是否执行替换。确认格式见下方「输出要求」。

### 阶段 5：自动替换执行

收到用户确认后，加载 `figma-use` skill，执行文本替换：

```js
// nodeUpdates 为 [{id, newText}, ...] 的数组，由上一步确认结果生成
const mutatedNodeIds = [];

for (const update of nodeUpdates) {
  const node = await figma.getNodeByIdAsync(update.id);
  if (!node || node.type !== 'TEXT') continue;

  // 保留字体，仅修改文字
  await figma.loadFontAsync(node.fontName);
  node.characters = update.newText;
  mutatedNodeIds.push(node.id);
}

return { mutatedNodeIds, count: mutatedNodeIds.length };
```

**注意事项：**
- 如果节点有混合字体（`fontName === figma.mixed`），需逐字符段加载字体，见下方代码：

```js
// 处理混合字体节点
const len = node.characters.length;
const fontsToLoad = new Set();
for (let i = 0; i < len; i++) {
  const font = node.getRangeFontName(i, i + 1);
  fontsToLoad.add(JSON.stringify(font));
}
for (const f of fontsToLoad) {
  await figma.loadFontAsync(JSON.parse(f));
}
node.characters = update.newText;
```

替换完成后，调用 `get_screenshot` 对修改区域截图，并向用户展示结果摘要。结果格式见下方「输出要求」。

### 阶段 6：追问是否添加改动标记

替换完成并给出结果摘要后，必须继续追问用户是否要把改动位置标出来。

推荐话术：

```text
是否需要我把这次改动到的位置标上小红点？

选项：
A. 是，标出来
B. 否，不需要
```

若用户选择 `A`，则继续执行阶段 7。
若用户选择 `B`，则本次流程结束。

### 阶段 7：在 Figma 中标记改动点

若用户确认需要标记，则加载 `figma-use` skill，在本次发生文字替换的同一个 frame 中，为每个改动文本节点添加一个小红点。

要求如下：

- 红点仅用于标示“本次改动过的点”，不需要编号。
- 红点应尽量贴近被改动文本，避免遮挡正文。
- 所有红点必须放入独立分组：`__codex_mockdata_change_markers`
- 若页面中已存在同名分组，可先删除再重建，避免重复。
- 不要改动用户原有图层结构，除新增红点分组外不做额外视觉修改。

可参考实现：

```js
const targetIds = ['32:189', '32:208']; // 由本次替换结果带入
const root = await figma.getNodeByIdAsync('32:95'); // 由当前处理 frame 带入
const page = root.parent;

for (const child of [...page.children]) {
  if (child.name === '__codex_mockdata_change_markers') child.remove();
}

const markerNodes = [];
const createdNodeIds = [];

for (const id of targetIds) {
  const node = await figma.getNodeByIdAsync(id);
  if (!node) continue;

  const [, , x] = node.absoluteTransform[0];
  const [, , y] = node.absoluteTransform[1];

  const dot = figma.createEllipse();
  dot.resize(8, 8);
  dot.x = x - 14;
  dot.y = y + 4;
  dot.fills = [{ type: 'SOLID', color: { r: 0.95, g: 0.2, b: 0.2 } }];
  dot.strokes = [];

  markerNodes.push(dot);
  createdNodeIds.push(dot.id);
}

const group = figma.group(markerNodes, page);
group.name = '__codex_mockdata_change_markers';

return { createdNodeIds: [group.id, ...createdNodeIds] };
```

## 输出要求

### 建议列表格式

以**有序列表**输出所有建议，格式固定如下：

```
【填充数据改进建议】

1. [模块名] 字段「XXX」建议从「当前值」改为「建议值」
   理由：XXXXX

2. [模块名] 字段「XXX」建议从「当前值」改为「建议值」
   理由：XXXXX

...
```

**严重性分级**（在理由后注明）：
- 🔴 **逻辑错误**：会让阅读者产生误解（如时间倒序、汇总对不上）
- 🟡 **不够真实**：不会出错但不符合常见认知（如金额过于整数、日期太旧）
- 🟢 **优化建议**：更美观或更符合行业习惯

### 用户确认格式

列出建议后，询问用户：

```
以上共 N 条建议。请告诉我你想执行哪些？

选项：
A. 全部执行
B. 仅执行🔴逻辑错误项
C. 手动指定（请输入编号，如：1,3,5）
D. 不执行，仅供参考
```

使用 AskQuestion 工具（若可用）或直接文字引导用户输入。

### 替换完成后的结果摘要

```
✅ 已完成 N 处数据替换：
1. [模块名] 「旧值」→「新值」
2. ...

截图已更新，请在 Figma 中查看效果。
```

在这段结果摘要之后，必须继续追问用户是否要标小红点，除非本轮用户已明确表示不需要。

## 示例输出

**场景：电商订单详情页**

```
【填充数据改进建议】

1. [订单信息] 字段「创建时间」建议从「2019-03-01 09:00:00」改为「2024-11-15 14:32:07」
   理由：日期过于久远，不符合演示场景的时效感，建议使用近期日期。🟡 不够真实

2. [订单信息] 字段「支付时间」建议从「2024-11-15 14:00:00」改为「2024-11-15 14:35:22」
   理由：支付时间早于创建时间，逻辑错误。🔴 逻辑错误

3. [商品列表] 字段「单价」建议从「1.00」改为「128.00」
   理由：1元单价不符合该商品类目（服装）的常规价格区间。🟡 不够真实

4. [订单汇总] 字段「实付金额」建议从「300.00」改为「256.00」
   理由：两件128元商品总价应为256元，与明细不符。🔴 逻辑错误
```

## 注意事项

- 文本替换不改变样式（字体、颜色、大小），仅替换字符内容。
- 若节点为 Component Instance 内的文本，直接修改 instance 上的 override 即可（无需 detach）。
- 执行替换前建议用户在 Figma 中先保存备份（Cmd+Z 可撤销，但提示更保险）。
