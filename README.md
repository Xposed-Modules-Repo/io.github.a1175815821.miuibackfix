# MIUI 侧滑返回修复

修复 **HyperOS / MIUI 上侧滑返回失效** 的 LSPosed 模块。

> 侧滑返回没反应，但返回键是好的？在 Chrome、Mihon 这类 App 里尤其明显？
> 就是它。装上、勾上作用域、重启，好了。

- **适用**：Android 14 的 HyperOS / MIUI（Android 15+ 没有这个 bug）
- **作用域**：`com.miui.home`（系统桌面）
- **源码 / 更新日志 / 详细原理**：https://github.com/a1175815821/MIUI-Back-Gesture-Fix

---

## 30 秒确认你要不要装

- [ ] HyperOS / MIUI，**Android 14**
- [ ] 侧滑返回**没反应**，按**返回键正常**
- [ ] 出问题的是 **Chrome / Mihon** 这类新 App，而「系统设置」里侧滑是好的

三条都中 → 就是这个问题。

---

## 安装

1. 在 LSPosed 里启用本模块
2. 作用域勾选 **系统桌面（`com.miui.home`）**
3. **重启手机**

> 勾错作用域（比如勾了浏览器）不会生效。

## 装完没效果？

看 LSPosed 日志（`/data/adb/lspd/log/`），正常应该出现：

```
(com.miui.home)[io.github.a1175815821.miuibackfix] [MiuiBackFix] built BackMotionEvent via 7-arg ctor
(com.miui.home)[io.github.a1175815821.miuibackfix] [MiuiBackFix] getInstance -> fixed
```

- 没有这两行 → 模块没加载，检查作用域 + 是否重启
- 有这两行但仍不行 → 别的原因，欢迎到源码仓库开 Issue（附机型和 ROM 版本）

---

## 为什么会坏

桌面（`com.miui.home`）在把侧滑手势转发给 App 之前，要先构造一个 `android.window.BackMotionEvent`，但它**硬编码用了 Android 15 才有的 8 参数构造函数**：

```
BackMotionEvent.<init>(float, float, float, float, float, boolean, int, RemoteAnimationTarget)
```

Android 14 的框架只有 7 参数版本 → 反射必然失败 → 拿到 `null` → 仍然传给 App：

```
ReflectUtils: NoSuchMethodException: android.window.BackMotionEvent.<init>
WindowOnBackDispatcher: onBackStarted error, backEvent is null
```

App 收到空事件，就**中止了整个返回手势**。

## 修法

hook `android.window.BackMotionEventProvider.getInstance`，改用框架里**真实存在**的构造函数重建事件（枚举 + 参数类型匹配，不硬编码签名）。

Android 15+ 上原逻辑本来就不会失败，hook 直接放行，无副作用。

---

## 影响范围

只有注册了「预测性返回动画回调」的 App 会中招：

| App | 侧滑返回 |
|---|:---:|
| Chrome | ❌ |
| Mihon | ❌ |
| 系统设置 | ✅ |
| Via | ✅ |

**所有支持 Android 预测性返回的 App 都受影响**，不只浏览器。

---

## 已验证环境

| 机型 | 系统 | Android |
|---|---|---|
| Redmi K50G（`22041216UC`） | HyperOS 2.0 `OS2.0.12.0.ULOCNXM` | 14 |

其他版本验证结果欢迎反馈。

---

## 许可

MIT · 源码：https://github.com/a1175815821/MIUI-Back-Gesture-Fix
