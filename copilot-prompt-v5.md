# GitHub Copilot Prompt — Jira Test Case Importer (estjira) v5
# COMPLETE PROJECT — Generate All Files From Scratch

> **Instructions for Copilot:** Generate the complete Maven Java 17 project described below,
> file by file, exactly as specified. Do NOT omit any file. After generating all files,
> create a ZIP archive named `jira-testcase-importer.zip`.

---

## 1. Project Overview

A Java 17 Swing desktop application that:

1. Reads an **"xray bulk upload importer" JSON config file** to get CSV column → Jira field mappings.  
   The **project key is extracted automatically** from this config file (`projectKey.project`).
2. Reads **CSV files** where rows sharing the same **Test Case Identifier** form one test case with multiple steps.
3. Each CSV row may include a **`User Story`** column specifying which Jira story to link to.
4. On import, either:
   - **Adds test cases to an EXISTING Test Execution** by entering its Jira ID (e.g. `FCMUS-99`) — **default mode**, or
   - **Creates a NEW Test Execution** by entering a summary title.
5. Imports all grouped test cases as Jira `Test` issues, with steps formatted in the description.
6. Adds each imported test to the Test Execution via the Xray REST API.
7. Links each imported test to its per-row User Story from the CSV.

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
    │   ├── ConfigService.java
    │   ├── ConfigServiceImpl.java
    │   ├── CsvService.java
    │   ├── CsvServiceImpl.java
    │   ├── JiraService.java
    │   └── JiraServiceImpl.java
    └── ui/
        └── MainFrame.java
```

> ⚠️ No `service/impl/` sub-package. All service files live directly in `com.estjira.service`.

---

## 4. `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.estjira</groupId>
    <artifactId>jira-testcase-importer</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.opencsv</groupId>
            <artifactId>opencsv</artifactId>
            <version>5.9</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.17.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.5.2</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <finalName>jira-testcase-importer</finalName>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.estjira.Main</mainClass>
                                </transformer>
                            </transformers>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 5. File-by-File Specifications

---

### 5.1 `Main.java`

```java
package com.estjira;

import com.estjira.ui.MainFrame;
import javax.swing.*;

public class Main {
    public static void main(String[] args) {
        try {
            UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());
        } catch (Exception ignored) {}
        SwingUtilities.invokeLater(() -> new MainFrame().setVisible(true));
    }
}
```

---

### 5.2 `model/FieldConfig.java`

- Package: `com.estjira.model`
- Annotate with `@JsonIgnoreProperties(ignoreUnknown = true)`
- **Public fields only** (no getters/setters), all `LinkedHashMap`:

```java
public Map<String, String> projectKey       = new LinkedHashMap<>();  // { "project": "FCMUS" }
public Map<String, String> field            = new LinkedHashMap<>();  // CSV header → Jira field ID
public Map<String, String> link             = new LinkedHashMap<>();
public Map<String, String> projectSelection = new LinkedHashMap<>();
public Map<String, String> projectName      = new LinkedHashMap<>();
public Map<String, String> value            = new LinkedHashMap<>();
```

Convenience method:
```java
public String resolvedProjectKey() {
    return projectKey.getOrDefault("project", "");
}
```

---

### 5.3 `model/TestCase.java`

- Package: `com.estjira.model`
- **All public fields, no getters/setters**

```java
public String testCaseIdentifier;
public String summary;
public String description;
public String priority;
public String assignee;
public String reporter;
public String environment;
public String testType;            // customfield_25902
public String testRepositoryPath;  // customfield_17400
public String labels;
public String outwardIssueLink;    // link-10900

// Set from the CSV "User Story" column (per test case)
public String userStory;

// One String[2] per CSV row: [0]=stepNo, [1]=action
public List<String[]> steps = new ArrayList<>();

// Dynamic Jira payload — from first row only; step-only fields never here
public Map<String, String> jiraFields = new LinkedHashMap<>();
```

---

### 5.4 `service/ConfigService.java`

```java
package com.estjira.service;
import com.estjira.model.FieldConfig;
import java.io.File;

public interface ConfigService {
    FieldConfig load(File configFile) throws Exception;
}
```

---

### 5.5 `service/ConfigServiceImpl.java`

- Uses `new ObjectMapper().readValue(configFile, FieldConfig.class)`
- Throws if file is null/missing or `config.field` is null/empty

---

### 5.6 `service/CsvService.java`

