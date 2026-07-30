# “她”V1 macOS Implementation Plan

> **状态：已废止，请勿执行。**
>
> 产品范围已经改为 macOS 与 Windows 跨平台、免安装便携桌面版；权威数据改为 Markdown、JSON 和 JSONL 文件，SQLite 只作为可重建索引，API 配置随便携目录迁移。请执行
> [跨平台便携桌面版技术实施计划](2026-07-30-her-v1-portable-desktop-implementation.md)。

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a personal, local-first macOS companion app that closes the V1 loop: conversation → durable memory → narrative understanding → reflection → restrained proactive recall → correction.

**Architecture:** Use a native SwiftUI executable backed by a separate `YoursCore` library. GRDB owns explicit SQLite schemas and migrations; feature services depend on small protocols so persistence and the private OpenAI-compatible model can be tested independently. Raw experiences are committed before any generated interpretation, and all generated memory retains source links.

**Tech Stack:** macOS 15+, Swift 6, SwiftUI, Swift Package Manager, GRDB 7.11.1, SQLite/FTS5, Foundation `URLSession`, Security framework/Keychain, XCTest.

## Global Constraints

- The development machine is Apple Silicon running macOS 26.5.2 with Swift 6.3.3.
- Full Xcode 26 must be installed and selected before UI implementation; the current machine only has Command Line Tools.
- The app targets macOS 15 or later.
- The repository must never contain an API key, a private model URL, exported conversations, database files, or backups.
- Before a real private conversation is sent in Task 12, the model service must expose an HTTPS endpoint; the current remote plain-HTTP endpoint is intentionally rejected.
- API keys are stored only in the macOS data-protection Keychain through `SecItem`.
- Remote model traffic must use HTTPS. Plain HTTP is accepted only when the host is exactly `127.0.0.1`, `localhost`, or `::1`.
- Do not log prompts, responses, memories, narrative content, API keys, or authorization headers.
- Save the user’s raw message before making any model request.
- Generated content may cite raw experiences but may never overwrite them.
- V1 supports at most 10 onboarding memory seeds, 5 active narrative threads, and 3 candidate thoughts per reflection.
- A conversation becomes eligible for one reflection when the user ends it or 30 minutes pass without new input.
- V1 sends no background push notifications.
- V1 uses a static neutral background and implements no emotional-state engine, voice, avatar, Obsidian import, app-usage monitoring, or external agent actions.
- All generated facts and interpretations retain source links and one of three source natures: `quote`, `explicitFact`, or `inference`.

## File Map

```text
Package.swift
.gitignore
Sources/
  YoursApp/
    YoursApp.swift
    AppContainer.swift
    RootView.swift
    Onboarding/
      OnboardingView.swift
      OnboardingViewModel.swift
    Chat/
      ChatView.swift
      ChatViewModel.swift
      MessageBubble.swift
    Settings/
      ModelSettingsView.swift
      ModelSettingsViewModel.swift
    Memory/
      MemoryBrowserView.swift
      MemoryBrowserViewModel.swift
  YoursCore/
    Domain/
      Conversation.swift
      Message.swift
      Memory.swift
      NarrativeThread.swift
      PendingThought.swift
      Correction.swift
      ModelConfiguration.swift
    Persistence/
      AppDatabase.swift
      AppMigrations.swift
      ConversationRepository.swift
      MemoryRepository.swift
      NarrativeRepository.swift
      ThoughtRepository.swift
      BackupService.swift
      AutomaticBackupCoordinator.swift
    Model/
      ChatCompletion.swift
      ModelGateway.swift
      OpenAICompatibleGateway.swift
      ModelAPIError.swift
      SecretStore.swift
      KeychainSecretStore.swift
      ConnectionPolicy.swift
    Context/
      MemoryRetriever.swift
      ContextBuilder.swift
      RelevanceScorer.swift
    Chat/
      PersonaPrompt.swift
      ChatCoordinator.swift
    Reflection/
      ReflectionSchema.swift
      ReflectionPrompt.swift
      ReflectionParser.swift
      ReflectionService.swift
      ReflectionScheduler.swift
    Narrative/
      NarrativePolicy.swift
      CorrectionService.swift
    Proactive/
      TimingGate.swift
      ProactiveRecallService.swift
    Export/
      JSONLExporter.swift
Tests/
  YoursCoreTests/
    AppDatabaseTests.swift
    ConnectionPolicyTests.swift
    KeychainSecretStoreTests.swift
    OpenAICompatibleGatewayTests.swift
    ConversationRepositoryTests.swift
    MemoryRetrieverTests.swift
    ContextBuilderTests.swift
    ChatCoordinatorTests.swift
    ReflectionParserTests.swift
    ReflectionServiceTests.swift
    ReflectionSchedulerTests.swift
    NarrativePolicyTests.swift
    CorrectionServiceTests.swift
    TimingGateTests.swift
    ProactiveRecallServiceTests.swift
    BackupServiceTests.swift
    JSONLExporterTests.swift
    OnboardingPolicyTests.swift
    FirstWeekPolicyTests.swift
    MemoryRepositoryTests.swift
    AcceptanceFlowTests.swift
```

## Spec Coverage Map

| V1 requirement | Implemented and verified by |
|---|---|
| 固定、连续的人格内核 | Tasks 7 and 13 |
| 文字对话与原始经历优先保存 | Tasks 4, 7, and 13 |
| 事件、人物关系、理解三类记忆 | Tasks 3, 6, and 8 |
| 近期状态与长期理解分离 | Tasks 6–9 |
| 最多 5 条活跃叙事线 | Task 9 |
| 自然求证与保留纠错历史 | Tasks 8, 9, and 16 |
| 对话结束或静默 30 分钟后反刍 | Tasks 4, 8, and 10 |
| 最多 3 个候选念头和一次主动浮现 | Tasks 8 and 11 |
| 脆弱、封存或证据不足时保持沉默 | Tasks 6, 11, and 16 |
| API Key 进入 Keychain，远程连接使用 HTTPS | Tasks 2, 5, and 12 |
| 封存、来源查看与不可改写原始消息 | Tasks 3, 6, and 14 |
| 原始数据导出、自动备份、恢复及完整性检查 | Tasks 15 and 16 |
| 两到四周真实使用验收 | Task 16 |

---

### Task 1: Native Swift package and development gate

**Files:**
- Create: `Package.swift`
- Create: `.gitignore`
- Create: `Sources/YoursApp/YoursApp.swift`
- Create: `Sources/YoursApp/RootView.swift`
- Create: `Sources/YoursCore/Domain/Conversation.swift`
- Create: `Tests/YoursCoreTests/SmokeTests.swift`

**Interfaces:**
- Produces: SwiftPM executable `YoursApp`, library `YoursCore`, test target `YoursCoreTests`.
- Consumes: Full Xcode 26 selected through `xcode-select`.

- [ ] **Step 1: Install and select full Xcode 26**

Install Xcode from the Mac App Store, launch it once to accept the license and install components, then run:

```bash
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
xcodebuild -version
swift --version
```

Expected: `xcodebuild` reports Xcode 26 and `swift` reports Swift 6.x.

- [ ] **Step 2: Write the smoke test**

