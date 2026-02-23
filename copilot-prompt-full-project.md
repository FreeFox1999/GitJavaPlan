# GitHub Copilot Prompt — Jira Test Case Importer (Full Project)

## Project Overview

Generate a complete Java 17 desktop application called **Jira Test Case Importer** (`com.estjira`).
It is a Java Swing GUI tool that:
1. Reads test cases from CSV files
2. Previews them in a table
3. Imports them as Jira issues (type: "Test") via the Jira REST API v2
4. Links each test case to a Jira Test Execution (new or existing) using the Xray REST API
5. Links each test case to a User Story (read from the CSV or typed in the UI)

Build tool: **Maven**. Package as a fat JAR using `maven-shade-plugin`.

---

## Project Structure

```
jira-testcase-importer/
├── pom.xml
├── sample-config.json
├── sample-testcases.csv
└── src/main/java/com/estjira/
    ├── Main.java
    ├── model/
    │   ├── TestCase.java
    │   └── FieldConfig.java
    ├── service/
    │   ├── CsvService.java
    │   ├── CsvServiceImpl.java
    │   ├── ConfigService.java
    │   ├── ConfigServiceImpl.java
    │   ├── JiraService.java
    │   └── JiraServiceImpl.java
    └── ui/
        └── MainFrame.java
```

---

## pom.xml

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

## Main.java

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

## model/TestCase.java

```java
package com.estjira.model;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Represents one complete test case, which may span multiple CSV rows.
 *
 * Rows sharing the same testCaseIdentifier are grouped into a single TestCase.
 * Metadata fields are taken from the FIRST row of the group.
 * Every row contributes one entry to `steps`.
 * `jiraFields` drives the Jira REST API payload.
 */
public class TestCase {

    // Identity & metadata (from first row of the group)
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

    // Linked User Story key (e.g. FCMUS-42) — from CSV column or UI fallback
    public String userStory;

    // Steps — one String[2] per CSV row: [0]=step number, [1]=action text
    public List<String[]> steps = new ArrayList<>();

    // Dynamic payload map: Jira field ID → raw value (from first row only)
    public Map<String, String> jiraFields = new LinkedHashMap<>();
}
```

---

## model/FieldConfig.java

```java
package com.estjira.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Maps to the xray bulk upload importer config JSON file.
 *
 * The "field" map:  key = CSV column header,  value = Jira field ID
 *
 * Example:
 * {
 *   "projectKey": { "project": "FCMUS" },
 *   "field": {
 *     "Summary"                          : "summary",
 *     "Priority"                         : "priority",
 *     "Assignee"                         : "assignee",
 *     "Reporter"                         : "reporter",
 *     "Description"                      : "description",
 *     "Environment"                      : "environment",
 *     "Labels"                           : "labels",
 *     "Test Case Identifier"             : "Test Case Identifier",
 *     "Test type"                        : "customfield_25902",
 *     "Outward issue link (Test)"        : "link-10900",
 *     "Custom field (Test Repository Path)" : "customfield_17400",
 *     "Action"                           : "Action",
 *     "Step no."                         : "Step Attachments"
 *   }
 * }
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public class FieldConfig {

    public Map<String, String> projectKey       = new LinkedHashMap<>();
    public Map<String, String> field            = new LinkedHashMap<>();
    public Map<String, String> link             = new LinkedHashMap<>();
    public Map<String, String> projectSelection = new LinkedHashMap<>();
    public Map<String, String> projectName      = new LinkedHashMap<>();
    public Map<String, String> value            = new LinkedHashMap<>();

    /** Returns the project key string (e.g. "FCMUS"). */
    public String resolvedProjectKey() {
        return projectKey.getOrDefault("project", "");
    }
}
```

---

## service/CsvService.java

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import java.io.File;
import java.util.List;

public interface CsvService {
    /**
     * Parse a CSV file into TestCase objects using the field config for column mapping.
     * Rows with the same "Test Case Identifier" are grouped into one TestCase.
     * @param userStory fallback User Story key if no "User Story" column in CSV
     */
    List<TestCase> parse(File file, FieldConfig config, String userStory) throws Exception;

