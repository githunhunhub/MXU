# 任务循环执行功能

## 需求

给每个任务添加「循环执行」开关。用户可以对任意任务标记循环，标记后的执行逻辑：

1. **首轮**：所有启用的任务按序完整执行一遍（A → B → C → D → E）
2. **循环阶段**：只重复执行标记了循环的任务（C → E → C → E → C → E → ...）
3. **终止**：用户手动停止

示例：任务列表 A(普通) B(普通) C(循环) D(普通) E(循环)，执行序列为：

```
A → B → C → D → E → C → E → C → E → C → E → ...
```

---

## 改动清单

### 1. 类型层

**文件：`src/types/interface.ts`**

`SelectedTask` 新增 `loop` 字段：

```typescript
export interface SelectedTask {
  id: string;
  taskName: string;
  customName?: string;
  enabled: boolean;
  loop: boolean;              // ← 新增：是否循环执行
  optionValues: Record<string, OptionValue>;
  expanded: boolean;
}
```

**文件：`src/types/config.ts`**

`SavedTask` 新增 `loop` 字段（持久化到 mxu.json）：

```typescript
export interface SavedTask {
  id: string;
  taskName: string;
  customName?: string;
  enabled: boolean;
  loop?: boolean;             // ← 新增，可选以兼容旧配置
  optionValues: Record<string, OptionValue>;
}
```

### 2. Store 层

**文件：`src/stores/types.ts`** — 新增 action 类型声明

```typescript
toggleTaskLoop: (instanceId: string, taskId: string) => void;
```

**文件：`src/stores/appStore.ts`** — 实现 toggleTaskLoop

```typescript
toggleTaskLoop: (instanceId, taskId) => {
  set((state) => ({
    instances: state.instances.map((inst) =>
      inst.id === instanceId
        ? {
            ...inst,
            selectedTasks: inst.selectedTasks.map((t) =>
              t.id === taskId ? { ...t, loop: !t.loop } : t,
            ),
          }
        : inst,
    ),
  }));
},
```

### 3. 配置同步

**文件：`src/services/configService.ts`**

在 `instanceToSaved` 函数中，导出时包含 `loop`：

```typescript
// SavedTask 自然包含 loop 字段，无需额外处理
// 读取时使用 loop ?? false 兼容旧配置
```

在 `savedToInstance` 或配置加载逻辑中，读取 `loop` 时做兼容：

```typescript
loop: savedTask.loop ?? false,
```

### 4. 执行逻辑（核心）

**文件：`src/components/Toolbar.tsx`**

在 `startTasksForInstance` 函数中，所有批次（primary + trailing）提交并等待完成后，检查是否有循环任务需要重启：

```typescript
// === 在 return true 之前插入循环检查 ===

// 等待所有任务完成（primary + trailing）
if (startedTaskIds.length > 0) {
  const result = await maaService.waitForTasks(targetId, startedTaskIds);
  if (!result.allDone || result.stopped) {
    return false;
  }
}

// 检查是否有循环任务
const currentTasks = useAppStore.getState().instances.find(i => i.id === targetId)?.selectedTasks || [];
const hasLoopingTasks = currentTasks.some(t => t.enabled && t.loop);

if (hasLoopingTasks) {
  // 循环阶段：每个周期独立提交，resetState=true
  while (true) {
    const latestInstance = useAppStore.getState().instances.find(i => i.id === targetId);
    if (!latestInstance || !latestInstance.isRunning) break;

    const loopTasks = latestInstance.selectedTasks.filter(t => t.enabled && t.loop);
    if (loopTasks.length === 0) break;

    // 构建循环任务的 RunnableTask[]
    const loopRunnableTasks = loopTasks.map(selectedTask => {
      const specialTask = getMxuSpecialTask(selectedTask.taskName);
      const taskDef = specialTask?.taskDef || projectInterface?.task.find(t => t.name === selectedTask.taskName);
      return { taskName: selectedTask.taskName, selectedTask, taskDef, specialTask };
    }).filter(r => r.taskDef);

    if (loopRunnableTasks.length === 0) break;

    // 提交循环批次（resetState=true，每个周期独立状态）
    // 不能用 resetState=false 追加，因为 handle_task_callback 不清空
    // pending_task_ids，多次追加会导致旧 ID 累积
    const loopTaskIds = await runTaskBatch(loopRunnableTasks, true, '循环');
    if (loopTaskIds.length === 0) break;

    // 等待循环批次完成
    const loopResult = await maaService.waitForTasks(targetId, loopTaskIds);
    if (!loopResult.allDone || loopResult.stopped) break;
  }
}
```

