---
name: mixin-optional-dependency-linking
description: 集成可选 Minecraft Mod 时，用 Stub + Conditional Mixin + MixinPlugin 把可选依赖隔离在类加载阶段。当需要安全调用一个「可能不存在」的 Mod API 时使用。
---

# Optional Mod Integration via Mixin

将可选 Mod 的依赖隔离在 **Conditional Mixin Implementation** 中，把运行时分支前移到类转换阶段。

## 核心模式

```text
主代码
  ↓
Stub（永远存在，空实现）
  ↓
Mixin Plugin 判断 Mod
  ├─ 不存在 → Stub 保持不变
  └─ 存在   → Mixin 覆盖 Stub
                ↓
             真实实现
```

效果：

* 无 Mod：调用空实现。
* 有 Mod：调用 Mixin 替换后的真实实现。
* 运行热路径无需 `isModLoaded`、反射、delegate/interface 判断。
* 本质是 **加载期条件链接 / 条件字节码织入**。

## 规则

1. **Stub 不得引用 Optional Mod 类型**。
2. 所有对 Optional Mod 的直接引用都放进 Mixin Implementation。
3. Optional 对象需要跨边界返回时，用 `Object`；`Object` 不会复制/包装对象，只隐藏静态类型。
4. 主代码不要出现 `instanceof OptionalType`、`(OptionalType)` 等引用；相关逻辑放进 Optional Implementation。
5. 用 `IMixinConfigPlugin#shouldApplyMixin` 按 `FabricLoader.isModLoaded()` 决定是否应用实现 Mixin。
6. 对自己掌控的 Stub，可用 `@Overwrite(remap = false)` 直接替换方法体，获得最简单的热路径。
7. 不要依赖 JVM 延迟解析来保证安全；**Optional 类型必须隔离在不会被加载/应用的类中**。

## 最小结构

```text
Stub.java
    ↓
MixinImpl.java  ← 引用 Optional API
    ↓
MixinPlugin.java
    ↓
shouldApplyMixin()
```

## 最小示例

```java
public class Compat {
    public static void run() {}
    public static Object get() { return null; }
}
```

```java
@Mixin(Compat.class)
public abstract class CompatImpl {
    @Overwrite(remap = false)
    public static void run() {
        OptionalApi.run();
    }

    @Overwrite(remap = false)
    public static Object get() {
        return OptionalApi.getObject();
    }
}
```

```java
public class CompatMixinPlugin implements IMixinConfigPlugin {
    @Override
    public boolean shouldApplyMixin(String target, String mixin) {
        return !mixin.endsWith("CompatImpl")
            || FabricLoader.getInstance().isModLoaded("optionalmod");
    }
}
```

**原则：Stub 提供稳定 ABI，Mixin Implementation 提供可选实现，Mixin Plugin 在类加载阶段决定最终实现。**