    /** Lists all .csv files in the given folder. */
    File[] listFiles(File folder);
}
```

---

## service/CsvServiceImpl.java

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import com.opencsv.CSVReader;

import java.io.File;
import java.io.FileReader;
import java.util.*;

/**
 * Groups CSV rows by "Test Case Identifier", collecting steps from every row
 * and metadata from the first row of each group.
 *
 * If the CSV has a "User Story" column its value overrides the fallback parameter.
 */
public class CsvServiceImpl implements CsvService {

    @Override
    public List<TestCase> parse(File file, FieldConfig config, String userStory) throws Exception {
        Map<String, TestCase> grouped = new LinkedHashMap<>();

        try (CSVReader reader = new CSVReader(new FileReader(file))) {
            String[] headers = reader.readNext();
            if (headers == null) throw new Exception("CSV file is empty: " + file.getName());

            Map<String, Integer> colIdx = new HashMap<>();
            for (int i = 0; i < headers.length; i++)
                colIdx.put(headers[i].trim().toLowerCase(), i);

            boolean csvHasUserStoryCol = colIdx.containsKey("user story");

            String[] row;
            while ((row = reader.readNext()) != null) {
                String id = resolveIdentifier(row, colIdx, config);
                if (id.isBlank()) continue;

                TestCase tc = grouped.get(id);
                boolean isFirstRow = (tc == null);

                if (isFirstRow) {
                    tc = new TestCase();
                    tc.testCaseIdentifier = id;
                    if (csvHasUserStoryCol) {
                        String csvStory = getCsvValueByHeader(row, colIdx, "user story");
                        tc.userStory = csvStory.isBlank() ? userStory : csvStory;
                    } else {
                        tc.userStory = userStory;
                    }
                    grouped.put(id, tc);
                }

                String stepNo = "";
                String action = "";

                for (Map.Entry<String, String> entry : config.field.entrySet()) {
                    String csvHeader = entry.getKey();
                    String jiraField = entry.getValue();
                    String value     = getCsvValue(row, colIdx, csvHeader);
                    if (value.isBlank()) continue;

                    String headerLc = csvHeader.trim().toLowerCase();

                    if (headerLc.equals("step no.")) {
                        stepNo = value;
                    } else if (headerLc.equals("action")) {
                        action = value;
                    } else if (headerLc.equals("test case identifier")) {
                        // internal key — already set on tc
                    } else if (isFirstRow) {
                        tc.jiraFields.put(jiraField, value);
                        populateNamedField(tc, csvHeader, value);
                    }
                }

                if (!stepNo.isBlank() || !action.isBlank())
                    tc.steps.add(new String[]{stepNo, action});
            }
        }
        return new ArrayList<>(grouped.values());
    }

    @Override
    public File[] listFiles(File folder) {
        if (folder == null || !folder.isDirectory()) return new File[0];
        File[] files = folder.listFiles(f -> f.getName().toLowerCase().endsWith(".csv"));
        return files != null ? files : new File[0];
    }

    private String resolveIdentifier(String[] row, Map<String, Integer> colIdx, FieldConfig config) {
        for (Map.Entry<String, String> e : config.field.entrySet())
            if (e.getKey().trim().equalsIgnoreCase("Test Case Identifier"))
                return getCsvValue(row, colIdx, e.getKey());
        return "";
    }

    private String getCsvValue(String[] row, Map<String, Integer> colIdx, String csvHeader) {
        return getCsvValueByHeader(row, colIdx, csvHeader.trim().toLowerCase());
    }

    private String getCsvValueByHeader(String[] row, Map<String, Integer> colIdx, String lowerHeader) {
        Integer idx = colIdx.get(lowerHeader);
        return (idx != null && idx < row.length) ? row[idx].trim() : "";
    }

    private void populateNamedField(TestCase tc, String csvHeader, String value) {
        switch (csvHeader.trim().toLowerCase()) {
            case "summary"                             -> tc.summary            = value;
            case "priority"                            -> tc.priority           = value;
            case "assignee"                            -> tc.assignee           = value;
            case "reporter"                            -> tc.reporter           = value;
            case "description"                         -> tc.description        = value;
            case "environment"                         -> tc.environment        = value;
            case "labels", "label"                     -> tc.labels             = value;
            case "test type"                           -> tc.testType           = value;
            case "outward issue link (test)"           -> tc.outwardIssueLink   = value;
            case "custom field (test repository path)" -> tc.testRepositoryPath = value;
        }
    }
}
```

---

## service/ConfigService.java

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import java.io.File;

public interface ConfigService {
    /** Load and parse the xray bulk upload importer config JSON into a FieldConfig. */
    FieldConfig load(File configFile) throws Exception;
}
```

---

## service/ConfigServiceImpl.java

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.File;

public class ConfigServiceImpl implements ConfigService {

    private final ObjectMapper json = new ObjectMapper();

    @Override
    public FieldConfig load(File configFile) throws Exception {
        if (configFile == null || !configFile.exists())
            throw new Exception("Config file not found: " + (configFile != null ? configFile.getPath() : "null"));
        FieldConfig config = json.readValue(configFile, FieldConfig.class);
        if (config.field == null || config.field.isEmpty())
            throw new Exception("Config file has no 'field' mappings defined.");
        return config;
    }
}
```

---

## service/JiraService.java

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import java.util.List;
import java.util.Map;

public interface JiraService {

    /** Returns true if credentials are valid and the instance is reachable. */
    boolean testConnection();