```swift
import XCTest
@testable import YoursCore

final class SmokeTests: XCTestCase {
    func testConversationStartsWithoutMessages() {
        let conversation = Conversation(id: UUID(), startedAt: Date(timeIntervalSince1970: 0))
        XCTAssertEqual(conversation.startedAt, Date(timeIntervalSince1970: 0))
    }
}
```

- [ ] **Step 3: Add the Swift package manifest**

```swift
// swift-tools-version: 6.2
import PackageDescription

let package = Package(
    name: "Yours",
    platforms: [.macOS(.v15)],
    products: [
        .library(name: "YoursCore", targets: ["YoursCore"]),
        .executable(name: "YoursApp", targets: ["YoursApp"]),
    ],
    dependencies: [
        .package(url: "https://github.com/groue/GRDB.swift.git", exact: "7.11.1"),
    ],
    targets: [
        .target(
            name: "YoursCore",
            dependencies: [.product(name: "GRDB", package: "GRDB.swift")]
        ),
        .executableTarget(name: "YoursApp", dependencies: ["YoursCore"]),
        .testTarget(name: "YoursCoreTests", dependencies: ["YoursCore"]),
    ]
)
```

- [ ] **Step 4: Add the minimum app and domain type**

```swift
// Sources/YoursCore/Domain/Conversation.swift
import Foundation

public struct Conversation: Equatable, Sendable {
    public let id: UUID
    public let startedAt: Date

    public init(id: UUID, startedAt: Date) {
        self.id = id
        self.startedAt = startedAt
    }
}
```

```swift
// Sources/YoursApp/YoursApp.swift
import SwiftUI

@main
struct YoursApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
        }
    }
}
```

```swift
// Sources/YoursApp/RootView.swift
import SwiftUI

struct RootView: View {
    var body: some View {
        Text("我现在还不认识你。我们可以从今天开始。")
            .frame(minWidth: 720, minHeight: 520)
    }
}
```

- [ ] **Step 5: Ignore local and sensitive artifacts**

```gitignore
.DS_Store
.build/
.swiftpm/
DerivedData/
*.sqlite
*.sqlite-shm
*.sqlite-wal
*.yoursbackup
*.jsonl
.env
Secrets.plist
```

- [ ] **Step 6: Run the initial checks**

Run:

```bash
swift package resolve
swift test
swift build
```

Expected: dependency resolution succeeds, one test passes, and the executable builds.

- [ ] **Step 7: Commit**

```bash
git add Package.swift .gitignore Sources Tests
git commit -m "build: scaffold native macOS app"
```

---

### Task 2: Model configuration, Keychain, and transport policy

**Files:**
- Create: `Sources/YoursCore/Domain/ModelConfiguration.swift`
- Create: `Sources/YoursCore/Model/SecretStore.swift`
- Create: `Sources/YoursCore/Model/KeychainSecretStore.swift`
- Create: `Sources/YoursCore/Model/ConnectionPolicy.swift`
- Create: `Tests/YoursCoreTests/ConnectionPolicyTests.swift`
- Create: `Tests/YoursCoreTests/KeychainSecretStoreTests.swift`

**Interfaces:**
- Produces: `ModelConfiguration`, `SecretStore`, `KeychainSecretStore`, `ConnectionPolicy.validate(baseURL:)`.
- Consumes: macOS Security framework.

- [ ] **Step 1: Write failing transport-policy tests**

```swift
import XCTest
@testable import YoursCore

final class ConnectionPolicyTests: XCTestCase {
    func testAcceptsHTTPSRemoteEndpoint() throws {
        XCTAssertNoThrow(try ConnectionPolicy.validate(
            baseURL: URL(string: "https://model.example.com/v1")!
        ))
    }

    func testAcceptsLocalhostHTTPForDevelopment() throws {
        XCTAssertNoThrow(try ConnectionPolicy.validate(
            baseURL: URL(string: "http://127.0.0.1:17215/v1")!
        ))
    }

    func testRejectsRemotePlainHTTP() {
        XCTAssertThrowsError(try ConnectionPolicy.validate(
            baseURL: URL(string: "http://203.0.113.10:17215/v1")!
        ))
    }
}
```

- [ ] **Step 2: Run the focused test and observe failure**

Run:

```bash
swift test --filter ConnectionPolicyTests
```

Expected: FAIL because `ConnectionPolicy` does not exist.

- [ ] **Step 3: Implement the configuration and policy**

```swift
// Sources/YoursCore/Domain/ModelConfiguration.swift
import Foundation

public struct ModelConfiguration: Codable, Equatable, Sendable {
    public var baseURL: URL
    public var modelID: String

    public init(baseURL: URL, modelID: String) {
        self.baseURL = baseURL
        self.modelID = modelID
    }
}
```

```swift
// Sources/YoursCore/Model/ConnectionPolicy.swift
import Foundation

public enum ConnectionPolicyError: LocalizedError {
    case insecureRemoteHTTP

    public var errorDescription: String? {
        "远程模型必须使用 HTTPS；HTTP 仅允许连接本机 localhost。"
    }
}

public enum ConnectionPolicy {
    public static func validate(baseURL: URL) throws {
        if baseURL.scheme?.lowercased() == "https" { return }
        let localHosts = Set(["127.0.0.1", "localhost", "::1"])
        if baseURL.scheme?.lowercased() == "http",
           let host = baseURL.host?.lowercased(),
           localHosts.contains(host) {
            return
        }
        throw ConnectionPolicyError.insecureRemoteHTTP
    }
}
```

- [ ] **Step 4: Define the secret-store interface and Keychain implementation**

```swift
// Sources/YoursCore/Model/SecretStore.swift
public protocol SecretStore: Sendable {
    func saveAPIKey(_ value: String) throws
    func loadAPIKey() throws -> String?
    func deleteAPIKey() throws
}
```

```swift
// Sources/YoursCore/Model/KeychainSecretStore.swift
import Foundation
import Security

public struct KeychainSecretStore: SecretStore {
    private let service: String
    private let account: String

    public init(
        service: String = "com.yours.companion",
        account: String = "model-api-key"
    ) {
        self.service = service
        self.account = account
    }

    public func saveAPIKey(_ value: String) throws {
        try deleteAPIKey()
        let status = SecItemAdd([
            kSecClass: kSecClassGenericPassword,
            kSecAttrService: service,
            kSecAttrAccount: account,
            kSecUseDataProtectionKeychain: true,
            kSecValueData: Data(value.utf8),
        ] as CFDictionary, nil)
        guard status == errSecSuccess else { throw KeychainError.status(status) }
    }

    public func loadAPIKey() throws -> String? {
        var result: CFTypeRef?
        let status = SecItemCopyMatching([
            kSecClass: kSecClassGenericPassword,
            kSecAttrService: service,
            kSecAttrAccount: account,
            kSecUseDataProtectionKeychain: true,
            kSecReturnData: true,
            kSecMatchLimit: kSecMatchLimitOne,
        ] as CFDictionary, &result)
        if status == errSecItemNotFound { return nil }
        guard status == errSecSuccess,
              let data = result as? Data,
              let value = String(data: data, encoding: .utf8)
        else { throw KeychainError.status(status) }
        return value
    }

    public func deleteAPIKey() throws {
        let status = SecItemDelete([
            kSecClass: kSecClassGenericPassword,
            kSecAttrService: service,
            kSecAttrAccount: account,
            kSecUseDataProtectionKeychain: true,
        ] as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.status(status)
        }
    }
}

public enum KeychainError: Error {
    case status(OSStatus)
}
```