```java
package com.estjira.service;
import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import java.io.File;
import java.util.List;

public interface CsvService {
    List<TestCase> parse(File file, FieldConfig config, String userStoryFallback) throws Exception;
    File[] listFiles(File folder);
}
```

---

### 5.7 `service/CsvServiceImpl.java` ← CRITICAL

#### `parse()` algorithm:

```
grouped = new LinkedHashMap<String, TestCase>()

read header row → build colIdx Map<String, Integer> (case-insensitive trim)

csvHasUserStoryCol = colIdx.containsKey("user story")

for each data row:
    id = resolveIdentifier(row, colIdx, config)
    if id blank → skip

    tc = grouped.get(id)
    isFirstRow = (tc == null)

    if isFirstRow:
        tc = new TestCase()
        tc.testCaseIdentifier = id

        if csvHasUserStoryCol:
            csvStory = getCsvValueByHeader(row, colIdx, "user story")
            tc.userStory = csvStory.isBlank() ? userStoryFallback : csvStory
        else:
            tc.userStory = userStoryFallback

        grouped.put(id, tc)

    stepNo = ""
    action = ""

    for each (csvHeader, jiraField) in config.field:
        value = getCsvValue(row, colIdx, csvHeader)
        if blank → skip

        headerLc = csvHeader.trim().toLowerCase()
        if headerLc == "step no." → stepNo = value
        else if headerLc == "action" → action = value
        else if isFirstRow:
            tc.jiraFields.put(jiraField, value)
            populateNamedField(tc, csvHeader, value)

    if stepNo or action not blank:
        tc.steps.add(new String[]{ stepNo, action })

return new ArrayList<>(grouped.values())
```

#### `resolveIdentifier()` (private):
Loop `config.field.entrySet()` to find the entry whose key `equalsIgnoreCase("Test Case Identifier")`.
Return `getCsvValue(row, colIdx, thatKey)`.

#### `listFiles()`:
Return `new File[0]` if folder is null or not a directory.
Filter: `f.getName().toLowerCase().endsWith(".csv")`.

#### Private helpers:
```java
// Delegates to getCsvValueByHeader with trimmed lowercase key
private String getCsvValue(String[] row, Map<String, Integer> colIdx, String csvHeader)

// Looks up exact lowercase header name; returns "" if absent
private String getCsvValueByHeader(String[] row, Map<String, Integer> colIdx, String lowerHeader)
```

#### `populateNamedField()` switch on `csvHeader.trim().toLowerCase()`:
```
"summary"                              → tc.summary
"priority"                             → tc.priority
"assignee"                             → tc.assignee
"reporter"                             → tc.reporter
"description"                          → tc.description
"environment"                          → tc.environment
"labels", "label"                      → tc.labels
"test case identifier"                 → { /* already set */ }
"test type"                            → tc.testType
"outward issue link (test)"            → tc.outwardIssueLink
"custom field (test repository path)"  → tc.testRepositoryPath
"step no.", "action"                   → { /* step, handled above */ }
"user story"                           → { /* handled before config loop */ }
```

---

### 5.8 `service/JiraService.java`

```java
package com.estjira.service;
import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import java.util.List;
import java.util.Map;

public interface JiraService {
    boolean testConnection();

    String createTestExecution(String projectKey, String userStory,
                               String executionSummary, String label);

    Map<String, String> importAll(List<TestCase> testCases, String projectKey,
                                  String issueType, String testExecutionKey,
                                  FieldConfig config);
}
```

---

### 5.9 `service/JiraServiceImpl.java`

- **Auth: `Authorization: Bearer <PAT>` — no username, no Basic auth**
- Constructor: `JiraServiceImpl(String baseUrl, String pat)`
  - Strip trailing slash; `this.auth = "Bearer " + pat`
- Use **`java.net.http.HttpClient`** only

```java
private static final String LINK_PREFIX         = "link-";
private static final String TEST_EXECUTION_TYPE = "Test Execution";
```

#### `testConnection()`:
GET `/rest/api/2/myself` → return `statusCode == 200`

#### `createTestExecution(projectKey, userStory, executionSummary, label)`:
- POST `/rest/api/2/issue` with `project.key`, `issuetype.name = TEST_EXECUTION_TYPE`, `summary = executionSummary`
- If `label` not blank: add to `labels[]`
- Parse `key` from 201 response
- If `userStory` not blank: POST `/rest/api/2/issueLink` (`type.name = "relates to"`)
- Return key or null on failure

#### `importAll(testCases, projectKey, issueType, testExecutionKey, config)`:
`LinkedHashMap<String, String>` keyed by identifier (fallback summary).

