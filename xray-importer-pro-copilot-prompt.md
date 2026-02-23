# GitHub Copilot Prompt — Generate: xray-importer-pro

## ROLE
You are a senior Java engineer. Generate a **complete, production-ready Maven desktop application** exactly matching every constraint below. Output every file in full — no placeholders, no "// TODO", no truncation.

---

## PROJECT IDENTITY

| Key | Value |
|-----|-------|
| Artifact ID | `xray-importer-pro` |
| Group ID | `com.estjira` |
| Version | `1.0.0` |
| Main class | `com.estjira.ui.MainApp` |
| Java version | 17 |
| Build tool | Maven 3.6+ |
| Window title | `"Jira Test Case Importer — Pro"` |

---

## PACKAGE STRUCTURE

Create exactly these 7 packages (no others):

```
com.estjira.ui        → Swing UI
com.estjira.csv       → CSV parsing
com.estjira.xray      → Xray Cloud API client
com.estjira.jira      → Jira REST client
com.estjira.config    → Config JSON loader
com.estjira.model     → Data models
com.estjira.util      → Shared utilities
```

---

## FILES TO GENERATE

Generate every file listed below in full.

### `pom.xml`
- Java 17 compiler source/target
- Dependencies (exact versions):
  - `com.fasterxml.jackson.core:jackson-databind:2.16.1`
  - `com.fasterxml.jackson.core:jackson-core:2.16.1`
  - `com.opencsv:opencsv:5.9`
  - `org.apache.httpcomponents:httpclient:4.5.14`
  - `org.apache.httpcomponents:httpmime:4.5.14`
- Maven Shade Plugin 3.5.1 bound to `package` phase
  - Produces runnable fat-jar
  - `mainClass` = `com.estjira.ui.MainApp`
  - Exclude `META-INF/*.SF`, `*.DSA`, `*.RSA`
  - Include `ServicesResourceTransformer`
  - Set `createDependencyReducedPom=false`

---

### `com.estjira.model.TestStep`
Fields: `int stepNo`, `String action`, `String data`, `String expectedResult`
- Full-arg constructor + no-arg constructor
- All getters/setters
- `toString()` returning `"Step N: action"`

---

### `com.estjira.model.TestCase`
Fields:
- `String identifier`
- `String summary`
- `String description`
- `String assignee` (stores employee ID, e.g. G01348533)
- `String reporter` (stores employee ID)
- `String priority`
- `List<String> labels` (initialized to `new ArrayList<>()`)
- `List<TestStep> steps` (initialized to `new ArrayList<>()`)
- `Map<String, Object> rawCsvRow` (LinkedHashMap)
- `Map<String, Object> extraFields` (LinkedHashMap, for customfield_xxxxx)
- `String importStatus` ("SUCCESS" | "FAILED" | "SKIPPED" | "DRY_RUN")
- `String jiraIssueKey`
- `String errorMessage`
- Method `addStep(TestStep)`, method `putExtraField(String, Object)`
- All getters/setters

---

### `com.estjira.model.ImportConfig`
Jackson-deserializable POJO. Fields with `@JsonProperty`:
- `String projectKey`
- `String issueType` (default `"Test"`)
- `Map<String, String> fieldMappings` (LinkedHashMap)
- `Map<String, String> stepMappings` (LinkedHashMap)
- `String testCaseIdentifierColumn` (default `"Test Case Identifier"`)
- `Map<String, String> priorityMap` (LinkedHashMap)
- `boolean dryRun` (default false)
- `int threadCount` (default 4)
- `int maxRetries` (default 3)
- `String xrayProjectKey`
- `Map<String, Object> extra` — catch-all via `@JsonAnySetter`
- Helper method: `String getStepColumn(String field)` → returns `stepMappings.getOrDefault(field, field)`

