# GitHub Copilot Prompt — Jira Test Case Importer (estjira)

Generate a complete Java 17 Maven 3.9.6 desktop application called **jira-testcase-importer** that reads test cases from CSV files and imports them into Jira (estjira) via the REST API. Follow every detail below exactly.

---

## Project Coordinates

```
groupId:    com.estjira
artifactId: jira-testcase-importer
version:    1.0.0
mainClass:  com.estjira.Main
```

---

## Package Structure

```
src/main/java/com/estjira/
├── Main.java
├── model/
│   └── TestCase.java
├── service/
│   ├── CsvService.java          ← interface
│   ├── JiraService.java         ← interface
│   ├── CsvServiceImpl.java
│   └── JiraServiceImpl.java
└── ui/
    └── MainFrame.java
```

---

## pom.xml Rules

- Java source/target: **17**
- Only **2 dependencies**:
  - `com.opencsv:opencsv:5.9`
  - `com.fasterxml.jackson.core:jackson-databind:2.17.1`
- Use **maven-shade-plugin 3.5.2** to produce a single fat JAR named `jira-testcase-importer.jar`
- Exclude `META-INF/*.SF`, `META-INF/*.DSA`, `META-INF/*.RSA` in the shade filter
- No other plugins or dependencies

---

## File-by-File Specifications

---

### `model/TestCase.java`

- Package: `com.estjira.model`
- A plain class with **only public fields** — no getters, no setters, no constructor:
  ```
  String id, summary, description, steps, expectedResult,
         priority, status, labels, component, userStory;
  ```

---

### `service/CsvService.java`

- Package: `com.estjira.service`
- Interface with exactly **2 methods**:
  ```java
  List<TestCase> parse(File file, String userStory) throws Exception;
  File[] listFiles(File folder);
  ```

---

### `service/JiraService.java`

- Package: `com.estjira.service`
- Interface with exactly **2 methods**:
  ```java
  boolean testConnection();
  Map<String, String> importAll(List<TestCase> testCases, String projectKey, String issueType);
  ```
- Add a javadoc comment on `importAll`: "Returns a map of TestCase ID → created Jira key or error message."

---

### `service/CsvServiceImpl.java`

- Package: `com.estjira.service`
- Implements `CsvService`
- Use `com.opencsv.CSVReader` with `FileReader`
- `parse()` method:
  - Read the first row as headers
  - Throw `Exception("Empty CSV file.")` if headers are null
  - Build a `Map<String, Integer>` of `header.trim().toLowerCase() → index`
  - For each subsequent row, create a `TestCase` and map fields using a private `col()` helper
  - Set `tc.userStory = userStory` parameter
  - Field-to-header mappings (try aliases in order, use first non-blank):
    - `id`            → `"test case id"`, `"id"`
    - `summary`       → `"summary"`, `"title"`
    - `description`   → `"description"`, `"desc"`
    - `steps`         → `"test steps"`, `"steps"`
    - `expectedResult`→ `"expected result"`, `"expected"`
    - `priority`      → `"priority"`
    - `status`        → `"status"`
    - `labels`        → `"labels"`, `"label"`
    - `component`     → `"component"`, `"components"`
- `listFiles()` method:
  - Return `new File[0]` if folder is null or not a directory
  - Filter files whose name ends with `.csv` (case-insensitive)
- Private `col(String[] row, Map<String, Integer> idx, String... keys)` helper:
  - Loop through keys, look up index, return first non-blank trimmed value
  - Return `""` if none found

---

### `service/JiraServiceImpl.java`

- Package: `com.estjira.service`
- Implements `JiraService`
- **Do NOT use Apache HttpClient**. Use Java 17 built-in `java.net.http.HttpClient`
- Fields:
  - `String baseUrl` — strip trailing slash in constructor
  - `String auth` — `"Basic " + Base64.getEncoder().encodeToString((username + ":" + token).getBytes(UTF_8))`
  - `ObjectMapper json = new ObjectMapper()`
  - `HttpClient http = HttpClient.newHttpClient()`
- Constructor: `JiraServiceImpl(String baseUrl, String username, String token)`
- `testConnection()`:
  - GET `/rest/api/2/myself`
  - Return `true` if status == 200, `false` on any exception