For each TestCase:
- `buildPayload(tc, projectKey, issueType)` → POST `/rest/api/2/issue`
- 201: parse key → `"✔ KEY"`
  - `addTestToExecution(testExecutionKey, key)` if execKey not blank
  - `handleOutwardLinks(tc, key)`
  - `linkToUserStory(key, tc.userStory)` if `tc.userStory` not blank
- else: `"✘ HTTP N — error"`
- exception: `"✘ message"`

#### `buildPayload(tc, projectKey, issueType)` (private):
```
fields.project.key = projectKey
fields.issuetype.name = issueType

for (fieldId, value) in tc.jiraFields:
    skip if fieldId starts with "link-"
    switch fieldId:
        "summary"     → fields.put("summary", value)
        "description" → fields.put("description", value)
        "priority"    → fields.priority.name = value
        "assignee"    → fields.assignee.name = value
        "reporter"    → fields.reporter.name = value
        "environment" → fields.put("environment", value)
        "labels"      → split on [,;] → labels[]
        default       → fields.put(fieldId, value)

// Fallback summary
if no summary or blank:
    fields.put("summary", tc.summary ?? tc.testCaseIdentifier ?? "Test Case")

// Append steps to description — MANDATORY
if tc.steps not empty:
    existing = fields["description"] or ""
    sb: if existing not blank → append + "\n\n"
    sb.append("*Test Steps:*\n")
    for i, step: stepNo = step[0] or (i+1); action = step[1] or ""
        sb.append(stepNo + ". " + action + "\n")
    fields.put("description", sb.stripTrailing())
```

#### `addTestToExecution(execKey, testKey)` (private):
POST `/rest/raven/1.0/api/testexecution/{execKey}/test` — body: `{ "add": ["testKey"] }`

#### `linkToUserStory(testKey, userStoryKey)` (private):
POST `/rest/api/2/issueLink` — `type.name = "relates to"`, `inwardIssue.key = testKey`, `outwardIssue.key = userStoryKey`

#### `handleOutwardLinks(tc, createdKey)` (private):
Loop `tc.jiraFields` where key starts with `LINK_PREFIX` → POST `/rest/api/2/issueLink` (`type.name = "Test"`)

#### HTTP helpers (private):
```java
HttpResponse<String> send(HttpRequest req)
HttpRequest get(String path)
HttpRequest post(String path, String body)
HttpRequest.Builder base(String path)   // Authorization + Accept headers
String extractError(String body)
```

---

### 5.10 `ui/MainFrame.java` ← CRITICAL — READ EVERY LINE

#### Package & title:
```java
package com.estjira.ui;
// title: "Jira Test Case Importer — estjira"
```

#### CardLayout card-name constants:
```java
private static final String CARD_EXISTING = "EXISTING";  // shown when checkbox IS checked (default)
private static final String CARD_NEW      = "NEW";        // shown when checkbox is NOT checked
```

#### All fields:
```java
// Services
CsvService    csvService    = new CsvServiceImpl();
ConfigService configService = new ConfigServiceImpl();
JiraService   jiraService;

// Jira connection
JTextField     tfUrl   = new JTextField("https://estjira.example.com", 30);  // *
JPasswordField tfToken = new JPasswordField(24);                             // * PAT

// ── Execution mode ──────────────────────────────────────────────────────────
// Checkbox: checked by default → shows CARD_EXISTING
JCheckBox  cbExistingTestExecution = new JCheckBox("Existing Test Execution", true);
CardLayout executionCardLayout     = new CardLayout();
JPanel     executionCardPanel      = new JPanel(executionCardLayout);

// CARD_EXISTING field  (default — checkbox is checked on startup)
JTextField tfExistingExecutionId = new JTextField(28);  // * e.g. FCMUS-99

// CARD_NEW field  (shown when checkbox is unchecked)
JTextField tfExecutionSummary = new JTextField(28);     // * title for the new execution
// ───────────────────────────────────────────────────────────────────────────

// File selection
JTextField      tfCsvFolder  = new JTextField(26);   // non-editable
JComboBox<File> cbCsvFiles   = new JComboBox<>();    // filename-only renderer
JTextField      tfConfigFile = new JTextField(26);   // non-editable

// Preview table — 9 columns
DefaultTableModel tableModel;
// columns: "Identifier","Summary","Priority","Assignee","Steps",
//          "Environment","Test Type","Labels","User Story"
// isCellEditable always returns false

// Log
JTextArea logArea = new JTextArea(7, 70);   // monospaced 11pt, non-editable

// State
List<TestCase> loadedTestCases;
FieldConfig    loadedConfig;
```