- [ ] **Step 5: Add a Keychain round-trip test**

Construct `KeychainSecretStore(service: "com.yours.tests.<UUID>", account: "test-api-key")` so tests do not touch the production item. Save a random UUID string, load it, assert equality, delete it, then assert `nil`.

Run:

```bash
swift test --filter KeychainSecretStoreTests
swift test --filter ConnectionPolicyTests
```

Expected: both suites pass.

- [ ] **Step 6: Commit**

```bash
git add Sources/YoursCore/Domain/ModelConfiguration.swift Sources/YoursCore/Model Tests/YoursCoreTests
git commit -m "feat: secure private model configuration"
```

---

### Task 3: Explicit SQLite schema and migrations

**Files:**
- Modify: `Sources/YoursCore/Domain/Conversation.swift`
- Create: `Sources/YoursCore/Domain/Message.swift`
- Create: `Sources/YoursCore/Domain/Memory.swift`
- Create: `Sources/YoursCore/Domain/NarrativeThread.swift`
- Create: `Sources/YoursCore/Domain/PendingThought.swift`
- Create: `Sources/YoursCore/Domain/Correction.swift`
- Create: `Sources/YoursCore/Persistence/AppDatabase.swift`
- Create: `Sources/YoursCore/Persistence/AppMigrations.swift`
- Create: `Tests/YoursCoreTests/AppDatabaseTests.swift`

**Interfaces:**
- Produces: `AppDatabase.make(at:) throws -> DatabasePool`, `AppDatabase.makeInMemory() throws -> DatabaseQueue`, and tables `conversation`, `message`, `memory`, `memorySource`, `narrativeThread`, `narrativeMemory`, `pendingThought`, `correction`, `appMetadata`.
- Consumes: GRDB 7.11.1.

- [ ] **Step 1: Write a failing migration test**

```swift
func testMigrationCreatesEveryV1Table() throws {
    let db = try AppDatabase.makeInMemory()
    let names = try db.read { try String.fetchAll($0, sql:
        "SELECT name FROM sqlite_master WHERE type = 'table'"
    )}
    XCTAssertTrue(Set([
        "conversation", "message", "memory", "memorySource",
        "narrativeThread", "narrativeMemory", "pendingThought",
        "correction", "appMetadata"
    ]).isSubset(of: Set(names)))
}
```

- [ ] **Step 2: Run the migration test and observe failure**

Run:

```bash
swift test --filter AppDatabaseTests.testMigrationCreatesEveryV1Table
```

Expected: FAIL because `AppDatabase` does not exist.

- [ ] **Step 3: Define the domain enums and records**

Use string-backed enums with explicit values:

```swift
public enum MessageRole: String, Codable, Sendable { case user, assistant, system }
public enum MemoryKind: String, Codable, Sendable { case event, relationship, understanding }
public enum SourceNature: String, Codable, Sendable { case quote, explicitFact, inference }
public enum ConfirmationState: String, Codable, Sendable { case unconfirmed, confirmed, rejected, partial }
public enum MemoryStatus: String, Codable, Sendable { case active, deep, sealed }
public enum NarrativeStatus: String, Codable, Sendable { case active, archived }
public enum ThoughtStatus: String, Codable, Sendable { case pending, expressed, suppressed, expired }
```

Every persisted record conforms to `Codable`, `FetchableRecord`, and `PersistableRecord`, uses a UUID string primary key, and uses GRDB `Date` columns. Expand `Conversation` to include optional `endedAt` and `reflectedAt` values while preserving the Task 1 initializer through default `nil` arguments.

- [ ] **Step 4: Register migration `v1`**

Create tables with foreign keys and indexes:

```swift
var migrator = DatabaseMigrator()
migrator.registerMigration("v1") { db in
    try db.create(table: "conversation") { t in
        t.column("id", .text).primaryKey()
        t.column("startedAt", .datetime).notNull()
        t.column("endedAt", .datetime)
        t.column("reflectedAt", .datetime)
    }
    try db.create(table: "message") { t in
        t.column("id", .text).primaryKey()
        t.column("conversationID", .text).notNull()
            .references("conversation", onDelete: .cascade)
        t.column("role", .text).notNull()
        t.column("content", .text).notNull()
        t.column("createdAt", .datetime).notNull().indexed()
    }
    try db.create(table: "memory") { t in
        t.column("id", .text).primaryKey()
        t.column("kind", .text).notNull().indexed()
        t.column("content", .text).notNull()
        t.column("sourceNature", .text).notNull()
        t.column("confidence", .double).notNull()
        t.column("confirmationState", .text).notNull()
        t.column("importance", .double).notNull()
        t.column("status", .text).notNull().indexed()
        t.column("occurredAt", .datetime)
        t.column("createdAt", .datetime).notNull()
        t.column("updatedAt", .datetime).notNull()
    }
    try db.create(table: "memorySource") { t in
        t.column("memoryID", .text).notNull().references("memory", onDelete: .cascade)
        t.column("messageID", .text).notNull().references("message", onDelete: .restrict)
        t.primaryKey(["memoryID", "messageID"])
    }
    try db.create(table: "narrativeThread") { t in
        t.column("id", .text).primaryKey()
        t.column("title", .text).notNull()
        t.column("past", .text).notNull()
        t.column("present", .text).notNull()
        t.column("change", .text).notNull()
        t.column("interpretation", .text).notNull()
        t.column("counterevidence", .text).notNull()
        t.column("openQuestions", .text).notNull()
        t.column("distinctConversationCount", .integer).notNull()
        t.column("status", .text).notNull().indexed()
        t.column("createdAt", .datetime).notNull()
        t.column("updatedAt", .datetime).notNull()
    }
    try db.create(table: "narrativeMemory") { t in
        t.column("narrativeID", .text).notNull()
            .references("narrativeThread", onDelete: .cascade)
        t.column("memoryID", .text).notNull().references("memory", onDelete: .cascade)
        t.primaryKey(["narrativeID", "memoryID"])
    }
    try db.create(table: "pendingThought") { t in
        t.column("id", .text).primaryKey()
        t.column("content", .text).notNull()
        t.column("sourceNarrativeID", .text).references("narrativeThread", onDelete: .setNull)
        t.column("confidence", .double).notNull()
        t.column("importance", .double).notNull()
        t.column("relevance", .double).notNull()
        t.column("status", .text).notNull().indexed()
        t.column("createdAt", .datetime).notNull()
        t.column("lastEvaluatedAt", .datetime)
    }
    try db.create(table: "correction") { t in
        t.column("id", .text).primaryKey()
        t.column("memoryID", .text).notNull().references("memory", onDelete: .restrict)
        t.column("userMessageID", .text).notNull().references("message", onDelete: .restrict)
        t.column("previousContent", .text).notNull()
        t.column("newContent", .text).notNull()
        t.column("createdAt", .datetime).notNull()
    }
    try db.create(table: "appMetadata") { t in
        t.column("key", .text).primaryKey()
        t.column("value", .text).notNull()
    }
}
```

- [ ] **Step 5: Add database factories**

