# GitHub Copilot Prompt — Jira Test Case Importer (estjira) v6
# COMPLETE PROJECT — Generate All Files From Scratch

> **Instructions for Copilot:** Generate the complete Maven Java 17 project described below,
> file by file, exactly as specified. Do NOT omit any file. After generating all files,
> create a ZIP archive named `jira-testcase-importer.zip`.

---

## 1. Project Overview

A Java 17 Swing desktop application that:

1. Reads an **"xray bulk upload importer" JSON config file** to get CSV column → Jira field mappings.
   The **project key is extracted automatically** from `projectKey.project` in this file.
2. Reads **CSV files** where rows sharing the same **Test Case Identifier** form one test case with multiple steps.
3. Each CSV row may include a **`User Story`** column specifying which Jira story to link to.
4. On import, either:
   - **Adds test cases to an EXISTING Test Execution** by entering its Jira ID — **default/checked state**, or
   - **Creates a NEW Test Execution** by entering a summary title — unchecked state.
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

```java
package com.estjira.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.LinkedHashMap;
import java.util.Map;

@JsonIgnoreProperties(ignoreUnknown = true)
public class FieldConfig {
    public Map<String, String> projectKey       = new LinkedHashMap<>();
    public Map<String, String> field            = new LinkedHashMap<>();
    public Map<String, String> link             = new LinkedHashMap<>();
    public Map<String, String> projectSelection = new LinkedHashMap<>();
    public Map<String, String> projectName      = new LinkedHashMap<>();
    public Map<String, String> value            = new LinkedHashMap<>();

    /** Returns the project key string (e.g. "FCMUS") from the config file. */
    public String resolvedProjectKey() {
        return projectKey.getOrDefault("project", "");
    }
}
```

No getters/setters — public fields only.

---

### 5.3 `model/TestCase.java`

```java
package com.estjira.model;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Rows sharing the same testCaseIdentifier are grouped into one TestCase.
 * Metadata comes from the first row. Every row contributes one step.
 */
public class TestCase {
    // Identity & metadata — from FIRST row of the group
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

    // From CSV "User Story" column (per test case)
    public String userStory;

    // One String[2] per CSV row: [0]=stepNo, [1]=action
    public List<String[]> steps = new ArrayList<>();

    // Dynamic Jira payload — from first row only; step-only & internal fields never here
    public Map<String, String> jiraFields = new LinkedHashMap<>();
}
```

No getters/setters — public fields only.

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

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.File;

public class ConfigServiceImpl implements ConfigService {
    @Override
    public FieldConfig load(File configFile) throws Exception {
        if (configFile == null || !configFile.exists()) {
            throw new Exception("Config file not found: " + configFile);
        }
        FieldConfig config = new ObjectMapper().readValue(configFile, FieldConfig.class);
        if (config.field == null || config.field.isEmpty()) {
            throw new Exception("Config file has no 'field' mappings.");
        }
        return config;
    }
}
```

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

### 5.7 `service/CsvServiceImpl.java` ← CRITICAL — READ EVERY LINE

```java
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import com.opencsv.CSVReader;

import java.io.File;
import java.io.FileReader;
import java.util.*;

public class CsvServiceImpl implements CsvService {

    @Override
    public List<TestCase> parse(File file, FieldConfig config, String userStory) throws Exception {
        Map<String, TestCase> grouped = new LinkedHashMap<>();

        try (CSVReader reader = new CSVReader(new FileReader(file))) {
            String[] headers = reader.readNext();
            if (headers == null) throw new Exception("CSV file is empty: " + file.getName());

            Map<String, Integer> colIdx = new HashMap<>();
            for (int i = 0; i < headers.length; i++) {
                colIdx.put(headers[i].trim().toLowerCase(), i);
            }

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

                    // CSV "User Story" column takes precedence; fallback used if absent/blank
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
                        // *** CRITICAL BUG FIX ***
                        // "Test Case Identifier" maps to itself in the xray config
                        // (e.g. "Test Case Identifier" : "Test Case Identifier").
                        // It is an INTERNAL GROUPING KEY — NOT a real Jira field ID.
                        // It must NEVER be written to tc.jiraFields.
                        // Sending it to Jira causes HTTP 400: "Field cannot be set."
                    } else if (isFirstRow) {
                        tc.jiraFields.put(jiraField, value);
                        populateNamedField(tc, csvHeader, value);
                    }
                }

