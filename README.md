# MIUI 侧滑返回修复 (MIUI Back Gesture Fix)

修复 **HyperOS / MIUI 上「侧滑返回」手势失效** 的 LSPosed 模块。

> **现象**：在 Chrome、Mihon 等 App 里从屏幕边缘侧滑，页面**完全不动**，返回不到上一层。
> 但按**返回键**正常，在「系统设置」里侧滑也正常。

- **作用域**：`com.miui.home`（系统桌面）
- **适用**：Android 14 上的 HyperOS / MIUI
- **源码 / 更新日志**：https://github.com/a1175815821/MIUI-Back-Gesture-Fix

---

## 问题原因

桌面（`com.miui.home`）在把系统手势**转发**给当前 App 之前，要先构造一个 `android.window.BackMotionEvent`。反编译桌面 APK 可以看到：

```
android.window.BackMotionEventProvider.getInstance:(FFFFFILjava/lang/Object;)Ljava/lang/Object;
```

它**硬编码**使用了 **Android 15 才有的 8 参数构造函数**：

```
BackMotionEvent.<init>(float, float, float, float, float, boolean, int, RemoteAnimationTarget)
```

而 **Android 14 的框架只有 7 参数版本**（没有那个 `boolean isAnimationCallback`），反射必然失败：

```
ReflectUtils: NoSuchMethodException: android.window.BackMotionEvent.<init>
    at GestureStubView$OnBackInvokedCallbackImplU.onBackProgressed(GestureStubView.java:1530)
    at GestureStubView.onTouchEvent(GestureStubView.java:1030)

WindowOnBackDispatcher: onBackStarted error, backEvent is null
```

桌面拿到 `null` 之后仍然继续把它传给 App，App 端的 `WindowOnBackInvokedDispatcher` 收到空事件，于是**中止整个返回手势**。

## 修复方式

hook `android.window.BackMotionEventProvider.getInstance`，改用框架里**真实存在**的构造函数来构造事件：

- 构造函数通过**枚举 + 参数类型匹配**选取，不硬编码签名
- 因此 Android 15+ 上会自然让原逻辑通过，不产生副作用
- 另有一层兜底 hook：`com.miui.launcher.utils.ReflectUtils.createNewInstance`

## 影响范围

所有支持 Android **预测性返回（predictive back）** 的 App 都会中招，例如 Chrome、Mihon。
系统设置、Via 等使用传统返回的 App 不受影响 —— 判据只有一条：**App 有没有注册「预测性返回动画回调」**。

---

## 安装

1. 安装 APK（从本仓库 Releases 下载）
2. 打开 **LSPosed** → 模块 → 启用「**MIUI 侧滑返回修复**」
3. 确认作用域勾选了 **系统桌面（`com.miui.home`）**
4. **重启手机**

> ⚠️ 从 1.0 升级到 1.1 的用户注意：包名由 `com.wbai.miuibackfix` 改为
> `io.github.a1175815821.miuibackfix`，需要**先卸载旧版**再安装。

## 已验证环境

| 项 | 值 |
|---|---|
| 机型 | Redmi K50G（`22041216UC` / `xagapro`） |
| 系统 | HyperOS 2.0 `OS2.0.12.0.ULOCNXM` |
| Android | 14 |
| LSPosed | Zygisk 版 |

> Android 15+ 本身没有这个 bug。

## 日志

模块会在 LSPosed 日志（`/data/adb/lspd/log/`）里输出：

```
(com.miui.home)[io.github.a1175815821.miuibackfix] [MiuiBackFix] built BackMotionEvent via 7-arg ctor
(com.miui.home)[io.github.a1175815821.miuibackfix] [MiuiBackFix] getInstance -> fixed
```

---

## 许可

MIT
