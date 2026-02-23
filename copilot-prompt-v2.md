# GitHub Copilot Prompt — Jira Test Case Importer (estjira) v3

> **Instructions for Copilot:** Generate the complete Maven Java 17 project described below, file by file, exactly as specified. After generating all files, create a ZIP archive of the project folder named `jira-testcase-importer.zip`.

---

## 1. Project Overview

A Java 17 Swing desktop application that:

1. Reads an **"xray bulk upload importer" JSON config file** to get CSV column → Jira field mappings
2. Reads **CSV files** where rows sharing the same **Test Case Identifier** belong to one test case with multiple steps
3. On import:
   - **Creates a Test Execution** issue in Jira (Xray), with a user-entered title and optional label
   - Links it to a specified User Story
   - **Imports all grouped test cases** as Jira `Test` issues, with steps formatted in the description
   - Adds each imported test to the Test Execution via the Xray REST API

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
    └── service/
        ├── ConfigService.java          ← interface
        ├── ConfigServiceImpl.java
        ├── CsvService.java             ← interface
        ├── CsvServiceImpl.java
        ├── JiraService.java            ← interface
        ├── JiraServiceImpl.java
        └── ui/
            └── MainFrame.java
```

> ⚠️ **No** `service/impl/` sub-package. All classes live directly in `com.estjira.service` (or `com.estjira.ui`).

---

## 4. `pom.xml`

- Java source/target: **17**, encoding: UTF-8
- **Only 2 dependencies:**
  - `com.opencsv:opencsv:5.9`
  - `com.fasterxml.jackson.core:jackson-databind:2.17.1`
- **maven-shade-plugin 3.5.2** fat JAR:
  - `<finalName>jira-testcase-importer</finalName>`
  - `<mainClass>com.estjira.Main</mainClass>`
  - Exclude `META-INF/*.SF`, `META-INF/*.DSA`, `META-INF/*.RSA`

---

## 5. File-by-File Specifications

---

### 5.1 `model/FieldConfig.java`

- Package: `com.estjira.model`
- Annotate with `@JsonIgnoreProperties(ignoreUnknown = true)`
- **Public fields** (no getters/setters), all `LinkedHashMap`:
  ```java
  Map<String, String> projectKey       // { "project": "FCMUS" }
  Map<String, String> field            // CSV column header → Jira field ID
  Map<String, String> link
  Map<String, String> projectSelection
  Map<String, String> projectName
  Map<String, String> value
  ```
- Convenience method:
  ```java
  public String resolvedProjectKey() {
      return projectKey.getOrDefault("project", "");
  }
  ```

---

### 5.2 `model/TestCase.java`

- Package: `com.estjira.model`
- **All public fields, no getters/setters:**

```java
// Identity & metadata — populated from the FIRST row of the group
String testCaseIdentifier;
String summary;
String description;
String priority;
String assignee;
String reporter;
String environment;
String testType;            // from customfield_25902
String testRepositoryPath;  // from customfield_17400
String labels;
String outwardIssueLink;    // from link-10900
String userStory;

// Steps — one entry per CSV row in the group
// String[2]: [0] = step number, [1] = action text
List<String[]> steps = new ArrayList<>();

// Dynamic Jira field payload — built from first row only
// Step-only fields (Action, Step no.) are NEVER put here
Map<String, String> jiraFields = new LinkedHashMap<>();
```

- Class Javadoc: "Rows sharing the same testCaseIdentifier are grouped into one TestCase. Metadata comes from the first row. Every row contributes one entry to steps."

---

### 5.3 `service/ConfigService.java`

Interface with one method:
```java
FieldConfig load(File configFile) throws Exception;
```

---

### 5.4 `service/ConfigServiceImpl.java`

- Uses `new ObjectMapper().readValue(configFile, FieldConfig.class)`
- Throws if file is null/missing or `config.field` is null/empty

---

### 5.5 `service/CsvService.java`

Interface:
```java
List<TestCase> parse(File file, FieldConfig config, String userStory) throws Exception;
File[] listFiles(File folder);
```

---

### 5.6 `service/CsvServiceImpl.java` ← CRITICAL — READ CAREFULLY

**Grouping rule:** Multiple CSV rows with the same value in the "Test Case Identifier" column belong to a single test case. They must be **grouped into one `TestCase` object**.

#### `parse()` algorithm:

```
grouped = new LinkedHashMap<String, TestCase>()   // preserves CSV order

read header row → build colIdx Map<String, Integer> (case-insensitive)

for each data row:
    id = resolveIdentifier(row, colIdx, config)
    if id is blank → skip row

    tc = grouped.get(id)
    isFirstRow = (tc == null)

    if isFirstRow:
        tc = new TestCase()
        tc.testCaseIdentifier = id
        tc.userStory = userStory
        grouped.put(id, tc)

    stepNo = ""
    action = ""

    for each (csvHeader, jiraField) in config.field:
        value = getCsvValue(row, colIdx, csvHeader)
        if value is blank → skip

        headerLc = csvHeader.trim().toLowerCase()

        if headerLc == "step no.":
            stepNo = value
        else if headerLc == "action":
            action = value
        else if isFirstRow:                     ← metadata ONLY from first row
            tc.jiraFields.put(jiraField, value)
            populateNamedField(tc, csvHeader, value)

    if stepNo or action is not blank:
        tc.steps.add(new String[]{ stepNo, action })   ← EVERY row adds a step

return new ArrayList<>(grouped.values())
```

#### `resolveIdentifier()` (private):
- Loop `config.field.entrySet()` to find the entry whose key equalsIgnoreCase `"Test Case Identifier"`
- Return `getCsvValue(row, colIdx, that key)`

#### `listFiles()`:
- Return `new File[0]` if folder is null or not a directory
- Filter: `f.getName().toLowerCase().endsWith(".csv")`

#### `populateNamedField()` switch on `csvHeader.trim().toLowerCase()`:
```
"summary"                              → tc.summary
"priority"                             → tc.priority
"assignee"                             → tc.assignee
"reporter"                             → tc.reporter
"description"                          → tc.description
"environment"                          → tc.environment
"labels", "label"                      → tc.labels
"test case identifier"                 → { /* already set, skip */ }
"test type"                            → tc.testType
"outward issue link (test)"            → tc.outwardIssueLink
"custom field (test repository path)"  → tc.testRepositoryPath
"step no.", "action"                   → { /* handled as step, never here */ }
```

---

### 5.7 `service/JiraService.java`

```java
boolean testConnection();

