# GitHub Copilot Prompt — Jira Test Case Importer (estjira) v2

> **Instructions for Copilot:** Generate the complete Maven Java 17 project described below, file by file, exactly as specified. After generating all files, create a ZIP archive of the project folder named `jira-testcase-importer.zip`.

---

## 1. Project Overview

A Java 17 Swing desktop application that:
1. Reads an **"xray bulk upload importer" JSON config file** to get the CSV column → Jira field mapping
2. Reads **CSV files** containing test cases using that mapping
3. On import:
   - **Creates a Test Execution** issue in Jira (linked to a Feature/User Story and tagged with a Label)
   - **Imports all test cases** as Jira issues
   - **Links** each imported test case back to the Test Execution via Jira's issue link API

---

## 2. Project Coordinates

```
groupId:    com.estjira
artifactId: jira-testcase-importer
version:    1.0.0
mainClass:  com.estjira.Main
```

---

## 3. Full File & Package Structure

```
jira-testcase-importer/
├── pom.xml
├── sample-config.json
├── sample-testcases.csv
└── src/main/java/com/estjira/
    ├── Main.java
    ├── model/
    │   ├── FieldConfig.java
    │   └── TestCase.java
    ├── service/
    │   ├── ConfigService.java          ← interface
    │   ├── ConfigServiceImpl.java
    │   ├── CsvService.java             ← interface
    │   ├── CsvServiceImpl.java
    │   ├── JiraService.java            ← interface
    │   └── JiraServiceImpl.java
    └── ui/
        └── MainFrame.java
```

> ⚠️ There is **no** `service/impl/` sub-package. All service classes live directly in `com.estjira.service`.

---

## 4. `pom.xml`

- Java source/target: **17**
- Encoding: UTF-8
- **Only 2 dependencies:**
  - `com.opencsv:opencsv:5.9`
  - `com.fasterxml.jackson.core:jackson-databind:2.17.1`
- Use **maven-shade-plugin 3.5.2** to produce a single fat JAR:
  - `<finalName>jira-testcase-importer</finalName>`
  - `<mainClass>com.estjira.Main</mainClass>`
  - Exclude `META-INF/*.SF`, `META-INF/*.DSA`, `META-INF/*.RSA`

---

## 5. File-by-File Specifications

---

### 5.1 `model/FieldConfig.java`

- Package: `com.estjira.model`
- Annotate with `@JsonIgnoreProperties(ignoreUnknown = true)` from Jackson
- **Public fields** (no getters/setters):
  ```java
  Map<String, String> projectKey      = new LinkedHashMap<>();
  Map<String, String> field           = new LinkedHashMap<>();  // CSV header → Jira field ID
  Map<String, String> link            = new LinkedHashMap<>();
  Map<String, String> projectSelection = new LinkedHashMap<>();
  Map<String, String> projectName     = new LinkedHashMap<>();
  Map<String, String> value           = new LinkedHashMap<>();
  ```
- Add a convenience method:
  ```java
  public String resolvedProjectKey() {
      return projectKey.getOrDefault("project", "");
  }
  ```
- Add a Javadoc comment explaining the `field` map:
  > "The field map key = CSV column header name. The field map value = Jira field ID / custom field ID."

---

### 5.2 `model/TestCase.java`

- Package: `com.estjira.model`
- **All public fields, no getters/setters:**
  ```java
  String testCaseIdentifier;
  String summary;
  String description;
  String priority;
  String assignee;
  String reporter;
  String environment;
  String testType;            // mapped from customfield_25902
  String testRepositoryPath;  // mapped from customfield_17400
  String labels;
  String outwardIssueLink;    // mapped from link-10900
  String action;
  String stepNo;
  String userStory;
  Map<String, String> jiraFields = new LinkedHashMap<>();  // dynamic: Jira field ID → value
  ```

---

### 5.3 `service/ConfigService.java`

- Package: `com.estjira.service`
- Interface with **one method**:
  ```java
  FieldConfig load(File configFile) throws Exception;
  ```
- Javadoc: "Reads and parses the xray bulk upload importer config JSON file."

