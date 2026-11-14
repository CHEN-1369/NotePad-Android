# NotePad 记事本应用 —— 移动软件开发期中实验报告

## 一、实验信息

| 项目 | 内容 |
| --- | --- |
| 实验名称 | NotePad 记事本应用功能扩展 |
| 实验类型 | 移动软件开发期中实验 |
| 开发平台 | Android |
| 开发语言 | Java |
| 完成日期 | 2026-11-14 |
| 作者 | CHEN-1369 |

## 二、实验目的

1. 阅读并理解 Android 官方 NotePad 示例的源码结构。
2. 掌握 `ContentProvider`、`SQLiteOpenHelper` 与 `ContentResolver` 的配合使用。
3. 在原有示例基础上完成笔记时间戳显示、笔记查询等基本功能。
4. 通过 UI 美化与笔记分类两个附加功能，提升应用的可用性与视觉效果。
5. 使用当前 Android 构建规范（AndroidX 时代的 SDK 配置、组件导出声明等）保证项目可编译、可运行。

## 三、实验环境

| 项目 | 版本 / 说明 |
| --- | --- |
| 操作系统 | Windows 10 |
| JDK | OpenJDK 21 |
| 构建工具 | Gradle 8.13 + Android Gradle Plugin 8.12.2 |
| compileSdk / targetSdk | 37 / 37 |
| minSdk | 24 |
| Java 编译版本 | Java 17 |
| 测试设备 | Android Emulator（API 37） |

## 四、实验要求与完成情况

| 类型 | 要求 | 完成情况 |
| --- | --- | --- |
| 基本要求 | NoteList 界面中笔记条目增加时间戳显示 | 已完成 |
| 基本要求 | 添加笔记查询功能（根据标题或内容查询） | 已完成 |
| 附加功能 | UI 美化 | 已完成 |
| 附加功能 | 笔记分类 | 已完成 |
| 额外完善 | 笔记变更实时刷新 | 已完成 |

## 五、功能实现

### 5.1 笔记列表与时间戳

笔记列表使用 `SimpleCursorAdapter` 绑定 `ContentProvider` 返回的游标。列表项展示以下信息：

- 笔记标题
- 笔记内容摘要（最多两行）
- 分类徽标
- 最后修改时间（`yyyy-MM-dd HH:mm`）

时间戳取自数据库中的 `modified` 字段。每次保存笔记时都会更新该字段，因此列表始终显示最近一次修改时间。

### 5.2 笔记查询

在列表页操作栏中集成 `SearchView`，支持按**标题或内容**实时搜索。查询条件由 `ContentProvider` 转换为 SQL `LIKE` 条件：

```sql
(title LIKE ? OR note LIKE ?)
```

搜索文本变化时立即重新查询，无需点击确认按钮；搜索条件可与分类筛选条件组合使用。

### 5.3 笔记分类

应用内置四个分类：**工作、学习、生活、其他**。

- 编辑器顶部提供分类下拉框，新建或编辑笔记时可选择分类。
- 列表页顶部的分类筛选器支持按分类过滤笔记。
- 列表项右侧显示彩色分类徽标，不同分类使用不同颜色，便于快速区分。
- 分类以稳定的英文键（`work` / `study` / `life` / `other`）存入数据库，展示时映射为中文名称。

### 5.4 笔记变更实时刷新

列表页在 `onStart()` 中向 `ContentResolver` 注册 `ContentObserver`，监听笔记数据变化：

- 新建、修改或删除笔记后，`ContentProvider` 调用 `notifyChange()`。
- 观察者收到通知后立即重新查询并刷新列表，无需手动下拉或重新进入页面。
- `onStop()` 中注销观察者，避免后台无效监听。

### 5.5 UI 美化

- 使用 Material 风格主题，统一主色、状态栏与导航栏颜色。
- 列表项采用圆角卡片、标题/摘要/时间/分类徽标的分层信息设计。
- 空列表时展示友好的空状态提示。
- 编辑器采用卡片式内容面板，保留原有横线笔记纸效果，并加入分类选择区。
- 操作栏图标替换为矢量图标（新建、保存、删除、撤销），适配不同屏幕密度。

## 六、系统设计与关键技术

### 6.1 总体架构

```
Activity（NotesList / NoteEditor / TitleEditor）
        │  ContentResolver
        ▼
NotePadProvider（ContentProvider）
        │  SQLiteOpenHelper
        ▼
SQLite 数据库 note_pad.db
```