The JSON config file it reads looks like:
```json
{
  "projectKey": "PROJ",
  "issueType": "Test",
  "xrayProjectKey": "PROJ",
  "testCaseIdentifierColumn": "Test Case Identifier",
  "fieldMappings": {
    "summary": "Test Case Name",
    "description": "Description",
    "assignee": "Assignee Employee ID",
    "reporter": "Reporter Employee ID",
    "priority": "Priority",
    "labels": "Labels",
    "customfield_10200": "Custom Field"
  },
  "stepMappings": {
    "stepNo": "Step no.",
    "action": "Action",
    "data": "Test Data",
    "expectedResult": "Expected Result"
  },
  "priorityMap": { "High": "High", "Critical": "Highest" },
  "dryRun": false,
  "threadCount": 4,
  "maxRetries": 3
}
```

---

### `com.estjira.model.ImportResult`
Immutable result per test case. Fields: `testCaseIdentifier`, `success`, `jiraKey`, `errorMessage`, `dryRun`, `timestamp` (LocalDateTime).
- Static factories: `ImportResult.success(id, jiraKey, dryRun)` and `ImportResult.failure(id, error)`
- `toString()` formats differently for dry-run, success, failure

---

### `com.estjira.config.ConfigLoader`
- `static ImportConfig load(File file)` — Jackson ObjectMapper reads JSON file
- `static ImportConfig loadFromString(String json)`
- `static void save(ImportConfig config, File file)` — pretty-printed
- `static String sampleConfigJson()` — returns a hardcoded sample JSON string

---

### `com.estjira.util.AppLogger`
- Constructor: `AppLogger(Consumer<String> logSink)`
- Methods: `info(String)`, `warn(String)`, `error(String)`, `success(String)`, `dryRun(String)`
- Each formats: `[HH:mm:ss] [LEVEL] message` using `DateTimeFormatter`
- Calls `logSink.accept(line)` or falls back to `System.out.println`

---

### `com.estjira.util.RetryUtil`
- `static <T> T withRetry(Callable<T> task, int maxRetries, long baseDelayMs, Consumer<String> logger)`
  - Attempt `maxRetries+1` times total
  - On failure: log retry message, sleep `delay`, double delay (exponential backoff)
  - On `InterruptedException`: restore interrupt flag and rethrow
  - After all retries exhausted: throw `Exception` wrapping last cause
- Overload without logger parameter

---

### `com.estjira.util.UserCache`
Resolves Jira user accountId from an employee ID string (e.g. `G01348533`).
- Constructor: `UserCache(CloseableHttpClient, String jiraBaseUrl, String authHeader, AppLogger)`
- Internal `ConcurrentHashMap<String, String>` cache (maps employeeId → accountId, caches nulls too)
- `String resolve(String employeeId)`:
  - Return `null` immediately for blank input
  - Check cache first
  - Call `GET {jiraBaseUrl}/rest/api/3/user/search?query={URLEncoded employeeId}`
  - Set headers: `Authorization`, `Accept: application/json`
  - Parse JSON array response, extract `accountId` from first element
  - Cache and return result (or cache null if not found)
  - Log appropriately at each path via `AppLogger`
- `void clearCache()`
- `int cacheSize()`

---

### `com.estjira.csv.CsvParser`
Static utility class.

`static List<TestCase> parse(File csvFile, ImportConfig config) throws IOException`

Behaviour:
1. Open CSV with OpenCSV `CSVReaderBuilder`
2. Read first row as headers (trim all header values)
3. For each subsequent row: build `Map<String, String>` of header→value
4. Get value at `config.getTestCaseIdentifierColumn()` as identifier; skip blank identifiers
5. Use `LinkedHashMap<String, TestCase>` keyed by identifier to preserve insertion order
6. On **first encounter** of an identifier: create `TestCase`, call `mapFields(...)`, store `rawCsvRow`
7. On every row (including first): call `buildStep(...)` and `addStep()` if non-null
8. After parsing, sort each TestCase's steps by `stepNo` ascending
9. Return `new ArrayList<>(caseMap.values())`

`mapFields` logic (switch on Jira field key):
- `"summary"` → `tc.setSummary(value)`
- `"description"` → `tc.setDescription(value)`
- `"assignee"` → `tc.setAssignee(value)` (raw employee ID stored, resolved later)
- `"reporter"` → `tc.setReporter(value)`
- `"priority"` → look up in `config.getPriorityMap()`, fallback to raw value; set if non-empty
- `"labels"` → split by `[,;|]`, trim each, add non-blank entries to `tc.getLabels()`
- default → if value non-empty, call `tc.putExtraField(jiraField, value)` (handles `customfield_xxxxx`)
- Fallback: if summary still empty after mapping, set it to the identifier

