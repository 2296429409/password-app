# VaultSync

跨平台密码管理器：密钥文件本地保管，密文通过 Git（GitHub / Gitee）同步。无后端、纯静态，支持 **Web（PC 浏览器）** 与 **Android**。

## 特性

- **密钥文件解锁**：创建密码库时生成 `.vaultkey` 密钥文件，无需主密码
- **多密码库**：每个库独立密钥 + Git 上独立 `.enc` 文件，可分别管理、删除
- **Git 同步**：通过 REST API 读写远程密文，支持 GitHub、Gitee
- **本地密钥缓存**：常用库自动记住，下次打开无需重复导入
- **密码工具**：随机密码生成、强度提示、一键复制用户名/密码
- **保存即同步**：新增或编辑条目后自动推送到 Git

## 架构概览

```
┌─────────────────┐     仅本地，不上传 Git
│  .vaultkey      │  ←  vault_id、库名、master_key、git_vault_path
│  密钥文件        │
└────────┬────────┘
         │ AES-256-CBC 加解密
         ▼
┌─────────────────┐     提交到 Git 仓库
│  名称_id.enc    │  ←  encrypted_data、iv、revision（不含密钥）
│  密文文件        │
└─────────────────┘
```

| 组件 | 说明 |
|------|------|
| **密钥文件** `.vaultkey` | 含 `vault_id`、`vault_name`、`master_key`（Base64）、`git_vault_path` |
| **密文文件** `工作_abc12345.enc` | 文件名由库名 + ID 前缀生成，仅存加密数据 |
| **加密算法** | AES-256-CBC，主密钥 32 字节保存在密钥文件中 |
| **Git 配置** | 全局一份（owner / repo / branch / token），各库使用自己的 `git_vault_path` |

## 技术栈

- [uni-app x](https://uniapp.dcloud.net.cn/)（Vue 3 + UTS）
- 纯前端实现：自研 AES、Git REST API 调用
- 本地存储：`uni.setStorageSync`（Git Token、密钥缓存、多库索引）

## 项目结构

```
密码/
├── pages/
│   ├── unlock/unlock.uvue    # 密码库列表、导入密钥、自动打开
│   ├── create/create.uvue    # 创建库、下载密钥文件
│   ├── index/index.uvue      # 密码列表、同步、切换库
│   ├── edit/edit.uvue        # 新增/编辑条目
│   └── settings/settings.uvue # Git 全局配置
├── store/
│   └── vaultStore.uts        # 解锁、同步、CRUD 状态
├── utils/
│   ├── keyfile.uts           # 密钥文件生成/解析/下载
│   ├── crypto.uts            # AES-256-CBC、Base64
│   ├── vault.uts             # 密码库加解密与条目操作
│   ├── git.uts               # GitHub/Gitee pull/push
│   ├── storage.uts           # 多库索引、密钥缓存
│   ├── fileio.uts            # H5/App 文件读取
│   └── password.uts          # 密码生成与强度
├── types/vault.uts           # 类型定义
├── manifest.json
└── pages.json
```

## 使用流程

### 1. 首次使用

1. 打开应用 → **Git 同步设置**，填写仓库所有者、仓库名、分支、Access Token
2. 点击 **测试连接** 确认配置正确
3. **创建新密码库** → 输入库名 → 自动下载 `.vaultkey` 并缓存到本地
4. 若已配置 Git，创建后会自动推送首个 `.enc` 文件

### 2. 日常使用

- 打开已缓存的密码库，或 **导入密钥文件**（`.vaultkey`）
- 在列表中搜索、复制、新增、编辑密码
- 点击 **同步** 手动拉取/合并远程数据
- **切换库** 会清除当前库的本地密钥缓存，返回选择页

### 3. 换设备 / 多设备

1. 新设备配置相同的 Git 仓库与 Token
2. 导入原设备的 `.vaultkey` 密钥文件（**不是** `.enc` 文件）
3. 应用从 Git 拉取对应 `.enc` 并解密

### 4. 删除某个密码库

- **Git**：删除仓库中对应的 `库名_id.enc` 文件
- **本地**：在解锁页长按库卡片可移除密钥缓存
- 妥善销毁本地的 `.vaultkey` 文件

## 密钥文件示例

```json
{
  "version": 1,
  "vault_id": "a1b2c3d4-...",
  "vault_name": "工作",
  "master_key": "Base64编码的32字节密钥",
  "git_vault_path": "工作_a1b2c3d4.enc",
  "created_at": "2026-06-30T12:00:00.000Z"
}
```

## 安全说明

- **密钥文件（`.vaultkey`）绝不提交到 Git**，请自行备份（U 盘、加密盘等）
- Git Token 仅保存在浏览器/设备本地存储中
- 丢失密钥文件 = 无法解密对应密码库，即使 Git 上仍有 `.enc`
- 建议在私有仓库中使用，Token 权限按需最小化（仅仓库读写）

## 开发与运行

### 环境要求

- [HBuilderX](https://www.dcloud.io/hbuilderx.html)（推荐）或支持 uni-app x 的 CLI
- Node.js（若使用 CLI 构建）

### 运行

1. 用 HBuilderX 打开本项目目录
2. **运行 → 运行到浏览器**（H5 / Web）
3. 或 **运行 → 运行到手机或模拟器 → Android**

H5 使用 hash 路由，入口为 `index.html`。

### 构建发布

- **H5**：发行 → 网站-H5，产物在 `unpackage/dist/build/h5`
- **Android**：发行 → 原生 App-Android，需配置签名与图标

## Git Token 获取

| 平台 | 说明 |
|------|------|
| **Gitee** | 设置 → 私人令牌 → 勾选 `projects` 权限 |
| **GitHub** | Settings → Developer settings → Personal access tokens → `repo` 权限 |

仓库可为空；首次推送会自动创建对应 `.enc` 文件。

## 常见问题

**导入提示「密钥文件无效」**  
请确认选择的是 `.vaultkey` 文件，而非 `.enc` 密文。H5 端使用浏览器原生文件选择读取 JSON 文本。

**切换库后又自动进入原库**  
切换库时会清除该库的本地密钥缓存；若仍异常，可在解锁页长按移除缓存后重新导入。

**同步 409 冲突**  
应用会在推送前获取最新 `sha` 并重试；若仍失败，可先手动同步再编辑。

**旧版 `vault.enc` 数据**  
早期主密码体系与当前密钥文件体系不兼容，需在 Git 删除旧文件后重新创建密码库。

## 许可证

本项目为个人/私有使用场景设计，按需自行部署与保管密钥。