---

### 5.4 `service/ConfigServiceImpl.java`

- Package: `com.estjira.service`
- Implements `ConfigService`
- Uses `new ObjectMapper().readValue(configFile, FieldConfig.class)`
- Throws `Exception("Config file not found: ...")` if file is null or does not exist
- Throws `Exception("Config file has no 'field' mappings defined.")` if `config.field` is null or empty

---

### 5.5 `service/CsvService.java`

- Package: `com.estjira.service`
- Interface with **two methods**:
  ```java
  List<TestCase> parse(File file, FieldConfig config, String userStory) throws Exception;
  File[] listFiles(File folder);
  ```

---

### 5.6 `service/CsvServiceImpl.java`

- Package: `com.estjira.service`
- Implements `CsvService`
- Use `com.opencsv.CSVReader` with `FileReader`

#### `parse()` logic:
1. Read first row as headers; throw `Exception("CSV file is empty: " + file.getName())` if null
2. Build `Map<String, Integer> colIdx` — key is `header.trim().toLowerCase()`, value is column index
3. For each subsequent row, create a `TestCase`:
   - Set `tc.userStory = userStory`
   - Loop over `config.field.entrySet()`:
     - `csvHeader` = entry key, `jiraField` = entry value
     - Look up value using `colIdx.get(csvHeader.trim().toLowerCase())`
     - If value is non-blank, put into `tc.jiraFields` and call `populateNamedField(tc, csvHeader, value)`

#### `listFiles()` logic:
- Return `new File[0]` if folder is null or not a directory
- Filter with: `f.getName().toLowerCase().endsWith(".csv")`

#### Private `populateNamedField(TestCase tc, String csvHeader, String value)`:
- Switch on `csvHeader.trim().toLowerCase()`:
  ```
  "summary"                             → tc.summary
  "priority"                            → tc.priority
  "assignee"                            → tc.assignee
  "reporter"                            → tc.reporter
  "description"                         → tc.description
  "environment"                         → tc.environment
  "labels", "label"                     → tc.labels
  "test case identifier"                → tc.testCaseIdentifier
  "test type"                           → tc.testType
  "outward issue link (test)"           → tc.outwardIssueLink
  "custom field (test repository path)" → tc.testRepositoryPath
  "action"                              → tc.action
  "step no."                            → tc.stepNo
  ```

---

### 5.7 `service/JiraService.java`

- Package: `com.estjira.service`
- Interface with **three methods**:
  ```java
  boolean testConnection();

  /**
   * Creates a Test Execution issue in Jira.
   * Called automatically before importing test cases.
   * Returns the created issue key (e.g. "FCMUS-99"), or null on failure.
   */
  String createTestExecution(String projectKey, String featureLink, String label);

  /**
   * Imports all test cases. Links each to the Test Execution.
   * Returns map of TestCase identifier → "✔ KEY" or "✘ error message".
   */
  Map<String, String> importAll(
      List<TestCase> testCases,
      String projectKey,
      String issueType,
      String testExecutionKey,
      FieldConfig config
  );
  ```

---

### 5.8 `service/JiraServiceImpl.java`

- Package: `com.estjira.service`
- Implements `JiraService`
- **Use `java.net.http.HttpClient` only — NO Apache HttpClient**
- Fields:
  ```java
  private static final String LINK_PREFIX          = "link-";
  private static final String TEST_EXECUTION_TYPE  = "Test Execution";
  private final String      baseUrl;   // trailing slash stripped in constructor
  private final String      auth;      // "Bearer <PAT>"
  private final ObjectMapper json      = new ObjectMapper();
  private final HttpClient   http      = HttpClient.newHttpClient();
  ```
- Constructor: `JiraServiceImpl(String baseUrl, String pat)`
  - `this.auth = "Bearer " + pat;`
  - No username — PAT is self-identifying

#### `testConnection()`:
- GET `/rest/api/2/myself`
- Return `true` if status == 200, `false` on any exception