- `importAll()`:
  - Loop through each `TestCase`, call `buildPayload()`, POST to `/rest/api/2/issue`
  - On HTTP 201: parse `key` from JSON response → put `"✔ " + key` in results map
  - On other status: put `"✘ HTTP " + statusCode`
  - On exception: put `"✘ " + e.getMessage()`
  - Return a `LinkedHashMap<String, String>` keyed by `tc.id`
- `buildPayload(TestCase tc, String projectKey, String issueType)`:
  - Build Jackson `ObjectNode` for `fields`:
    - `project.key` = projectKey
    - `issuetype.name` = issueType
    - `summary` = `"[" + tc.id + "] " + tc.summary` (skip brackets if id is blank)
    - `description` = `"*Steps:*\n" + tc.steps + "\n\n*Expected:*\n" + tc.expectedResult + "\n\n*Description:*\n" + tc.description`
    - If `priority` not blank: `priority.name` = tc.priority
    - If `userStory` not blank: `customfield_10014` = tc.userStory  *(Epic Link — adjust to match estjira)*
    - If `labels` not blank: add `labels` array by splitting on lines
    - If `component` not blank: add `components` array with single object `{name: tc.component}`
  - Wrap fields in root node and return `json.writeValueAsString(root)`
- Private `base(String path)` helper:
  - Returns `HttpRequest.newBuilder(URI.create(baseUrl + path)).header("Authorization", auth)`

---

### `ui/MainFrame.java`

- Package: `com.estjira.ui`
- Extends `JFrame`
- Title: `"Jira Test Case Importer — estjira"`
- `defaultCloseOperation`: `EXIT_ON_CLOSE`
- Root pane has `EmptyBorder(10,10,10,10)`
- `BorderLayout(8,8)` on the frame

#### Fields

```java
// Services
CsvService csv = new CsvServiceImpl();
JiraService jira;                          // built lazily in buildJira()

// Jira config
JTextField tfUrl       = new JTextField("https://estjira.example.com", 26);
JTextField tfUser      = new JTextField(16);
JPasswordField tfToken = new JPasswordField(16);
JTextField tfProject   = new JTextField("EST", 6);
JTextField tfUserStory = new JTextField("EST-1", 10);
JComboBox<String> cbType = new JComboBox<>(new String[]{"Test","Task","Sub-task","Story","Bug"});

// CSV
JTextField tfFolder         = new JTextField(22);
JComboBox<File> cbFiles     = new JComboBox<>();

// Preview
DefaultTableModel tableModel  // columns: "ID","Summary","Priority","Status","Steps","Expected Result"
                               // isCellEditable always returns false

// Log
JTextArea log = new JTextArea(6, 60);  // non-editable, monospaced 11pt

// State
List<TestCase> loaded;
```

#### Layout — 3 regions

**NORTH** — `GridLayout(2,1,4,4)` containing:
  1. `buildJiraPanel()` — `TitledBorder("Jira Settings")`, `GridBagLayout`
     - Row 0: label "Jira URL:", `tfUrl` (gridwidth=2), button "Test Connection"
     - Row 1: label "Username:", `tfUser`, label "API Token:", `tfToken`
     - Row 2: label "Project Key:", `tfProject`, label "Issue Type:", `cbType`
     - Row 3: label "User Story:", `tfUserStory` (gridwidth=3)
     - "Test Connection" button calls `onTestConnection()`

  2. `buildCsvPanel()` — `TitledBorder("CSV File Selection")`, `GridBagLayout`
     - Row 0: label "Folder:", `tfFolder` (non-editable, gridwidth=2), button "Browse…"
     - Row 1: label "CSV File:", `cbFiles` (gridwidth=2), button "Preview"
     - `cbFiles` renderer shows only `file.getName()`
     - "Browse…" calls `onBrowse()`; "Preview" calls `onPreview()`

**CENTER** — `TitledBorder("Preview")`, `BorderLayout`, contains `JScrollPane(new JTable(tableModel))`
  - Table: `setFillsViewportHeight(true)`

**SOUTH** — `BorderLayout(4,4)`:
  - NORTH: centered panel with single button `"▶  Import to Jira"` (Bold, 13pt) → calls `onImport()`
  - CENTER: `TitledBorder("Log")` panel containing `JScrollPane(log)`

