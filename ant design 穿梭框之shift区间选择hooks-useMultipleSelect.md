# useMultipleSelect Hook 源码解析

> 本文分析 Ant Design 中 `useMultipleSelect` 这个自定义 Hook 的实现原理、使用方法和 Shift 键区间选择机制。

## 1. Hook 概述

`useMultipleSelect` 是 Ant Design 中一个通用的自定义 Hook，主要用于实现类似文件管理器的"Shift 键区间多选"功能。这个 Hook 位于 `components/_util/hooks/useMultipleSelect.ts`，在 Transfer 穿梭框等组件中使用，为用户提供了更便捷的多选交互体验。

### 1.1 核心功能

- **记忆上次选择位置**：保存上一次选择的索引位置，用于确定区间起点
- **区间选择**：当用户按住 Shift 键点击时，自动选中/取消选中从上次选择到当前选择的所有项
- **智能判断操作类型**：自动判断是"全选区间"还是"取消区间"
- **支持泛型**：可处理任意数据类型，通过 `getKey` 函数提取唯一键

## 2. 源码详解

```typescript
import { useCallback, useState } from 'react';

export type PrevSelectedIndex = null | number;

/**
 * @title multipleSelect hooks
 * @description multipleSelect by hold down shift key
 */
export default function useMultipleSelect<T, K>(getKey: (item: T) => K) {
  const [prevSelectedIndex, setPrevSelectedIndex] = useState<PrevSelectedIndex>(null);

  const multipleSelect = useCallback(
    (currentSelectedIndex: number, data: T[], selectedKeys: Set<K>) => {
      const configPrevSelectedIndex = prevSelectedIndex ?? currentSelectedIndex;

      // add/delete the selected range
      const startIndex = Math.min(configPrevSelectedIndex || 0, currentSelectedIndex);
      const endIndex = Math.max(configPrevSelectedIndex || 0, currentSelectedIndex);
      const rangeKeys = data.slice(startIndex, endIndex + 1).map((item) => getKey(item));
      const shouldSelected = rangeKeys.some((rangeKey) => !selectedKeys.has(rangeKey));
      const changedKeys: K[] = [];

      rangeKeys.forEach((item) => {
        if (shouldSelected) {
          if (!selectedKeys.has(item)) {
            changedKeys.push(item);
          }
          selectedKeys.add(item);
        } else {
          selectedKeys.delete(item);
          changedKeys.push(item);
        }
      });

      setPrevSelectedIndex(shouldSelected ? endIndex : null);

      return changedKeys;
    },
    [prevSelectedIndex],
  );

  const updatePrevSelectedIndex = (val: PrevSelectedIndex) => {
    setPrevSelectedIndex(val);
  };

  return [multipleSelect, updatePrevSelectedIndex] as const;
}
```

### 2.1 参数与返回值

- **输入参数**：
  - `getKey: (item: T) => K`：从数据项中提取唯一键的函数

- **返回值**：`[multipleSelect, updatePrevSelectedIndex]`
  - `multipleSelect`：区间选择处理函数
  - `updatePrevSelectedIndex`：手动更新上次选择索引的函数

### 2.2 核心逻辑解析

1. **状态管理**：
   - 使用 `prevSelectedIndex` 记录上一次选择的索引位置
   - 初始值为 `null`，表示没有上一次选择

2. **区间计算**：
   - 如果 `prevSelectedIndex` 为 `null`，则使用当前索引作为起点
   - 计算区间的起点和终点：`startIndex` 和 `endIndex`
   - 从数据中提取区间内所有项的唯一键：`rangeKeys`

3. **操作类型判断**：
   - 通过 `shouldSelected` 判断是选中还是取消选中
   - 如果区间内有任何一项未被选中，则执行"选中"操作
   - 否则执行"取消选中"操作

4. **执行操作**：
   - 对区间内的每一项执行相应操作（添加或删除）
   - 记录变化的键到 `changedKeys` 数组
   - 更新 `prevSelectedIndex`：如果是"选中"操作，设为当前终点；否则重置为 `null`

5. **返回变化的键**：
   - 返回 `changedKeys` 数组，便于调用者知道哪些项发生了变化

## 3. Shift 键监听机制

在 Ant Design 中，Shift 键的监听并不是在 `useMultipleSelect` Hook 中直接实现的，而是在使用这个 Hook 的组件中通过原生事件对象获取。以 Transfer 组件为例：

### 3.1 事件传递链路

1. **ListItem 组件**：
   ```typescript
   // components/transfer/ListItem.tsx
   liProps.onClick = disabled || item.disabled ? undefined : (event) => onClick(item, event);
   ```
   - 点击列表项时，将原生事件对象 `event` 传递给 `onClick` 回调

2. **ListBody 组件**：
   ```typescript
   // components/transfer/ListBody.tsx
   const onInternalClick = (item: KeyWiseTransferItem, e: React.MouseEvent<Element, MouseEvent>) => {
     onItemSelect(item.key, !selectedKeys.includes(item.key), e);
   };
   ```
   - 接收事件对象并传递给 `onItemSelect`

3. **List 组件**：
   - 将 `onItemSelect` 传递给 `ListBody`

4. **Transfer 组件**：
   ```typescript
   // components/transfer/index.tsx
   const onLeftItemSelect = (selectedKey, checked, e) => {
     onItemSelect('left', selectedKey, checked, e?.shiftKey);
   };
   ```
   - 从事件对象中提取 `shiftKey` 属性，传递给通用处理函数

