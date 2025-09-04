# react性能优化之 rc-util(useEvent)

## 概述

`useEvent` 是一个 rc-util 的 一个自定义React Hook，用于创建一个稳定的函数引用，避免因为依赖变化而导致的重新渲染问题。

## 函数签名

```typescript
function useEvent<T extends Function>(callback: T): T
```

## 实现原理

### 核心代码分析

```typescript
function useEvent<T extends Function>(callback: T): T {
  const fnRef = React.useRef<any>();
  fnRef.current = callback;

  const memoFn = React.useCallback<T>(
    ((...args: any) => fnRef.current?.(...args)) as any,
    [],
  );

  return memoFn;
}
```

### 实现步骤

1. **使用 `useRef` 存储最新函数**：
   ```typescript
   const fnRef = React.useRef<any>();
   fnRef.current = callback;
   ```
   每次渲染时，都将最新的 `callback` 存储到 `fnRef.current` 中。

2. **创建稳定的函数引用**：
   ```typescript
   const memoFn = React.useCallback<T>(
     ((...args: any) => fnRef.current?.(...args)) as any,
     [],
   );
   ```
   使用 `useCallback` 创建一个函数，这个函数内部调用 `fnRef.current`，但依赖数组为空 `[]`，所以这个函数引用在组件生命周期内保持稳定。

3. **返回稳定的函数**：
   返回的 `memoFn` 引用永远不会改变，但内部总是调用最新的 `callback`。

## 问题场景

### 不使用 useEvent 的问题

```typescript
function MyComponent() {
  const [count, setCount] = useState(0);
  
  // 每次渲染都会创建新的函数引用
  const handleClick = () => {
    console.log('Current count:', count);
  };
  
  useEffect(() => {
    console.log('useEffect 执行了！'); // 每次 count 变化都会打印
    document.addEventListener('click', handleClick);
    return () => document.removeEventListener('click', handleClick);
  }, [handleClick]); // handleClick 每次都是新的引用
  
  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

**问题分析：**
1. 每次 `count` 状态更新时，组件重新渲染
2. 重新渲染时，`handleClick` 被重新创建，是一个新的函数引用
3. `useEffect` 检测到 `handleClick` 依赖项变化，重新执行
4. 重新执行意味着：
   - 移除旧的事件监听器
   - 添加新的事件监听器
   - 执行清理函数

### 使用 useEvent 的解决方案

```typescript
function MyComponent() {
  const [count, setCount] = useState(0);
  
  // 函数引用稳定，不会变化
  const handleClick = useEvent(() => {
    console.log('Current count:', count);
  });
  
  useEffect(() => {
    console.log('useEffect 执行了！'); // 只在组件挂载时打印一次
    document.addEventListener('click', handleClick);
    return () => document.removeEventListener('click', handleClick);
  }, [handleClick]); // handleClick 引用稳定，不会变化
  
  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

## 对比分析

| 方面 | 不使用 useEvent | 使用 useEvent |
|------|----------------|---------------|
| **函数引用** | 每次渲染都变化 | 引用稳定不变 |
| **useEffect 执行** | 每次状态变化都执行 | 只在挂载时执行一次 |
| **事件监听器** | 频繁添加/移除 | 只添加一次 |
| **性能** | 较差，有重复操作 | 优化，避免不必要操作 |
| **内存风险** | 可能有泄漏风险 | 更安全 |

## 使用场景

### 1. 事件处理函数

```typescript
const handleResize = useEvent(() => {
  // 处理窗口大小变化
  setWindowSize({ width: window.innerWidth, height: window.innerHeight });
});

useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, [handleResize]);
```

### 2. 回调函数

```typescript
const handleSubmit = useEvent((data) => {
  // 提交数据，总是使用最新的状态
  submitData(data, currentUser);
});

// 在子组件中使用，不会因为父组件重新渲染而重新创建
<Form onSubmit={handleSubmit} />
```

### 3. 定时器回调

```typescript
const handleTick = useEvent(() => {
  // 定时器回调，总是使用最新的状态
  setTime(new Date());
});

useEffect(() => {
  const timer = setInterval(handleTick, 1000);
  return () => clearInterval(timer);
}, [handleTick]);
```

## 优势总结

1. **避免不必要的重新渲染**：函数引用稳定，不会触发依赖它的 Hook 重新执行
2. **总是使用最新值**：虽然引用稳定，但内部总是调用最新的函数
3. **简化依赖管理**：不需要在依赖数组中包含函数本身
4. **性能优化**：减少不必要的 DOM 操作和副作用执行
5. **代码简洁**：避免复杂的依赖管理逻辑

## 总结

`useEvent` 是一个非常有用的 Hook，它解决了 React 中函数依赖导致的性能问题。通过创建一个稳定的函数引用，同时保证内部总是调用最新的函数，它既保证了性能优化，又保持了功能的正确性。在需要将函数作为依赖传递给其他 Hook 的场景中，`useEvent` 是一个很好的解决方案。
