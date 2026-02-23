# GitHub Copilot Prompt — Jira Test Case Importer (estjira) v9
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
5. Imports all grouped test cases as Jira `Test` issues.
6. Adds each test's steps via the **Xray REST API v1** step endpoint — NOT via the Jira description field.
7. Adds each imported test to the Test Execution via the Xray REST API v1.
8. Links each imported test to its per-row User Story from the CSV.

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
                        // *** CRITICAL BUG FIX — Rule 28 ***
                        // Internal grouping key — NEVER write to tc.jiraFields
                        // Sending it causes HTTP 400: "Field cannot be set."
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
     * *** CRITICAL BUG FIX — Rule 27 ***
     * These appear as Jira "field ID" values in the xray config but are NOT real Jira field IDs.
     * Sending them causes HTTP 400 "Field cannot be set / unknown field."
     * MUST be skipped in buildPayload() BEFORE the switch statement.
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
                    // Rule 42: steps via Xray REST API v1 — NOT in Jira description
                    if (!tc.steps.isEmpty()) {
                        addXraySteps(key, tc.steps);
                    }
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

    private String buildPayload(TestCase tc, String projectKey, String issueType)
            throws Exception {
        ObjectNode fields = json.createObjectNode();
        fields.putObject("project").put("key", projectKey);
        fields.putObject("issuetype").put("name", issueType);

        for (Map.Entry<String, String> entry : tc.jiraFields.entrySet()) {
            String fieldId = entry.getKey();
            String value   = entry.getValue();

            if (fieldId.startsWith(LINK_PREFIX)) continue;          // Rule 5
            if (INTERNAL_FIELD_IDS.contains(fieldId)) continue;    // Rule 27

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

        // Rule 42: DO NOT append steps to description — steps go via addXraySteps()

        return json.writeValueAsString(json.createObjectNode().set("fields", fields));
    }

    /**
     * Rule 42: Xray REST API v1 — POST /rest/raven/1.0/api/test/{testKey}/step
     * Sends one step at a time; API appends them in order.
     * Body: { "action": "...", "data": "", "result": "" }
     * DO NOT put steps in the Jira description field.
     */
    private void addXraySteps(String testKey, List<String[]> steps) {
        for (String[] step : steps) {
            try {
                String action = (step[1] != null && !step[1].isBlank()) ? step[1] : "(no action)";
                ObjectNode body = json.createObjectNode();
                body.put("action", action);
                body.put("data",   "");
                body.put("result", "");
                HttpResponse<String> res = send(post(
                        "/rest/raven/1.0/api/test/" + testKey + "/step",
                        json.writeValueAsString(body)));
                if (res.statusCode() != 200 && res.statusCode() != 201) {
                    LOG.warning("Failed to add step to " + testKey
                            + ": HTTP " + res.statusCode() + " — " + res.body());
                }
            } catch (Exception e) {
                LOG.warning("addXraySteps error for " + testKey + ": " + e.getMessage());
            }
        }
    }

    /**
     * Rule 16: CORRECT Xray v1 endpoint is /rest/raven/1.0/api/testexec/{key}/test
     * NOT /testexecution/ — that path does not exist and returns 404.
     * Body: { "add": ["TEST-KEY"] }
     */
    private void addTestToExecution(String execKey, String testKey) {
        try {
            ObjectNode body = json.createObjectNode();
            body.putArray("add").add(testKey);
            HttpResponse<String> res = send(post(
                    "/rest/raven/1.0/api/testexec/" + execKey + "/test",
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
            // Rule 25: POST /rest/api/2/issueLink, type.name = "relates to"
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
                .header("Authorization", auth)  // Rule 6: Bearer PAT
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
import javax.swing.table.DefaultTableCellRenderer;
import javax.swing.table.DefaultTableModel;
import javax.swing.table.JTableHeader;
import java.awt.*;
import java.io.File;
import java.util.List;
import java.util.Map;

public class MainFrame extends JFrame {

    // ── UI Constants ──────────────────────────────────────────────────────────
    private static final Color  ACCENT_COLOR = new Color(0, 82, 204);
    private static final Color  SUCCESS_COLOR= new Color(0, 135, 90);
    private static final Color  HEADER_BG   = new Color(37, 56, 88);
    private static final Color  HEADER_FG   = Color.WHITE;
    private static final Color  ROW_ALT     = new Color(244, 246, 250);
    private static final Color  STATUS_BG   = new Color(247, 248, 250);
    private static final Font   MONO_FONT   = new Font(Font.MONOSPACED, Font.PLAIN, 12);
    private static final Font   LABEL_FONT  = new Font(Font.SANS_SERIF, Font.PLAIN, 13);
    private static final Font   BOLD_FONT   = new Font(Font.SANS_SERIF, Font.BOLD, 13);
    private static final Font   HEADER_FONT = new Font(Font.SANS_SERIF, Font.BOLD, 14);

    // CardLayout card names
    private static final String CARD_EXISTING = "EXISTING";  // default (checkbox checked)
    private static final String CARD_NEW      = "NEW";

    // Services
    private final CsvService    csvService    = new CsvServiceImpl();
    private final ConfigService configService = new ConfigServiceImpl();
    private JiraService jiraService;

    // Jira connection
    private final JTextField     tfUrl   = new JTextField("https://estjira.example.com", 32);
    private final JPasswordField tfToken = new JPasswordField(24);

    // Execution mode — Rule 18: checkbox TRUE by default
    private final JCheckBox  cbExistingTestExecution = new JCheckBox("Use Existing Test Execution", true);
    private final CardLayout executionCardLayout     = new CardLayout();
    private final JPanel     executionCardPanel      = new JPanel(executionCardLayout);

    private final JTextField tfExistingExecutionId = new JTextField(28);  // CARD_EXISTING
    private final JTextField tfExecutionSummary    = new JTextField(28);  // CARD_NEW

    // File selection
    private final JTextField      tfCsvFolder  = new JTextField(26);
    private final JComboBox<File> cbCsvFiles   = new JComboBox<>();
    private final JTextField      tfConfigFile = new JTextField(26);

    // Preview table — Rule 23: 9 columns, one row per grouped TestCase
    private final DefaultTableModel tableModel = new DefaultTableModel(
            new String[]{"Identifier", "Summary", "Priority", "Assignee", "Steps",
                         "Environment", "Test Type", "Labels", "User Story"}, 0) {
        @Override public boolean isCellEditable(int r, int c) { return false; }
    };

    private final JLabel       statusBar   = new JLabel("Ready", SwingConstants.LEFT);
    private final JTextArea    logArea     = new JTextArea(8, 70);
    private final JProgressBar progressBar = new JProgressBar();

    private List<TestCase> loadedTestCases;
    private FieldConfig    loadedConfig;

    public MainFrame() {
        super("Jira Test Case Importer — estjira v1.0");
        setDefaultCloseOperation(EXIT_ON_CLOSE);

        JPanel root = new JPanel(new BorderLayout(0, 0));
        root.add(buildAppHeader(), BorderLayout.NORTH);

        JPanel body = new JPanel(new BorderLayout(0, 8));
        body.setBorder(new EmptyBorder(10, 14, 6, 14));
        body.add(buildFormPanel(),    BorderLayout.NORTH);
        body.add(buildPreviewPanel(), BorderLayout.CENTER);
        body.add(buildBottomPanel(),  BorderLayout.SOUTH);
        root.add(body,             BorderLayout.CENTER);
        root.add(buildStatusBar(), BorderLayout.SOUTH);

        setContentPane(root);
        pack();
        setMinimumSize(getSize());   // Rule 9
        setLocationRelativeTo(null);
    }

    private JPanel buildAppHeader() {
        JPanel header = new JPanel(new BorderLayout());
        header.setBackground(HEADER_BG);
        header.setBorder(new EmptyBorder(10, 16, 10, 16));
        JLabel title = new JLabel("Jira Test Case Importer");
        title.setFont(new Font(Font.SANS_SERIF, Font.BOLD, 17));
        title.setForeground(HEADER_FG);
        JLabel subtitle = new JLabel("estjira  ·  Xray Bulk Upload");
        subtitle.setFont(new Font(Font.SANS_SERIF, Font.PLAIN, 12));
        subtitle.setForeground(new Color(180, 200, 230));
        JPanel left = new JPanel(new GridLayout(2, 1, 1, 1));
        left.setOpaque(false);
        left.add(title);
        left.add(subtitle);
        header.add(left, BorderLayout.WEST);
        return header;
    }

    private JPanel buildStatusBar() {
        JPanel bar = new JPanel(new BorderLayout(8, 0));
        bar.setBackground(STATUS_BG);
        bar.setBorder(BorderFactory.createCompoundBorder(
                BorderFactory.createMatteBorder(1, 0, 0, 0, new Color(210, 215, 220)),
                new EmptyBorder(4, 12, 4, 12)));
        statusBar.setFont(new Font(Font.SANS_SERIF, Font.PLAIN, 12));
        statusBar.setForeground(new Color(80, 90, 100));
        progressBar.setVisible(false);
        progressBar.setPreferredSize(new Dimension(150, 16));
        bar.add(statusBar,   BorderLayout.CENTER);
        bar.add(progressBar, BorderLayout.EAST);
        return bar;
    }

    // ── Single flat form panel ─────────────────────────────────────────────────
    //
    //  Rule 33: All configuration in ONE GridBagLayout panel.
    //  Col 0: right-aligned label (weightx=0, EAST anchor)
    //  Col 1: field / combo       (weightx=1, fill HORIZONTAL)
    //  Col 2: button or filler    (weightx=0, fill NONE)

    private JPanel buildFormPanel() {
        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBorder(BorderFactory.createCompoundBorder(
                BorderFactory.createTitledBorder(
                        BorderFactory.createLineBorder(new Color(200, 210, 225)),
                        "Configuration", TitledBorder.LEFT, TitledBorder.TOP, BOLD_FONT, ACCENT_COLOR),
                new EmptyBorder(4, 6, 6, 6)));

        int row = 0;

        addSectionLabel(panel, "— Jira Connection", row++);

        tfUrl.setFont(LABEL_FONT);
        JButton btnTest = accentButton("Test Connection");
        btnTest.addActionListener(e -> onTestConnection());
        addFormRow(panel, row++, "Jira URL:", tfUrl, btnTest);

        tfToken.setFont(LABEL_FONT);
        addFormRow(panel, row++, "Personal Access Token:", tfToken, null);

        addSeparator(panel, row++);
        addSectionLabel(panel, "— Files", row++);

        tfCsvFolder.setEditable(false);
        tfCsvFolder.setFont(LABEL_FONT);
        JButton btnBrowseFolder = new JButton("Browse…");
        btnBrowseFolder.addActionListener(e -> onBrowseCsvFolder());
        addFormRow(panel, row++, "CSV Folder:", tfCsvFolder, btnBrowseFolder);

        cbCsvFiles.setFont(LABEL_FONT);
        cbCsvFiles.setRenderer(fileNameRenderer());
        JButton btnPreview = accentButton("Preview ▶");
        btnPreview.addActionListener(e -> onPreview());
        addFormRow(panel, row++, "CSV File:", cbCsvFiles, btnPreview);

        tfConfigFile.setEditable(false);
        tfConfigFile.setFont(LABEL_FONT);
        JButton btnBrowseConfig = new JButton("Browse…");
        btnBrowseConfig.addActionListener(e -> onBrowseConfigFile());
        addFormRow(panel, row++, "Config File (.json):", tfConfigFile, btnBrowseConfig);

        addSeparator(panel, row++);
        addSectionLabel(panel, "— Test Execution", row++);

        cbExistingTestExecution.setFont(BOLD_FONT);
        cbExistingTestExecution.setForeground(ACCENT_COLOR);
        cbExistingTestExecution.setToolTipText(
                "Checked: add tests to an existing Test Execution (enter its Jira ID).\n" +
                "Unchecked: create a new Test Execution (enter a summary title).");
        cbExistingTestExecution.addActionListener(e -> onToggleExecutionMode());
        GridBagConstraints cbGbc = formGbc(row++, 1, 2);
        cbGbc.fill = GridBagConstraints.NONE;
        panel.add(cbExistingTestExecution, cbGbc);

        buildExecutionCardPanel();
        addFormRow(panel, row, "", executionCardPanel, null);

        return panel;
    }

    /** Small grey section-heading spanning all 3 columns. */
    private void addSectionLabel(JPanel p, String text, int row) {
        JLabel lbl = new JLabel(text);
        lbl.setFont(new Font(Font.SANS_SERIF, Font.BOLD, 11));
        lbl.setForeground(new Color(120, 130, 145));
        GridBagConstraints c = new GridBagConstraints();
        c.gridx = 0; c.gridy = row; c.gridwidth = 3;
        c.insets = new Insets(6, 4, 0, 4);
        c.anchor = GridBagConstraints.WEST;
        p.add(lbl, c);
    }

    /** Full-width JSeparator spanning all 3 columns. */
    private void addSeparator(JPanel p, int row) {
        JSeparator sep = new JSeparator();
        GridBagConstraints c = new GridBagConstraints();
        c.gridx = 0; c.gridy = row; c.gridwidth = 3;
        c.fill = GridBagConstraints.HORIZONTAL;
        c.insets = new Insets(4, 0, 2, 0);
        p.add(sep, c);
    }

    /**
     * Rule 39: Strict 3-column form row.
     *   col 0 → right-aligned label (EAST anchor, fixed)
     *   col 1 → field (weightx=1, fill HORIZONTAL)
     *   col 2 → button (fill NONE), or invisible 110px JPanel filler when btn==null
     */
    private void addFormRow(JPanel p, int row, String labelText, JComponent field, JButton btn) {
        if (!labelText.isBlank()) {
            JLabel lbl = new JLabel(labelText);
            lbl.setFont(LABEL_FONT);
            lbl.setHorizontalAlignment(SwingConstants.RIGHT);
            GridBagConstraints lc = new GridBagConstraints();
            lc.gridx = 0; lc.gridy = row; lc.gridwidth = 1;
            lc.anchor = GridBagConstraints.EAST;
            lc.insets = new Insets(4, 8, 4, 6);
            p.add(lbl, lc);
        }
        GridBagConstraints fc = new GridBagConstraints();
        fc.gridx = 1; fc.gridy = row;
        fc.fill = GridBagConstraints.HORIZONTAL;
        fc.weightx = 1.0;
        fc.insets = new Insets(4, 0, 4, btn != null ? 4 : 8);
        p.add(field, fc);

        GridBagConstraints bc = new GridBagConstraints();
        bc.gridx = 2; bc.gridy = row; bc.gridwidth = 1;
        bc.fill = GridBagConstraints.NONE;
        bc.anchor = GridBagConstraints.WEST;
        bc.insets = new Insets(4, 0, 4, 8);
        if (btn != null) {
            p.add(btn, bc);
        } else {
            JPanel filler = new JPanel();
            filler.setOpaque(false);
            filler.setPreferredSize(new Dimension(110, 1));
            p.add(filler, bc);
        }
    }

    /** GBC spanning colStart..colStart+colSpan-1, for checkbox and card panel rows. */
    private GridBagConstraints formGbc(int row, int colStart, int colSpan) {
        GridBagConstraints c = new GridBagConstraints();
        c.gridx = colStart; c.gridy = row; c.gridwidth = colSpan;
        c.anchor = GridBagConstraints.WEST;
        c.fill   = GridBagConstraints.HORIZONTAL;
        c.weightx = 1.0;
        c.insets = new Insets(4, 0, 4, 8);
        return c;
    }

    /** Rule 41: cards use BorderLayout — fixed-width label WEST, text field CENTER. */
    private void buildExecutionCardPanel() {
        JPanel existingCard = new JPanel(new BorderLayout());
        existingCard.setOpaque(false);
        tfExistingExecutionId.setFont(LABEL_FONT);
        tfExistingExecutionId.setToolTipText("e.g. FCMUS-99");
        JLabel lbl1 = new JLabel("Execution ID:");
        lbl1.setFont(LABEL_FONT);
        lbl1.setPreferredSize(new Dimension(160, 26));
        lbl1.setHorizontalAlignment(SwingConstants.RIGHT);
        existingCard.add(lbl1, BorderLayout.WEST);
        existingCard.add(tfExistingExecutionId, BorderLayout.CENTER);

        JPanel newCard = new JPanel(new BorderLayout());
        newCard.setOpaque(false);
        tfExecutionSummary.setFont(LABEL_FONT);
        tfExecutionSummary.setToolTipText("Title for the new Test Execution issue");
        JLabel lbl2 = new JLabel("Execution Summary:");
        lbl2.setFont(LABEL_FONT);
        lbl2.setPreferredSize(new Dimension(160, 26));
        lbl2.setHorizontalAlignment(SwingConstants.RIGHT);
        newCard.add(lbl2, BorderLayout.WEST);
        newCard.add(tfExecutionSummary, BorderLayout.CENTER);

        executionCardPanel.add(existingCard, CARD_EXISTING);  // FIRST = default
        executionCardPanel.add(newCard,      CARD_NEW);
        executionCardLayout.show(executionCardPanel, CARD_EXISTING);  // Rule 19
    }

    private void onToggleExecutionMode() {
        boolean useExisting = cbExistingTestExecution.isSelected();
        executionCardLayout.show(executionCardPanel, useExisting ? CARD_EXISTING : CARD_NEW);
        setStatus(useExisting
                ? "Mode: Add to EXISTING Test Execution — enter its Jira ID."
                : "Mode: Create NEW Test Execution — enter a summary title.");
    }

    private JPanel buildPreviewPanel() {
        JTable table = new JTable(tableModel);
        table.setFillsViewportHeight(true);
        table.setRowHeight(24);
        table.setFont(LABEL_FONT);
        table.setShowGrid(false);
        table.setIntercellSpacing(new Dimension(0, 0));
        table.getTableHeader().setReorderingAllowed(false);

        table.setDefaultRenderer(Object.class, new DefaultTableCellRenderer() {
            @Override public Component getTableCellRendererComponent(
                    JTable t, Object v, boolean sel, boolean foc, int row, int col) {
                super.getTableCellRendererComponent(t, v, sel, foc, row, col);
                setBorder(BorderFactory.createEmptyBorder(2, 6, 2, 6));
                if (!sel) setBackground(row % 2 == 0 ? Color.WHITE : ROW_ALT);
                return this;
            }
        });

        // Rule 43: MUST use th.setDefaultRenderer — th.setBackground/setForeground are
        // silently ignored by system L&Fs. setOpaque(true) is mandatory inside the renderer.
        JTableHeader th = table.getTableHeader();
        th.setPreferredSize(new Dimension(0, 28));
        th.setDefaultRenderer(new DefaultTableCellRenderer() {
            @Override public Component getTableCellRendererComponent(
                    JTable t, Object v, boolean sel, boolean foc, int row, int col) {
                super.getTableCellRendererComponent(t, v, sel, foc, row, col);
                setBackground(HEADER_BG);
                setForeground(HEADER_FG);
                setFont(HEADER_FONT);
                setHorizontalAlignment(SwingConstants.LEFT);
                setBorder(BorderFactory.createCompoundBorder(
                    BorderFactory.createMatteBorder(0, 0, 0, 1, new Color(60, 80, 110)),
                    BorderFactory.createEmptyBorder(0, 6, 0, 6)));
                setOpaque(true);
                return this;
            }
        });

        int[] widths = {70, 200, 65, 90, 65, 90, 80, 80, 80};
        for (int i = 0; i < widths.length; i++) {
            table.getColumnModel().getColumn(i).setPreferredWidth(widths[i]);
        }

        JScrollPane scroll = new JScrollPane(table);
        scroll.setBorder(BorderFactory.createLineBorder(new Color(210, 215, 220)));
        scroll.setPreferredSize(new Dimension(760, 160));

        JPanel panel = titled("Test Cases Preview", new JPanel(new BorderLayout()));
        panel.add(scroll, BorderLayout.CENTER);
        return panel;
    }

    private JPanel buildBottomPanel() {
        JPanel panel = new JPanel(new BorderLayout(4, 6));

        JButton btnImport = importButton("▶  Import to Jira");
        btnImport.addActionListener(e -> onImport());
        // Rule 38: Clear Log uses styledButton() (not accentButton)
        JButton btnClear = styledButton("Clear Log");
        btnClear.addActionListener(e -> { logArea.setText(""); setStatus("Log cleared."); });

        JPanel btnRow = new JPanel(new FlowLayout(FlowLayout.CENTER, 10, 2));
        btnRow.add(btnImport);
        btnRow.add(btnClear);
        panel.add(btnRow, BorderLayout.NORTH);

        logArea.setEditable(false);
        logArea.setFont(MONO_FONT);
        logArea.setBackground(new Color(30, 34, 40));
        logArea.setForeground(new Color(190, 210, 190));
        logArea.setCaretColor(Color.WHITE);
        logArea.setMargin(new Insets(6, 8, 6, 8));
        logArea.setLineWrap(true);
        logArea.setWrapStyleWord(true);

        JScrollPane logScroll = new JScrollPane(logArea);
        logScroll.setBorder(BorderFactory.createLineBorder(new Color(60, 70, 80)));
        logScroll.setPreferredSize(new Dimension(760, 130));

        JPanel logPanel = titled("Import Log", new JPanel(new BorderLayout()));
        logPanel.add(logScroll, BorderLayout.CENTER);
        panel.add(logPanel, BorderLayout.CENTER);

        return panel;
    }

    // ── Actions ────────────────────────────────────────────────────────────────

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
            String msg = "Found " + files.length + " CSV file(s) in: " + folder.getName();
            log(msg); setStatus(msg);
        }
    }

    private void onBrowseConfigFile() {
        JFileChooser ch = new JFileChooser();
        ch.setDialogTitle("Select xray bulk upload importer config (.json)");
        // Rule 11: JSON-only filter
        ch.setFileFilter(new javax.swing.filechooser.FileNameExtensionFilter(
                "JSON Config File (*.json)", "json"));
        if (ch.showOpenDialog(this) == JFileChooser.APPROVE_OPTION) {
            File f = ch.getSelectedFile();
            tfConfigFile.setText(f.getAbsolutePath());
            try {
                loadedConfig = configService.load(f);
                // Rule 30: log must include resolved project key
                String msg = "Config loaded: " + loadedConfig.field.size()
                        + " field mapping(s) — project: " + loadedConfig.resolvedProjectKey();
                log(msg); setStatus("✔ " + msg);
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
        setBusy(true, "Parsing CSV…");
        // Rule 7: all IO in SwingWorker
        SwingWorker<List<TestCase>, Void> w = new SwingWorker<>() {
            @Override protected List<TestCase> doInBackground() throws Exception {
                // Rule 24: pass "" as userStory fallback — comes from CSV column
                return csvService.parse(sel, loadedConfig, "");
            }
            @Override protected void done() {
                setBusy(false, null);
                try {
                    loadedTestCases = get();
                    refreshTable(loadedTestCases);
                    int rows = loadedTestCases.stream().mapToInt(tc -> tc.steps.size()).sum();
                    String msg = "Preview: " + loadedTestCases.size() + " test case(s) from "
                            + rows + " CSV row(s) — " + sel.getName();
                    log(msg); setStatus("✔ " + msg);
                } catch (Exception ex) { showError("Failed to parse CSV: " + ex.getMessage()); }
            }
        };
        w.execute();
    }

    private void onTestConnection() {
        if (!validateJiraFields()) return;
        buildJiraService();
        setBusy(true, "Testing connection…");
        SwingWorker<Boolean, Void> w = new SwingWorker<>() {
            @Override protected Boolean doInBackground() { return jiraService.testConnection(); }
            @Override protected void done() {
                setBusy(false, null);
                try {
                    if (get()) {
                        log("✔ Connection successful!"); setStatus("✔ Connected.");
                        JOptionPane.showMessageDialog(MainFrame.this,
                                "Connected to Jira successfully.", "Connection OK",
                                JOptionPane.INFORMATION_MESSAGE);
                    } else {
                        log("✘ Connection failed."); setStatus("✘ Connection failed.");
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

        // Rule 14: project key from config only — no UI field
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
        setBusy(true, "Importing…");

        // Rule 10: SwingWorker.process() for real-time log streaming
        SwingWorker<Void, String> w = new SwingWorker<>() {
            @Override protected Void doInBackground() {
                String execKey;
                if (useExisting) {
                    execKey = existingExecutionId;
                    publish("Using existing Test Execution: " + execKey);
                } else {
                    publish("Creating Test Execution '" + executionSummary
                            + "' in project '" + projectKey + "'…");
                    // Rule 26: call with empty strings for removed UI fields
                    execKey = jiraService.createTestExecution(projectKey, "", executionSummary, "");
                    publish(execKey != null
                            ? "✔ Test Execution created: " + execKey
                            : "⚠ Could not create execution — continuing without linking.");
                }
                publish("Importing " + loadedTestCases.size() + " test case(s)…");
                // Rule 13: issueType hardcoded as "Test"
                Map<String, String> results = jiraService.importAll(
                        loadedTestCases, projectKey, "Test", execKey, loadedConfig);
                long ok = results.values().stream().filter(v -> v.startsWith("✔")).count();
                results.forEach((id, res) -> publish("  " + id + "  →  " + res));
                publish("────────────────────────────────────────────────────────");
                publish("Done: " + ok + "/" + results.size() + " imported successfully.");
                return null;
            }
            @Override protected void process(List<String> chunks) { chunks.forEach(MainFrame.this::log); }
            @Override protected void done() {
                setBusy(false, null);
                try { get(); } catch (Exception ex) {
                    showError("Import error: " + ex.getMessage());
                }
            }
        };
        w.execute();
    }

    // ── Helpers ────────────────────────────────────────────────────────────────

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

    private void setStatus(String msg) {
        SwingUtilities.invokeLater(() -> statusBar.setText(msg));
    }

    private void setBusy(boolean busy, String statusMsg) {
        SwingUtilities.invokeLater(() -> {
            progressBar.setVisible(busy);
            progressBar.setIndeterminate(busy);
            if (statusMsg != null) setStatus(statusMsg);
        });
    }

    private void showError(String msg) {
        JOptionPane.showMessageDialog(this, msg, "Error", JOptionPane.ERROR_MESSAGE);
        setStatus("⚠ " + msg);
    }

    private <T extends JPanel> T titled(String title, T panel) {
        TitledBorder border = BorderFactory.createTitledBorder(
                BorderFactory.createLineBorder(new Color(200, 210, 225), 1), title);
        border.setTitleFont(BOLD_FONT);
        border.setTitleColor(ACCENT_COLOR);
        panel.setBorder(BorderFactory.createCompoundBorder(
                border, new EmptyBorder(4, 4, 4, 4)));
        return panel;
    }

    /** Plain unstyled button — consistent font, no color override. Use for Browse… and Clear Log. */
    private JButton styledButton(String text) {
        JButton btn = new JButton(text);
        btn.setFocusPainted(false);
        btn.setFont(LABEL_FONT);
        btn.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        return btn;
    }

    /**
     * Rule 44: setOpaque(true) + setBorderPainted(false) are MANDATORY.
     * Without them, setBackground/setForeground are silently ignored on most system L&Fs.
     */
    private JButton accentButton(String text) {
        JButton btn = new JButton(text);
        btn.setBackground(ACCENT_COLOR);
        btn.setForeground(Color.WHITE);
        btn.setOpaque(true);
        btn.setBorderPainted(false);
        btn.setFocusPainted(false);
        btn.setFont(BOLD_FONT);
        btn.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        return btn;
    }

    private JButton importButton(String text) {
        JButton btn = new JButton(text);
        btn.setBackground(SUCCESS_COLOR);
        btn.setForeground(Color.WHITE);
        btn.setOpaque(true);
        btn.setBorderPainted(false);
        btn.setFocusPainted(false);
        btn.setFont(new Font(Font.SANS_SERIF, Font.BOLD, 14));
        btn.setPreferredSize(new Dimension(200, 36));
        btn.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        return btn;
    }

    // Rule 8: cbCsvFiles renders filename only
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
| 16 | **Xray add-to-execution endpoint: `POST /rest/raven/1.0/api/testexec/{key}/test`** — body `{"add":["KEY"]}`. The path is `/testexec/` — NOT `/testexecution/` (404) |
| 17 | **No `tfUserStory` or `tfLabel`** — removed from UI entirely |
| 18 | **`cbExistingTestExecution` is checked by default** — `new JCheckBox("Use Existing Test Execution", true)` |
| 19 | **Default startup state**: checkbox checked → `CARD_EXISTING` visible → Existing ID field shown |
| 20 | **`buildExecutionCardPanel()` MUST register both cards**: `CARD_EXISTING` first, then `CARD_NEW`. Start: `executionCardLayout.show(executionCardPanel, CARD_EXISTING)` |
| 21 | **GROUPING IS MANDATORY**: rows with same Test Case Identifier = one TestCase; metadata from first row; every row adds one step |
| 22 | **Steps go through the Xray REST API — NOT the Jira description field.** See Rule 42. |
| 23 | Preview table: **one row per grouped TestCase**; 9 columns; `"Steps"` shows `"N step(s)"` |
| 24 | **`"User Story"` CSV column** sets `tc.userStory` per test case; `onPreview()` passes `""` as fallback |
| 25 | **`linkToUserStory(testKey, storyKey)`** — `POST /rest/api/2/issueLink`, `type.name = "relates to"` |
| 26 | `createTestExecution()` called with **empty strings**: `jiraService.createTestExecution(projectKey, "", executionSummary, "")` |
| 27 | **BUG FIX — `INTERNAL_FIELD_IDS` skip-set**: `{"Test Case Identifier", "Action", "Step Attachments"}` declared `private static final Set<String>` — checked with `if (INTERNAL_FIELD_IDS.contains(fieldId)) continue;` in `buildPayload()` BEFORE the switch |
| 28 | **BUG FIX — `"test case identifier"` skip in `CsvServiceImpl.parse()`**: explicit `else if (headerLc.equals("test case identifier")) { /* skip */ }` branch BEFORE `else if (isFirstRow)` |
| 29 | **HTTP 400 error message** in `importAll()` must include human-readable hint about Create Issue screen scheme |
| 30 | **`onBrowseConfigFile()` log** must include the resolved project key: `"Config loaded: N field mapping(s) — project: KEY"` |
| 31 | **UI**: App has a dark **header banner** (`#253858`) with title and subtitle labels |
| 32 | **UI**: A **status bar** at the bottom shows feedback; `JProgressBar` (indeterminate) visible when busy |
| 33 | **UI**: All configuration in a **single flat `GridBagLayout` form panel** — NO `GridLayout(1,2)` side-by-side sub-panels. Three columns: label col 0 (`EAST`), field col 1 (`weightx=1`), button/filler col 2 (`fill=NONE`) |
| 34 | **UI**: Log area dark-themed (bg `#1E2228`, fg `#BED2BE`), monospaced font, line wrap on |
| 35 | **UI**: Preview table alternating row colors and preset column widths |
| 36 | **UI**: Import button green (`#00875A`), accent buttons Jira blue (`#0052CC`), both white text |
| 37 | **UI**: `TitledBorder` with accent blue title and `BOLD_FONT` |
| 38 | **UI**: `styledButton("Clear Log")` beside Import button — uses `styledButton()` (no color override), NOT `accentButton()` |
| 39 | **UI**: `addFormRow(panel, row, label, field, btn)` — label col 0 `EAST`, field col 1 `weightx=1 fill=HORIZONTAL`, btn col 2 `fill=NONE`. When `btn==null`, add invisible `JPanel` filler 110px wide in col 2 |
| 40 | **UI**: `addSectionLabel()` spans 3 cols, small grey bold font. `addSeparator()` inserts `JSeparator` spanning 3 cols |
| 41 | **UI**: CardLayout cards use `new JPanel(new BorderLayout())` — fixed 160px-wide `JLabel` in `WEST`, text field in `CENTER`. No nested `GridBagLayout` inside cards |
| 42 | **XRAY STEPS — CRITICAL**: After `POST /rest/api/2/issue` → 201, call `addXraySteps(key, tc.steps)`. Each step: `POST /rest/raven/1.0/api/test/{testKey}/step` body `{"action":"...","data":"","result":""}` one at a time. `buildPayload()` must **NOT** touch `tc.steps` at all — no steps in description |
| 43 | **UI — TABLE HEADER**: `JTableHeader.setBackground()` ignored by system L&Fs. MUST use `th.setDefaultRenderer(new DefaultTableCellRenderer(){...})` — call `setBackground()`, `setForeground()`, `setOpaque(true)` inside the renderer |
| 44 | **UI — BUTTON COLORS**: `JButton.setBackground()` ignored by system L&Fs. All colored buttons MUST call `btn.setOpaque(true)` AND `btn.setBorderPainted(false)` — without these the custom color is invisible |