String createTestExecution(String projectKey, String userStory,
                           String executionName, String label);

Map<String, String> importAll(List<TestCase> testCases, String projectKey,
                              String issueType, String testExecutionKey,
                              FieldConfig config);
```

---

### 5.8 `service/JiraServiceImpl.java`

- **Authentication: `Authorization: Bearer <PAT>` — NO username, NO Basic auth**
- Constructor: `JiraServiceImpl(String baseUrl, String pat)`
  - Strip trailing slash from baseUrl
  - `this.auth = "Bearer " + pat`
- Use **`java.net.http.HttpClient`** only — no Apache HttpClient
- Constants:
  ```java
  private static final String LINK_PREFIX        = "link-";
  private static final String TEST_EXECUTION_TYPE = "Test Execution";
  ```

#### `testConnection()`:
- GET `/rest/api/2/myself` → return `statusCode == 200`

#### `createTestExecution(projectKey, userStory, executionName, label)`:
- Build fields: `project.key`, `issuetype.name = TEST_EXECUTION_TYPE`
- `summary = executionName` ← user-entered title, shown on Jira board
- If `label` not blank: add to `labels[]` array ← separate from title
- POST `/rest/api/2/issue`; parse `key` from 201 response
- If `userStory` not blank: POST `/rest/api/2/issueLink` with:
  - `type.name = "relates to"`, `inwardIssue.key = execKey`, `outwardIssue.key = userStory`
- Return execKey or null on failure

#### `importAll(testCases, projectKey, issueType, testExecutionKey, config)`:
- Return `LinkedHashMap<String, String>` keyed by `tc.testCaseIdentifier` (fallback `tc.summary`)
- For each TestCase:
  - `buildPayload(tc, projectKey, issueType)` → POST `/rest/api/2/issue`
  - HTTP 201: parse key, put `"✔ KEY"`
    - call `addTestToExecution(testExecutionKey, key)` if execKey not blank
    - call `handleOutwardLinks(tc, key)`
  - else: put `"✘ HTTP N — error"`
  - catch Exception: put `"✘ message"`

#### `buildPayload(tc, projectKey, issueType)` (private):

```
fields = new ObjectNode
fields.project.key = projectKey
fields.issuetype.name = issueType