`buildStep` logic:
- Read stepNo column (default `"Step no."`), action, data, expectedResult columns from `stepMappings`
- Return `null` if both stepNo and action are blank
- Parse stepNo as int (default 0 on parse failure)
- Return `new TestStep(stepNo, action, data, expectedResult)`

---

### `com.estjira.jira.JiraClient` (implements `AutoCloseable`)
Constructor A: `JiraClient(String jiraBaseUrl, String email, String apiToken, AppLogger)`
- Build Basic auth header from `email:apiToken` encoded in Base64
- Create `CloseableHttpClient` via `HttpClients.createDefault()`

Constructor B: `JiraClient(String jiraBaseUrl, String basicAuthHeader, AppLogger)`
- Use provided header directly

Methods:
- `String testConnection()` — `GET /rest/api/3/myself`, return `displayName`, log success; throw `IOException` on non-200
- `String searchUserAccountId(String query)` — `GET /rest/api/3/user/search?query=...`, return first `accountId` or null
- Expose getters: `getJiraBaseUrl()`, `getAuthHeader()`, `getHttpClient()`
- Private `normalizeUrl(String)` — strip trailing slash
- Private `truncate(String, int)` — safe substring for log messages
- `close()` — closes `httpClient`

---

### `com.estjira.xray.XrayClient` (implements `AutoCloseable`)
Base URL constant: `"https://xray.cloud.getxray.app"`

Constructor: `XrayClient(String clientId, String clientSecret, String jiraBaseUrl, String jiraAuthHeader, AppLogger)`
- Creates its own `CloseableHttpClient`
- Caches `xrayToken` field (String, initially null)

**Authentication:**
`String authenticate() throws IOException`
- `POST https://xray.cloud.getxray.app/api/v2/authenticate`
- Body JSON: `{"client_id": "...", "client_secret": "..."}`
- Response is a quoted string — strip quotes, store in `xrayToken`
- Log success or throw `IOException` on non-200

Private `String getToken()` — calls `authenticate()` if token null/blank, returns token

**Single test import:**
`private String importSingleTest(TestCase tc, ImportConfig config, String executionKey, UserCache userCache)`
- Build payload via `buildSingleTestPayload(...)`
- `POST /api/v2/import/execution` with Bearer token
- Wrap in `RetryUtil.withRetry(...)` using `config.getMaxRetries()` and 1000ms base delay
- Parse `key` from response JSON (fallback: `testExecIssue.key`)

**Bulk import:**
`List<ImportResult> bulkImport(List<TestCase>, ImportConfig, String executionKey, UserCache, Consumer<int[]> progressCb, Consumer<String> logCb)`
- If `config.isDryRun()`: iterate, log dry-run per test case, return `ImportResult.success(id, null, true)` for each
- Otherwise: create `ExecutorService` with `config.getThreadCount()` threads
- Submit one `Future<ImportResult>` per test case, each calling `importSingleTest`
- In finally block of each future: increment progress counter, call `progressCb.accept(new int[]{current, total})`
- Log `[OK]` or `[FAIL]` per result via `logCb`
- Await all futures, collect `ImportResult` list, return it

**Payload builders (private):**

`ObjectNode buildExecutionPayload(List<TestCase>, ImportConfig, String executionKey, UserCache)`
- Root: `{"testExecutionInfo": {...}, "tests": [...]}`
- If executionKey non-blank: `testExecutionInfo` = `{"testExecutionKey": "..."}` 
- Else: `testExecutionInfo` = `{"project": "...", "summary": "Test Execution - Bulk Import"}`
- Call `buildTestNode(tc, config, userCache)` for each TestCase

`ObjectNode buildSingleTestPayload(TestCase, ImportConfig, String executionKey, UserCache)`
- Same structure but `summary` = `"Test Execution - {identifier}"`