5. **处理函数**：
   ```typescript
   const onItemSelect = (direction, selectedKey, checked, multiple) => {
     // multiple 即 e.shiftKey
     if (multiple && holder.length > 0) {
       handleMultipleSelect(direction, data, holderSet, currentSelectedIndex);
     } else {
       handleSingleSelect(direction, holderSet, selectedKey, checked, currentSelectedIndex);
     }
   };
   ```
   - 根据 `multiple`（即 `e.shiftKey`）决定是单选还是区间选择

### 3.2 关键点

1. **原生事件属性**：React 事件系统保留了浏览器原生事件的 `shiftKey` 属性，无需额外监听
2. **事件冒泡链**：从底层 UI 组件一直向上传递事件对象
3. **条件判断**：仅当 `shiftKey` 为 true 且已有选中项时，才触发区间选择
4. **状态重置**：在某些操作（如移动项目）后，需要重置 `prevSelectedIndex`

## 4. 在 Transfer 组件中的应用

Transfer 组件是 `useMultipleSelect` 的典型使用场景：

```typescript
// 初始化 hooks
const [leftMultipleSelect, updateLeftPrevSelectedIndex] = useMultipleSelect<KeyWise<RecordType>, TransferKey>(
  (item) => item.key
);
const [rightMultipleSelect, updateRightPrevSelectedIndex] = useMultipleSelect<KeyWise<RecordType>, TransferKey>(
  (item) => item.key
);

// 处理多选
const handleMultipleSelect = (
  direction: TransferDirection,
  data: KeyWise<RecordType>[],
  holder: Set<TransferKey>,
  currentSelectedIndex: number,
) => {
  const isLeftDirection = direction === 'left';
  const multipleSelect = isLeftDirection ? leftMultipleSelect : rightMultipleSelect;
  multipleSelect(currentSelectedIndex, data, holder);
};

// 处理单选
const handleSingleSelect = (
  direction: TransferDirection,
  holder: Set<TransferKey>,
  selectedKey: TransferKey,
  checked: boolean,
  currentSelectedIndex: number,
) => {
  const isSelected = holder.has(selectedKey);
  if (isSelected) {
    holder.delete(selectedKey);
    setPrevSelectedIndex(direction, null);
  }
  if (checked) {
    holder.add(selectedKey);
    setPrevSelectedIndex(direction, currentSelectedIndex);
  }
};

// 重置索引（例如在移动项目后）
const moveToLeft = () => {
  moveTo('left');
  setPrevSelectedIndex('left', null);
};
```

### 4.1 实现要点

1. **左右列表分别初始化**：Transfer 为左右两个列表分别创建独立的 Hook 实例
2. **方向判断**：根据操作方向选择对应的 Hook 实例
3. **索引更新时机**：
   - 单选时：选中项时更新索引，取消选中时重置索引
   - 移动操作后：重置索引，避免跨操作的错误区间选择

## 5. 使用建议

### 5.1 适用场景

- 列表、表格等需要区间多选的场景
- 文件管理器类似的交互模式
- 需要提升用户批量操作效率的界面

### 5.2 使用注意事项

1. **数据一致性**：确保 `data` 参数是当前可见/可操作的数据集合，已排除禁用项
2. **键的唯一性**：确保 `getKey` 函数返回的键是唯一的
3. **状态重置**：在数据变化或移动操作后，适时重置 `prevSelectedIndex`
4. **事件传递**：确保从底层 UI 组件到处理函数的事件对象完整传递

### 5.3 简化使用示例

```tsx
import { useState } from 'react';
import useMultipleSelect from './useMultipleSelect';

interface Item {
  id: string;
  name: string;
  disabled?: boolean;
}

const MyList = ({ items }: { items: Item[] }) => {
  const [selectedKeys, setSelectedKeys] = useState<Set<string>>(new Set());
  const [multipleSelect, updatePrevSelectedIndex] = useMultipleSelect<Item, string>(item => item.id);

  const handleItemClick = (item: Item, e: React.MouseEvent) => {
    const currentIndex = items.findIndex(i => i.id === item.id);
    const newSelectedKeys = new Set(selectedKeys);

    if (e.shiftKey && selectedKeys.size > 0) {
      // 区间选择
      multipleSelect(currentIndex, items, newSelectedKeys);
    } else {
      // 单选
      if (newSelectedKeys.has(item.id)) {
        newSelectedKeys.delete(item.id);
        updatePrevSelectedIndex(null);
      } else {
        newSelectedKeys.add(item.id);
        updatePrevSelectedIndex(currentIndex);
      }
    }

    setSelectedKeys(newSelectedKeys);
  };

  return (
    <ul>
      {items.map((item, index) => (
        <li
          key={item.id}
          className={selectedKeys.has(item.id) ? 'selected' : ''}
          onClick={(e) => handleItemClick(item, e)}
        >
          {item.name}
        </li>
      ))}
    </ul>
  );
};
```

## 6. 总结

`useMultipleSelect` 是一个简洁而强大的 Hook，通过巧妙的设计实现了复杂的区间选择功能。它不仅提升了用户体验，也为开发者提供了易用的 API。通过与原生事件系统的结合，实现了无缝的 Shift 键多选交互，是 React 组件库中优秀的交互设计实践。