for (fieldId, value) in tc.jiraFields:
    if fieldId starts with "link-" → skip (handled by handleOutwardLinks)
    switch fieldId:
        "summary"     → fields.put("summary", value)
        "description" → fields.put("description", value)
        "priority"    → fields.priority.name = value
        "assignee"    → fields.assignee.name = value
        "reporter"    → fields.reporter.name = value
        "environment" → fields.put("environment", value)
        "labels"      → split on [,;], add each to labels[]
        default       → fields.put(fieldId, value)   // customfield_* etc.

// Fallback summary
if fields has no "summary" or it is blank:
    fields.put("summary", tc.summary ?? tc.testCaseIdentifier ?? "Test Case")

// ── FORMAT STEPS INTO DESCRIPTION ───────────────────────────────────
// This is MANDATORY — steps must appear in the Jira issue description
if tc.steps is not empty:
    existingDesc = fields["description"].asText() or ""
    sb = new StringBuilder()
    if existingDesc not blank:
        sb.append(existingDesc).append("\n\n")
    sb.append("*Test Steps:*\n")
    for i, step in enumerate(tc.steps):
        stepNo = step[0] if not blank else String.valueOf(i + 1)
        action = step[1] if not blank else ""
        sb.append(stepNo).append(". ").append(action).append("\n")
    fields.put("description", sb.toString().stripTrailing())

return json.writeValueAsString({ "fields": fields })
```

#### `addTestToExecution(execKey, testKey)` (private) — Xray REST API:
- POST `/rest/raven/1.0/api/testexecution/{execKey}/test`
- Body: `{ "add": ["testKey"] }`
- Log warning if status != 200 and != 201

#### `handleOutwardLinks(tc, createdKey)` (private):
- Loop `tc.jiraFields` entries where key starts with `LINK_PREFIX`
- For each non-blank value: POST `/rest/api/2/issueLink`
  - `type.name = "Test"`, `inwardIssue.key = createdKey`, `outwardIssue.key = linkedValue`

#### HTTP helpers (private):
```java
HttpResponse<String> send(HttpRequest req)
HttpRequest get(String path)
HttpRequest post(String path, String body)   // Content-Type: application/json
HttpRequest.Builder base(String path)        // Authorization: auth, Accept: application/json
String extractError(String body)             // parse errorMessages[0] or errors node
```

---

### 5.9 `ui/MainFrame.java`

- Package: `com.estjira.ui`
- Title: `"Jira Test Case Importer — estjira"`

#### Fields:
```java
// Services
CsvService    csvService    = new CsvServiceImpl();
ConfigService configService = new ConfigServiceImpl();
JiraService   jiraService;  // built lazily in buildJiraService()

// Jira Settings  (* = mandatory)
JTextField     tfUrl          = new JTextField("https://estjira.example.com", 30);  // *
JPasswordField tfToken        = new JPasswordField(24);   // * Personal Access Token
JTextField     tfUserStory    = new JTextField(14);       // * Linked User Story key
JTextField     tfExecutionName = new JTextField(28);      // * Test Execution title
JTextField     tfLabel        = new JTextField(14);       // optional tag

