# Dime

基于 SwiftUI 的 iOS 个人记账应用，支持收支记录、分类预算、统计图表、语音记账和票据识别。

本仓库是 [Rafael Soh 的 Dime](https://github.com/rafsoh/dimeApp) 的个人维护分支，增加了 Gemini 辅助录入、快速记账和本地存储配置。[App Store 上的 Dime](https://apps.apple.com/sg/app/dime-budget-expense-tracker/id1635280255) 是上游发行版，不代表本分支的功能或构建产物。

<p align="center">
  <img src="docs/assets/hero.png" width="451" alt="Dime 应用界面预览" />
</p>

## 功能

| 功能 | 使用方式 |
| --- | --- |
| 日常记账 | 记录收入、支出、分类、备注和日期，支持周期交易、搜索与 CSV 导入导出。 |
| 预算与统计 | 按分类或整体设置预算，查看不同时间范围的收支、趋势和环形图。 |
| 语音记账 | 长按底部“＋”开始说话，点按停止；识别后进入交易确认页面。 |
| 票据与账单识别 | 在新增交易页面点扫描按钮，拍照或选择图片，提取金额、商户、日期及建议分类。 |
| 批量录入 | 从账单截图或一段语音中提取多笔交易，勾选后逐笔导入，或合并为一笔。 |
| Siri 与快捷指令 | 支持新建交易、查询统计/预算，以及先记金额再补分类的 Quick Log。Quick Log 会发送补分类通知。 |
| 小组件与个性化 | 主屏幕/锁屏小组件、提醒、生物识别应用锁、深色模式和自定义图标。 |

### 语音输入调用了什么

语音记账分为转写和交易解析两步：

```text
长按“＋”
  -> AVAudioEngine 采集音频
  -> Apple Speech / SFSpeechRecognizer 转成文字
  -> GeminiService.parseVoiceInput 调用 Gemini 2.5 Flash
  -> 返回金额、备注、日期、分类、收支类型
  -> 单笔填入交易表单，多笔进入批量导入页面
  -> 用户确认后保存到 Core Data
```

例如可以说“昨天午饭 35，今天打车 18”。录音使用系统当前 Locale，最长 30 秒；解析请求会附上当前日期和已有分类名称。能否识别对应语言取决于系统语音识别的可用性，结果需要核对。

对应代码：转写在 [SpeechRecognizer.swift](app/dime/Utilities/SpeechRecognizer.swift)，语音触发在 [CustomTabBar.swift](app/dime/Views/CustomTabBar.swift)，Gemini 请求在 [GeminiService.swift](app/dime/Utilities/GeminiService.swift)，结果填充和批量确认在 [TransactionView.swift](app/dime/Views/TransactionView.swift)。

### 数据与网络

交易保存在本地 Core Data。当前分支使用 `NSPersistentContainer`，没有启用 CloudKit 交易同步；源码仍保留部分上游 iCloud 设置和依赖。

手动记账不依赖 Gemini。语音解析会将转写文字、当前日期及分类名称发给 Google Gemini；票据识别会将压缩后的图片发给 Gemini。它们需要联网和自备 API key。语音转写由 Apple Speech 处理，代码没有强制使用设备端识别，不能保证全程离线。

## 本地运行

### 环境

- macOS、完整安装的 Xcode 及对应 iOS 平台/模拟器组件。
- 工程的 iOS deployment target 为 15.0；App Intents 快捷入口要求 iOS 16.4 或更新版本。
- 首次解析 Swift Package Manager 依赖需要网络；语音、相机和签名行为需要真机验证。

### 获取源码与配置密钥

```sh
git clone https://github.com/Vanxun-Hank/dimeApp.git
cd dimeApp
cp app/dime/Utilities/Secrets.example.swift app/dime/Utilities/Secrets.swift
open app/dime.xcodeproj
```

`Secrets.swift` 已在工程中配置为编译源文件，但被 Git 忽略。模板内容如下：

```swift
enum Secrets {
    static let geminiAPIKey = ""
}
```

需要语音解析或票据识别时，在本地 `Secrets.swift` 中填写自己的 Gemini API key；留空不影响编译配置和手动记账，AI 请求会因缺少密钥而失败。不要提交真实密钥。此配置仍会把密钥编译进客户端，适合个人构建；公开分发前应改为服务端管理密钥。

在 Xcode 中选择 `dime` scheme 和模拟器，让 Swift Package Manager 解析依赖。依赖解析异常时使用 `File > Packages > Resolve Package Versions`，并保留仓库锁定版本，避免无关升级。

### 真机签名

项目包含 `dime`、`ExpenditureWidgetExtension`、`BudgetIntent` 和 `BudgetIntentUI` 四个 target。真机运行时为各 target 选择自己的 Development Team，并按需调整 Bundle Identifier。

这些 target 的 entitlements、Core Data 路径和 `UserDefaults` 共享 `group.com.vanxun.dime`。如果改 App Group，必须同步修改全部引用；只改签名设置可能导致数据不可见或小组件无法读取记录。若当前签名账户无法配置 App Groups，需要另外处理共享存储和扩展能力，不能保证仅换 Team 就能运行。当前工程已移除 iCloud entitlement。

### 命令行检查

从仓库根目录执行；先完成上面的 `Secrets.swift` 配置：

```sh
xcodebuild -list -project app/dime.xcodeproj

xcodebuild \
  -project app/dime.xcodeproj \
  -scheme dime \
  -configuration Debug \
  -destination 'generic/platform=iOS Simulator' \
  -derivedDataPath /tmp/dimeApp-build \
  -disableAutomaticPackageResolution \
  CODE_SIGNING_ALLOWED=NO \
  build

git diff --check
```

仓库尚未配置自动化测试 target 或 CI 工作流。模拟器构建不等于通过真机权限、网络服务、快捷指令和小组件验收。改动相关功能后，至少检查手动记账保存、语音权限拒绝与单笔/多笔解析、票据导入确认，以及 Quick Log 通知补分类流程。

若构建报告 `iOS 18.2 is not installed` 或找不到 iOS Simulator destination，请先在 Xcode 设置的平台/组件下载页面安装当前 Xcode 要求的 iOS 组件。

## 代码导航

| 路径 | 用途 |
| --- | --- |
| `app/dime/Views/` | 交易、预算、统计和设置页面。 |
| `app/dime/Components/` | 数字键盘、分类选择器和共用 UI。 |
| `app/dime/Data/` | `DataController`、Core Data 模型和数据辅助方法。实际编译模型为此目录下的 `MainModel.xcdatamodeld`。 |
| `app/dime/Utilities/` | 语音识别、Gemini 请求、图片选择、应用锁等。 |
| `app/dime/Shortcuts/` | App Intents 与快捷指令实体。 |
| `app/ExpenditureWidget/` | WidgetKit 小组件。 |
| `app/BudgetIntent/`、`app/BudgetIntentUI/` | 小组件配置 intent 及对应 UI 扩展。 |
| `app/Localizations/` | 英文、法文和俄文本地化资源；部分字符串还位于 `Views/Localizable.strings`。 |
| [AGENTS.md](AGENTS.md) | 面向编码代理的仓库约束和验证要求。 |

依赖由 [Xcode 工程](app/dime.xcodeproj/project.pbxproj) 和 [Package.resolved](app/dime.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved) 管理，包括 Alamofire、CloudKitSyncMonitor、ConfettiSwiftUI、CrookedText、IsScrolling、Popovers 和 SwiftUI Introspect。语音转写使用系统框架，Gemini 通过 `URLSession` 直接调用 HTTP API。

## 参与和许可

问题与改进建议请提交到 [本仓库 Issues](https://github.com/Vanxun-Hank/dimeApp/issues)。提交改动时注明验证环境和结果；涉及数据模型、App Group 或 AI 数据发送行为时，请说明影响。

原始 Dime 由 Rafael Soh 及上游贡献者开发，界面预览素材沿用上游。项目使用 [GNU GPL v3.0](LICENSE) 许可。仓库中的 [原始隐私政策](docs/privacypolicy.md) 来自上游，尚未针对本分支新增的 Gemini 功能更新。
