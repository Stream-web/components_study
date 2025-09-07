# 如何基于react useEffect实现一个类似vue的watch功能

## 概述

rc-util 的 `useEffect` 是对 React 原生 `useEffect` 的增强封装，提供了类似 Vue `watch` 的功能，能够访问前一次的依赖值并自动处理依赖变化。

## 函数签名

```typescript
function useEffect(
  callback: (prevDeps: any[]) => void,
  deps: any[],
): void
```

## 实现原理

### 核心代码分析

```typescript
export default function useEffect(
  callback: (prevDeps: any[]) => void,
  deps: any[],
) {
  const prevRef = React.useRef(deps);
  React.useEffect(() => {
    if (
      deps.length !== prevRef.current.length ||
      deps.some((dep, index) => dep !== prevRef.current[index])
    ) {
      callback(prevRef.current);
    }
    prevRef.current = deps;
  });
}
```

### 实现步骤

1. **保存前一次依赖值**：
   ```typescript
   const prevRef = React.useRef(deps);
   ```
   使用 `useRef` 保存前一次的依赖数组。

2. **检测变化**：
   ```typescript
   if (
     deps.length !== prevRef.current.length ||
     deps.some((dep, index) => dep !== prevRef.current[index])
   ) {
     callback(prevRef.current);
   }
   ```
   检查依赖数组长度或内容是否发生变化。

3. **更新前值**：
   ```typescript
   prevRef.current = deps;
   ```
   将当前依赖值保存为下一次的前值。

### rc-util useEffect 示例

```typescript
// React with rc-util
import { useEffect } from 'rc-util'

const [count, setCount] = useState(0)
const [user, setUser] = useState({ name: 'Alice', age: 25 })

// 监听单个值
useEffect((prevCount) => {
  console.log('count 变化了：', prevCount, '->', count)
}, [count])

// 监听对象
useEffect((prevUser) => {
  console.log('用户信息变化了：', prevUser, '->', user)
}, [user])

// 监听多个值
useEffect((prevDeps) => {
  const [prevCount, prevUser] = prevDeps
  console.log('多个值变化了')
}, [count, user])
```

## 功能对比表

| 特性 | Vue watch | rc-util useEffect | React 原生 useEffect |
|------|-----------|-------------------|---------------------|
| **前值访问** | ✅ `(newVal, oldVal)` | ✅ `(prevDeps)` | ❌ 无法获取 |
| **自动比较** | ✅ 自动检测变化 | ✅ 自动检测变化 | ❌ 需要手动实现 |
| **多值监听** | ✅ 数组形式 | ✅ 数组形式 | ✅ 数组形式 |
| **深度监听** | ✅ `{ deep: true }` | ❌ 需要手动实现 | ❌ 需要手动实现 |
| **立即执行** | ✅ `{ immediate: true }` | ❌ 只在变化时执行 | ❌ 只在变化时执行 |
| **依赖长度变化** | ✅ 自动处理 | ✅ 自动处理 | ⚠️ 需要手动处理 |

## 使用场景对比

### 1. 路由变化监听

#### Vue watch
```javascript
// 监听路由变化
watch(() => route.path, (newPath, oldPath) => {
  console.log('路由从', oldPath, '变化到', newPath)
  // 执行路由变化后的逻辑
})
```

#### rc-util useEffect
```typescript
// 监听路由变化
useEffect((prevPath) => {
  console.log('路由从', prevPath, '变化到', pathname)
  // 执行路由变化后的逻辑
}, [pathname])
```

### 2. 表单数据监听

#### Vue watch
```javascript
// 监听表单数据
watch(formData, (newData, oldData) => {
  if (newData.email !== oldData.email) {
    validateEmail(newData.email)
  }
  if (newData.phone !== oldData.phone) {
    validatePhone(newData.phone)
  }
}, { deep: true })
```

#### rc-util useEffect
```typescript
// 监听表单数据
useEffect((prevFormData) => {
  if (prevFormData?.email !== formData.email) {
    validateEmail(formData.email)
  }
  if (prevFormData?.phone !== formData.phone) {
    validatePhone(formData.phone)
  }
}, [formData])
```

### 3. 用户信息变化监听

#### Vue watch
```javascript
// 监听用户信息
watch(user, (newUser, oldUser) => {
  if (oldUser && newUser.id !== oldUser.id) {
    // 用户切换了
    loadUserData(newUser.id)
  }
  if (newUser.permissions !== oldUser.permissions) {
    // 权限变化了
    updateUI(newUser.permissions)
  }
}, { deep: true })
```

#### rc-util useEffect
```typescript
// 监听用户信息
useEffect((prevUser) => {
  if (prevUser && user.id !== prevUser.id) {
    // 用户切换了
    loadUserData(user.id)
  }
  if (user.permissions !== prevUser?.permissions) {
    // 权限变化了
    updateUI(user.permissions)
  }
}, [user])
```

## 为什么需要这种设计？

### React 原生 useEffect 的局限性

```typescript
// React 原生 useEffect 的问题
const [count, setCount] = useState(0)

useEffect(() => {
  // 问题1：无法获取前一次的值
  // 问题2：需要手动比较变化
  // 问题3：依赖数组长度变化时需要特殊处理
  console.log('count 变化了，但不知道之前的值')
}, [count])
```

### rc-util useEffect 的解决方案

```typescript
// rc-util useEffect 的解决方案
const [count, setCount] = useState(0)

useEffect((prevCount) => {
  // 优势1：可以获取前一次的值
  // 优势2：自动比较变化
  // 优势3：自动处理依赖数组长度变化
  console.log('count 从', prevCount, '变化到', count)
}, [count])
```

## 优势总结

1. **提供前值访问**：可以比较当前值和前一次的值
2. **自动变化检测**：无需手动实现比较逻辑
3. **处理长度变化**：自动处理依赖数组长度变化
4. **简化代码**：减少样板代码，提高开发效率
5. **类似 Vue 体验**：让 React 开发者享受 Vue watch 的便利

## 注意事项

1. **深度比较**：对于对象和数组，需要手动实现深度比较
2. **性能考虑**：每次渲染都会执行比较逻辑，对于大量数据需要注意性能
3. **依赖管理**：仍然需要正确管理依赖数组

## 总结

rc-util 的 `useEffect` 是对 Vue `watch` 的 React 实现，它：

1. **弥补了 React 原生 useEffect 的不足**
2. **提供了类似 Vue watch 的 API 体验**
3. **让状态变化监听变得更加直观和易用**
4. **是工具库增强原生 API 的典型例子**

最后：ahooks里面也有类似的实现，后面再补充，如果面试被问到vue和react的区别，又多了个回答了。
