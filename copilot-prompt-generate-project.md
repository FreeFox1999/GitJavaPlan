# GitHub Copilot Prompt — Jira Test Case Importer: Full Project Generator

## What This Prompt Does

Paste this entire prompt into GitHub Copilot Chat. It contains the **complete, ready-to-run Python script** that generates every source file of the Jira Test Case Importer project and packages it into `jira-testcase-importer.zip`.

**No additional code generation is needed.** Copy the script below, save it as `generate_project.py`, and run it.

---

## UI Alignment Fixes Verified ✅

The following bugs were identified and fixed in `MainFrame.java`:

| # | Bug | Fix |
|---|-----|-----|
| 1 | `addRow(extra=2)` on button rows caused field to span cols 1–3, overlapping button at col 3 | Changed to `extra=1` on all rows that have a trailing action button |
| 2 | `weightx` never set → text fields didn't stretch when window resized | `addRow()` now sets `weightx=1.0` on the field, `0` on the label |
| 3 | Buttons had inconsistent widths based on label text length | New `uniformButton(text)` helper enforces 120 px preferred width on all 4 action buttons |

### Column layout after fix:
```
Col 0 (label)    │ Col 1 ── Col 2 (field, weightx=1.0) │ Col 3 (button, weightx=0)
─────────────────┼──────────────────────────────────────┼───────────────────────────
* Jira URL:      │ tfUrl          (extra=1)             │ [Test Connection]
* Personal AT:   │ tfToken        (extra=2, no button)
Checkbox         │ (gridwidth=4, spans all cols)
CardPanel        │ (gridwidth=4, spans all cols)
* CSV Folder:    │ tfCsvFolder    (extra=1)             │ [Browse…]
* CSV File:      │ cbCsvFiles     (extra=1)             │ [Preview]
* Config File:   │ tfConfigFile   (extra=1)             │ [Browse…]
```

---

## Generated Project Structure

```
jira-testcase-importer/
├── pom.xml                          Maven build — Java 17, opencsv 5.9, jackson-databind 2.17.1
├── sample-config.json               Example xray bulk upload importer field-mapping config
├── sample-testcases.csv             Example CSV with 4 test cases and 13 rows
└── src/main/java/com/estjira/
    ├── Main.java                    Entry point — sets system L&F, launches MainFrame
    ├── model/
    │   ├── TestCase.java            Data model — fields + steps list + jiraFields map
    │   └── FieldConfig.java         Config model — CSV header → Jira field ID mapping
    ├── service/
    │   ├── CsvService.java          Interface
    │   ├── CsvServiceImpl.java      Groups CSV rows by identifier; reads steps from every row
    │   ├── ConfigService.java       Interface
    │   ├── ConfigServiceImpl.java   Loads JSON config via Jackson
    │   ├── JiraService.java         Interface
    │   └── JiraServiceImpl.java     REST client — test conn, create execution, import, link
    └── ui/
        └── MainFrame.java           ✅ UI FIXED — GridBagLayout buttons aligned correctly
```

---

## The Script — Save as `generate_project.py` and Run