    /**
     * Creates a Test Execution issue in Jira (Xray).
     * @param projectKey  e.g. "FCMUS"
     * @param userStory   User Story key to link execution to (e.g. "FCMUS-42"), or ""
     * @param executionName  Summary/title shown on the Jira board
     * @param label       Optional label tag, or ""
     * @return created Test Execution issue key (e.g. "FCMUS-99"), or null on failure
     */
    String createTestExecution(String projectKey, String userStory,
                               String executionName, String label);

    /**
     * Imports all test cases to Jira, linking each to the given Test Execution.
     * @return map of TestCase identifier → "✔ KEY" or "✘ error"
     */
    Map<String, String> importAll(List<TestCase> testCases, String projectKey,
                                   String issueType, String testExecutionKey,
                                   FieldConfig config);
}
```

---

## service/JiraServiceImpl.java

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;

import java.net.URI;
import java.net.http.*;
import java.util.*;
import java.util.logging.Logger;

/**
 * Calls the estjira REST API v2 using Java 17's built-in HttpClient.
 * Auth: Bearer PAT (Personal Access Token — no username needed).
 */
public class JiraServiceImpl implements JiraService {

    private static final Logger LOG = Logger.getLogger(JiraServiceImpl.class.getName());
    private static final String LINK_PREFIX          = "link-";
    private static final String TEST_EXECUTION_TYPE  = "Test Execution";

    /** Field IDs from config that are NOT real Jira fields — never sent in payload. */
    private static final Set<String> INTERNAL_FIELD_IDS = Set.of(
            "Test Case Identifier", "Action", "Step Attachments"
    );

    private final String       baseUrl;
    private final String       auth;
    private final ObjectMapper json = new ObjectMapper();
    private final HttpClient   http = HttpClient.newHttpClient();

    public JiraServiceImpl(String baseUrl, String pat) {
        this.baseUrl = baseUrl.replaceAll("/$", "");
        this.auth    = "Bearer " + pat;
    }

    // ── Interface ─────────────────────────────────────────────────────────────

    @Override
    public boolean testConnection() {
        try {
            return send(get("/rest/api/2/myself")).statusCode() == 200;
        } catch (Exception e) {
            LOG.warning("Connection test failed: " + e.getMessage());
            return false;
        }
    }

    @Override
    public String createTestExecution(String projectKey, String userStory,
                                      String executionName, String label) {
        try {
            ObjectNode fields = json.createObjectNode();
            fields.putObject("project").put("key", projectKey);
            fields.putObject("issuetype").put("name", TEST_EXECUTION_TYPE);
            fields.put("summary", executionName);
            if (!label.isBlank()) fields.putArray("labels").add(label);

            String body = json.writeValueAsString(json.createObjectNode().set("fields", fields));
            HttpResponse<String> res = send(post("/rest/api/2/issue", body));

            if (res.statusCode() == 201) {
                String execKey = json.readTree(res.body()).path("key").asText();
                LOG.info("Test Execution created: " + execKey);
                if (!userStory.isBlank()) {
                    ObjectNode linkBody = json.createObjectNode();
                    linkBody.putObject("type").put("name", "relates to");
                    linkBody.putObject("inwardIssue").put("key", execKey);
                    linkBody.putObject("outwardIssue").put("key", userStory);
                    send(post("/rest/api/2/issueLink", json.writeValueAsString(linkBody)));
                }
                return execKey;
            } else {
                LOG.warning("Failed to create Test Execution: HTTP " + res.statusCode());
                return null;
            }
        } catch (Exception e) {
            LOG.severe("createTestExecution error: " + e.getMessage());
            return null;
        }
    }

    @Override
    public Map<String, String> importAll(List<TestCase> testCases, String projectKey,
                                          String issueType, String testExecutionKey,
                                          FieldConfig config) {
        Map<String, String> results = new LinkedHashMap<>();
        for (TestCase tc : testCases) {
            String id = tc.testCaseIdentifier != null ? tc.testCaseIdentifier : tc.summary;
            try {
                String body = buildPayload(tc, projectKey, issueType);
                HttpResponse<String> res = send(post("/rest/api/2/issue", body));
                if (res.statusCode() == 201) {
                    String key = json.readTree(res.body()).path("key").asText();
                    results.put(id, "✔ " + key);
                    if (testExecutionKey != null && !testExecutionKey.isBlank())
                        addTestToExecution(testExecutionKey, key);
                    handleOutwardLinks(tc, key);
                    if (tc.userStory != null && !tc.userStory.isBlank())
                        linkToUserStory(key, tc.userStory);
                } else {
                    String err = extractError(res.body());
                    if (res.statusCode() == 400)
                        err += " | HTTP 400: check field is on the Create Issue screen for this project/issue type.";
                    results.put(id, "✘ HTTP " + res.statusCode() + " — " + err);
                }
            } catch (Exception e) {
                results.put(id, "✘ " + e.getMessage());
            }
        }
        return results;
    }

    // ── Payload builder ───────────────────────────────────────────────────────

    private String buildPayload(TestCase tc, String projectKey, String issueType) throws Exception {
        ObjectNode fields = json.createObjectNode();
        fields.putObject("project").put("key", projectKey);
        fields.putObject("issuetype").put("name", issueType);

        for (Map.Entry<String, String> entry : tc.jiraFields.entrySet()) {
            String fieldId = entry.getKey();
            String value   = entry.getValue();
            if (fieldId.startsWith(LINK_PREFIX))       continue;
            if (INTERNAL_FIELD_IDS.contains(fieldId))  continue;

            switch (fieldId) {
                case "summary"     -> fields.put("summary", value);
                case "description" -> fields.put("description", value);
                case "priority"    -> fields.putObject("priority").put("name", value);
                case "assignee"    -> fields.putObject("assignee").put("name", value);
                case "reporter"    -> fields.putObject("reporter").put("name", value);
                case "environment" -> fields.put("environment", value);
                case "labels"      -> {
                    ArrayNode arr = fields.putArray("labels");
                    Arrays.stream(value.split("[,;]")).map(String::trim)
                          .filter(s -> !s.isEmpty()).forEach(arr::add);
                }
                default -> fields.put(fieldId, value);  // customfield_* etc.
            }
        }

        // Fallback summary
        if (!fields.has("summary") || fields.get("summary").asText().isBlank()) {
            String s = tc.summary != null ? tc.summary : tc.testCaseIdentifier;
            fields.put("summary", s != null ? s : "Test Case");
        }

        // Append steps to description
        if (!tc.steps.isEmpty()) {
            String existing = fields.has("description") ? fields.get("description").asText() : "";
            StringBuilder sb = new StringBuilder();
            if (!existing.isBlank()) sb.append(existing).append("\n\n");
            sb.append("*Test Steps:*\n");
            for (int i = 0; i < tc.steps.size(); i++) {
                String[] step = tc.steps.get(i);
                String no     = (step[0] != null && !step[0].isBlank()) ? step[0] : String.valueOf(i + 1);
                String act    = (step[1] != null) ? step[1] : "";
                sb.append(no).append(". ").append(act).append("\n");
            }
            fields.put("description", sb.toString().stripTrailing());
        }

        return json.writeValueAsString(json.createObjectNode().set("fields", fields));
    }

    // ── Xray / issue linking ──────────────────────────────────────────────────

    /** POST /rest/raven/1.0/api/testexecution/{execKey}/test  body: { "add": ["testKey"] } */
    private void addTestToExecution(String execKey, String testKey) {
        try {
            ObjectNode body = json.createObjectNode();
            body.putArray("add").add(testKey);
            HttpResponse<String> res = send(post(
                    "/rest/raven/1.0/api/testexecution/" + execKey + "/test",
                    json.writeValueAsString(body)));
            if (res.statusCode() != 200 && res.statusCode() != 201)
                LOG.warning("Failed to add " + testKey + " to execution " + execKey);
        } catch (Exception e) { LOG.warning("addTestToExecution error: " + e.getMessage()); }
    }

    /** Links test issue → User Story via "relates to" link type. */
    private void linkToUserStory(String testKey, String userStoryKey) {
        try {
            ObjectNode body = json.createObjectNode();
            body.putObject("type").put("name", "relates to");
            body.putObject("inwardIssue").put("key", testKey);
            body.putObject("outwardIssue").put("key", userStoryKey);
            HttpResponse<String> res = send(post("/rest/api/2/issueLink",
                    json.writeValueAsString(body)));
            if (res.statusCode() != 201 && res.statusCode() != 204)
                LOG.warning("Failed to link " + testKey + " → " + userStoryKey);
        } catch (Exception e) { LOG.warning("linkToUserStory error: " + e.getMessage()); }
    }

    /** Handles link-* field IDs from config as outward Jira issue links. */
    private void handleOutwardLinks(TestCase tc, String createdKey) {
        for (Map.Entry<String, String> entry : tc.jiraFields.entrySet()) {
            if (!entry.getKey().startsWith(LINK_PREFIX)) continue;
            String linkedKey = entry.getValue();
            if (linkedKey.isBlank()) continue;
            try {
                ObjectNode body = json.createObjectNode();
                body.putObject("type").put("name", "Test");
                body.putObject("inwardIssue").put("key", createdKey);
                body.putObject("outwardIssue").put("key", linkedKey);
                send(post("/rest/api/2/issueLink", json.writeValueAsString(body)));
            } catch (Exception e) { LOG.warning("handleOutwardLinks error: " + e.getMessage()); }
        }
    }

    // ── HTTP helpers ──────────────────────────────────────────────────────────

    private HttpResponse<String> send(HttpRequest req) throws Exception {
        return http.send(req, HttpResponse.BodyHandlers.ofString());
    }

    private HttpRequest get(String path) { return base(path).GET().build(); }

    private HttpRequest post(String path, String body) {
        return base(path).POST(HttpRequest.BodyPublishers.ofString(body))
                .header("Content-Type", "application/json").build();
    }

    private HttpRequest.Builder base(String path) {
        return HttpRequest.newBuilder(URI.create(baseUrl + path))
                .header("Authorization", auth).header("Accept", "application/json");
    }

    private String extractError(String responseBody) {
        try {
            var root = json.readTree(responseBody);
            if (root.has("errorMessages") && root.get("errorMessages").size() > 0)
                return root.get("errorMessages").get(0).asText();
            if (root.has("errors")) return root.get("errors").toString();
        } catch (Exception ignored) {}
        return responseBody;
    }
}
```