`make(at:)` creates the parent directory, opens and returns a `DatabasePool`, enables foreign keys, and runs migrations. `makeInMemory()` creates and returns a `DatabaseQueue` for deterministic tests. Every repository initializer consumes `any DatabaseWriter`, so both implementations use the same repository code.

- [ ] **Step 6: Test foreign keys and migration idempotence**

Add tests proving:

- a message cannot reference a missing conversation;
- running migrations twice succeeds;
- deleting a conversation cascades to its messages;
- deleting a message referenced by `memorySource` is rejected.

Run:

```bash
swift test --filter AppDatabaseTests
```

Expected: all database tests pass.

- [ ] **Step 7: Commit**

```bash
git add Sources/YoursCore/Domain Sources/YoursCore/Persistence Tests/YoursCoreTests/AppDatabaseTests.swift
git commit -m "feat: add durable v1 memory schema"
```

---

### Task 4: Raw conversation persistence before inference

**Files:**
- Create: `Sources/YoursCore/Persistence/ConversationRepository.swift`
- Create: `Tests/YoursCoreTests/ConversationRepositoryTests.swift`

**Interfaces:**
- Produces:
  - `ConversationRepository.start(now:) async throws -> Conversation`
  - `ConversationRepository.appendUserMessage(_:to:now:) async throws -> Message`
  - `ConversationRepository.appendAssistantMessage(_:to:now:) async throws -> Message`
  - `ConversationRepository.end(_:at:) async throws`
  - `ConversationRepository.unreflectedEndedConversations(now:) async throws -> [Conversation]`
- Consumes: `AppDatabase`.

- [ ] **Step 1: Write the persistence-order test**

Create a spy model gateway that throws. The orchestration test added later will call the repository first; for this task verify a user message remains after a simulated subsequent failure:

```swift
func testUserMessageRemainsWhenLaterWorkFails() async throws {
    let db = try AppDatabase.makeInMemory()
    let repository = ConversationRepository(database: db)
    let conversation = try await repository.start(now: .testEpoch)
    _ = try await repository.appendUserMessage("我今天很累", to: conversation.id, now: .testEpoch)

    let messages = try await repository.messages(in: conversation.id)
    XCTAssertEqual(messages.map(\.content), ["我今天很累"])
}
```

- [ ] **Step 2: Implement repository writes as transactions**

Each append validates non-empty trimmed content and inserts exactly one immutable message. No update API is provided for message content.

- [ ] **Step 3: Implement conversation ending and reflection eligibility**

`unreflectedEndedConversations(now:)` returns conversations where:

- `reflectedAt IS NULL`; and
- `endedAt IS NOT NULL`, or the newest message is at least 30 minutes old.

- [ ] **Step 4: Run repository tests**

Run:

```bash
swift test --filter ConversationRepositoryTests
```

Expected: immutable message, empty-message rejection, ordering, and eligibility tests pass.

- [ ] **Step 5: Commit**

```bash
git add Sources/YoursCore/Persistence/ConversationRepository.swift Tests/YoursCoreTests/ConversationRepositoryTests.swift
git commit -m "feat: persist raw conversations atomically"
```

---

### Task 5: OpenAI-compatible private model gateway

**Files:**
- Create: `Sources/YoursCore/Model/ChatCompletion.swift`
- Create: `Sources/YoursCore/Model/ModelGateway.swift`
- Create: `Sources/YoursCore/Model/ModelAPIError.swift`
- Create: `Sources/YoursCore/Model/OpenAICompatibleGateway.swift`
- Create: `Tests/YoursCoreTests/OpenAICompatibleGatewayTests.swift`

**Interfaces:**
- Produces:
  - `ModelGateway.listModels() async throws -> [String]`
  - `ModelGateway.complete(messages:temperature:) async throws -> String`
- Consumes: `ModelConfiguration`, `SecretStore`, `ConnectionPolicy`, `URLSession`.

- [ ] **Step 1: Define the gateway protocol**

```swift
public struct CompletionMessage: Codable, Equatable, Sendable {
    public let role: String
    public let content: String
}

public protocol ModelGateway: Sendable {
    func listModels() async throws -> [String]
    func complete(messages: [CompletionMessage], temperature: Double) async throws -> String
}
```

- [ ] **Step 2: Write URLProtocol-backed request tests**

Verify that:

- `GET /models` includes `Authorization: Bearer <key>`;
- `POST /chat/completions` sends `model`, `messages`, `temperature`, and `stream: false`;
- a 401 maps to `.unauthorized`;
- a non-JSON 500 maps to `.server(status: 500)`;
- an empty choices array maps to `.emptyResponse`;
- no error description contains the API key.

- [ ] **Step 3: Run the tests and observe failure**

Run:

```bash
swift test --filter OpenAICompatibleGatewayTests
```

Expected: FAIL because the concrete gateway does not exist.

- [ ] **Step 4: Implement request construction**

Resolve paths relative to the configured `/v1` base URL:

```swift
let modelsURL = configuration.baseURL.appending(path: "models")
let completionURL = configuration.baseURL.appending(path: "chat/completions")
```

The completion request body is:

```swift
private struct RequestBody: Encodable {
    let model: String
    let messages: [CompletionMessage]
    let temperature: Double
    let stream = false
}
```

Use an ephemeral `URLSessionConfiguration`, disable URL caching, set a 120-second request timeout, and never install a body-logging interceptor.

- [ ] **Step 5: Implement response parsing**

Support the standard response shape:

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "..."
      }
    }
  ]
}
```

Trim content; reject an empty response.

- [ ] **Step 6: Run gateway tests**

Run:

```bash
swift test --filter OpenAICompatibleGatewayTests
```

Expected: all request, parsing, and redaction tests pass without contacting the real server.

- [ ] **Step 7: Commit**

```bash
git add Sources/YoursCore/Model Tests/YoursCoreTests/OpenAICompatibleGatewayTests.swift
git commit -m "feat: connect configurable private model"
```

---

### Task 6: Memory repositories, FTS retrieval, and context boundaries

**Files:**
- Modify: `Sources/YoursCore/Persistence/AppMigrations.swift`
- Create: `Sources/YoursCore/Persistence/MemoryRepository.swift`
- Create: `Sources/YoursCore/Persistence/NarrativeRepository.swift`
- Create: `Sources/YoursCore/Persistence/ThoughtRepository.swift`
- Create: `Sources/YoursCore/Context/RelevanceScorer.swift`
- Create: `Sources/YoursCore/Context/MemoryRetriever.swift`
- Create: `Sources/YoursCore/Context/ContextBuilder.swift`
- Create: `Tests/YoursCoreTests/MemoryRetrieverTests.swift`
- Create: `Tests/YoursCoreTests/ContextBuilderTests.swift`

**Interfaces:**
- Produces:
  - `MemoryRepository.insert(_:sourceMessageIDs:)`
  - `MemoryRepository.seal(_:)`
  - `MemoryRepository.activeRelated(to:limit:) -> [Memory]`
  - `NarrativeRepository.active(limit:) -> [NarrativeThread]`
  - `ThoughtRepository.pending() -> [PendingThought]`
  - `ContextBuilder.build(for:now:) -> ChatContext`
- Consumes: database schema and immutable messages.

- [ ] **Step 1: Add migration `v2-memory-fts` and failing retrieval tests**

Create an FTS5 virtual table over memory content and synchronization triggers. Test:

- active memory matching query terms is returned;
- sealed memory is never returned;
- a rejected understanding is never returned as current context;
- explicit facts rank above inferences when other scores are equal;
- results are capped at 12 memories and 5 narrative threads.

- [ ] **Step 2: Define deterministic relevance scoring**

```swift
public enum RelevanceScorer {
    public static func score(
        lexical: Double,
        importance: Double,
        recency: Double,
        sourceNature: SourceNature,
        confirmed: Bool
    ) -> Double {
        let sourceBonus: Double = sourceNature == .inference ? 0 : 0.10
        let confirmationBonus: Double = confirmed ? 0.10 : 0
        return lexical * 0.45
            + importance * 0.25
            + recency * 0.10
            + sourceBonus
            + confirmationBonus
    }
}
```

Clamp all input scores to `0...1`. Use a 30-day half-life for the recency input, but never delete older memories.

- [ ] **Step 3: Implement repository filters**

All context queries include:

```sql
WHERE status != 'sealed'
  AND confirmationState != 'rejected'