#### `createTestExecution(projectKey, featureLink, label)`:
- Build `fields` ObjectNode:
  - `project.key` = projectKey
  - `issuetype.name` = `TEST_EXECUTION_TYPE`
  - `summary` = `"Test Execution — " + (label.isBlank() ? projectKey : label)`
  - If label is not blank: add `labels` array with label value
- POST to `/rest/api/2/issue`
- On HTTP 201: parse `key` from response JSON
- If `featureLink` is not blank: call `linkIssues(execKey, featureLink, "relates to")`
- Return exec key or `null` on failure; log warnings on failure

#### `importAll(testCases, projectKey, issueType, testExecutionKey, config)`:
- Return a `LinkedHashMap<String, String>` keyed by `tc.testCaseIdentifier` (fallback: `tc.summary`)
- For each TestCase:
  - Call `buildPayload(tc, projectKey, issueType)`, POST to `/rest/api/2/issue`
  - HTTP 201: parse `key`, put `"✔ " + key`
    - If `testExecutionKey` is not blank: call `linkIssues(testExecutionKey, key, "Test")`
    - Call `handleOutwardLinks(tc, key)` for `link-*` fields
  - Other status: put `"✘ HTTP " + status + " — " + extractError(body)`
  - Exception: put `"✘ " + e.getMessage()`

#### `buildPayload(tc, projectKey, issueType)` (private):
- Build `fields` ObjectNode:
  - `project.key` = projectKey
  - `issuetype.name` = issueType
  - Loop over `tc.jiraFields.entrySet()`:
    - **Skip** entries where key starts with `LINK_PREFIX` (handled by `handleOutwardLinks`)
    - Switch on `fieldId`:
      - `"summary"`     → `fields.put("summary", value)`
      - `"description"` → `fields.put("description", value)`
      - `"priority"`    → `fields.putObject("priority").put("name", value)`
      - `"assignee"`    → `fields.putObject("assignee").put("name", value)`
      - `"reporter"`    → `fields.putObject("reporter").put("name", value)`
      - `"environment"` → `fields.put("environment", value)`
      - `"labels"`      → split on `[,;]`, add each to `fields.putArray("labels")`
      - default         → `fields.put(fieldId, value)` (covers `customfield_*`)
- Fallback: if `fields` has no `summary` or it's blank, use `tc.summary` or `tc.testCaseIdentifier`
- Return `json.writeValueAsString(root)` where root wraps `fields`

#### `addTestToExecution(execKey, testKey)` (private) — **Xray REST API**:
- POST to `/rest/raven/1.0/api/testexecution/{execKey}/test`
- Body: `{ "add": ["testKey"] }`
- Log warning if status is not 200 or 201

#### `handleOutwardLinks(tc, createdKey, config)` (private):
- Loop `tc.jiraFields.entrySet()` where key starts with `LINK_PREFIX`
- For each non-blank value: POST to `/rest/api/2/issueLink` with `type.name = "Test"`, `inwardIssue.key = createdKey`, `outwardIssue.key = linkedKey`
- Log warning if status is not 201 or 204

#### HTTP helper methods (private):
```java
HttpResponse<String> send(HttpRequest req)    // http.send(req, BodyHandlers.ofString())
HttpRequest get(String path)                  // base(path).GET().build()
HttpRequest post(String path, String body)    // base(path).POST(...).header("Content-Type","application/json").build()
HttpRequest.Builder base(String path)         // newBuilder(URI(baseUrl+path)).header("Authorization", auth).header("Accept","application/json")
String extractError(String body)              // parse errorMessages[0] or errors from Jira error JSON
```

---

### 5.9 `ui/MainFrame.java`

- Package: `com.estjira.ui`
- Extends `JFrame`
- Title: `"Jira Test Case Importer — estjira"`
- `setDefaultCloseOperation(EXIT_ON_CLOSE)`
- Root panel: `BorderLayout(8, 8)` with `EmptyBorder(12, 12, 12, 12)`