```python
#!/usr/bin/env python3
"""
generate_project.py
Run this script to generate the full jira-testcase-importer Maven project
and package it as jira-testcase-importer.zip

  python generate_project.py
"""
import os, zipfile

ROOT = "jira-testcase-importer"
FILES = {}

FILES["pom.xml"] = """\
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

"""

FILES["sample-config.json"] = """\
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

"""

FILES["sample-testcases.csv"] = """\
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

"""

FILES["src/main/java/com/estjira/Main.java"] = """\
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

"""

FILES["src/main/java/com/estjira/model/TestCase.java"] = """\
package com.estjira.model;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Represents one complete test case, which may span multiple CSV rows.
 *
 * <p>Rows sharing the same {@code testCaseIdentifier} are grouped into a single
 * TestCase. Metadata fields (summary, priority, etc.) are taken from the first
 * row of the group. Every row in the group contributes one entry to {@code steps}.
 *
 * <p>Named fields drive the preview table.
 * {@code jiraFields} drives the Jira REST API payload.
 * {@code steps} drives the formatted step table in the issue description.
 */
public class TestCase {

    // ── Identity & metadata (from first row of the group) ─────────────────────
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

    // ── Linked User Story key (e.g. FCMUS-42) ────────────────────────────────
    public String userStory;

    // ── Steps — one entry per CSV row in this group ───────────────────────────
    // Each String[2]: [0] = step number, [1] = action/description of the step
    public List<String[]> steps = new ArrayList<>();

    // ── Dynamic payload map: Jira field ID → raw value ────────────────────────
    // Populated from the first row; step-only fields (Action, Step no.) excluded.
    public Map<String, String> jiraFields = new LinkedHashMap<>();
}

"""

FILES["src/main/java/com/estjira/model/FieldConfig.java"] = """\
package com.estjira.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Represents the "xray bulk upload importer" config JSON file.
 *
 * Example config.json:
 * <pre>
 * {
 *   "projectKey"       : { "project": "FCMUS" },
 *   "field"            : {
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
 *   },
 *   "link"             : {},
 *   "projectSelection" : { "project": "true" },
 *   "projectName"      : {},
 *   "value"            : {}
 * }
 * </pre>
 *
 * The "field" map key   = CSV column header name
 * The "field" map value = Jira field ID / custom field ID
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public class FieldConfig {

    /** { "project": "FCMUS" } */
    public Map<String, String> projectKey   = new LinkedHashMap<>();

    /** CSV column header → Jira field ID */
    public Map<String, String> field        = new LinkedHashMap<>();

    public Map<String, String> link         = new LinkedHashMap<>();
    public Map<String, String> projectSelection = new LinkedHashMap<>();
    public Map<String, String> projectName  = new LinkedHashMap<>();
    public Map<String, String> value        = new LinkedHashMap<>();

    /** Convenience: returns the project key string (e.g. "FCMUS"). */
    public String resolvedProjectKey() {
        return projectKey.getOrDefault("project", "");
    }
}

"""

FILES["src/main/java/com/estjira/service/CsvService.java"] = """\
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;

import java.io.File;
import java.util.List;

/** Reads and parses CSV files into {@link TestCase} objects. */
public interface CsvService {

    /**
     * Parse a CSV file using the supplied field config for column mapping.
     *
     * @param file      CSV file to parse
     * @param config    field mapping config (CSV header → Jira field ID)
     * @param userStory Jira User Story key to link each test case
     * @return list of parsed test cases
     * @throws Exception if the file is unreadable or has no headers
     */
    List<TestCase> parse(File file, FieldConfig config, String userStory) throws Exception;

    /** List all CSV files inside the given folder. */
    File[] listFiles(File folder);
}

"""

FILES["src/main/java/com/estjira/service/CsvServiceImpl.java"] = """\
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;
import com.opencsv.CSVReader;

import java.io.File;
import java.io.FileReader;
import java.util.*;

/**
 * Parses CSV files into {@link TestCase} objects, grouping rows that share the
 * same Test Case Identifier into a single TestCase with multiple steps.
 *
 * <p><b>Grouping rule:</b> Every CSV row with the same value in the
 * "Test Case Identifier" column belongs to the same test case.
 * <ul>
 *   <li>Metadata fields (Summary, Priority, Assignee, etc.) are read from the
 *       <em>first</em> row of each group.</li>
 *   <li>Step fields ("Step no." and "Action") are collected from <em>every</em>
 *       row in the group and stored in {@code TestCase.steps}.</li>
 * </ul>
 *
 * <p>The CSV may include a <b>"User Story"</b> column to specify the Jira issue
 * each test case should be linked to. If present, it overrides the fallback
 * {@code userStory} parameter passed to {@link #parse}. If absent, the fallback
 * value is used for every test case.
 *
 * <p>Column-to-field mapping is driven entirely by the user-supplied
 * {@link FieldConfig} JSON config file.
 */
public class CsvServiceImpl implements CsvService {

    @Override
    public List<TestCase> parse(File file, FieldConfig config, String userStory) throws Exception {

        // Preserve insertion order so test cases appear in CSV order
        Map<String, TestCase> grouped = new LinkedHashMap<>();

        try (CSVReader reader = new CSVReader(new FileReader(file))) {

            String[] headers = reader.readNext();
            if (headers == null) {
                throw new Exception("CSV file is empty: " + file.getName());
            }

            // Build case-insensitive header → column-index map
            Map<String, Integer> colIdx = new HashMap<>();
            for (int i = 0; i < headers.length; i++) {
                colIdx.put(headers[i].trim().toLowerCase(), i);
            }

            // Check if the CSV contains a "User Story" column directly
            boolean csvHasUserStoryCol = colIdx.containsKey("user story");

            String[] row;
            while ((row = reader.readNext()) != null) {

                // ── Determine the test case identifier for this row ──────────
                String id = resolveIdentifier(row, colIdx, config);
                if (id.isBlank()) {
                    continue;   // skip rows with no identifier
                }

                // ── Get or create the TestCase for this identifier ───────────
                TestCase tc = grouped.get(id);
                boolean isFirstRow = (tc == null);

                if (isFirstRow) {
                    tc = new TestCase();
                    tc.testCaseIdentifier = id;

                    // User story from CSV column takes precedence over UI fallback
                    if (csvHasUserStoryCol) {
                        String csvStory = getCsvValueByHeader(row, colIdx, "user story");
                        tc.userStory = csvStory.isBlank() ? userStory : csvStory;
                    } else {
                        tc.userStory = userStory;
                    }

                    grouped.put(id, tc);
                }

                // ── Process each config field mapping ────────────────────────
                String stepNo  = "";
                String action  = "";

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
                        // Internal grouping key — never a real Jira field; already set on tc
                    } else if (isFirstRow) {
                        // Metadata only from the first row
                        tc.jiraFields.put(jiraField, value);
                        populateNamedField(tc, csvHeader, value);
                    }
                }

                // ── Add this row's step (every row contributes one step) ─────
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

    // ── Helpers ───────────────────────────────────────────────────────────────

    /**
     * Finds the value of the "Test Case Identifier" column for the given row,
     * using the config field map to locate the correct column header.
     */
    private String resolveIdentifier(String[] row, Map<String, Integer> colIdx,
                                     FieldConfig config) {
        for (Map.Entry<String, String> entry : config.field.entrySet()) {
            if (entry.getKey().trim().equalsIgnoreCase("Test Case Identifier")) {
                return getCsvValue(row, colIdx, entry.getKey());
            }
        }
        return "";
    }

    /** Returns a trimmed cell value by config-mapped CSV header, or "" if absent. */
    private String getCsvValue(String[] row, Map<String, Integer> colIdx, String csvHeader) {
        return getCsvValueByHeader(row, colIdx, csvHeader.trim().toLowerCase());
    }

    /** Returns a trimmed cell value by exact lowercase header name, or "" if absent. */
    private String getCsvValueByHeader(String[] row, Map<String, Integer> colIdx, String lowerHeader) {
        Integer idx = colIdx.get(lowerHeader);
        if (idx != null && idx < row.length) {
            return row[idx].trim();
        }
        return "";
    }

    /**
     * Maps well-known CSV column names to named fields on {@link TestCase}
     * so the preview table can display them without parsing jiraFields.
     *
     * <p>"User Story" is handled during row processing (before config field loop)
     * and is therefore not listed here to avoid double-assignment.
     */
    private void populateNamedField(TestCase tc, String csvHeader, String value) {
        switch (csvHeader.trim().toLowerCase()) {
            case "summary"                             -> tc.summary            = value;
            case "priority"                            -> tc.priority           = value;
            case "assignee"                            -> tc.assignee           = value;
            case "reporter"                            -> tc.reporter           = value;
            case "description"                         -> tc.description        = value;
            case "environment"                         -> tc.environment        = value;
            case "labels", "label"                     -> tc.labels             = value;
            case "test case identifier"                -> { /* already set */ }
            case "test type"                           -> tc.testType           = value;
            case "outward issue link (test)"           -> tc.outwardIssueLink   = value;
            case "custom field (test repository path)" -> tc.testRepositoryPath = value;
            case "user story"                          -> { /* handled before config loop */ }
        }
    }
}

"""

FILES["src/main/java/com/estjira/service/ConfigService.java"] = """\
package com.estjira.service;

import com.estjira.model.FieldConfig;

import java.io.File;

/** Reads and parses the xray bulk upload importer config JSON file. */
public interface ConfigService {

    /**
     * Load and parse a config JSON file into a {@link FieldConfig}.
     *
     * @param configFile the .json config file
     * @return parsed FieldConfig
     * @throws Exception if file is missing or malformed
     */
    FieldConfig load(File configFile) throws Exception;
}

"""

FILES["src/main/java/com/estjira/service/ConfigServiceImpl.java"] = """\
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.File;

/** Loads the xray bulk upload importer config JSON using Jackson. */
public class ConfigServiceImpl implements ConfigService {

    private final ObjectMapper json = new ObjectMapper();

    @Override
    public FieldConfig load(File configFile) throws Exception {
        if (configFile == null || !configFile.exists()) {
            throw new Exception("Config file not found: " + (configFile != null ? configFile.getPath() : "null"));
        }
        FieldConfig config = json.readValue(configFile, FieldConfig.class);
        if (config.field == null || config.field.isEmpty()) {
            throw new Exception("Config file has no 'field' mappings defined.");
        }
        return config;
    }
}

"""

FILES["src/main/java/com/estjira/service/JiraService.java"] = """\
package com.estjira.service;

import com.estjira.model.FieldConfig;
import com.estjira.model.TestCase;

import java.util.List;
import java.util.Map;

/** Communicates with the estjira Jira REST API. */
public interface JiraService {

    /** Returns true if credentials are valid and the instance is reachable. */
    boolean testConnection();

    /**
     * Creates a Test Execution issue in Jira (Xray).
     * This is called automatically before importing test cases.
     *
     * @param projectKey    Jira project key (e.g. "FCMUS")
     * @param userStory     User Story issue key to link the execution to (e.g. "FCMUS-42")
     * @param executionName Title/summary of the Test Execution issue — shown on the Jira board
     * @param label         Optional label tag on the Test Execution (e.g. "regression")
     * @return the created Test Execution issue key (e.g. "FCMUS-99"), or null on failure
     */
    String createTestExecution(String projectKey, String userStory, String executionName, String label);

    /**
     * Imports all test cases to Jira using the field config for payload construction.
     * Links each imported issue to the given Test Execution key.
     *
     * @param testCases        test cases to import
     * @param projectKey       Jira project key
     * @param issueType        Jira issue type (e.g. "Test")
     * @param testExecutionKey key of the Test Execution to link cases to
     * @param config           field mapping config
     * @return map of TestCase identifier → "✔ KEY" or "✘ error"
     */
    Map<String, String> importAll(
            List<TestCase> testCases,
            String projectKey,
            String issueType,
            String testExecutionKey,
            FieldConfig config
    );
}

"""

FILES["src/main/java/com/estjira/service/JiraServiceImpl.java"] = """\
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
 * REST implementation of {@link JiraService} targeting estjira.
 *
 * <p>Uses Java 17's built-in {@code java.net.http.HttpClient} with Jira REST API v2.
 * Authentication: Bearer token (Personal Access Token — no username required).
 */
public class JiraServiceImpl implements JiraService {

    private static final Logger LOG = Logger.getLogger(JiraServiceImpl.class.getName());

    /** Prefix used in config field values to denote outward issue link types. */
    private static final String LINK_PREFIX = "link-";

    /** Xray issue type name for test executions. */
    private static final String TEST_EXECUTION_TYPE = "Test Execution";

    private final String       baseUrl;
    private final String       auth;   // "Bearer <PAT>"
    private final ObjectMapper json = new ObjectMapper();
    private final HttpClient   http = HttpClient.newHttpClient();

    /**
     * @param baseUrl estjira base URL, e.g. {@code https://estjira.company.com}
     * @param pat     Personal Access Token (Jira Profile → Security → Personal Access Tokens)
     */
    public JiraServiceImpl(String baseUrl, String pat) {
        this.baseUrl = baseUrl.replaceAll("/$", "");
        this.auth    = "Bearer " + pat;
    }

    // ── Interface ─────────────────────────────────────────────────────────────

    @Override
    public boolean testConnection() {
        try {
            HttpResponse<String> res = send(get("/rest/api/2/myself"));
            return res.statusCode() == 200;
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
            fields.put("summary", executionName);   // user-entered title shown on Jira board

            if (!label.isBlank()) {
                fields.putArray("labels").add(label);   // optional tag — separate from title
            }

            String body = json.writeValueAsString(json.createObjectNode().set("fields", fields));
            HttpResponse<String> res = send(post("/rest/api/2/issue", body));

            if (res.statusCode() == 201) {
                String execKey = json.readTree(res.body()).path("key").asText();
                LOG.info("Test Execution created: " + execKey + " — " + executionName);

                // Link the Test Execution to the User Story via standard Jira issue link
                if (!userStory.isBlank()) {
                    ObjectNode linkBody = json.createObjectNode();
                    linkBody.putObject("type").put("name", "relates to");
                    linkBody.putObject("inwardIssue").put("key", execKey);
                    linkBody.putObject("outwardIssue").put("key", userStory);
                    send(post("/rest/api/2/issueLink", json.writeValueAsString(linkBody)));
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
    public Map<String, String> importAll(List<TestCase> testCases,
                                         String projectKey,
                                         String issueType,
                                         String testExecutionKey,
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

                    // Add test issue to the Test Execution using the Xray REST API
                    if (testExecutionKey != null && !testExecutionKey.isBlank()) {
                        addTestToExecution(testExecutionKey, key);
                    }

                    // Handle outward link fields (e.g. link-10900 → FCMUS-42)
                    handleOutwardLinks(tc, key);

                    // Link this test case to its User Story from the CSV (if set)
                    if (tc.userStory != null && !tc.userStory.isBlank()) {
                        linkToUserStory(key, tc.userStory);
                    }

                } else {
                    String errDetail = extractError(res.body());
                    if (res.statusCode() == 400) {
                        errDetail += " | HTTP 400: a field in the payload is not on the Create Issue " +
                                     "screen for this project/issue type, or the field ID is invalid. " +
                                     "Check your Jira screen scheme for any custom fields " +
                                     "(e.g. customfield_25902, customfield_17400).";
                    }
                    results.put(id, "✘ HTTP " + res.statusCode() + " — " + errDetail);
                }
            } catch (Exception e) {
                results.put(id, "✘ " + e.getMessage());
            }
        }
        return results;
    }

    // ── Payload builder ───────────────────────────────────────────────────────

    /**
     * Builds the Jira "create issue" JSON payload.
     *
     * <p>Standard fields (summary, priority, assignee, etc.) come from
     * {@code tc.jiraFields}. Steps collected from multiple CSV rows are
     * formatted as a numbered wiki-markup list and appended to the description.
     */
    /**
     * Field IDs that appear in the xray config as mapping values but are NOT
     * real Jira field IDs. Sending these in the create-issue payload causes
     * HTTP 400 "Field cannot be set / unknown field" errors.
     */
    private static final Set<String> INTERNAL_FIELD_IDS = Set.of(
            "Test Case Identifier",  // internal grouping key
            "Action",                // step content — goes into description
            "Step Attachments"       // step number — goes into description
    );

    private String buildPayload(TestCase tc, String projectKey, String issueType)
            throws Exception {

        ObjectNode fields = json.createObjectNode();
        fields.putObject("project").put("key", projectKey);
        fields.putObject("issuetype").put("name", issueType);

        // ── Map jiraFields → Jira API fields ──────────────────────────────────
        for (Map.Entry<String, String> entry : tc.jiraFields.entrySet()) {
            String fieldId = entry.getKey();
            String value   = entry.getValue();

            // Skip outward-link fields (handled by handleOutwardLinks)
            if (fieldId.startsWith(LINK_PREFIX)) continue;

            // Skip internal config keys that are NOT real Jira field IDs
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
                          .map(String::trim)
                          .filter(s -> !s.isEmpty())
                          .forEach(arr::add);
                }
                default -> {
                    // customfield_* and other unmapped fields — include in payload.
                    // If the field is not on the project's Create Issue screen scheme
                    // Jira will reject the entire request with HTTP 400.
                    // Log the field so the user knows what to add to the screen.
                    LOG.fine("Including custom field in payload: " + fieldId + " = " + value);
                    fields.put(fieldId, value);
                }
            }
        }

        // ── Fallback summary ──────────────────────────────────────────────────
        if (!fields.has("summary") || fields.get("summary").asText().isBlank()) {
            String s = tc.summary != null ? tc.summary : tc.testCaseIdentifier;
            fields.put("summary", s != null ? s : "Test Case");
        }

        // ── Append steps to description ───────────────────────────────────────
        // Steps from multiple CSV rows are formatted as a numbered list and
        // appended after any existing description text.
        if (!tc.steps.isEmpty()) {
            String existingDesc = fields.has("description")
                    ? fields.get("description").asText()
                    : "";

            StringBuilder sb = new StringBuilder();
            if (!existingDesc.isBlank()) {
                sb.append(existingDesc).append("\\n\\n");
            }
            sb.append("*Test Steps:*\\n");
            for (int i = 0; i < tc.steps.size(); i++) {
                String[] step   = tc.steps.get(i);
                String stepNo   = (step[0] != null && !step[0].isBlank()) ? step[0] : String.valueOf(i + 1);
                String action   = (step[1] != null && !step[1].isBlank()) ? step[1] : "";
                sb.append(stepNo).append(". ").append(action).append("\\n");
            }

            fields.put("description", sb.toString().stripTrailing());
        }

        return json.writeValueAsString(json.createObjectNode().set("fields", fields));
    }

    // ── Xray linking ──────────────────────────────────────────────────────────

    /**
     * Adds a test issue to a Test Execution via the Xray REST API.
     * {@code POST /rest/raven/1.0/api/testexecution/{execKey}/test}
     * Body: {@code { "add": ["TEST-KEY"] }}
     */
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

    /**
     * Links a newly created test issue to its associated User Story (from the CSV).
     * Uses link type "relates to" so both sides show the connection in Jira.
     */
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

    /**
     * Handles outward issue links defined in the config as {@code link-*} field IDs.
     * Uses the standard Jira issue link API.
     * {@code POST /rest/api/2/issueLink}
     */
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

                HttpResponse<String> res = send(post(
                        "/rest/api/2/issueLink",
                        json.writeValueAsString(body)));

                if (res.statusCode() != 201 && res.statusCode() != 204) {
                    LOG.warning("Outward link failed (" + createdKey + " → "
                            + linkedKey + "): HTTP " + res.statusCode());
                }
            } catch (Exception e) {
                LOG.warning("handleOutwardLinks error: " + e.getMessage());
            }
        }
    }

    // ── HTTP helpers ──────────────────────────────────────────────────────────

    private HttpResponse<String> send(HttpRequest req) throws Exception {
        return http.send(req, HttpResponse.BodyHandlers.ofString());
    }

    private HttpRequest get(String path) {
        return base(path).GET().build();
    }

    private HttpRequest post(String path, String body) {
        return base(path)
                .POST(HttpRequest.BodyPublishers.ofString(body))
                .header("Content-Type", "application/json")
                .build();
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
            if (root.has("errors")) {
                return root.get("errors").toString();
            }
        } catch (Exception ignored) {}
        return responseBody;
    }
}

"""

FILES["src/main/java/com/estjira/ui/MainFrame.java"] = """\
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

    // Execution mode — checkbox (checked by default = existing) + CardLayout switcher
    private final JCheckBox  cbExistingTestExecution = new JCheckBox("Existing Test Execution", true);
    private final CardLayout executionCardLayout      = new CardLayout();
    private final JPanel     executionCardPanel       = new JPanel(executionCardLayout);

    // EXISTING execution card field  (shown when checkbox is checked — the default)
    private final JTextField tfExistingExecutionId = new JTextField(28);

    // NEW execution card field  (shown when checkbox is unchecked)
    private final JTextField tfExecutionSummary = new JTextField(28);

    // File selection
    private final JTextField      tfCsvFolder  = new JTextField(26);
    private final JComboBox<File> cbCsvFiles   = new JComboBox<>();
    private final JTextField      tfConfigFile = new JTextField(26);

    // Preview table — 9 columns including "User Story" read per-row from CSV
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

    // Panel builders

    private JPanel buildTopPanel() {
        JPanel panel = new JPanel(new GridLayout(2, 1, 6, 6));
        panel.add(buildJiraSettingsPanel());
        panel.add(buildFileSelectionPanel());
        return panel;
    }

    private JPanel buildJiraSettingsPanel() {
        JPanel panel = titled("Jira Settings", new JPanel(new GridBagLayout()));
        GridBagConstraints c = defaultGbc();

        // Give the text-field column (col 1+2) all the extra horizontal weight
        c.weightx = 0; // reset — label col
        // (weightx is set per-component below via the helper)

        // Row 0: URL + Test Connection
        addRow(panel, c, 0, "* Jira URL:", tfUrl, 1);           // field spans cols 1-2
        c.gridx = 3; c.gridy = 0; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnTest = uniformButton("Test Connection");
        btnTest.addActionListener(e -> onTestConnection());
        panel.add(btnTest, c);

        // Row 1: PAT (no button — field spans cols 1-3)
        addRow(panel, c, 1, "* Personal Access Token:", tfToken, 2);

        // Row 2: Checkbox (full width) — checked by default → starts on EXISTING card
        c.gridx = 0; c.gridy = 2; c.gridwidth = 4; c.weightx = 1;
        cbExistingTestExecution.setToolTipText(
            "Checked: add test cases to an existing Test Execution (enter its Jira ID).\\n" +
            "Unchecked: create a brand-new Test Execution (enter a summary title).");
        cbExistingTestExecution.addActionListener(e -> onToggleExecutionMode());
        panel.add(cbExistingTestExecution, c);

        // Row 3: CardLayout panel with NEW vs EXISTING sub-panels
        buildExecutionCardPanel();
        c.gridx = 0; c.gridy = 3; c.gridwidth = 4; c.weightx = 1;
        panel.add(executionCardPanel, c);

        return panel;
    }

    private void buildExecutionCardPanel() {
        // ── Card: EXISTING execution (default — checkbox starts checked) ──────
        JPanel existingCard = new JPanel(new GridBagLayout());
        GridBagConstraints c1 = defaultGbc();
        addRow(existingCard, c1, 0, "* Existing Test Execution ID:", tfExistingExecutionId, 2);

        // ── Card: NEW execution (shown when checkbox is unchecked) ────────────
        JPanel newCard = new JPanel(new GridBagLayout());
        GridBagConstraints c2 = defaultGbc();
        addRow(newCard, c2, 0, "* Test Execution Summary:", tfExecutionSummary, 2);

        executionCardPanel.add(existingCard, CARD_EXISTING);
        executionCardPanel.add(newCard,      CARD_NEW);

        // Start on EXISTING card (checkbox is selected by default)
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
        addRow(panel, c, 0, "* CSV Folder:", tfCsvFolder, 1);       // field cols 1-2
        c.gridx = 3; c.gridy = 0; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnF = uniformButton("Browse…"); btnF.addActionListener(e -> onBrowseCsvFolder());
        panel.add(btnF, c);

        cbCsvFiles.setRenderer(fileNameRenderer());
        addRow(panel, c, 1, "* CSV File:", cbCsvFiles, 1);           // field cols 1-2
        c.gridx = 3; c.gridy = 1; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnP = uniformButton("Preview"); btnP.addActionListener(e -> onPreview());
        panel.add(btnP, c);

        tfConfigFile.setEditable(false);
        addRow(panel, c, 2, "* Config File:", tfConfigFile, 1);      // field cols 1-2
        c.gridx = 3; c.gridy = 2; c.gridwidth = 1;
        c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
        JButton btnC = uniformButton("Browse…"); btnC.addActionListener(e -> onBrowseConfigFile());
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
                log("Config loaded: " + loadedConfig.field.size() + " field mapping(s) found.");
            } catch (Exception ex) {
                showError("Failed to load config: " + ex.getMessage());
                loadedConfig = null;
            }
        }
    }

    private void onPreview() {
        File sel = (File) cbCsvFiles.getSelectedItem();
        if (sel == null)         { showError("Please select a CSV file."); return; }
        if (loadedConfig == null) { showError("Please load a config file first."); return; }

        // userStory fallback — comes from CSV "User Story" column; pass empty when no UI field
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
        if (loadedConfig == null) {
            showError("No config file loaded."); return;
        }
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

        // Project name is extracted from the loaded config file
        String execLine = useExisting
                ? "Add to existing Execution: " + existingExecutionId
                : "Create new Execution: '" + executionSummary + "' in project '" + projectKey + "'";

        int confirm = JOptionPane.showConfirmDialog(this,
                "This will:\\n  1. " + execLine + "\\n"
                + "  2. Import " + loadedTestCases.size() + " test case(s) into it\\n\\nProceed?",
                "Confirm Import", JOptionPane.YES_NO_OPTION);
        if (confirm != JOptionPane.YES_OPTION) return;

        buildJiraService();

        SwingWorker<Void, String> w = new SwingWorker<>() {
            @Override
            protected Void doInBackground() {
                String execKey;
                if (useExisting) {
                    execKey = existingExecutionId;
                    publish("Using existing Test Execution: " + execKey);
                } else {
                    publish("Creating Test Execution '" + executionSummary
                            + "' in project '" + projectKey + "'…");
                    // Project key from config; no userStory/label — pass empty strings
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

    // Helpers

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
            logArea.append(msg + "\\n");
            logArea.setCaretPosition(logArea.getDocument().getLength());
        });
    }

    private void showError(String msg) {
        JOptionPane.showMessageDialog(this, msg, "Error", JOptionPane.ERROR_MESSAGE);
    }

    private void addRow(JPanel p, GridBagConstraints c, int row, String lbl, JComponent field, int extra) {
        c.gridx = 0; c.gridy = row; c.gridwidth = 1; c.weightx = 0;
        p.add(label(lbl), c);
        c.gridx = 1; c.gridwidth = 1 + extra; c.weightx = 1.0;
        p.add(field, c);
        c.weightx = 0; c.gridwidth = 1;
    }

    /** Creates a button with a consistent preferred width so all action buttons align. */
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

"""


def write_files():
    for path, content in FILES.items():
        full = os.path.join(ROOT, path)
        os.makedirs(os.path.dirname(full), exist_ok=True)
        with open(full, "w", encoding="utf-8") as f:
            f.write(content)
        print(f"  wrote: {full}")

def zip_project():
    zip_name = ROOT + ".zip"
    with zipfile.ZipFile(zip_name, "w", zipfile.ZIP_DEFLATED) as zf:
        for root, _, files in os.walk(ROOT):
            for file in files:
                fp = os.path.join(root, file)
                arcname = fp  # keep relative path inside zip
                zf.write(fp, arcname)
    return zip_name

if __name__ == "__main__":
    print("\n=== Jira Test Case Importer — Project Generator ===\n")
    write_files()
    zip_name = zip_project()
    print(f"\n✔ ZIP created: {zip_name}")
    print(f"  Files: {len(FILES)}")
    print("\nNext steps:")
    print("  unzip jira-testcase-importer.zip")
    print("  cd jira-testcase-importer")
    print("  mvn clean package")
    print("  java -jar target/jira-testcase-importer.jar")

```

---

## How to Use

```bash
# 1. Save the script above as generate_project.py
# 2. Run it
python generate_project.py

# 3. Build and launch
unzip jira-testcase-importer.zip
cd jira-testcase-importer
mvn clean package
java -jar target/jira-testcase-importer.jar
```

---

## What the App Does

A Java 17 Swing desktop tool for QA teams:

1. **Browse** a folder of CSV test case files
2. **Load** an xray bulk upload importer JSON config (maps CSV columns → Jira field IDs)  
3. **Preview** parsed test cases in a table (identifier, summary, priority, steps, user story, etc.)
4. **Import** to Jira as issue type "Test" via REST API v2 with Bearer PAT auth
5. **Link** each test case to a Jira **Test Execution** (new or existing) via the Xray REST API
6. **Link** each test case to its **User Story** (read from the CSV `User Story` column)
7. **Log** all results in real time with ✔/✘ per test case