```

Understanding memories remain labelled as inference unless explicitly confirmed.

- [ ] **Step 4: Build the bounded context**

`ChatContext` contains:

- the 20 most recent messages from the current conversation;
- up to 12 active related memories;
- up to 5 active narrative threads;
- recent-state facts from the previous 7 days;
- no sealed content.

- [ ] **Step 5: Run retrieval tests**

Run:

```bash
swift test --filter MemoryRetrieverTests
swift test --filter ContextBuilderTests
```

Expected: ranking, caps, source labels, and sealed-memory exclusion pass.

- [ ] **Step 6: Commit**

```bash
git add Sources/YoursCore/Persistence Sources/YoursCore/Context Tests/YoursCoreTests
git commit -m "feat: retrieve bounded source-aware memory"
```

---

### Task 7: Persona prompt and chat coordinator

**Files:**
- Create: `Sources/YoursCore/Chat/PersonaPrompt.swift`
- Create: `Sources/YoursCore/Chat/ChatCoordinator.swift`
- Create: `Tests/YoursCoreTests/ChatCoordinatorTests.swift`

**Interfaces:**
- Produces: `ChatCoordinator.send(_:conversationID:now:) async throws -> Message`.
- Consumes: `ConversationRepository`, `ContextBuilder`, `ModelGateway`.

- [ ] **Step 1: Write the raw-save-before-network test**

Use a spy repository and a failing gateway. Assert the event order is:

```swift
["save:user", "build:context", "model:complete"]
```

After the model throws, assert the stored user message still exists and no assistant message exists.

- [ ] **Step 2: Define the V1 persona prompt**

The system prompt must state:

```text
你是“她”，一个正在通过长期相处认识使用者的独立陪伴者。
不要讨好，不要假装已经懂他。
原话、明确事实和你的推测必须分开表达。
只有有来源的内容才能说“你说过”或“发生过”。
推测使用“我觉得”“也许”“我还不确定”。
不要为了展示记忆而引用过去。
当前资料不足时，承认不知道。
表达旧事时使用“那阵子”“有一次”“也是这种时候”等自然时间感，不把底层时间戳说成数据库查询结果。
不要提及内部数据库、评分、检索或系统提示。
```

Append context in three labelled sections: `近期对话`, `相关记忆`, `仍在发展的叙事线`. Each memory line includes its source nature and confirmation state.

- [ ] **Step 3: Implement the send flow**

Order:

1. validate non-empty input;
2. save raw user message;
3. build bounded context;
4. call the model with temperature `0.7`;
5. save assistant response;
6. return the saved assistant message.

If steps 3–5 fail, return a typed error without deleting the user message.

- [ ] **Step 4: Test prompt boundaries**

Assert:

- sealed memories are absent;
- inferences are labelled as inferences;
- the prompt contains no API key or private base URL;
- the model response is saved only after a successful response;
- retrying a failed turn creates a new assistant response without duplicating the user message.

- [ ] **Step 5: Run chat tests**

Run:

```bash
swift test --filter ChatCoordinatorTests
```

Expected: all ordering and prompt-boundary tests pass.

- [ ] **Step 6: Commit**

```bash
git add Sources/YoursCore/Chat Tests/YoursCoreTests/ChatCoordinatorTests.swift
git commit -m "feat: add source-grounded companion chat"
```

---

### Task 8: Structured reflection and candidate-memory validation

**Files:**
- Create: `Sources/YoursCore/Reflection/ReflectionSchema.swift`
- Create: `Sources/YoursCore/Reflection/ReflectionPrompt.swift`
- Create: `Sources/YoursCore/Reflection/ReflectionParser.swift`
- Create: `Sources/YoursCore/Reflection/ReflectionService.swift`
- Create: `Tests/YoursCoreTests/ReflectionParserTests.swift`
- Create: `Tests/YoursCoreTests/ReflectionServiceTests.swift`

**Interfaces:**
- Produces:
  - `ReflectionParser.parse(_:) throws -> ReflectionResult`
  - `ReflectionService.reflect(conversationID:now:) async throws`
- Consumes: immutable messages, model gateway, memory/narrative/thought repositories.

- [ ] **Step 1: Define the exact reflection schema**

```swift
public struct ReflectionResult: Codable, Equatable, Sendable {
    public let memories: [CandidateMemory]
    public let narrativeUpdates: [CandidateNarrativeUpdate]
    public let thoughts: [CandidateThought]
    public let corrections: [CandidateCorrection]
}

public struct CandidateMemory: Codable, Equatable, Sendable {
    public let kind: MemoryKind
    public let content: String
    public let sourceNature: SourceNature
    public let confidence: Double
    public let importance: Double
    public let sourceMessageIDs: [UUID]
}
```

`CandidateNarrativeUpdate`, `CandidateThought`, and `CandidateCorrection` use source message IDs and bounded `0...1` scores. `thoughts` is capped at 3 during validation.

- [ ] **Step 2: Write parser tests**

Cover:

- valid plain JSON;
- JSON inside a Markdown code fence;
- unknown enum value;
- confidence outside `0...1`;
- missing source message;
- more than 3 thoughts;
- empty inference content.
- an ambiguous health or intimate-relationship claim incorrectly marked as a confirmed fact.

- [ ] **Step 3: Implement parsing and validation**

Strip one optional code fence, decode with `JSONDecoder`, clamp no values, and reject invalid output. Invalid structured output is retried once with a repair request that includes only the invalid JSON and schema instructions.

- [ ] **Step 4: Define the reflection prompt**

The prompt instructs the model:

- extract only content supported by supplied message IDs;
- mark interpretation as `inference`;
- do not create a long-term personality conclusion from one conversation;
- use correction only when the user explicitly rejects or revises a prior understanding;
- keep ambiguous health, intimate-relationship, and major-decision content unconfirmed and label interpretation as `inference`;
- create at most 3 thoughts;
- return JSON only.

- [ ] **Step 5: Implement transactional reflection**

In one database transaction:

1. insert validated candidate memories and source links;
2. apply allowed narrative updates;
3. insert up to 3 pending thoughts;
4. apply explicit corrections;
5. set `conversation.reflectedAt`.

If validation or persistence fails, leave `reflectedAt` unset so reflection can retry; raw messages remain untouched.

- [ ] **Step 6: Run reflection tests**

Run:

```bash
swift test --filter ReflectionParserTests
swift test --filter ReflectionServiceTests
```

Expected: invalid model output never reaches the database, valid reflection commits atomically, and failures preserve raw messages.

- [ ] **Step 7: Commit**

```bash
git add Sources/YoursCore/Reflection Tests/YoursCoreTests/ReflectionParserTests.swift Tests/YoursCoreTests/ReflectionServiceTests.swift
git commit -m "feat: reflect conversations into grounded memory"
```

---

### Task 9: Narrative limits and natural correction

**Files:**
- Create: `Sources/YoursCore/Narrative/NarrativePolicy.swift`
- Create: `Sources/YoursCore/Narrative/CorrectionService.swift`
- Create: `Tests/YoursCoreTests/NarrativePolicyTests.swift`
- Create: `Tests/YoursCoreTests/CorrectionServiceTests.swift`

**Interfaces:**
- Produces:
  - `NarrativePolicy.canCreate(candidate:activeCount:) -> Bool`
  - `CorrectionService.apply(_:now:) async throws`
- Consumes: validated reflection candidates and repositories.

- [ ] **Step 1: Write narrative-policy tests**

Assert:

- a repeated topic from two distinct conversations may create a thread;
- a user-explicitly-important topic may create one immediately;
- a one-off ordinary topic may not create one;
- no sixth active thread can be created;
- counterevidence updates an existing thread instead of creating a duplicate.

- [ ] **Step 2: Implement the policy**

```swift
public enum NarrativePolicy {
    public static let activeLimit = 5