#### Overall layout (root `BorderLayout`):
- **NORTH** — `GridLayout(2,1,6,6)`: Jira Settings panel, then File Selection panel
- **CENTER** — Test Cases Preview panel
- **SOUTH** — Import button + Log panel

#### `buildJiraSettingsPanel()` (`TitledBorder("Jira Settings")`, `GridBagLayout`):

```
Row 0: "* Jira URL:"              [tfUrl — gridwidth=2]           [Test Connection btn]
Row 1: "* Personal Access Token:" [tfToken — gridwidth=2]
Row 2: [cbExistingTestExecution — gridwidth=4, full row]
Row 3: [executionCardPanel — gridwidth=4, full row]
```

`cbExistingTestExecution` tooltip:
> "Checked: add test cases to an existing Test Execution (enter its Jira ID).
> Unchecked: create a brand-new Test Execution (enter a summary title)."

#### `buildExecutionCardPanel()` — CRITICAL LAYOUT DETAIL:

```java
// ── CARD_EXISTING (default, checkbox starts checked) ─────────────────────
JPanel existingCard = new JPanel(new GridBagLayout());
// Row 0: label "* Existing Test Execution ID:"  +  tfExistingExecutionId (gridwidth=2)
addRow(existingCard, c1, 0, "* Existing Test Execution ID:", tfExistingExecutionId, 2);

// ── CARD_NEW (shown when checkbox unchecked) ──────────────────────────────
JPanel newCard = new JPanel(new GridBagLayout());
// Row 0: label "* Test Execution Summary:"  +  tfExecutionSummary (gridwidth=2)
addRow(newCard, c2, 0, "* Test Execution Summary:", tfExecutionSummary, 2);

// Register — EXISTING must be added first (it's the default)
executionCardPanel.add(existingCard, CARD_EXISTING);
executionCardPanel.add(newCard,      CARD_NEW);

// Show EXISTING on startup (checkbox is selected by default)
executionCardLayout.show(executionCardPanel, CARD_EXISTING);
```

#### `onToggleExecutionMode()`:
```java
boolean useExisting = cbExistingTestExecution.isSelected();
executionCardLayout.show(executionCardPanel, useExisting ? CARD_EXISTING : CARD_NEW);
// log which mode is now active
```

#### `buildFileSelectionPanel()` (`TitledBorder("File Selection")`, `GridBagLayout`):
```
Row 0: "* CSV Folder:"   [tfCsvFolder — non-editable, gridwidth=2]  [Browse…]
Row 1: "* CSV File:"     [cbCsvFiles — gridwidth=2]                  [Preview]
Row 2: "* Config File:"  [tfConfigFile — non-editable, gridwidth=2]  [Browse…]
```

#### CENTER: `TitledBorder("Test Cases Preview")`:
`JScrollPane(JTable(tableModel))`, `fillsViewportHeight=true`, `rowHeight=22`, reordering disabled.

#### SOUTH `BorderLayout(4,6)`:
- NORTH: `[▶  Import to Jira]` button — Bold 13pt, `FlowLayout.CENTER`
- CENTER: `TitledBorder("Log")` + `JScrollPane(logArea)`

After `pack()`: `setMinimumSize(getSize())` then `setLocationRelativeTo(null)`.

#### `onBrowseCsvFolder()`:
Directory chooser → `csvService.listFiles()` → populate `cbCsvFiles`.

#### `onBrowseConfigFile()`:
JSON file filter → `configService.load()` → store in `loadedConfig`.
Log: `"Config loaded: N field mapping(s) found."` — project key comes from this file.

#### `onPreview()`:
```java
// No userStory UI field — pass empty string; CSV column takes over per test case
csvService.parse(selectedFile, loadedConfig, "")
// Log: "Preview: N test case(s) from M CSV rows — filename.csv"
//   M = sum of tc.steps.size() across all test cases
```

#### `onTestConnection()`:
`validateJiraFields()` → `buildJiraService()` → `testConnection()` in SwingWorker.

#### `onImport()` — FULL LOGIC:

```
1. Check loadedTestCases not null/empty
2. Check loadedConfig not null
3. validateJiraFields()
4. projectKey = loadedConfig.resolvedProjectKey()   ← from config file, NOT a UI field
   if blank → error and return

5. useExisting = cbExistingTestExecution.isSelected()

6. if useExisting:
       existingExecutionId = tfExistingExecutionId.getText().trim()
       if blank → showError("* Existing Test Execution ID is required (e.g. FCMUS-99).")
                  return

   else:
       executionSummary = tfExecutionSummary.getText().trim()
       if blank → showError("* Test Execution Summary is required.")
                  return

7. Confirm dialog:
   "This will:
     1. [Add to existing Execution: FCMUS-99]
         OR
        [Create new Execution: 'SUMMARY' in project 'KEY']
     2. Import N test case(s) into it
   Proceed?"

8. buildJiraService()

9. SwingWorker<Void, String> with publish() for streaming log:

   if useExisting:
       execKey = existingExecutionId
       publish("Using existing Test Execution: " + execKey)
   else:
       publish("Creating Test Execution '" + executionSummary + "' in project '" + projectKey + "'…")
       execKey = jiraService.createTestExecution(projectKey, "", executionSummary, "")
       //  ↑ userStory="" and label="" — both removed from UI
       publish(execKey != null
           ? "✔ Test Execution created: " + execKey
           : "⚠ Could not create — continuing without linking.")

   publish("Importing N test case(s)…")
   results = jiraService.importAll(loadedTestCases, projectKey, "Test", execKey, loadedConfig)

   successCount = count results starting with "✔"
   results.forEach → publish("  ID  →  result")
   publish("────────────────────────────────────────")
   publish("Done: N/total imported successfully.")

10. process(chunks) → log() each chunk
```