#### Fields:
```java
// Services
CsvService    csvService    = new CsvServiceImpl();
ConfigService configService = new ConfigServiceImpl();
JiraService   jiraService;   // built lazily

// Jira Settings
JTextField     tfUrl         = new JTextField("https://estjira.example.com", 30);
JPasswordField tfToken       = new JPasswordField(24);  // Personal Access Token — no username needed
JTextField     tfFeatureLink = new JTextField(14);       // User Story to link Test Execution to
JTextField     tfLabel       = new JTextField(14);       // Label for the Test Execution

// File Selection
JTextField      tfCsvFolder  = new JTextField(26);  // non-editable
JComboBox<File> cbCsvFiles   = new JComboBox<>();
JTextField      tfConfigFile = new JTextField(26);  // non-editable

// Preview Table
DefaultTableModel tableModel;  // columns: "Identifier","Summary","Priority","Assignee","Reporter","Environment","Test Type","Labels"
                               // isCellEditable always returns false

// Log
JTextArea logArea = new JTextArea(7, 70);  // non-editable, monospaced 11pt

// State
List<TestCase> loadedTestCases;
FieldConfig    loadedConfig;
```

#### Layout — 3 BorderLayout regions:

**`NORTH`** — `GridLayout(2, 1, 6, 6)` containing:

1. **`buildJiraSettingsPanel()`** — `TitledBorder("Jira Settings")`, `GridBagLayout`:
   - Row 0: `label("Jira URL:")`, `tfUrl` (gridwidth=2), button `"Test Connection"` → `onTestConnection()`
   - Row 1: `label("Personal Access Token:")`, `tfToken` (gridwidth=2, full row)
   - Row 2: `label("Feature Link:")`, `tfFeatureLink`, `label("Label:")`, `tfLabel`

2. **`buildFileSelectionPanel()`** — `TitledBorder("File Selection")`, `GridBagLayout`:
   - Row 0: `label("CSV Folder:")`, `tfCsvFolder` (gridwidth=2), button `"Browse…"` → `onBrowseCsvFolder()`
   - Row 1: `label("CSV File:")`, `cbCsvFiles` (gridwidth=2), button `"Preview"` → `onPreview()`
   - Row 2: `label("Config File:")`, `tfConfigFile` (gridwidth=2), button `"Browse…"` → `onBrowseConfigFile()`
   - `cbCsvFiles` renderer: show only `file.getName()`

**`CENTER`** — `TitledBorder("Test Cases Preview")`, `BorderLayout`, contains `JScrollPane(new JTable(tableModel))`:
- `table.setFillsViewportHeight(true)`
- `table.setRowHeight(22)`
- `table.getTableHeader().setReorderingAllowed(false)`

**`SOUTH`** — `BorderLayout(4, 6)`:
- NORTH: `FlowLayout(CENTER)` with button `"▶  Import to Jira"` (Bold, 13pt) → `onImport()`
- CENTER: `TitledBorder("Log")` panel containing `JScrollPane(logArea)`

After `pack()`: call `setMinimumSize(getSize())`, then `setLocationRelativeTo(null)`.

#### Action methods:

**`onBrowseCsvFolder()`:**
- `JFileChooser` in `DIRECTORIES_ONLY` mode
- On approve: set `tfCsvFolder`, clear and repopulate `cbCsvFiles` via `csvService.listFiles(folder)`
- Log: `"Found N CSV file(s) in folder."`

**`onBrowseConfigFile()`:**
- `JFileChooser` with `FileNameExtensionFilter("JSON Config File (*.json)", "json")`
- On approve: set `tfConfigFile`, call `configService.load(configFile)`, store in `loadedConfig`
- Log: `"Config loaded: N field mapping(s) found."`
- Auto-fill `tfProjectKey` from `loadedConfig.resolvedProjectKey()` if `tfProjectKey` is blank
- On error: show error dialog, set `loadedConfig = null`

**`onPreview()`:**
- Validate: show error if `cbCsvFiles.getSelectedItem()` is null or `loadedConfig` is null
- Run `csvService.parse(selectedFile, loadedConfig, tfFeatureLink.getText().trim())` in `SwingWorker`
- On done: store in `loadedTestCases`, call `refreshTable(loadedTestCases)`
- Log: `"Preview: N test case(s) loaded from filename"`