---

## ui/MainFrame.java

> ⚠️ **UI Alignment Fix Applied** — See section below for the specific changes made and why.

```java
package com.estjira.ui;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import com.estjira.service.*;

import javax.swing.*;
import javax.swing.border.EmptyBorder;
import javax.swing.border.TitledBorder;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.io.File;
import java.util.List;
import java.util.Map;

public class MainFrame extends JFrame {

    private static final String CARD_NEW      = "NEW";
    private static final String CARD_EXISTING = "EXISTING";

    // Services
    private final CsvService    csvService    = new CsvServiceImpl();
    private final ConfigService configService = new ConfigServiceImpl();
    private JiraService jiraService;

    // Jira Settings
    private final JTextField     tfUrl   = new JTextField("https://estjira.example.com", 30);
    private final JPasswordField tfToken = new JPasswordField(24);

    // Execution mode — checkbox (checked = existing) + CardLayout switcher
    private final JCheckBox  cbExistingTestExecution = new JCheckBox("Existing Test Execution", true);
    private final CardLayout executionCardLayout      = new CardLayout();
    private final JPanel     executionCardPanel       = new JPanel(executionCardLayout);

    private final JTextField tfExistingExecutionId = new JTextField(28); // shown when checked
    private final JTextField tfExecutionSummary    = new JTextField(28); // shown when unchecked

    // File selection
    private final JTextField      tfCsvFolder  = new JTextField(26);
    private final JComboBox<File> cbCsvFiles   = new JComboBox<>();
    private final JTextField      tfConfigFile = new JTextField(26);

    // Preview table — 9 columns
    private final DefaultTableModel tableModel = new DefaultTableModel(
            new String[]{"Identifier","Summary","Priority","Assignee","Steps",
                         "Environment","Test Type","Labels","User Story"}, 0) {
        @Override public boolean isCellEditable(int r, int c) { return false; }
    };

    private final JTextArea logArea = new JTextArea(7, 70);

    private List<TestCase> loadedTestCases;
    private FieldConfig    loadedConfig;

    public MainFrame() {
        super("Jira Test Case Importer — estjira");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        JPanel root = new JPanel(new BorderLayout(8, 8));
        root.setBorder(new EmptyBorder(12, 12, 12, 12));
        root.add(buildTopPanel(),     BorderLayout.NORTH);
        root.add(buildPreviewPanel(), BorderLayout.CENTER);
        root.add(buildBottomPanel(),  BorderLayout.SOUTH);
        setContentPane(root);
        pack();
        setMinimumSize(getSize());
        setLocationRelativeTo(null);
    }

    private JPanel buildTopPanel() {
        JPanel panel = new JPanel(new GridLayout(2, 1, 6, 6));
        panel.add(buildJiraSettingsPanel());
        panel.add(buildFileSelectionPanel());
        return panel;
    }

    private JPanel buildJiraSettingsPanel() {
        JPanel panel = titled("Jira Settings", new JPanel(new GridBagLayout()));
        GridBagConstraints c = defaultGbc();

        // Row 0: URL (cols 1-2) + Test Connection button (col 3)
        addRow(panel, c, 0, "* Jira URL:", tfUrl, 1);
        c.gridx = 3; c.gridy = 0; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnTest = uniformButton("Test Connection");
        btnTest.addActionListener(e -> onTestConnection());
        panel.add(btnTest, c);

        // Row 1: PAT — no button, field spans cols 1-3
        addRow(panel, c, 1, "* Personal Access Token:", tfToken, 2);

        // Row 2: Checkbox — full width (all 4 cols)
        c.gridx = 0; c.gridy = 2; c.gridwidth = 4; c.weightx = 1;
        cbExistingTestExecution.setToolTipText(
            "Checked: add test cases to an existing Test Execution (enter its Jira ID).\n" +
            "Unchecked: create a brand-new Test Execution (enter a summary title).");
        cbExistingTestExecution.addActionListener(e -> onToggleExecutionMode());
        panel.add(cbExistingTestExecution, c);

        // Row 3: CardLayout panel — full width (all 4 cols)
        buildExecutionCardPanel();
        c.gridx = 0; c.gridy = 3; c.gridwidth = 4; c.weightx = 1;
        panel.add(executionCardPanel, c);

        return panel;
    }

    private void buildExecutionCardPanel() {
        JPanel existingCard = new JPanel(new GridBagLayout());
        GridBagConstraints c1 = defaultGbc();
        addRow(existingCard, c1, 0, "* Existing Test Execution ID:", tfExistingExecutionId, 2);

        JPanel newCard = new JPanel(new GridBagLayout());
        GridBagConstraints c2 = defaultGbc();
        addRow(newCard, c2, 0, "* Test Execution Summary:", tfExecutionSummary, 2);

        executionCardPanel.add(existingCard, CARD_EXISTING);
        executionCardPanel.add(newCard,      CARD_NEW);
        executionCardLayout.show(executionCardPanel, CARD_EXISTING);
    }

    private void onToggleExecutionMode() {
        boolean useExisting = cbExistingTestExecution.isSelected();
        executionCardLayout.show(executionCardPanel, useExisting ? CARD_EXISTING : CARD_NEW);
        log(useExisting
            ? "Mode: Add to EXISTING Test Execution — enter its Jira ID (e.g. FCMUS-99)."
            : "Mode: Create NEW Test Execution — enter a summary title.");
    }

    private JPanel buildFileSelectionPanel() {
        JPanel panel = titled("File Selection", new JPanel(new GridBagLayout()));
        GridBagConstraints c = defaultGbc();

        // Row 0: CSV Folder (cols 1-2) + Browse button (col 3)
        tfCsvFolder.setEditable(false);
        addRow(panel, c, 0, "* CSV Folder:", tfCsvFolder, 1);
        c.gridx = 3; c.gridy = 0; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnF = uniformButton("Browse…");
        btnF.addActionListener(e -> onBrowseCsvFolder());
        panel.add(btnF, c);

        // Row 1: CSV File combo (cols 1-2) + Preview button (col 3)
        cbCsvFiles.setRenderer(fileNameRenderer());
        addRow(panel, c, 1, "* CSV File:", cbCsvFiles, 1);
        c.gridx = 3; c.gridy = 1; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnP = uniformButton("Preview");
        btnP.addActionListener(e -> onPreview());
        panel.add(btnP, c);

        // Row 2: Config File (cols 1-2) + Browse button (col 3)
        tfConfigFile.setEditable(false);
        addRow(panel, c, 2, "* Config File:", tfConfigFile, 1);
        c.gridx = 3; c.gridy = 2; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnC = uniformButton("Browse…");
        btnC.addActionListener(e -> onBrowseConfigFile());
        panel.add(btnC, c);

        return panel;
    }

    private JPanel buildPreviewPanel() {
        JTable table = new JTable(tableModel);
        table.setFillsViewportHeight(true);
        table.setRowHeight(22);
        table.getTableHeader().setReorderingAllowed(false);
        JPanel panel = titled("Test Cases Preview", new JPanel(new BorderLayout()));
        panel.add(new JScrollPane(table), BorderLayout.CENTER);
        return panel;
    }

    private JPanel buildBottomPanel() {
        JPanel panel = new JPanel(new BorderLayout(4, 6));
        JButton btnImport = new JButton("▶  Import to Jira");
        btnImport.setFont(btnImport.getFont().deriveFont(Font.BOLD, 13f));
        btnImport.addActionListener(e -> onImport());
        JPanel btnRow = new JPanel(new FlowLayout(FlowLayout.CENTER));
        btnRow.add(btnImport);
        panel.add(btnRow, BorderLayout.NORTH);
        logArea.setEditable(false);
        logArea.setFont(new Font(Font.MONOSPACED, Font.PLAIN, 11));
        JPanel logPanel = titled("Log", new JPanel(new BorderLayout()));
        logPanel.add(new JScrollPane(logArea), BorderLayout.CENTER);
        panel.add(logPanel, BorderLayout.CENTER);
        return panel;
    }

    // ── Actions ───────────────────────────────────────────────────────────────

    private void onBrowseCsvFolder() {
        JFileChooser ch = new JFileChooser();
        ch.setFileSelectionMode(JFileChooser.DIRECTORIES_ONLY);
        ch.setDialogTitle("Select folder containing CSV files");
        if (ch.showOpenDialog(this) == JFileChooser.APPROVE_OPTION) {
            File folder = ch.getSelectedFile();
            tfCsvFolder.setText(folder.getAbsolutePath());
            cbCsvFiles.removeAllItems();
            File[] files = csvService.listFiles(folder);
            for (File f : files) cbCsvFiles.addItem(f);
            log("Found " + files.length + " CSV file(s) in folder.");
        }
    }

    private void onBrowseConfigFile() {
        JFileChooser ch = new JFileChooser();
        ch.setDialogTitle("Select xray bulk upload importer config (.json)");
        ch.setFileFilter(new javax.swing.filechooser.FileNameExtensionFilter(
                "JSON Config File (*.json)", "json"));
        if (ch.showOpenDialog(this) == JFileChooser.APPROVE_OPTION) {
            File f = ch.getSelectedFile();
            tfConfigFile.setText(f.getAbsolutePath());
            try {
                loadedConfig = configService.load(f);
                log("Config loaded: " + loadedConfig.field.size() + " field mapping(s) found.");
            } catch (Exception ex) {
                showError("Failed to load config: " + ex.getMessage());
                loadedConfig = null;
            }
        }
    }

    private void onPreview() {
        File sel = (File) cbCsvFiles.getSelectedItem();
        if (sel == null)          { showError("Please select a CSV file.");         return; }
        if (loadedConfig == null) { showError("Please load a config file first."); return; }
        String userStoryFallback = "";
        SwingWorker<List<TestCase>, Void> w = new SwingWorker<>() {
            @Override protected List<TestCase> doInBackground() throws Exception {
                return csvService.parse(sel, loadedConfig, userStoryFallback);
            }
            @Override protected void done() {
                try {
                    loadedTestCases = get();
                    refreshTable(loadedTestCases);
                    int rows = loadedTestCases.stream().mapToInt(tc -> tc.steps.size()).sum();
                    log("Preview: " + loadedTestCases.size() + " test case(s) from "
                            + rows + " CSV rows — " + sel.getName());
                } catch (Exception ex) { showError("Failed to parse CSV: " + ex.getMessage()); }
            }
        };
        w.execute();
    }

    private void onTestConnection() {
        if (!validateJiraFields()) return;
        buildJiraService();
        SwingWorker<Boolean, Void> w = new SwingWorker<>() {
            @Override protected Boolean doInBackground() { return jiraService.testConnection(); }
            @Override protected void done() {
                try {
                    boolean ok = get();
                    if (ok) {
                        log("✔ Connection successful!");
                        JOptionPane.showMessageDialog(MainFrame.this,
                            "Connected to estjira successfully.", "Success",
                            JOptionPane.INFORMATION_MESSAGE);
                    } else {
                        log("✘ Connection failed.");
                        showError("Could not connect. Verify Jira URL and Personal Access Token.");
                    }
                } catch (Exception ex) { showError(ex.getMessage()); }
            }
        };
        w.execute();
    }

    private void onImport() {
        if (loadedTestCases == null || loadedTestCases.isEmpty()) {
            showError("No test cases loaded. Please preview a CSV file first."); return;
        }
        if (loadedConfig == null) { showError("No config file loaded."); return; }
        if (!validateJiraFields()) return;

        String projectKey = loadedConfig.resolvedProjectKey();
        if (projectKey.isBlank()) {
            showError("Project key not found in config. Check 'projectKey.project'."); return;
        }

        boolean useExisting = cbExistingTestExecution.isSelected();
        final String executionSummary;
        final String existingExecutionId;

        if (useExisting) {
            existingExecutionId = tfExistingExecutionId.getText().trim();
            executionSummary    = null;
            if (existingExecutionId.isBlank()) {
                showError("* Existing Test Execution ID is required (e.g. FCMUS-99)."); return;
            }
        } else {
            executionSummary    = tfExecutionSummary.getText().trim();
            existingExecutionId = null;
            if (executionSummary.isBlank()) {
                showError("* Test Execution Summary is required."); return;
            }
        }

        String execLine = useExisting
                ? "Add to existing Execution: " + existingExecutionId
                : "Create new Execution: '" + executionSummary + "' in project '" + projectKey + "'";

        int confirm = JOptionPane.showConfirmDialog(this,
                "This will:\n  1. " + execLine + "\n"
                + "  2. Import " + loadedTestCases.size() + " test case(s) into it\n\nProceed?",
                "Confirm Import", JOptionPane.YES_NO_OPTION);
        if (confirm != JOptionPane.YES_OPTION) return;

        buildJiraService();
        SwingWorker<Void, String> w = new SwingWorker<>() {
            @Override protected Void doInBackground() {
                String execKey;
                if (useExisting) {
                    execKey = existingExecutionId;
                    publish("Using existing Test Execution: " + execKey);
                } else {
                    publish("Creating Test Execution '" + executionSummary
                            + "' in project '" + projectKey + "'…");
                    execKey = jiraService.createTestExecution(projectKey, "", executionSummary, "");
                    publish(execKey != null
                        ? "✔ Test Execution created: " + execKey
                        : "⚠ Test Execution could not be created — continuing without linking.");
                }
                publish("Importing " + loadedTestCases.size() + " test case(s)…");
                Map<String, String> results = jiraService.importAll(
                        loadedTestCases, projectKey, "Test", execKey, loadedConfig);
                long ok = results.values().stream().filter(v -> v.startsWith("✔")).count();
                results.forEach((id, res) -> publish("  " + id + "  →  " + res));
                publish("────────────────────────────────────────");
                publish("Done: " + ok + "/" + results.size() + " imported successfully.");
                return null;
            }
            @Override protected void process(List<String> chunks) { chunks.forEach(MainFrame.this::log); }
            @Override protected void done() {
                try { get(); } catch (Exception ex) { showError("Import error: " + ex.getMessage()); }
            }
        };
        w.execute();
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private void buildJiraService() {
        jiraService = new JiraServiceImpl(tfUrl.getText().trim(), new String(tfToken.getPassword()));
    }

    private boolean validateJiraFields() {
        if (tfUrl.getText().isBlank())         { showError("* Jira URL is required.");               return false; }
        if (tfToken.getPassword().length == 0) { showError("* Personal Access Token is required."); return false; }
        return true;
    }

    private void refreshTable(List<TestCase> list) {
        tableModel.setRowCount(0);
        for (TestCase tc : list) {
            tableModel.addRow(new Object[]{
                tc.testCaseIdentifier, tc.summary, tc.priority, tc.assignee,
                tc.steps.size() + " step(s)", tc.environment, tc.testType,
                tc.labels, tc.userStory
            });
        }
    }

    private void log(String msg) {
        SwingUtilities.invokeLater(() -> {
            logArea.append(msg + "\n");
            logArea.setCaretPosition(logArea.getDocument().getLength());
        });
    }

    private void showError(String msg) {
        JOptionPane.showMessageDialog(this, msg, "Error", JOptionPane.ERROR_MESSAGE);
    }

    /**
     * Adds a label + field row to a GridBagLayout panel.
     * label: col 0,       weightx=0
     * field: cols 1..(1+extra), weightx=1.0  ← stretches to fill available width
     */
    private void addRow(JPanel p, GridBagConstraints c, int row,
                        String lbl, JComponent field, int extra) {
        c.gridx = 0; c.gridy = row; c.gridwidth = 1; c.weightx = 0;
        p.add(label(lbl), c);
        c.gridx = 1; c.gridwidth = 1 + extra; c.weightx = 1.0;
        p.add(field, c);
        c.weightx = 0; c.gridwidth = 1;
    }

    /**
     * Creates an action button with a fixed preferred width (120 px) so all
     * buttons in the right column are the same size regardless of label text length.
     */
    private JButton uniformButton(String text) {
        JButton btn = new JButton(text);
        btn.setPreferredSize(new Dimension(120, btn.getPreferredSize().height));
        return btn;
    }

    private JLabel label(String t) { return new JLabel(t); }

    private GridBagConstraints defaultGbc() {
        GridBagConstraints c = new GridBagConstraints();
        c.insets = new Insets(5, 8, 5, 8);
        c.anchor = GridBagConstraints.WEST;
        c.fill   = GridBagConstraints.HORIZONTAL;
        return c;
    }

    private <T extends JPanel> T titled(String title, T panel) {
        panel.setBorder(new TitledBorder(title));
        return panel;
    }

    private DefaultListCellRenderer fileNameRenderer() {
        return new DefaultListCellRenderer() {
            @Override public Component getListCellRendererComponent(
                    JList<?> list, Object value, int index, boolean isSelected, boolean hasFocus) {
                super.getListCellRendererComponent(list, value, index, isSelected, hasFocus);
                if (value instanceof File f) setText(f.getName());
                return this;
            }
        };
    }
}
```