`ObjectNode buildTestNode(TestCase, ImportConfig, UserCache)`
- `fields` object containing:
  - `summary` (required)
  - `issuetype: {name: config.getIssueType()}`
  - `project: {key: config.getProjectKey()}`
  - `description` (if non-blank)
  - `priority: {name: tc.getPriority()}` (if non-null/non-blank)
  - `labels: [...]` (if list non-empty)
  - `assignee: {accountId: ...}` — resolve via `userCache.resolve(tc.getAssignee())`, only set if resolved non-null
  - `reporter: {accountId: ...}` — same pattern
  - Extra fields from `tc.getExtraFields()` — if value is `String`, call `fields.put(key, value)`
- `testInfo` object:
  - `type: "Manual"`
  - `steps: [{action, data, result}, ...]` for each `TestStep`
- Return node with both `fields` and `testInfo`

---

### `com.estjira.ui.MainApp`
```java
public static void main(String[] args) {
    System.setProperty("sun.java2d.uiScale", "1.0");
    SwingUtilities.invokeLater(() -> {
        try { UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName()); } catch (Exception ignored) {}
        new MainWindow();
    });
}
```

---

### `com.estjira.ui.MainWindow extends JFrame`

**Window:** title `"Jira Test Case Importer — Pro"`, `EXIT_ON_CLOSE`, minimum `1100×750`, preferred `1200×820`

**Color palette (private static final):**
```java
BG_DARK    = new Color(30, 33, 40)
BG_PANEL   = new Color(42, 45, 54)
ACCENT     = new Color(99, 179, 237)
FG_TEXT    = new Color(220, 220, 220)
BORDER_COLOR = new Color(70, 73, 82)
```

**UI Components (private fields):**
- `JTextField txtJiraUrl` (cols 30)
- `JTextField txtXrayClientId` (cols 20)
- `JPasswordField txtXrayClientSecret` (cols 20)
- `JTextField txtCsvFile` (cols 30, non-editable)
- `JTextField txtConfigFile` (cols 30, non-editable)
- `JCheckBox chkUseExistingExecution` ("Use Existing Test Execution")
- `JTextField txtExecutionKey` (cols 15, initially disabled)
- `JButton btnTestConnection` ("Test Connection")
- `JButton btnPreview` ("Preview")
- `JButton btnImport` ("Import to Jira")
- `JButton btnClearLog` ("Clear Log")
- `DefaultTableModel previewTableModel`
- `JTable previewTable`
- `JTextArea logArea` (12 rows, 60 cols, non-editable, word-wrap)
- `JProgressBar progressBar` (0–100, string-painted)
- `List<TestCase> previewedTestCases` (field, initialized to empty ArrayList)
- `AppLogger logger` (initialized in constructor with `this::appendLog` sink)

**Layout:**
- Root panel: `BorderLayout(8,8)`, 10px `EmptyBorder`
- `NORTH` → settings panel (GridBagLayout, rows for each field + button row)
- `CENTER` → `JSplitPane(VERTICAL_SPLIT)` with resizeWeight=0.55
  - Top pane: preview panel (JScrollPane wrapping JTable), TitledBorder "Preview"
  - Bottom pane: log panel (JScrollPane wrapping JTextArea), TitledBorder "Live Log"
- `SOUTH` → bottom panel (BorderLayout): `progressBar` CENTER, version label EAST

**Settings panel rows (GridBagLayout):**
- Row 0: label "Jira URL:" | `txtJiraUrl`
- Row 1: label "Xray Client ID:" | `txtXrayClientId`
- Row 2: label "Xray Client Secret:" | `txtXrayClientSecret`
- Row 3: label "CSV File:" | file-chooser row for CSV (extension filter: csv)
- Row 4: label "Config JSON:" | file-chooser row for JSON (extension filter: json)
- Row 5: label "Execution:" | row with `chkUseExistingExecution` + `txtExecutionKey`
- Row 6: (gridwidth=2) button row with all 4 buttons

**File chooser row:** `JPanel(BorderLayout)` with `JTextField` CENTER and `JButton` EAST. Button opens `JFileChooser` with appropriate extension filter; on approve sets text field path.