                if (!stepNo.isBlank() || !action.isBlank()) {
                    tc.steps.add(new String[]{stepNo, action});
                }
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
        for (Map.Entry<String, String> entry : config.field.entrySet()) {
            if (entry.getKey().trim().equalsIgnoreCase("Test Case Identifier")) {
                return getCsvValue(row, colIdx, entry.getKey());
            }
        }
        return "";
    }

    private String getCsvValue(String[] row, Map<String, Integer> colIdx, String csvHeader) {
        return getCsvValueByHeader(row, colIdx, csvHeader.trim().toLowerCase());
    }

    private String getCsvValueByHeader(String[] row, Map<String, Integer> colIdx, String lowerHeader) {
        Integer idx = colIdx.get(lowerHeader);
        if (idx != null && idx < row.length) return row[idx].trim();
        return "";
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
            case "test case identifier"                -> { /* already set as tc.testCaseIdentifier */ }
            case "test type"                           -> tc.testType           = value;
            case "outward issue link (test)"           -> tc.outwardIssueLink   = value;
            case "custom field (test repository path)" -> tc.testRepositoryPath = value;
            case "step no.", "action"                  -> { /* handled as steps, never here */ }
            case "user story"                          -> { /* handled before config loop */ }
        }
    }
}
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

### 5.9 `service/JiraServiceImpl.java` ← CRITICAL — READ EVERY LINE

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

public class JiraServiceImpl implements JiraService {

    private static final Logger LOG = Logger.getLogger(JiraServiceImpl.class.getName());

    private static final String LINK_PREFIX         = "link-";
    private static final String TEST_EXECUTION_TYPE = "Test Execution";

    /**
     * *** CRITICAL BUG FIX ***
     * These values appear as Jira "field ID" mappings in the xray config file,
     * but they are NOT real Jira field IDs. Sending them in the create-issue
     * payload causes HTTP 400: "Field cannot be set / unknown field."
     *
     * "Test Case Identifier" — internal grouping key, maps to itself in the config
     * "Action"               — step content, written to tc.steps, not to Jira fields
     * "Step Attachments"     — step number alias used by xray config for "Step no."
     *
     * These MUST be skipped in buildPayload() before the field switch statement.
     */
    private static final Set<String> INTERNAL_FIELD_IDS = Set.of(
            "Test Case Identifier",
            "Action",
            "Step Attachments"
    );

    private final String       baseUrl;
    private final String       auth;
    private final ObjectMapper json = new ObjectMapper();
    private final HttpClient   http = HttpClient.newHttpClient();

    public JiraServiceImpl(String baseUrl, String pat) {
        this.baseUrl = baseUrl.replaceAll("/$", "");
        this.auth    = "Bearer " + pat;
    }