**停止检测**：通过 `waitForTasks` 返回的 `stopped` 字段判断用户是否手动停止。

**为什么用 `resetState=true`**：后端 `handle_task_callback` 在所有任务完成后只清空 `task_ids`，不清空 `pending_task_ids`。如果用追加模式（`false`），多次循环后 `pending_task_ids` 会不断累积旧 ID，造成脏数据。使用 `reset=true` 让每个周期独立：清空旧状态、重建映射、`overall_status` 重置为 Running。

### 5. UI 层 — 双入口

#### 5a. 右键菜单入口

**文件：`src/components/listItemShared.tsx`**

`ListItemMenuLabels` 新增标签：

```typescript
export interface ListItemMenuLabels {
  // ...existing labels
  loopOn: string;      // "循环执行"
  loopOff: string;     // "取消循环"
}
```

`ListItemMenuConfig` 新增配置：

```typescript
export interface ListItemMenuConfig {
  // ...existing config
  isLooping: boolean;
  onToggleLoop: () => void;
}
```

`buildListItemMenuItems` 在「启用/禁用」菜单项后插入循环开关：

```typescript
{
  id: 'toggle-loop',
  label: isLooping ? labels.loopOff : labels.loopOn,
  icon: Repeat,  // lucide-react 的 Repeat 图标
  disabled: isLocked,
  onClick: cfg.onToggleLoop,
},
```

#### 5b. 任务项图标按钮入口

**文件：`src/components/TaskItem.tsx`**

在任务名称旁边（勾选框右侧或名称右侧）添加循环图标按钮：

```tsx
<button
  className={clsx(
    'p-0.5 rounded transition-colors',
    task.loop ? 'text-[var(--color-accent)]' : 'text-[var(--color-text-muted)] hover:text-[var(--color-text-primary)]',
  )}
  onClick={() => toggleTaskLoop(instanceId, task.id)}
  disabled={isInstanceRunning}
  title={task.loop ? t('task.loopOn') : t('task.loopOff')}
>
  <Repeat size={14} />
</button>
```

图标使用 lucide-react 的 `Repeat` 图标（两支循环箭头），循环开启时高亮为强调色。

### 6. 国际化

**文件：`src/i18n/locales/*.json`**（5 个语言文件）

新增翻译键：

| 键 | zh-CN | zh-TW | en-US | ja-JP | ko-KR |
|---|---|---|---|---|---|
| `task.loopOn` | 循環執行 | 循環執行 | Loop | ループ | 반복 |
| `task.loopOff` | 取消循環 | 取消循環 | Stop Looping | ループ解除 | 반복 해제 |

### 7. 组件调用更新

**文件：`src/components/TaskItem.tsx`**

`handleContextMenu` 中构建菜单时传入新字段：

```typescript
const menuItems = buildListItemMenuItems({
  // ...existing fields
  isLooping: task.loop ?? false,
  onToggleLoop: () => toggleTaskLoop(instanceId, task.id),
});
```

---

## 改动文件汇总

| 文件 | 改动内容 |
|------|----------|
| `src/types/interface.ts` | `SelectedTask` 加 `loop: boolean` |
| `src/types/config.ts` | `SavedTask` 加 `loop?: boolean` |
| `src/stores/types.ts` | `AppState` 加 `toggleTaskLoop` action 类型 |
| `src/stores/appStore.ts` | 实现 `toggleTaskLoop` |
| `src/services/configService.ts` | 配置读写兼容 `loop` 字段 |
| `src/components/Toolbar.tsx` | 所有批次完成后加循环重提交逻辑 |
| `src/components/listItemShared.tsx` | 右键菜单加循环开关项 |
| `src/components/TaskItem.tsx` | 任务项加循环图标按钮 + 菜单调用更新 |
| `src/i18n/locales/*.json` | 新增 `task.loopOn` / `task.loopOff` 翻译 |

---

## 兼容性

- `loop` 字段在 `SavedTask` 中为可选（`loop?: boolean`），旧版 mxu.json 读取时默认 `false`
- 不影响定时执行、前置动作等现有功能
- 循环阶段的任务使用追加模式（`resetState=false`），不重置已完成状态