**Button colors:**
- Test Connection: `new Color(52, 152, 219)`
- Preview: `new Color(39, 174, 96)`
- Import to Jira: `new Color(155, 89, 182)`
- Clear Log: `new Color(100, 100, 110)`
- All buttons: white foreground, no border paint, no focus paint, `HAND_CURSOR`, `Segoe UI Bold 13`, preferred height 32

**Styling helpers (private):**
- `darkPanel(LayoutManager)` — JPanel with `BG_PANEL` background
- `styledLabel(String)` — JLabel with `FG_TEXT`, `Segoe UI Plain 13`
- `style(JTextField)` — applies dark background, `FG_TEXT`, compound border (line + empty insets), `Segoe UI Plain 13`
- `styleButton(JButton, Color)`
- `titledBorder(String)` — `TitledBorder` with `BORDER_COLOR` line, `ACCENT` title color, `Segoe UI Bold 12`

**Wire listeners:**
- `chkUseExistingExecution` → enable/disable `txtExecutionKey`
- `btnClearLog` → `logArea.setText("")`, reset progress bar
- `btnTestConnection` → `runAsync(this::doTestConnection)`
- `btnPreview` → `runAsync(this::doPreview)`
- `btnImport` → show `JOptionPane.showConfirmDialog` YES/NO; on YES → `runAsync(this::doImport)`

**`doTestConnection()`** (runs on worker thread):
1. `setUiBusy(true, "Testing connection...")`
2. Validate Jira URL non-blank
3. Create `JiraClient(jiraUrl, clientId, clientSecret, logger)` in try-with-resources → call `testConnection()`
4. If clientId and clientSecret non-blank: create `XrayClient(...)` in try-with-resources → call `authenticate()`
5. On any exception: `appendLog("[ERROR] ...")` + `showError(...)`
6. Finally: `setUiBusy(false, null)`, `setProgress(100, "Connection OK")` on success

**`doPreview()`** (runs on worker thread):
1. `setUiBusy(true, "Parsing CSV...")`
2. Validate and load `csvFile` and `configFile`
3. `ConfigLoader.load(configFile)` → `ImportConfig`
4. `CsvParser.parse(csvFile, config)` → `List<TestCase>`
5. Store in `previewedTestCases`
6. `SwingUtilities.invokeLater(() -> populatePreviewTable(cases))`
7. Log count of test cases and total steps
8. If `config.isDryRun()` → log dry-run notice
9. `setProgress(100, "Preview complete — N test cases")`

**`doImport()`** (runs on worker thread):
1. `setUiBusy(true, "Importing...")`
2. Gather and validate all inputs
3. Load config; re-parse CSV if `previewedTestCases` is empty
4. If still empty → log warn and return
5. Determine `executionKey` from checkbox + text field
6. Build Basic auth header for Jira: Base64(`clientId:clientSecret`)
7. Create `AppLogger`, `UserCache` (using `HttpClients.createDefault()`)
8. Create `XrayClient` in try-with-resources; call `authenticate()`
9. Call `xray.bulkImport(...)` with progress callback that:
   - Computes `pct = (current / total) * 100`
   - Calls `SwingUtilities.invokeLater(() -> { progressBar.setValue(pct); progressBar.setString("x/y imported"); })`
10. After completion: log summary (success count, failure count)
11. `setProgress(100, "Done: OK/total imported")`
12. `SwingUtilities.invokeLater(() -> updateTableWithResults(results))`
13. On exception: log full stack trace via `StringWriter/PrintWriter`, `showError(...)`

**`populatePreviewTable(List<TestCase>)`** (call on EDT):
- Columns: `"#"`, `"Identifier"`, `"Summary"`, `"Priority"`, `"Labels"`, `"Assignee"`, `"Steps"`, `"Status"`
- One row per TestCase
- Labels: `String.join(", ", tc.getLabels())`
- Steps: `tc.getSteps().size()`
- Status: `"Pending"`
- After adding rows: auto-size each column width by measuring cell content (cap at 300px)

**`updateTableWithResults(List<ImportResult>)`** (call on EDT):
- Find "Status" column index via `previewTableModel.findColumn("Status")`
- For dry-run: set `"DRY-RUN"`; success: set `"✓ {key}"`; failure: set `"✗ FAILED"`

