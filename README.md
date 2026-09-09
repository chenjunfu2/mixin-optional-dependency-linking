# mixin-optional-dependency-linking

一个 Claude Code skill：在 Minecraft（Fabric）mod 开发中，安全集成一个「可选」的其他 mod 的 API。

核心思路：把对可选 Mod 的依赖全部隔离进 **Conditional Mixin Implementation**，用 Mixin 在**类加载阶段**的条件织入，替代运行时的 `isModLoaded` / 反射 / `instanceof` 判断。

## 适用场景

你的 mod 想增强另一个 mod（例如 `optionalmod`）的功能，但那个 mod 可能不在玩家的 `mods` 文件夹里。直接引用它的 API 会在类加载时抛 `NoClassDefFoundError`。

## 核心模式

```text
主代码
  ↓
Stub（永远存在，空实现）          ← 主代码只依赖这个
  ↓
Mixin Plugin 判断 Mod 是否加载
  ├─ 不存在 → Stub 保持空实现（不织入）
  └─ 存在   → Mixin 覆盖 Stub 的方法体
                ↓
             真实实现（引用 Optional API）
```

- **无 Mod**：调用空实现。
- **有 Mod**：调用 Mixin 替换后的真实实现。
- 热路径零判断、零反射、零崩溃风险。

## 安装

### 方式一：clone 到用户级 skills 目录

```bash
git clone https://github.com/<你的用户名>/mixin-optional-dependency-linking.git \
  ~/.claude/skills/mixin-optional-dependency-linking
```

### 方式二：手动复制

把整个文件夹复制到 Claude Code 的用户级 skills 目录：

- Linux / macOS：`~/.claude/skills/mixin-optional-dependency-linking`
- Windows：`C:\Users\<你>\.claude\skills\mixin-optional-dependency-linking`

装好后重启 Claude Code，即可在需要集成可选 Mod 时被自动调用。

## 文件结构

```text
mixin-optional-dependency-linking/
├── SKILL.md      # skill 本体（含 frontmatter）
├── README.md     # 本文件
└── LICENSE       # 开源协议
```

## 最小结构（skill 内容摘要）

```text
Stub.java          → 稳定 ABI，空实现，不引用可选类型
    ↓
MixinImpl.java     → 唯一允许引用 Optional API 的地方
    ↓
MixinPlugin.java   → shouldApplyMixin() 在类加载阶段决定是否织入
```

## 适用注意

针对 **Fabric** 的 Mixin。Forge / NeoForge 的 `IMixinConfigPlugin` 签名略有差异，迁移时需对应调整。

## License

见 [LICENSE](./LICENSE)。