// File Selection  (* = mandatory)
JTextField      tfCsvFolder  = new JTextField(26);    // * non-editable
JComboBox<File> cbCsvFiles   = new JComboBox<>();     // * filename-only renderer
JTextField      tfConfigFile = new JTextField(26);    // * non-editable

// Preview table — one row per grouped TestCase (NOT per CSV row)
DefaultTableModel tableModel;
// columns: "Identifier", "Summary", "Priority", "Assignee", "Steps", "Environment", "Test Type", "Labels"
// isCellEditable always returns false

// Log
JTextArea logArea = new JTextArea(7, 70);  // monospaced 11pt, non-editable

// State
List<TestCase> loadedTestCases;   // grouped test cases
FieldConfig    loadedConfig;
```

#### Layout:

**NORTH** — `GridLayout(2,1,6,6)`:

1. **Jira Settings** (`TitledBorder`, `GridBagLayout`):
   - Row 0: `"* Jira URL:"` + `tfUrl` (gridwidth=2) + `[Test Connection]` button
   - Row 1: `"* Personal Access Token:"` + `tfToken` (gridwidth=2, full row)
   - Row 2: `"* Linked User Story:"` + `tfUserStory` + `"Label:"` + `tfLabel`
   - Row 3: `"* Execution Name:"` + `tfExecutionName` (gridwidth=2, full row)

2. **File Selection** (`TitledBorder`, `GridBagLayout`):
   - Row 0: `"* CSV Folder:"` + `tfCsvFolder` (gridwidth=2) + `[Browse…]`
   - Row 1: `"* CSV File:"` + `cbCsvFiles` (gridwidth=2) + `[Preview]`
   - Row 2: `"* Config File:"` + `tfConfigFile` (gridwidth=2) + `[Browse…]`

**CENTER** — `TitledBorder("Test Cases Preview")`:
- `JScrollPane` wrapping `JTable(tableModel)`
- `fillsViewportHeight=true`, `rowHeight=22`, header reordering disabled

**SOUTH** — `BorderLayout(4,6)`:
- NORTH: `[▶  Import to Jira]` button — Bold 13pt
- CENTER: `TitledBorder("Log")` + `JScrollPane(logArea)`

After `pack()`: `setMinimumSize(getSize())` then `setLocationRelativeTo(null)`

#### Actions:

**`onBrowseCsvFolder()`**: directory chooser → populate `cbCsvFiles` via `csvService.listFiles()`

**`onBrowseConfigFile()`**: JSON file chooser → `configService.load()` → store in `loadedConfig`
- Auto-fill nothing (projectKey comes from config at import time)
- Log: `"Config loaded: N field mapping(s) found."`
- On error: show error dialog, `loadedConfig = null`

**`onPreview()`**:
- Validate: csv file selected, config loaded
- `SwingWorker`: `csvService.parse(file, loadedConfig, tfUserStory.getText().trim())`
- On done: store in `loadedTestCases`, call `refreshTable(loadedTestCases)`
- Log: `"Preview: N test case(s) from M CSV rows"` — N = grouped count, M = sum of all steps

**`onTestConnection()`**:
- `validateJiraFields()` → `buildJiraService()` → `jiraService.testConnection()` in SwingWorker
- Success: info dialog `"Connected to estjira successfully."`
- Failure: error dialog `"Could not connect. Verify Jira URL and Personal Access Token."`

**`onImport()`**:
1. Check `loadedTestCases` not null/empty, `loadedConfig` not null, `validateJiraFields()` passes
2. `projectKey = loadedConfig.resolvedProjectKey()` — if blank show error and return
3. `executionName = tfExecutionName.getText().trim()` — if blank show error and return
4. Show YES/NO confirm:
   ```
   This will:
     1. Create Test Execution: 'EXECUTION NAME'
     2. Link it to User Story: FCMUS-42  (or "(none)" if blank)
     3. Import N test case(s) into it
   Proceed?
   ```
5. `buildJiraService()`
6. `SwingWorker<Void, String>` with `publish()` for streaming log:
   - Publish: `"Creating Test Execution 'NAME' in project 'KEY'…"`
   - `createTestExecution(projectKey, userStory, executionName, label)`
   - Publish: `"✔ Test Execution created: KEY"` or `"⚠ could not be created…"`
   - Publish: `"Importing N test case(s)…"`
   - `importAll(...)` with `"Test"` hardcoded as issue type
   - Publish each result: `"  ID  →  result"`
   - Publish separator + `"Done: N/total imported successfully."`
7. `process(chunks)` calls `log()` for each chunk

#### Private helpers:

```java
void buildJiraService()
// new JiraServiceImpl(tfUrl.getText().trim(), new String(tfToken.getPassword()))