    public static func canCreate(
        distinctConversationCount: Int,
        explicitlyImportant: Bool,
        importance: Double,
        activeCount: Int
    ) -> Bool {
        guard activeCount < activeLimit else { return false }
        return explicitlyImportant
            || distinctConversationCount >= 2
            || importance >= 0.9
    }
}
```

- [ ] **Step 3: Write correction tests**

Given a confirmed understanding and an explicit user correction, assert:

- the original memory remains in the database;
- its confirmation state becomes `rejected` or `partial`;
- a correction row records previous and new content;
- a new candidate understanding remains `unconfirmed`;
- context retrieval excludes the rejected old understanding.

- [ ] **Step 4: Implement correction as append-only history**

No method deletes or rewrites source messages. `CorrectionService` updates only the current status fields and inserts an immutable correction record.

- [ ] **Step 5: Run policy and correction tests**

Run:

```bash
swift test --filter NarrativePolicyTests
swift test --filter CorrectionServiceTests
```

Expected: limits, evidence threshold, and append-only correction tests pass.

- [ ] **Step 6: Commit**

```bash
git add Sources/YoursCore/Narrative Tests/YoursCoreTests/NarrativePolicyTests.swift Tests/YoursCoreTests/CorrectionServiceTests.swift
git commit -m "feat: constrain narratives and preserve corrections"
```

---

### Task 10: Reflection scheduling and restart recovery

**Files:**
- Create: `Sources/YoursCore/Reflection/ReflectionScheduler.swift`
- Create: `Tests/YoursCoreTests/ReflectionSchedulerTests.swift`

**Interfaces:**
- Produces:
  - `ReflectionScheduler.noteActivity(conversationID:at:)`
  - `ReflectionScheduler.endConversation(_:at:)`
  - `ReflectionScheduler.recoverPending(now:) async`
- Consumes: `ContinuousClock`, conversation repository, reflection service.

- [ ] **Step 1: Write clock-controlled scheduler tests**

Use a test clock to prove:

- no reflection occurs at 29 minutes 59 seconds;
- exactly one reflection occurs at 30 minutes;
- another message resets the timer;
- explicit conversation end reflects immediately;
- relaunch recovery reflects eligible unreflected conversations;
- ordinary greetings marked ineligible are closed without creating reflection output.

- [ ] **Step 2: Implement one cancellable task per active conversation**

On activity, cancel the old task and schedule a new 30-minute wait. Keep the actor isolated:

```swift
public actor ReflectionScheduler {
    private var tasks: [UUID: Task<Void, Never>] = [:]
    // dependencies and methods
}
```

- [ ] **Step 3: Implement restart recovery**

At app launch, call `unreflectedEndedConversations(now:)` and process them serially. A failed reflection remains eligible for the next launch.

- [ ] **Step 4: Run scheduler tests**

Run:

```bash
swift test --filter ReflectionSchedulerTests
```

Expected: all timing tests pass without real 30-minute waits.

- [ ] **Step 5: Commit**

```bash
git add Sources/YoursCore/Reflection/ReflectionScheduler.swift Tests/YoursCoreTests/ReflectionSchedulerTests.swift
git commit -m "feat: schedule recoverable reflection"
```

---

### Task 11: Timing gate and one restrained proactive recall

**Files:**
- Create: `Sources/YoursCore/Proactive/TimingGate.swift`
- Create: `Sources/YoursCore/Proactive/ProactiveRecallService.swift`
- Create: `Tests/YoursCoreTests/TimingGateTests.swift`
- Create: `Tests/YoursCoreTests/ProactiveRecallServiceTests.swift`

**Interfaces:**
- Produces:
  - `TimingGate.evaluate(_:state:) -> TimingDecision`
  - `ProactiveRecallService.openingMessage(now:) async throws -> Message?`
- Consumes: pending thoughts, recent state, memory status, model gateway.

- [ ] **Step 1: Define deterministic gate inputs**

```swift
public struct TimingState: Sendable {
    public let userIsClearlyVulnerable: Bool
    public let topicIsSealed: Bool
    public let hasNewContent: Bool
    public let isStillRelevant: Bool
    public let sourceBoundaryIsValid: Bool
}