#### Actions

`onBrowse()`:
- Open `JFileChooser` in `DIRECTORIES_ONLY` mode, title "Select folder containing CSV files"
- On approve: set `tfFolder` text, clear and repopulate `cbFiles` using `csv.listFiles(folder)`
- Log: `"Found N CSV file(s)."`

`onPreview()`:
- Get selected `File` from `cbFiles`; show error "Select a CSV file first." if null
- Run `csv.parse(file, tfUserStory.getText().trim())` in `SwingWorker`
- On success: store in `loaded`, clear `tableModel`, add one row per TestCase:
  `{tc.id, tc.summary, tc.priority, tc.status, tc.steps, tc.expectedResult}`
- Log: `"Preview: N test case(s) loaded from filename"`
- On error: show error dialog with parse error message

`onTestConnection()`:
- Call `buildJira()`, then run `jira.testConnection()` in `SwingWorker`
- On `true`: log `"✔ Connection successful!"` + show info dialog "Connected to estjira!"
- On `false`: log `"✘ Connection failed — check URL/credentials."` + show error dialog

`onImport()`:
- If `loaded` is null or empty: show error "Preview a CSV file first."
- If `tfProject` is blank: show error "Project Key is required."
- Call `buildJira()`
- Show `YES_NO` confirm dialog: `"Import N test case(s) to 'PROJECT'?"`
- Run `jira.importAll(loaded, project, type)` in `SwingWorker`
- On done: log each entry as `"  ID → result"`, then log summary:
  `"Done: N/total imported successfully."` (count entries starting with "✔")

#### Private helpers

```java
void buildJira()  // creates new JiraServiceImpl from tfUrl, tfUser, tfToken
void log(String)  // SwingUtilities.invokeLater → append + scroll to bottom
void error(String) // JOptionPane.showMessageDialog ERROR_MESSAGE
void row(JPanel, GridBagConstraints, int rowIndex, String labelText, JComponent field)
   // adds label at gridx=0, field at gridx=1 with gridwidth=2
JLabel lbl(String)
GridBagConstraints gbc()  // insets(4,6,4,6), WEST anchor, HORIZONTAL fill
<T extends Container> T titled(String title, T panel)
   // sets TitledBorder on panel if it is a JPanel, returns panel
```

After `pack()`, set `minimumSize` to the packed size, then `setLocationRelativeTo(null)`.

---

### `Main.java`

- Package: `com.estjira`
- Set `UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName())` — wrap in try/catch
- `SwingUtilities.invokeLater(() -> new MainFrame().setVisible(true))`

---

## Sample CSV File

Create `sample-testcases.csv` in the project root with exactly these columns and 3 data rows:

```
Test Case ID,Summary,Description,Test Steps,Expected Result,Priority,Status,Labels,Component
TC-001,Verify login with valid credentials,User login flow,"1. Open login page
2. Enter valid credentials
3. Click Login",Dashboard is displayed,High,Ready,smoke,Auth
TC-002,Verify login with invalid password,Negative login test,"1. Open login page
2. Enter wrong password
3. Click Login",Error message shown,High,Ready,regression,Auth
TC-003,Verify password reset email,Forgot password flow,"1. Click Forgot Password
2. Enter email
3. Submit",Reset email received within 2 min,Medium,Ready,regression,Auth
```

---

## Build & Run

```bash
mvn clean package
java -jar target/jira-testcase-importer.jar
```

---

## Key Notes for Copilot

1. **No Apache HttpClient** — use only `java.net.http.HttpClient` (built into Java 11+)
2. **No getters/setters on TestCase** — use direct public field access everywhere
3. **No separate `ImportResult` class** — use `Map<String, String>` for import results
4. **No `service/impl/` sub-package** — all 4 service classes live in `com.estjira.service`
5. `customfield_10014` is the Epic Link field — add a comment saying it may need adjustment
6. All Jira API calls target `/rest/api/2/` (Jira REST API v2)
7. Authentication is HTTP Basic: `Base64(username:apiToken)`
8. SwingWorker must be used for all network operations (never block the EDT)
9. The CSV dropdown (`cbFiles`) must render only the file name, not the full path
10. After `pack()`, call `setMinimumSize(getSize())` to prevent the window from being resized smaller