**Utility methods:**
- `appendLog(String)` — `SwingUtilities.invokeLater` → `logArea.append(msg + "\n")` + scroll to end
- `setUiBusy(boolean, String)` — EDT: enable/disable 3 action buttons; set progress bar string + indeterminate
- `setProgress(int, String)` — EDT: stop indeterminate, set value and string
- `showError(String, String)` — EDT: `JOptionPane.showMessageDialog(..., ERROR_MESSAGE)`
- `getJiraUrl()` — trim and strip trailing slash from `txtJiraUrl`
- `validateField(String, String)` — throw `IllegalArgumentException` if blank
- `validateFileField(JTextField, String)` — throw if blank or file doesn't exist, return `File`
- `runAsync(Runnable)` — start daemon thread named `"importer-worker"`

---

### `src/main/resources/sample-config.json`
Full working example config (same JSON shown in `ImportConfig` section above, with all keys populated).

### `src/main/resources/sample-test-cases.csv`
CSV with headers:
```
Test Case Identifier, Test Case Name, Description, Priority, Labels,
Assignee Employee ID, Reporter Employee ID, Custom Field,
Step no., Action, Test Data, Expected Result
```
Include 3 test cases (`TC-001`, `TC-002`, `TC-003`) with 4, 4, and 3 steps respectively — all data realistic.

---

## CONSTRAINTS & RULES

1. **No placeholders.** Every method must be fully implemented.
2. **No test classes.** No `src/test` directory.
3. **Thread safety.** `bulkImport` must use `CopyOnWriteArrayList` for result collection.
4. **EDT safety.** All Swing mutations must happen via `SwingUtilities.invokeLater`.
5. **AutoCloseable.** `JiraClient` and `XrayClient` both implement `AutoCloseable` and close their `CloseableHttpClient`.
6. **Error isolation.** Each per-test-case import thread catches its own exceptions and records `ImportResult.failure(...)` — one failure must not abort others.
7. **Negative caching.** `UserCache` must cache null results to avoid repeated API calls for unknown employee IDs.
8. **Step ordering.** `CsvParser` must sort steps by `stepNo` int value after grouping — not lexicographically.
9. **Config flexibility.** `CsvParser.mapFields` must use a `switch` on Jira field key so `customfield_xxxxx` keys fall through to `tc.putExtraField(...)` automatically.
10. **Fat JAR.** The Maven Shade plugin config must produce a single self-contained executable JAR. Do not use `maven-assembly-plugin`.
11. **No external icon/image assets.** The UI must render without any bundled image files.
12. **Label splitting.** Labels column value must be split on `[,;|]` regex and each segment trimmed.
13. **Priority mapping.** Priority value from CSV must be looked up in `config.getPriorityMap()` before being set; raw value used as fallback if no mapping entry exists.
14. **Xray token format.** The `/api/v2/authenticate` response is a JSON-quoted string. Strip surrounding double-quotes before storing.

---

## OUTPUT FORMAT

Output files in this order, each preceded by a markdown code block with the filename as the label:

1. `pom.xml`
2. `src/main/java/com/estjira/model/TestStep.java`
3. `src/main/java/com/estjira/model/TestCase.java`
4. `src/main/java/com/estjira/model/ImportConfig.java`
5. `src/main/java/com/estjira/model/ImportResult.java`
6. `src/main/java/com/estjira/config/ConfigLoader.java`
7. `src/main/java/com/estjira/util/AppLogger.java`
8. `src/main/java/com/estjira/util/RetryUtil.java`
9. `src/main/java/com/estjira/util/UserCache.java`
10. `src/main/java/com/estjira/csv/CsvParser.java`
11. `src/main/java/com/estjira/jira/JiraClient.java`
12. `src/main/java/com/estjira/xray/XrayClient.java`
13. `src/main/java/com/estjira/ui/MainApp.java`
14. `src/main/java/com/estjira/ui/MainWindow.java`
15. `src/main/resources/sample-config.json`
16. `src/main/resources/sample-test-cases.csv`

Do not output anything other than these 16 files. Do not add explanatory prose between files. Every file must be complete and compilable as-is with `mvn clean package` on Java 17.