    // ── Interface ──────────────────────────────────────────────────────────────

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
                                      String executionSummary, String label) {
        try {
            ObjectNode fields = json.createObjectNode();
            fields.putObject("project").put("key", projectKey);
            fields.putObject("issuetype").put("name", TEST_EXECUTION_TYPE);
            fields.put("summary", executionSummary);
            if (label != null && !label.isBlank()) {
                fields.putArray("labels").add(label);
            }

            String body = json.writeValueAsString(json.createObjectNode().set("fields", fields));
            HttpResponse<String> res = send(post("/rest/api/2/issue", body));

            if (res.statusCode() == 201) {
                String execKey = json.readTree(res.body()).path("key").asText();
                LOG.info("Test Execution created: " + execKey);

                if (userStory != null && !userStory.isBlank()) {
                    ObjectNode link = json.createObjectNode();
                    link.putObject("type").put("name", "relates to");
                    link.putObject("inwardIssue").put("key", execKey);
                    link.putObject("outwardIssue").put("key", userStory);
                    send(post("/rest/api/2/issueLink", json.writeValueAsString(link)));
                }
                return execKey;
            } else {
                LOG.warning("Failed to create Test Execution: HTTP " + res.statusCode()
                        + " — " + extractError(res.body()));
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
                HttpResponse<String> res = send(post("/rest/api/2/issue",
                        buildPayload(tc, projectKey, issueType)));

                if (res.statusCode() == 201) {
                    String key = json.readTree(res.body()).path("key").asText();
                    results.put(id, "✔ " + key);
                    if (testExecutionKey != null && !testExecutionKey.isBlank()) {
                        addTestToExecution(testExecutionKey, key);
                    }
                    handleOutwardLinks(tc, key);
                    if (tc.userStory != null && !tc.userStory.isBlank()) {
                        linkToUserStory(key, tc.userStory);
                    }
                } else {
                    String errDetail = extractError(res.body());
                    if (res.statusCode() == 400) {
                        errDetail += " | HTTP 400: a field in the payload is not on the " +
                                "Create Issue screen for this project/issue type, or the " +
                                "field ID is invalid. Check your Jira screen scheme for " +
                                "custom fields (e.g. customfield_25902, customfield_17400).";
                    }
                    results.put(id, "✘ HTTP " + res.statusCode() + " — " + errDetail);
                }
            } catch (Exception e) {
                results.put(id, "✘ " + e.getMessage());
            }
        }
        return results;
    }

    // ── Payload builder ────────────────────────────────────────────────────────

    private String buildPayload(TestCase tc, String projectKey, String issueType)
            throws Exception {

        ObjectNode fields = json.createObjectNode();
        fields.putObject("project").put("key", projectKey);
        fields.putObject("issuetype").put("name", issueType);

        for (Map.Entry<String, String> entry : tc.jiraFields.entrySet()) {
            String fieldId = entry.getKey();
            String value   = entry.getValue();

            // Skip outward-link fields — handled separately
            if (fieldId.startsWith(LINK_PREFIX)) continue;

            // *** CRITICAL: skip internal xray config keys that are not real Jira field IDs ***
            // Sending these causes HTTP 400 "Field cannot be set / unknown field"
            if (INTERNAL_FIELD_IDS.contains(fieldId)) continue;

            switch (fieldId) {
                case "summary"     -> fields.put("summary", value);
                case "description" -> fields.put("description", value);
                case "priority"    -> fields.putObject("priority").put("name", value);
                case "assignee"    -> fields.putObject("assignee").put("name", value);
                case "reporter"    -> fields.putObject("reporter").put("name", value);
                case "environment" -> fields.put("environment", value);
                case "labels"      -> {
                    ArrayNode arr = fields.putArray("labels");
                    Arrays.stream(value.split("[,;]"))
                          .map(String::trim).filter(s -> !s.isEmpty())
                          .forEach(arr::add);
                }
                default -> {
                    // customfield_* etc. — included in payload.
                    // NOTE: if the field is not on the Create Issue screen scheme in Jira,
                    // the entire request will fail with HTTP 400. The user must add the
                    // custom field to the project's screen scheme in Jira admin.
                    LOG.fine("Including custom field: " + fieldId + " = " + value);
                    fields.put(fieldId, value);
                }
            }
        }

        // Fallback summary
        if (!fields.has("summary") || fields.get("summary").asText().isBlank()) {
            String s = tc.summary != null ? tc.summary : tc.testCaseIdentifier;
            fields.put("summary", s != null ? s : "Test Case");
        }

        // Append steps to description — MANDATORY
        if (!tc.steps.isEmpty()) {
            String existingDesc = fields.has("description")
                    ? fields.get("description").asText() : "";
            StringBuilder sb = new StringBuilder();
            if (!existingDesc.isBlank()) sb.append(existingDesc).append("\n\n");
            sb.append("*Test Steps:*\n");
            for (int i = 0; i < tc.steps.size(); i++) {
                String[] step  = tc.steps.get(i);
                String stepNo  = (step[0] != null && !step[0].isBlank()) ? step[0] : String.valueOf(i + 1);
                String action  = (step[1] != null && !step[1].isBlank()) ? step[1] : "";
                sb.append(stepNo).append(". ").append(action).append("\n");
            }
            fields.put("description", sb.toString().stripTrailing());
        }

        return json.writeValueAsString(json.createObjectNode().set("fields", fields));
    }

    // ── Xray & linking helpers ─────────────────────────────────────────────────

    private void addTestToExecution(String execKey, String testKey) {
        try {
            ObjectNode body = json.createObjectNode();
            body.putArray("add").add(testKey);
            HttpResponse<String> res = send(post(
                    "/rest/raven/1.0/api/testexecution/" + execKey + "/test",
                    json.writeValueAsString(body)));
            if (res.statusCode() != 200 && res.statusCode() != 201) {
                LOG.warning("Failed to add " + testKey + " to execution "
                        + execKey + ": HTTP " + res.statusCode());
            }
        } catch (Exception e) {
            LOG.warning("addTestToExecution error: " + e.getMessage());
        }
    }

    private void linkToUserStory(String testKey, String userStoryKey) {
        try {
            ObjectNode body = json.createObjectNode();
            body.putObject("type").put("name", "relates to");
            body.putObject("inwardIssue").put("key", testKey);
            body.putObject("outwardIssue").put("key", userStoryKey);
            HttpResponse<String> res = send(post("/rest/api/2/issueLink",
                    json.writeValueAsString(body)));
            if (res.statusCode() != 201 && res.statusCode() != 204) {
                LOG.warning("Failed to link " + testKey + " → User Story "
                        + userStoryKey + ": HTTP " + res.statusCode());
            }
        } catch (Exception e) {
            LOG.warning("linkToUserStory error: " + e.getMessage());
        }
    }

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
            } catch (Exception e) {
                LOG.warning("handleOutwardLinks error: " + e.getMessage());
            }
        }
    }

    // ── HTTP helpers ───────────────────────────────────────────────────────────

    private HttpResponse<String> send(HttpRequest req) throws Exception {
        return http.send(req, HttpResponse.BodyHandlers.ofString());
    }

    private HttpRequest get(String path) {
        return base(path).GET().build();
    }

    private HttpRequest post(String path, String body) {
        return base(path).POST(HttpRequest.BodyPublishers.ofString(body))
                .header("Content-Type", "application/json").build();
    }

    private HttpRequest.Builder base(String path) {
        return HttpRequest.newBuilder(URI.create(baseUrl + path))
                .header("Authorization", auth)
                .header("Accept", "application/json");
    }

    private String extractError(String responseBody) {
        try {
            var root = json.readTree(responseBody);
            if (root.has("errorMessages") && root.get("errorMessages").size() > 0) {
                return root.get("errorMessages").get(0).asText();
            }
            if (root.has("errors")) return root.get("errors").toString();
        } catch (Exception ignored) {}
        return responseBody;
    }
}
```

---

### 5.10 `ui/MainFrame.java` ← CRITICAL — READ EVERY LINE

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

    // CardLayout card names
    private static final String CARD_EXISTING = "EXISTING";  // default (checkbox checked)
    private static final String CARD_NEW      = "NEW";        // checkbox unchecked

    // Services
    private final CsvService    csvService    = new CsvServiceImpl();
    private final ConfigService configService = new ConfigServiceImpl();
    private JiraService jiraService;

    // Jira connection
    private final JTextField     tfUrl   = new JTextField("https://estjira.example.com", 30);
    private final JPasswordField tfToken = new JPasswordField(24);

    // Execution mode
    // Checkbox is TRUE by default → shows CARD_EXISTING on startup
    private final JCheckBox  cbExistingTestExecution = new JCheckBox("Existing Test Execution", true);
    private final CardLayout executionCardLayout     = new CardLayout();
    private final JPanel     executionCardPanel      = new JPanel(executionCardLayout);

    // CARD_EXISTING field — default visible (checkbox starts checked)
    private final JTextField tfExistingExecutionId = new JTextField(28);

    // CARD_NEW field — visible when checkbox is unchecked
    private final JTextField tfExecutionSummary = new JTextField(28);

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

    // Log
    private final JTextArea logArea = new JTextArea(7, 70);

    // State
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
        JPanel p = new JPanel(new GridLayout(2, 1, 6, 6));
        p.add(buildJiraSettingsPanel());
        p.add(buildFileSelectionPanel());
        return p;
    }

    private JPanel buildJiraSettingsPanel() {
        JPanel panel = titled("Jira Settings", new JPanel(new GridBagLayout()));
        GridBagConstraints c = defaultGbc();

        // Row 0: URL + Test Connection
        addRow(panel, c, 0, "* Jira URL:", tfUrl, 2);
        c.gridx = 3; c.gridy = 0; c.gridwidth = 1; c.fill = GridBagConstraints.NONE;
        JButton btnTest = new JButton("Test Connection");
        btnTest.addActionListener(e -> onTestConnection());
        panel.add(btnTest, c);
        c.fill = GridBagConstraints.HORIZONTAL;

        // Row 1: PAT
        addRow(panel, c, 1, "* Personal Access Token:", tfToken, 2);

        // Row 2: Checkbox full width
        c.gridx = 0; c.gridy = 2; c.gridwidth = 4;
        cbExistingTestExecution.setToolTipText(
                "Checked: add to an existing Test Execution — enter its Jira ID.\n" +
                "Unchecked: create a new Test Execution — enter a summary title.");
        cbExistingTestExecution.addActionListener(e -> onToggleExecutionMode());
        panel.add(cbExistingTestExecution, c);

        // Row 3: CardLayout — EXISTING or NEW sub-panel
        buildExecutionCardPanel();
        c.gridx = 0; c.gridy = 3; c.gridwidth = 4;
        panel.add(executionCardPanel, c);

        return panel;
    }

    private void buildExecutionCardPanel() {
        // CARD_EXISTING — default (checkbox starts checked)
        JPanel existingCard = new JPanel(new GridBagLayout());
        GridBagConstraints c1 = defaultGbc();
        addRow(existingCard, c1, 0, "* Existing Test Execution ID:", tfExistingExecutionId, 2);

        // CARD_NEW — shown when checkbox is unchecked
        JPanel newCard = new JPanel(new GridBagLayout());
        GridBagConstraints c2 = defaultGbc();
        addRow(newCard, c2, 0, "* Test Execution Summary:", tfExecutionSummary, 2);

        // EXISTING must be added first — it is the default card
        executionCardPanel.add(existingCard, CARD_EXISTING);
        executionCardPanel.add(newCard,      CARD_NEW);

        // Show EXISTING on startup (checkbox default = true)
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

        tfCsvFolder.setEditable(false);
        addRow(panel, c, 0, "* CSV Folder:", tfCsvFolder, 2);
        c.gridx = 3; c.gridy = 0; c.gridwidth = 1; c.fill = GridBagConstraints.NONE;
        JButton btnF = new JButton("Browse…"); btnF.addActionListener(e -> onBrowseCsvFolder());
        panel.add(btnF, c); c.fill = GridBagConstraints.HORIZONTAL;

        cbCsvFiles.setRenderer(fileNameRenderer());
        addRow(panel, c, 1, "* CSV File:", cbCsvFiles, 2);
        c.gridx = 3; c.gridy = 1; c.gridwidth = 1; c.fill = GridBagConstraints.NONE;
        JButton btnP = new JButton("Preview"); btnP.addActionListener(e -> onPreview());
        panel.add(btnP, c); c.fill = GridBagConstraints.HORIZONTAL;

        tfConfigFile.setEditable(false);
        addRow(panel, c, 2, "* Config File:", tfConfigFile, 2);
        c.gridx = 3; c.gridy = 2; c.gridwidth = 1; c.fill = GridBagConstraints.NONE;
        JButton btnC = new JButton("Browse…"); btnC.addActionListener(e -> onBrowseConfigFile());
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

    // Actions

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
                log("Config loaded: " + loadedConfig.field.size()
                        + " field mapping(s) — project: " + loadedConfig.resolvedProjectKey());
            } catch (Exception ex) {
                showError("Failed to load config: " + ex.getMessage());
                loadedConfig = null;
            }
        }
    }

    private void onPreview() {
        File sel = (File) cbCsvFiles.getSelectedItem();
        if (sel == null)          { showError("Please select a CSV file."); return; }
        if (loadedConfig == null) { showError("Please load a config file first."); return; }

        SwingWorker<List<TestCase>, Void> w = new SwingWorker<>() {
            @Override protected List<TestCase> doInBackground() throws Exception {
                // Pass empty string — User Story comes from CSV column, not from UI
                return csvService.parse(sel, loadedConfig, "");
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
                    if (get()) {
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

        // Project key is extracted from the config file — there is no UI field for it
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
                    // userStory and label are empty — removed from UI
                    execKey = jiraService.createTestExecution(projectKey, "", executionSummary, "");
                    publish(execKey != null
                            ? "✔ Test Execution created: " + execKey
                            : "⚠ Could not create — continuing without linking.");
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

    // Helpers

    private void buildJiraService() {
        jiraService = new JiraServiceImpl(
                tfUrl.getText().trim(), new String(tfToken.getPassword()));
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

    private void addRow(JPanel p, GridBagConstraints c, int row, String lbl,
                        JComponent field, int extra) {
        c.gridx = 0; c.gridy = row; c.gridwidth = 1; p.add(new JLabel(lbl), c);
        c.gridx = 1; c.gridwidth = 1 + extra;         p.add(field, c);
        c.gridwidth = 1;
    }

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
                    JList<?> list, Object value, int index,
                    boolean isSelected, boolean hasFocus) {
                super.getListCellRendererComponent(list, value, index, isSelected, hasFocus);
                if (value instanceof File f) setText(f.getName());
                return this;
            }
        };
    }
}
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
| 6 | Auth: **`Authorization: Bearer <PAT>`** — no username, no Basic auth |
| 7 | All network calls run in `SwingWorker` — never block the EDT |
| 8 | `cbCsvFiles` dropdown renders **filename only** (`file.getName()`) |
| 9 | Call `setMinimumSize(getSize())` after `pack()` |
| 10 | `SwingWorker.process()` used in `onImport()` for real-time streaming |
| 11 | Config file browser uses `FileNameExtensionFilter` for `.json` only |
| 12 | Indent all Java with **4 spaces** |
| 13 | **No `cbIssueType`** — issue type hardcoded as `"Test"` |
| 14 | **No `tfProjectKey`** — project key from `loadedConfig.resolvedProjectKey()` only |
| 15 | **No username field** — PAT is self-identifying |
| 16 | Xray link: `POST /rest/raven/1.0/api/testexecution/{key}/test` with `{"add":["KEY"]}` |
| 17 | **No `tfUserStory` or `tfLabel`** — removed from UI entirely |
| 18 | **`cbExistingTestExecution` is checked by default** — `new JCheckBox("Existing Test Execution", true)` |
| 19 | **Default startup state**: checkbox checked → `CARD_EXISTING` visible → "* Existing Test Execution ID:" shown |
| 20 | **`buildExecutionCardPanel()` MUST register both cards**: `executionCardPanel.add(existingCard, CARD_EXISTING)` AND `executionCardPanel.add(newCard, CARD_NEW)`. Start: `executionCardLayout.show(executionCardPanel, CARD_EXISTING)` |
| 21 | **GROUPING IS MANDATORY**: rows with same Test Case Identifier = one TestCase; metadata from first row; every row adds one step |
| 22 | **Steps MUST appear in Jira description**: `"*Test Steps:*\n1. action\n2. action..."` appended after existing description |
| 23 | Preview table: **one row per grouped TestCase**; 9 columns; `"Steps"` shows `"N step(s)"` |
| 24 | **`"User Story"` CSV column** sets `tc.userStory` per test case; `onPreview()` passes `""` as fallback |
| 25 | **`linkToUserStory(testKey, storyKey)`** — `POST /rest/api/2/issueLink`, `type.name = "relates to"` |
| 26 | `createTestExecution()` called with **empty strings**: `jiraService.createTestExecution(projectKey, "", executionSummary, "")` |
| 27 | **BUG FIX — `INTERNAL_FIELD_IDS` skip-set in `JiraServiceImpl`**: The set `{"Test Case Identifier", "Action", "Step Attachments"}` MUST be declared as a `private static final Set<String>` and checked with `if (INTERNAL_FIELD_IDS.contains(fieldId)) continue;` in `buildPayload()` BEFORE the switch statement. These are xray config values that look like Jira field IDs but are not. Sending them causes HTTP 400. |
| 28 | **BUG FIX — `"test case identifier"` skip in `CsvServiceImpl.parse()`**: In the config field loop, add an explicit `else if (headerLc.equals("test case identifier")) { /* skip */ }` branch BEFORE the `else if (isFirstRow)` branch. This prevents `"Test Case Identifier"` from being written to `tc.jiraFields` and subsequently sent to Jira. |
| 29 | **HTTP 400 error message** in `importAll()` must include a human-readable hint: "a field in the payload is not on the Create Issue screen for this project/issue type" so users know to check their Jira screen scheme |
| 30 | **`onBrowseConfigFile()` log** must include the resolved project key: `"Config loaded: N field mapping(s) — project: KEY"` |