#### Private helpers:
```java
void buildJiraService()
// new JiraServiceImpl(tfUrl.getText().trim(), new String(tfToken.getPassword()))

boolean validateJiraFields()
// tfUrl blank → "* Jira URL is required."
// tfToken empty → "* Personal Access Token is required."

void refreshTable(List<TestCase> list)
// tableModel.setRowCount(0)
// for each tc: addRow {identifier, summary, priority, assignee,
//     steps.size()+" step(s)", environment, testType, labels, userStory}

void log(String msg)
// SwingUtilities.invokeLater → append + scroll to bottom

void showError(String msg)
// JOptionPane ERROR_MESSAGE

void addRow(JPanel, GridBagConstraints, int row, String labelText, JComponent field, int extraWidth)
GridBagConstraints defaultGbc()   // insets(5,8,5,8), WEST, HORIZONTAL
<T extends JPanel> T titled(String, T)
DefaultListCellRenderer fileNameRenderer()  // shows f.getName()
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

4 test cases, 3–4 steps each, with a `User Story` column per row.

```csv
Test Case Identifier,Summary,Description,Action,Priority,Assignee,Reporter,Environment,Outward issue link (Test),Labels,Test type,Step no.,Custom field (Test Repository Path),User Story
1,Verify login with valid credentials,User should be able to log in,Navigate to login page,High,john.doe,jane.smith,QA,FCMUS-10,smoke,Functional,1,/Authentication/Login,FCMUS-42
1,Verify login with valid credentials,User should be able to log in,Enter valid username and password,High,john.doe,jane.smith,QA,FCMUS-10,smoke,Functional,2,/Authentication/Login,FCMUS-42
1,Verify login with valid credentials,User should be able to log in,Click the Login button,High,john.doe,jane.smith,QA,FCMUS-10,smoke,Functional,3,/Authentication/Login,FCMUS-42
2,Verify login with invalid password,Login should fail for wrong password,Navigate to login page,High,john.doe,jane.smith,QA,FCMUS-10,regression,Functional,1,/Authentication/Login,FCMUS-42
2,Verify login with invalid password,Login should fail for wrong password,Enter valid username and wrong password,High,john.doe,jane.smith,QA,FCMUS-10,regression,Functional,2,/Authentication/Login,FCMUS-42
2,Verify login with invalid password,Login should fail for wrong password,Click the Login button and verify error,High,john.doe,jane.smith,QA,FCMUS-10,regression,Functional,3,/Authentication/Login,FCMUS-42
3,Verify password reset email,User receives reset email,Navigate to login page,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,1,/Authentication/Password,FCMUS-43
3,Verify password reset email,User receives reset email,Click Forgot Password,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,2,/Authentication/Password,FCMUS-43
3,Verify password reset email,User receives reset email,Enter registered email address,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,3,/Authentication/Password,FCMUS-43
3,Verify password reset email,User receives reset email,Check inbox for reset email,Medium,john.doe,jane.smith,QA,FCMUS-11,regression,Functional,4,/Authentication/Password,FCMUS-43
4,Verify user profile update,User can update display name,Navigate to Profile page,Low,john.doe,jane.smith,QA,FCMUS-12,smoke,Functional,1,/Profile/Update,FCMUS-44
4,Verify user profile update,User can update display name,Edit the display name field,Low,john.doe,jane.smith,QA,FCMUS-12,smoke,Functional,2,/Profile/Update,FCMUS-44
4,Verify user profile update,User can update display name,Click Save and verify change,Low,john.doe,jane.smith,QA,FCMUS-12,smoke,Functional,3,/Profile/Update,FCMUS-44
```

---

## 8. Build & Run

```bash
mvn clean package
java -jar target/jira-testcase-importer.jar
```

---

## 9. Critical Rules for Copilot

| # | Rule |
|---|------|
| 1 | Use **`java.net.http.HttpClient`** only — never Apache HttpClient |
| 2 | **No getters/setters** on `TestCase` or `FieldConfig` — public fields only |
| 3 | **No `service/impl/` sub-package** — all service classes in `com.estjira.service` |
| 4 | **No `ImportResult` class** — use `Map<String, String>` |
| 5 | `link-*` fields in `jiraFields` must be **skipped** in `buildPayload()` and handled in `handleOutwardLinks()` |
| 6 | `customfield_25902` = Test type, `customfield_17400` = Test Repository Path — plain string fields |
| 7 | `createTestExecution()` is called **only when** `cbExistingTestExecution` is **NOT** checked |
| 8 | All Jira API calls target `/rest/api/2/`; Xray calls target `/rest/raven/1.0/` |
| 9 | Auth: **`Authorization: Bearer <PAT>`** — no username, no Basic auth |
| 10 | All network calls run in `SwingWorker` — never block the EDT |
| 11 | `cbCsvFiles` dropdown renders **filename only** (`file.getName()`) |
| 12 | Call `setMinimumSize(getSize())` after `pack()` |
| 13 | `SwingWorker.process()` used in `onImport()` for real-time streaming |
| 14 | Config file browser uses `FileNameExtensionFilter` for `.json` only |
| 15 | Indent all Java with **4 spaces** |
| 16 | **No `cbIssueType`** — issue type hardcoded as `"Test"` |
| 17 | **No `tfProjectKey`** — project key from `loadedConfig.resolvedProjectKey()` only |
| 18 | **No username field** — PAT is self-identifying |
| 19 | Xray link: `POST /rest/raven/1.0/api/testexecution/{key}/test` with `{"add":["KEY"]}` |
| 20 | **No `tfUserStory` or `tfLabel` fields** — both removed from UI entirely |
| 21 | **No `tfExecutionName`** — replaced by `tfExecutionSummary` (new execution) and `tfExistingExecutionId` (existing execution) |
| 22 | **`cbExistingTestExecution` is checked by default** (`new JCheckBox("Existing Test Execution", true)`) |
| 23 | **Default startup state**: checkbox checked → `CARD_EXISTING` visible → user sees "* Existing Test Execution ID:" input |
| 24 | **`buildExecutionCardPanel()` MUST add both cards**: `executionCardPanel.add(existingCard, CARD_EXISTING)` AND `executionCardPanel.add(newCard, CARD_NEW)`. Start: `executionCardLayout.show(executionCardPanel, CARD_EXISTING)` |
| 25 | **GROUPING IS MANDATORY**: rows with same Test Case Identifier = one TestCase; metadata from first row only; every row adds one step |
| 26 | **Steps MUST appear in Jira description**: `"*Test Steps:*\n1. action\n2. action..."` appended after existing description |
| 27 | Preview table: **one row per grouped TestCase**; `"Steps"` column shows `"N step(s)"` |
| 28 | `"Step no."` and `"Action"` are **never** written to `tc.jiraFields` — only to `tc.steps` |
| 29 | **`"User Story"` CSV column** (case-insensitive) sets `tc.userStory` per test case. `onPreview()` passes `""` as the fallback (no UI field for it) |
| 30 | **`linkToUserStory(testKey, storyKey)`** — `POST /rest/api/2/issueLink`, `type.name = "relates to"`. Called in `importAll()` when `tc.userStory` not blank |
| 31 | **Preview table has 9 columns**: `"Identifier","Summary","Priority","Assignee","Steps","Environment","Test Type","Labels","User Story"` |
| 32 | `createTestExecution()` is called with **empty strings** for `userStory` and `label`: `jiraService.createTestExecution(projectKey, "", executionSummary, "")` |