boolean validateJiraFields()
// tfUrl blank → "* Jira URL is required."
// tfToken empty → "* Personal Access Token is required."

void refreshTable(List<TestCase> list)
// tableModel.setRowCount(0)
// for each tc: add row {identifier, summary, priority, assignee,
//              tc.steps.size() + " step(s)", environment, testType, labels}

void log(String msg)
// SwingUtilities.invokeLater → logArea.append(msg+"\n") + scroll to bottom

void showError(String msg)
// JOptionPane.showMessageDialog ERROR_MESSAGE

void addRow(JPanel, GridBagConstraints, int row, String label, JComponent field, int extraWidth)
GridBagConstraints defaultGbc()   // insets(5,8,5,8), WEST, HORIZONTAL
<T extends JPanel> T titled(String, T)
DefaultListCellRenderer fileNameRenderer()  // shows f.getName() for File items
```

---

### 5.10 `Main.java`

```java
UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());  // in try/catch
SwingUtilities.invokeLater(() -> new MainFrame().setVisible(true));
```

---

## 6. `sample-config.json`

```json
{
    "projectKey": { "project": "FCMUS" },
    "field": {
        "Test Case Identifier"                  : "Test Case Identifier",
        "Summary"                               : "summary",
        "Description"                           : "description",
        "Action"                                : "Action",
        "Priority"                              : "priority",
        "Assignee"                              : "assignee",
        "Reporter"                              : "reporter",
        "Environment"                           : "environment",
        "Outward issue link (Test)"             : "link-10900",
        "Labels"                                : "labels",
        "Test type"                             : "customfield_25902",
        "Step no."                              : "Step Attachments",
        "Custom field (Test Repository Path)"   : "customfield_17400"
    },
    "link": {},
    "projectSelection": { "project": "true" },
    "projectName": {},
    "value": {}
}
```

---

## 7. `sample-testcases.csv`

Demonstrates 4 test cases with 3–4 steps each. Rows with the same identifier are grouped:

```csv
Test Case Identifier,Summary,Description,Action,Priority,Assignee,Reporter,Environment,Outward issue link (Test),Labels,Test type,Step no.,Custom field (Test Repository Path)
1,Verify login with valid credentials,User should be able to log in,Navigate to login page,High,john.doe,jane.smith,QA,FCMUS-10,smoke,Functional,1,/Authentication/Login
1,Verify login with valid credentials,User should be able to log in,Enter valid username and password,High,john.doe,jane.smith,QA,FCMUS-10,smoke,Functional,2,/Authentication/Login
1,Verify login with valid credentials,User should be able to log in,Click the Login button,High,john.doe,jane.smith,QA,FCMUS-10,smoke,Functional,3,/Authentication/Login
2,Verify login with invalid password,Login should fail for wrong password,Navigate to login page,High,john.doe,jane.smith,QA,FCMUS-10,regression,Functional,1,/Authentication/Login
2,Verify login with invalid password,Login should fail for wrong password,Enter valid username and wrong password,High,john.doe,jane.smith,QA,FCMUS-10,regression,Functional,2,/Authentication/Login
2,Verify login with invalid password,Login should fail for wrong password,Click the Login button and verify error,High,john.doe,jane.smith,QA,FCMUS-10,regression,Functional,3,/Authentication/Login
3,Verify password reset email,User receives reset email,Navigate to login page,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,1,/Authentication/Password
3,Verify password reset email,User receives reset email,Click Forgot Password,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,2,/Authentication/Password
3,Verify password reset email,User receives reset email,Enter registered email address,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,3,/Authentication/Password
3,Verify password reset email,User receives reset email,Check inbox for reset email,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,4,/Authentication/Password
4,Verify user profile update,User can update display name,Navigate to Profile page,Low,john.doe,jane.smith,QA,FCMUS-12,smoke,Functional,1,/Profile/Update
4,Verify user profile update,User can update display name,Edit the display name field,Low,john.doe,jane.smith,QA,FCMUS-12,smoke,Functional,2,/Profile/Update
4,Verify user profile update,User can update display name,Click Save and verify change,Low,john.doe,jane.smith,QA,FCMUS-12,smoke,Functional,3,/Profile/Update
```

Result after parsing: **4 TestCase objects**, identifiers 1–4, with 3, 3, 4, and 3 steps respectively.

---

## 8. Build & Run

```bash
mvn clean package
java -jar target/jira-testcase-importer.jar
```

---

## 9. Creating the ZIP

```bash
# Linux / macOS
zip -r jira-testcase-importer.zip jira-testcase-importer/