---

## sample-config.json

```json
{
    "projectKey": { "project": "FCMUS" },
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
    "projectSelection": { "project": "true" },
    "projectName": {},
    "value": {}
}
```

---

## sample-testcases.csv

```
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

## UI Alignment Fix — What Changed and Why

### Root Cause
In the original `addRow()`, passing `extra=2` made the field span **columns 1, 2, and 3**.
The action button was then placed at `gridx=3`, overlapping the field — causing misaligned buttons.

### Column Layout (after fix)
```
Col 0 (label)  | Col 1 ── Col 2 (field, weightx=1) | Col 3 (button, weightx=0)
───────────────┼───────────────────────────────────┼──────────────────────────
"* Jira URL:"  │  tfUrl  (extra=1 → cols 1-2)      │  [Test Connection]
"* PAT:"       │  tfToken (extra=2 → cols 1-3, no button)
Checkbox       │  (gridwidth=4, spans all cols)
CardPanel      │  (gridwidth=4, spans all cols)
"* CSV Folder:"│  tfCsvFolder (extra=1 → cols 1-2) │  [Browse…]
"* CSV File:"  │  cbCsvFiles  (extra=1 → cols 1-2) │  [Preview]
"* Config File"│  tfConfigFile (extra=1 → cols 1-2)│  [Browse…]
```

### Key code changes
1. **`addRow()` sets `weightx=1.0` on the field** so it stretches horizontally.
2. **Rows with buttons use `extra=1`** (field spans cols 1–2 only, button gets col 3).
3. **`uniformButton(text)`** gives every action button a fixed 120 px preferred width,
   so the right column stays neat regardless of label text length.
4. **Checkbox and CardPanel rows** use `gridwidth=4, weightx=1` to span full width.

---

## How to Build and Run

```bash
mvn clean package
java -jar target/jira-testcase-importer.jar
```