**`onTestConnection()`:**
- Call `validateJiraFields()` — return early if invalid
- Call `buildJiraService()`
- Run `jiraService.testConnection()` in `SwingWorker`
- On `true`: log `"✔ Connection to estjira successful!"` + info dialog `"Connected to estjira successfully."`
- On `false`: log `"✘ Connection failed — check URL and credentials."` + error dialog

**`onImport()`:**
- Validate: `loadedTestCases` not null/empty, `loadedConfig` not null, `validateJiraFields()` passes
- Read `projectKey` from `loadedConfig.resolvedProjectKey()` — **no text field for this**
- If `projectKey` is blank: show error `"Project key not found in config file. Check that 'projectKey.project' is set."` and return
- Show `YES_NO` confirm dialog:
  ```
  "This will:
    1. Create a Test Execution in project 'KEY'
    2. Import N test case(s)
    3. Link them to the Test Execution

  Proceed?"
  ```
- Call `buildJiraService()`
- Use `SwingWorker<Void, String>` with `publish()` for real-time log updates:
  1. Publish `"Creating Test Execution in project 'KEY'…"`
  2. Call `jiraService.createTestExecution(projectKey, featureLink, label)`
  3. Publish `"✔ Test Execution created: KEY"` or `"⚠ Test Execution could not be created — import will continue without linking."`
  4. Publish `"Importing N test case(s)…"`
  5. Call `jiraService.importAll(loadedTestCases, projectKey, "Test", execKey, loadedConfig)` — issue type is hardcoded to `"Test"`
  6. Publish each result as `"  ID  →  result"`
  7. Count successes (`startsWith("✔")`), publish `"────────────────────────────────────────"`
  8. Publish `"Done: N/total imported successfully."`
- Override `process(List<String> chunks)` to call `log()` for each chunk

#### Private helpers:

```java
void buildJiraService()
// new JiraServiceImpl(tfUrl.getText().trim(), new String(tfToken.getPassword()))
// PAT only — no username

boolean validateJiraFields()
// Check tfUrl and tfToken are not blank; show error dialog and return false if missing
// "Personal Access Token is required." — no username check needed

void refreshTable(List<TestCase> list)
// tableModel.setRowCount(0); add one row per tc: {identifier, summary, priority, assignee, reporter, environment, testType, labels}

void log(String msg)
// SwingUtilities.invokeLater → logArea.append(msg + "\n") + scroll to bottom

void showError(String msg)
// JOptionPane.showMessageDialog(this, msg, "Error", JOptionPane.ERROR_MESSAGE)

void addRow(JPanel panel, GridBagConstraints c, int row, String labelText, JComponent field, int extraWidth)
// c.gridx=0, c.gridy=row, gridwidth=1 → add label
// c.gridx=1, gridwidth=1+extraWidth → add field

JLabel label(String text)
// return new JLabel(text)

GridBagConstraints defaultGbc()
// insets(5,8,5,8), anchor=WEST, fill=HORIZONTAL

<T extends JPanel> T titled(String title, T panel)
// panel.setBorder(new TitledBorder(title)); return panel

DefaultListCellRenderer fileNameRenderer()
// Returns renderer that calls setText(f.getName()) when value instanceof File
```

---

### 5.10 `Main.java`

- Package: `com.estjira`
- `UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName())` in try/catch
- `SwingUtilities.invokeLater(() -> new MainFrame().setVisible(true))`

---

## 6. `sample-config.json` (project root)

Create this file exactly as shown — it mirrors the Xray bulk upload importer format from the image:

```json
{
    "projectKey": {
        "project": "FCMUS"
    },
    "field": {
        "Assignee"                              : "assignee",
        "Action"                                : "Action",
        "Test Case Identifier"                  : "Test Case Identifier",
        "Description"                           : "description",
        "Priority"                              : "priority",
        "Reporter"                              : "reporter",
        "Outward issue link (Test)"             : "link-10900",
        "Labels"                                : "labels",
        "Test type"                             : "customfield_25902",
        "Step no."                              : "Step Attachments",
        "Environment"                           : "environment",
        "Summary"                               : "summary",
        "Custom field (Test Repository Path)"   : "customfield_17400"
    },
    "link": {},
    "projectSelection": {
        "project": "true"
    },
    "projectName": {},
    "value": {}
}
```