- **NotesList**：展示笔记列表，负责搜索、分类筛选和内容观察。
- **NoteEditor**：新建、查看、编辑、删除笔记，选择笔记分类。
- **TitleEditor**：以对话框形式编辑笔记标题。
- **NotePadProvider**：统一的数据访问入口，负责增删改查与变更通知。
- **NotePad**：数据契约类，集中定义表名、列名、URI、MIME 类型与分类常量。

### 6.2 数据库设计

数据库名：`note_pad.db`，当前版本：`3`。

表 `notes` 的字段如下：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `_id` | INTEGER | 主键，自增 |
| `title` | TEXT | 笔记标题 |
| `note` | TEXT | 笔记内容 |
| `created` | INTEGER | 创建时间（毫秒时间戳） |
| `modified` | INTEGER | 最后修改时间（毫秒时间戳） |
| `category` | TEXT | 分类键，默认 `other` |

数据库从版本 2 升级到版本 3 时使用 `ALTER TABLE` 增加 `category` 字段，升级过程不会丢失已有笔记。

### 6.3 ContentProvider URI

| URI | 说明 |
| --- | --- |
| `content://com.google.provider.NotePad/notes` | 笔记集合 |
| `content://com.google.provider.NotePad/notes/#` | 单条笔记 |
| `content://com.google.provider.NotePad/live_folders/notes` | Live Folder 列表 |

### 6.4 关键类说明

| 文件 | 作用 |
| --- | --- |
| `NotesList.java` | 笔记列表、搜索、分类筛选、时间戳、内容观察与实时刷新 |
| `NoteEditor.java` | 笔记编辑、分类选择、粘贴、保存与删除 |
| `NotePadProvider.java` | 数据增删改查、数据库创建与升级、变更通知 |
| `NotePad.java` | 数据契约（表名、列名、URI、MIME、分类常量） |
| `TitleEditor.java` | 笔记标题编辑对话框 |
| `NotesLiveFolder.java` | Live Folder 支持（原有示例功能） |

## 七、项目结构

```
NotePad-main/
├── app/
│   ├── build.gradle
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/example/android/notepad/
│       │   ├── NotePad.java
│       │   ├── NotePadProvider.java
│       │   ├── NotesList.java
│       │   ├── NoteEditor.java
│       │   ├── TitleEditor.java
│       │   └── NotesLiveFolder.java
│       └── res/
│           ├── drawable/          # 卡片、分类徽标、矢量图标等
│           ├── layout/            # 列表、列表项、编辑器布局
│           ├── menu/              # 列表菜单、编辑器菜单
│           └── values/            # 字符串、颜色、主题、分类数组
├── build.gradle
├── gradle.properties
├── gradlew / gradlew.bat
└── settings.gradle
```

## 八、运行与验证

### 8.1 编译与安装

```bash
# Windows
gradlew.bat assembleDebug

# macOS / Linux
./gradlew assembleDebug
```

生成的安装包位于：

```
app/build/outputs/apk/debug/app-debug.apk
```

### 8.2 功能验证

| 验证项 | 操作步骤 | 结果 |
| --- | --- | --- |
| 新建笔记 | 点击右上角“+”，输入内容并保存 | 列表立即出现新笔记 |
| 时间戳 | 查看列表项底部 | 正确显示修改时间 |
| 保存分类 | 新建时选择“学习”并保存 | 列表显示“学习”彩色徽标 |
| 分类筛选 | 顶部筛选器选择“学习” | 仅显示学习分类笔记 |
| 空状态 | 筛选一个没有笔记的分类 | 显示空状态提示 |
| 标题/内容搜索 | 搜索框输入笔记内容关键字 | 实时返回匹配笔记 |
| 实时刷新 | 保存笔记后返回列表 | 无需手动刷新即可看到最新数据 |

### 8.3 构建检查

- `./gradlew clean build`：构建成功
- Android Lint：**0 errors**
- 剩余警告为示例应用旧图标缺少多密度目录，不影响功能运行

## 九、实验总结

本次实验在官方 NotePad 示例的基础上完成了列表时间戳、标题/内容搜索、UI 美化与笔记分类四项功能，并额外实现了笔记变更的实时刷新。

通过实验加深了对 Android 数据持久化与组件通信的理解：`ContentProvider` 负责统一的数据访问，`ContentResolver` 负责跨组件查询，`ContentObserver` 负责数据变化的实时感知；三者配合可以在保证数据一致性的同时获得良好的交互体验。在 UI 层面，通过主题、卡片布局、分类徽标与矢量图标提升了应用的整体观感。

同时，项目按当前 Android 构建规范进行了升级：使用 `compileSdk 37`、`targetSdk 37`、`minSdk 24` 与 Java 17，并在清单中补齐组件导出声明，移除了已失效的旧测试代码与不再使用的图片资源，使工程结构更加清晰、可维护。