# Windows PowerShell
Compress-Archive -Path jira-testcase-importer -DestinationPath jira-testcase-importer.zip
```

---

## 10. Critical Rules for Copilot

| # | Rule |
|---|------|
| 1 | Use **`java.net.http.HttpClient`** only — never Apache HttpClient |
| 2 | **No getters/setters** on `TestCase` or `FieldConfig` — public fields only |
| 3 | **No `service/impl/` sub-package** — all service files in `com.estjira.service` |
| 4 | **No `ImportResult` class** — use `Map<String, String>` |
| 5 | `link-*` fields in `jiraFields` must be **skipped** in `buildPayload()` and handled in `handleOutwardLinks()` |
| 6 | `customfield_25902` = Test type, `customfield_17400` = Test Repository Path — pass as plain string fields |
| 7 | `createTestExecution()` must be called **before** `importAll()` on every import |
| 8 | All Jira API calls target `/rest/api/2/` (REST API v2) |
| 9 | Authentication: **`Authorization: Bearer <PAT>`** — no username, no Basic auth |
| 10 | All network calls run in `SwingWorker` — never block the EDT |
| 11 | `cbCsvFiles` dropdown renders **filename only** (`file.getName()`) |
| 12 | Call `setMinimumSize(getSize())` after `pack()` |
| 13 | `SwingWorker.process()` used in `onImport()` for real-time log streaming |
| 14 | Config file browser uses `FileNameExtensionFilter` for `.json` files only |
| 15 | Indent all Java with **4 spaces** |
| 16 | **No `cbIssueType`** — issue type is always `"Test"` (hardcoded) |
| 17 | **No `tfProjectKey`** — project key read from `loadedConfig.resolvedProjectKey()` |
| 18 | **No username field** — PAT is self-identifying |
| 19 | Add tests to execution via **Xray API**: `POST /rest/raven/1.0/api/testexecution/{key}/test` with `{"add":["KEY"]}` |
| 20 | Mandatory field labels start with `"* "` — Jira URL, PAT, Linked User Story, Execution Name, CSV Folder, CSV File, Config File |
| 21 | `tfExecutionName` = **Test Execution title** (summary on Jira board). `tfLabel` = **optional tag**. Never merge them |
| 22 | **GROUPING IS MANDATORY**: rows with same Test Case Identifier = one TestCase. Metadata from first row only. Every row adds one step to `tc.steps` |
| 23 | **Steps MUST appear in the Jira description**: format as `"*Test Steps:*\n1. action\n2. action\n..."` appended after any existing description text |
| 24 | Preview table shows **one row per grouped TestCase**, with a `"Steps"` column showing `"N step(s)"` — NOT one row per CSV row |
| 25 | `"Step no."` and `"Action"` CSV columns are **never** written to `tc.jiraFields` — they only populate `tc.steps` |