---

## 7. `sample-testcases.csv` (project root)

```csv
Test Case Identifier,Summary,Description,Action,Priority,Assignee,Reporter,Environment,Outward issue link (Test),Labels,Test type,Step no.,Custom field (Test Repository Path)
TC-001,Verify login with valid credentials,User should be able to log in,Login,High,john.doe,jane.smith,QA,FCMUS-10,smoke,Functional,1,/Authentication/Login
TC-002,Verify login with invalid password,Login should fail for wrong password,Login,High,john.doe,jane.smith,QA,FCMUS-10,regression,Functional,1,/Authentication/Login
TC-003,Verify password reset email,User receives reset email,Password Reset,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,1,/Authentication/Password
TC-004,Verify user profile update,User can update display name,Profile,Low,john.doe,jane.smith,QA,FCMUS-12,smoke,Functional,1,/Profile/Update
```

---

## 8. Build & Run

```bash
mvn clean package
java -jar target/jira-testcase-importer.jar
```

---

## 9. Creating the ZIP

After generating all files in the folder `jira-testcase-importer/`, create a ZIP archive using one of the following:

**Linux / macOS:**
```bash
zip -r jira-testcase-importer.zip jira-testcase-importer/
```

**Windows (PowerShell):**
```powershell
Compress-Archive -Path jira-testcase-importer -DestinationPath jira-testcase-importer.zip
```

The resulting `jira-testcase-importer.zip` should contain the full project tree as shown in Section 3.

---

## 10. Critical Rules for Copilot

| # | Rule |
|---|------|
| 1 | Use **`java.net.http.HttpClient`** only — never Apache HttpClient |
| 2 | **No getters/setters** on `TestCase` or `FieldConfig` — use public fields directly |
| 3 | **No `service/impl/` sub-package** — all 6 service files live in `com.estjira.service` |
| 4 | **No `ImportResult` class** — use `Map<String, String>` for import results |
| 5 | `link-*` fields in `jiraFields` must be skipped in `buildPayload()` and handled via `handleOutwardLinks()` instead |
| 6 | `customfield_25902` = Test type, `customfield_17400` = Test Repository Path — pass these as plain string fields in the Jira payload |
| 7 | `createTestExecution()` must be called **before** `importAll()` every time the user clicks Import |
| 8 | All Jira API calls must target `/rest/api/2/` (REST API v2) |
| 9 | Authentication is HTTP Basic: `Base64(username:apiToken)` |
| 10 | All network calls must run inside `SwingWorker` — never block the Event Dispatch Thread |
| 11 | `cbCsvFiles` dropdown must display only `file.getName()`, not the full path |
| 12 | Call `setMinimumSize(getSize())` after `pack()` to prevent the window shrinking below its natural size |
| 13 | `SwingWorker.process()` must be used in `onImport()` for real-time log streaming |
| 14 | The config file browse must use `FileNameExtensionFilter` to show only `.json` files |
| 15 | Indent all Java code with **4 spaces** consistently throughout |
| 16 | **No `cbIssueType` combo box** — issue type is always `"Test"` (hardcoded). Do not add any UI element for selecting the issue type |
| 17 | **No `tfProjectKey` text field** — project key is read directly from `loadedConfig.resolvedProjectKey()` at import time. The config JSON already contains it under `projectKey.project` |
| 18 | **No username field** — PAT authentication is self-identifying. Auth header must be `Authorization: Bearer <token>`, never `Basic base64(user:token)` |
| 19 | **Use Xray REST API to add tests to execution** — `POST /rest/raven/1.0/api/testexecution/{execKey}/test` with body `{"add":["TEST-KEY"]}`. Do NOT use the generic Jira issueLink API for this |