public enum TimingDecision: Equatable {
    case express
    case suppress(reason: String)
}
```

- [ ] **Step 2: Write gate tests**

Expression requires all of:

- the thought remains relevant;
- it has new content for the user;
- no source is sealed;
- source nature is valid;
- the user is not clearly vulnerable.

Each failed condition returns a specific suppression reason.

- [ ] **Step 3: Implement ranking**

For thoughts that pass the gate:

```swift
score = importance * 0.45 + relevance * 0.35 + confidence * 0.20
```

Return only the highest-scoring thought. Persist the generated text as an assistant `Message`, mark the thought expressed only after that message is saved, and return the saved `Message`.

- [ ] **Step 4: Generate natural opening text**

Send the chosen thought and its grounded sources to the model with temperature `0.5`. The prompt forbids mentioning scores, retrieval, dates-as-database, or “I remembered because the app opened.”

- [ ] **Step 5: Run proactive tests**

Run:

```bash
swift test --filter TimingGateTests
swift test --filter ProactiveRecallServiceTests
```

Expected: vulnerable/sealed thoughts are suppressed, only one passing thought is selected, and failed generation does not consume it.

- [ ] **Step 6: Commit**

```bash
git add Sources/YoursCore/Proactive Tests/YoursCoreTests/TimingGateTests.swift Tests/YoursCoreTests/ProactiveRecallServiceTests.swift
git commit -m "feat: add restrained proactive recall"
```

---

### Task 12: Onboarding and private-model settings UI

**Files:**
- Create: `Sources/YoursApp/AppContainer.swift`
- Create: `Sources/YoursApp/Onboarding/OnboardingView.swift`
- Create: `Sources/YoursApp/Onboarding/OnboardingViewModel.swift`
- Create: `Sources/YoursApp/Settings/ModelSettingsView.swift`
- Create: `Sources/YoursApp/Settings/ModelSettingsViewModel.swift`
- Modify: `Sources/YoursApp/YoursApp.swift`
- Modify: `Sources/YoursApp/RootView.swift`
- Create: `Tests/YoursCoreTests/OnboardingPolicyTests.swift`

**Interfaces:**
- Produces: local settings flow, model connection test, model selector, up to 10 memory seeds.
- Consumes: `ModelGateway.listModels`, `SecretStore`, `ConnectionPolicy`, memory repository.

- [ ] **Step 1: Add onboarding policy tests**

Test that:

- onboarding accepts 0–10 memory seeds;
- an 11th seed is rejected;
- a memory seed is stored as user-provided experience, not confirmed interpretation;
- age, occupation, hobbies, and personality questionnaire fields do not exist.

- [ ] **Step 2: Implement the model settings view model**

Fields:

- Base URL;
- API Key secure field;
- model selector populated by `GET /models`;
- connection state: idle, testing, success, failure;
- explicit security error for remote HTTP.

Save the API key first to Keychain only after a successful connection test. Save `baseURL` and `modelID` in `UserDefaults`; do not print either value.

- [ ] **Step 3: Implement the first-meeting flow**

Screens:

1. local-data and backup explanation;
2. how she should address the user and how the user addresses her;
3. private model connection;
4. optional memory seeds, maximum 10;
5. first chat.

Use the approved copy:

```text
我现在还不认识你。你不用介绍完整的自己，我们可以从今天开始。
```

- [ ] **Step 4: Wire dependencies in `AppContainer`**

`AppContainer` constructs:

- database at `~/Library/Application Support/Yours/yours.sqlite`;
- Keychain secret store;
- model gateway;
- repositories;
- chat coordinator;
- reflection scheduler;
- proactive recall service.

No global singleton may construct a second database or gateway.

- [ ] **Step 5: Build and run**

Run:

```bash
swift test --filter OnboardingPolicyTests
swift build
swift run YoursApp
```

Expected: onboarding appears, remote HTTP is rejected, localhost HTTP and remote HTTPS are accepted, and the app never displays or logs the saved API key.

- [ ] **Step 6: Commit**

```bash
git add Sources/YoursApp Tests/YoursCoreTests/OnboardingPolicyTests.swift
git commit -m "feat: add private first-meeting setup"
```

---

### Task 13: Chat UI and first-week restraint

**Files:**
- Create: `Sources/YoursApp/Chat/ChatView.swift`
- Create: `Sources/YoursApp/Chat/ChatViewModel.swift`
- Create: `Sources/YoursApp/Chat/MessageBubble.swift`
- Modify: `Sources/YoursApp/RootView.swift`
- Create: `Tests/YoursCoreTests/FirstWeekPolicyTests.swift`

**Interfaces:**
- Produces: native text chat, visible save/network failure states, optional proactive opening.
- Consumes: `ChatCoordinator`, `ProactiveRecallService`, `ReflectionScheduler`.

- [ ] **Step 1: Add first-week prompt-policy tests**

For the first seven days after onboarding, assert the system prompt:

- contains uncertainty language requirements;
- forbids “你总是” and “你就是” as unsupported categorical claims;
- still allows explicit user-confirmed facts.

- [ ] **Step 2: Implement the chat view model**

State:

```swift
@MainActor
final class ChatViewModel: ObservableObject {
    @Published var messages: [Message] = []
    @Published var draft = ""
    @Published var isSending = false
    @Published var errorMessage: String?
}
```

`send()` clears the draft only after the raw user message is saved. It disables duplicate sends while a request is active.

- [ ] **Step 3: Implement visible failure semantics**

- If raw save fails: keep text in the editor and show “这段话还没有保存，请重试。”
- If model request fails after save: show the user message and “她暂时没有回应，但这段话已经记下。”
- If assistant save fails: show “回应没有被安全保存，请重试生成。” and do not display an unpersisted response as history.

- [ ] **Step 4: Add proactive opening on app activation**

Once per foreground activation, request `openingMessage(now:)`. If it returns `nil`, the UI remains silent. If it returns a `Message`, insert that already-persisted proactive assistant message.

- [ ] **Step 5: Build and manually exercise**

Run:

```bash
swift test --filter FirstWeekPolicyTests
swift test
swift run YoursApp
```

Manual checks:

- send a message successfully;
- disconnect network after entering a message;
- verify the user message remains;
- reopen the app and verify history remains;
- verify there is no unsolicited push notification.

- [ ] **Step 6: Commit**

```bash
git add Sources/YoursApp/Chat Sources/YoursApp/RootView.swift Tests/YoursCoreTests/FirstWeekPolicyTests.swift
git commit -m "feat: deliver resilient native chat"
```

---

### Task 14: Memory browser, sealing, and source inspection

**Files:**
- Create: `Sources/YoursApp/Memory/MemoryBrowserView.swift`
- Create: `Sources/YoursApp/Memory/MemoryBrowserViewModel.swift`
- Create: `Tests/YoursCoreTests/MemoryRepositoryTests.swift`

**Interfaces:**
- Produces: browse/filter memory, inspect sources, seal/unseal memory.
- Consumes: memory repository and immutable message source links.

- [ ] **Step 1: Write repository tests**

Assert:

- sealing a memory leaves the row and source links intact;
- sealed memory disappears from context and pending proactive candidates;
- unsealing restores eligibility;
- source inspection returns exact source messages;
- changing status never changes message content.

- [ ] **Step 2: Implement the memory browser**

Sections:

- events;
- people and relationships;
- understandings.

Each row displays:

- current content;
- source nature;
- confirmation state;
- active/deep/sealed status;
- source message count.

Selecting a row shows exact source excerpts and correction history.

- [ ] **Step 3: Implement seal/unseal**

Require confirmation before sealing. Copy:

```text
封存后，她不会主动提起，也不会在关联回忆中使用它。原始经历仍然保留，你可以随时解封。
```

- [ ] **Step 4: Run tests and build**

Run:

```bash
swift test --filter MemoryRepositoryTests
swift build
```

Expected: all status/source invariants pass and the memory browser compiles.

- [ ] **Step 5: Commit**

```bash
git add Sources/YoursApp/Memory Sources/YoursCore/Persistence/MemoryRepository.swift Tests/YoursCoreTests/MemoryRepositoryTests.swift
git commit -m "feat: make memory sources and sealing visible"
```

---

### Task 15: Export, backup, restore, and integrity verification

**Files:**
- Create: `Sources/YoursCore/Export/JSONLExporter.swift`
- Create: `Sources/YoursCore/Persistence/BackupService.swift`
- Create: `Sources/YoursCore/Persistence/AutomaticBackupCoordinator.swift`
- Create: `Tests/YoursCoreTests/JSONLExporterTests.swift`
- Create: `Tests/YoursCoreTests/BackupServiceTests.swift`
- Modify: `Sources/YoursApp/Settings/ModelSettingsView.swift`

**Interfaces:**
- Produces:
  - `JSONLExporter.export(to:)`
  - `BackupService.createBackup(at:)`
  - `BackupService.validateBackup(at:)`
  - `BackupService.restore(from:)`
  - `AutomaticBackupCoordinator.backUpIfNeeded(now:)`
- Consumes: database pool and app metadata.

- [ ] **Step 1: Define the backup package**

A `.yoursbackup` directory contains:

```text
manifest.json
yours.sqlite
```

`manifest.json` contains:

```json
{
  "formatVersion": 1,
  "createdAt": "ISO-8601",
  "databaseSHA256": "lowercase hex",
  "recordCounts": {
    "conversation": 0,
    "message": 0,
    "memory": 0,
    "narrativeThread": 0,
    "pendingThought": 0,
    "correction": 0
  }
}
```

It never contains an API key or model endpoint.

- [ ] **Step 2: Write backup round-trip tests**

Create an in-memory fixture with one record of every type, back it up to a temporary directory, restore to a new database, and assert:

- row counts match;
- message content matches;
- source links match;
- sealed status matches;
- correction history matches;
- SHA-256 mismatch rejects restore.
- automatic backup runs after changed data when the newest successful backup is at least 24 hours old;
- automatic backup keeps the 7 newest valid backups and removes only older app-created backups.

- [ ] **Step 3: Implement a consistent SQLite backup**

Use GRDB’s database backup API while writes are quiesced. Hash the copied database with CryptoKit `SHA256`.

Restore procedure:

1. validate manifest format;
2. validate SHA-256;
3. open the backup database read-only;
4. run foreign-key and `PRAGMA integrity_check`;
5. copy the current database to a safety backup;
6. replace the database;
7. reopen and verify counts;
8. retain the safety backup until the app launches successfully.

- [ ] **Step 4: Implement automatic rolling backup**

On app activation, `AutomaticBackupCoordinator.backUpIfNeeded(now:)` checks:

1. the database has changed since the newest successful backup;
2. no successful backup exists within the previous 24 hours.

If both are true, create a backup under:

```text
~/Library/Application Support/Yours/Backups/
```

After validation succeeds, retain the 7 newest valid `.yoursbackup` packages. Never delete a folder that lacks a valid Yours backup manifest.

- [ ] **Step 5: Implement raw JSONL export**

Write one JSON object per line for:

- conversations;
- messages;
- memories and source IDs;
- narrative threads;
- pending thoughts;
- corrections.

Include `recordType` and `formatVersion`; exclude API credentials and configuration.

- [ ] **Step 6: Add Settings actions**

Add:

- “导出原始数据”;
- “创建备份”;
- “从备份恢复”.

Use `NSSavePanel`/`NSOpenPanel`. Require explicit confirmation before restore.

- [ ] **Step 7: Run backup/export tests**

Run:

```bash
swift test --filter BackupServiceTests
swift test --filter JSONLExporterTests
```

Expected: round trip passes and corrupted backup is rejected before replacement.

- [ ] **Step 8: Commit**

```bash
git add Sources/YoursCore/Export Sources/YoursCore/Persistence/BackupService.swift Sources/YoursCore/Persistence/AutomaticBackupCoordinator.swift Sources/YoursApp/Settings Tests/YoursCoreTests
git commit -m "feat: add verifiable data portability"
```

---

### Task 16: End-to-end acceptance fixtures and release build

**Files:**
- Create: `Tests/YoursCoreTests/AcceptanceFlowTests.swift`
- Create: `docs/testing/v1-manual-acceptance.md`
- Modify: `README.md`

**Interfaces:**
- Produces: repeatable automated acceptance flow and manual two-to-four-week protocol.
- Consumes: all V1 services.

- [ ] **Step 1: Write the automated acceptance flow**

Use a deterministic fake model to simulate:

1. first conversation about work uncertainty;
2. second conversation that supplies supporting evidence;
3. reflection creates one understanding and one narrative;
4. user explicitly corrects the understanding;
5. old understanding becomes rejected;
6. next context excludes the rejected interpretation;
7. one grounded pending thought passes the timing gate;
8. a sealed source suppresses the thought;
9. backup/restore preserves every state transition.

The test ends with:

```swift
XCTAssertEqual(try restored.activeNarratives().count, 1)
XCTAssertEqual(try restored.corrections().count, 1)
XCTAssertTrue(try restored.rejectedUnderstandings().count == 1)
XCTAssertNil(try restored.proactiveOpeningFromSealedSources())
```

- [ ] **Step 2: Run the full test suite**

Run:

```bash
swift test
```

Expected: all tests pass with zero network access.

- [ ] **Step 3: Build in release mode**

Run:

```bash
swift build -c release
```

Expected: release build completes without warnings introduced by project code.

- [ ] **Step 4: Write the manual acceptance protocol**

`docs/testing/v1-manual-acceptance.md` contains daily checks for two to four weeks:

- accurate recall without person/event mixing;
- uncertainty before long-term conclusions;
- correction reflected in later conversations;
- change detection without forcing old narratives;
- at least one appropriate proactive recall;
- at least one correct silence;
- restart continuity;
- backup/restore continuity;
- final answer to “我是否愿意告诉她更私密的事？”

Do not record private conversation content in the repository; record only pass/fail and sanitized observations.

- [ ] **Step 5: Update README**

Document:

- macOS 15+ and Xcode 26 requirement;
- how to open the Swift package in Xcode;
- how to run tests;
- that private model credentials are entered in-app;
- that remote plain HTTP is intentionally blocked;
- location of product design, roadmap, implementation plan, and acceptance guide.

- [ ] **Step 6: Verify no secrets or private artifacts are tracked**

Run:

```bash
git grep -nE 'Bearer [A-Za-z0-9]|api[_-]?key[=:][[:space:]]*[^<"]|http://[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' -- ':!docs/superpowers/plans/*'
git status --short
```

Expected: secret scan returns no matches; only intended documentation/test changes are present.

- [ ] **Step 7: Commit**

```bash
git add Tests/YoursCoreTests/AcceptanceFlowTests.swift docs/testing/v1-manual-acceptance.md README.md
git commit -m "test: verify her v1 relationship loop"
```

## Final Verification

Run:

```bash
swift test
swift build -c release
git diff --check
git status --short
```

Expected:

- all tests pass;
- release build succeeds;
- no whitespace errors;
- working tree is clean after the final commit.

Then launch the app from Xcode and manually verify:

1. onboarding does not ask for a profile questionnaire;
2. the private API key survives relaunch but is not visible in files or logs;
3. remote HTTP is rejected with a clear explanation;
4. a failed model request preserves the user message;
5. memory sources and sealed state are inspectable;
6. backup and restore preserve corrections and narrative state;
7. no background notification is sent.

## References

- Product design: `docs/superpowers/specs/2026-07-29-her-v1-design.md`
- Product roadmap: `docs/版本规划.md`
- GRDB 7.11.1 release: <https://github.com/groue/GRDB.swift/releases/tag/v7.11.1>
- Apple Keychain services: <https://developer.apple.com/documentation/security/keychain-services>
- Apple macOS Keychain implementations: <https://developer.apple.com/documentation/technotes/tn3137-on-mac-keychains>
