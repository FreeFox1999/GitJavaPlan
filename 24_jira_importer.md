# Copilot Workspace Task — Recreate Project: xray-importer-pro (Jira Data Center)

Create a Maven Java 17 project by writing each file below **exactly as shown**.
Build: `mvn clean package` — Run: `java -jar target/xray-importer-pro-1.0.0.jar`

-----

## File 1 — `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.estjira</groupId>
    <artifactId>xray-importer-pro</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>Xray Importer Pro</name>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <jackson.version>2.16.1</jackson.version>
        <opencsv.version>5.9</opencsv.version>
        <httpclient.version>4.5.14</httpclient.version>
    </properties>

    <dependencies>
        <!-- Jackson -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>${jackson.version}</version>
        </dependency>

        <!-- OpenCSV -->
        <dependency>
            <groupId>com.opencsv</groupId>
            <artifactId>opencsv</artifactId>
            <version>${opencsv.version}</version>
        </dependency>

        <!-- Apache HttpClient -->
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>${httpclient.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpmime</artifactId>
            <version>${httpclient.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
            <!-- Fat JAR via Maven Shade -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.5.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <createDependencyReducedPom>false</createDependencyReducedPom>
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
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.estjira.ui.MainApp</mainClass>
                                </transformer>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

-----

## File 2 — `src/main/java/com/estjira/model/TestStep.java`

```java
package com.estjira.model;

public class TestStep {
    private int stepNo;
    private String action;
    private String data;
    private String expectedResult;

    public TestStep() {}

    public TestStep(int stepNo, String action, String data, String expectedResult) {
        this.stepNo = stepNo;
        this.action = action;
        this.data = data;
        this.expectedResult = expectedResult;
    }

    public int getStepNo() { return stepNo; }
    public void setStepNo(int stepNo) { this.stepNo = stepNo; }

    public String getAction() { return action; }
    public void setAction(String action) { this.action = action; }

    public String getData() { return data; }
    public void setData(String data) { this.data = data; }

    public String getExpectedResult() { return expectedResult; }
    public void setExpectedResult(String expectedResult) { this.expectedResult = expectedResult; }

    @Override
    public String toString() {
        return "Step " + stepNo + ": " + action;
    }
}
```

-----

## File 3 — `src/main/java/com/estjira/model/TestCase.java`

```java
package com.estjira.model;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class TestCase {
    private String identifier;
    private String summary;
    private String description;
    private String assignee;
    private String reporter;
    private String priority;
    private List<String> labels = new ArrayList<>();
    private List<TestStep> steps = new ArrayList<>();
    private Map<String, Object> rawCsvRow = new LinkedHashMap<>();
    private Map<String, Object> extraFields = new LinkedHashMap<>();

    // Import result tracking
    private String importStatus; // SUCCESS, FAILED, SKIPPED, DRY_RUN
    private String jiraIssueKey;
    private String errorMessage;

    public TestCase() {}

    public String getIdentifier() { return identifier; }
    public void setIdentifier(String identifier) { this.identifier = identifier; }

    public String getSummary() { return summary; }
    public void setSummary(String summary) { this.summary = summary; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public String getAssignee() { return assignee; }
    public void setAssignee(String assignee) { this.assignee = assignee; }

    public String getReporter() { return reporter; }
    public void setReporter(String reporter) { this.reporter = reporter; }

    public String getPriority() { return priority; }
    public void setPriority(String priority) { this.priority = priority; }

    public List<String> getLabels() { return labels; }
    public void setLabels(List<String> labels) { this.labels = labels; }

    public List<TestStep> getSteps() { return steps; }
    public void setSteps(List<TestStep> steps) { this.steps = steps; }

    public void addStep(TestStep step) { this.steps.add(step); }

    public Map<String, Object> getRawCsvRow() { return rawCsvRow; }
    public void setRawCsvRow(Map<String, Object> rawCsvRow) { this.rawCsvRow = rawCsvRow; }

    public Map<String, Object> getExtraFields() { return extraFields; }
    public void setExtraFields(Map<String, Object> extraFields) { this.extraFields = extraFields; }
    public void putExtraField(String key, Object value) { this.extraFields.put(key, value); }

    public String getImportStatus() { return importStatus; }
    public void setImportStatus(String importStatus) { this.importStatus = importStatus; }

    public String getJiraIssueKey() { return jiraIssueKey; }
    public void setJiraIssueKey(String jiraIssueKey) { this.jiraIssueKey = jiraIssueKey; }

    public String getErrorMessage() { return errorMessage; }
    public void setErrorMessage(String errorMessage) { this.errorMessage = errorMessage; }
}
```

-----

## File 4 — `src/main/java/com/estjira/model/ImportConfig.java`

```java
package com.estjira.model;

import com.fasterxml.jackson.annotation.JsonAnySetter;
import com.fasterxml.jackson.annotation.JsonProperty;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Represents the JSON config file that maps CSV columns to Jira fields.
 *
 * Example config.json:
 * {
 *   "projectKey": "PROJ",
 *   "issueType": "Test",
 *   "fieldMappings": {
 *     "summary": "Test Case Name",
 *     "description": "Description",
 *     "assignee": "Assignee Employee ID",
 *     "reporter": "Reporter Employee ID",
 *     "priority": "Priority",
 *     "labels": "Labels",
 *     "customfield_10200": "Custom Field Column"
 *   },
 *   "stepMappings": {
 *     "stepNo": "Step no.",
 *     "action": "Action",
 *     "data": "Test Data",
 *     "expectedResult": "Expected Result"
 *   },
 *   "testCaseIdentifierColumn": "Test Case Identifier",
 *   "priorityMap": {
 *     "High": "High",
 *     "Medium": "Medium",
 *     "Low": "Low"
 *   },
 *   "dryRun": false,
 *   "threadCount": 4,
 *   "maxRetries": 3
 * }
 */
public class ImportConfig {

    @JsonProperty("projectKey")
    private String projectKey;

    @JsonProperty("issueType")
    private String issueType = "Test";

    @JsonProperty("fieldMappings")
    private Map<String, String> fieldMappings = new LinkedHashMap<>();

    @JsonProperty("stepMappings")
    private Map<String, String> stepMappings = new LinkedHashMap<>();

    @JsonProperty("testCaseIdentifierColumn")
    private String testCaseIdentifierColumn = "Test Case Identifier";

    @JsonProperty("priorityMap")
    private Map<String, String> priorityMap = new LinkedHashMap<>();

    @JsonProperty("dryRun")
    private boolean dryRun = false;

    @JsonProperty("threadCount")
    private int threadCount = 4;

    @JsonProperty("maxRetries")
    private int maxRetries = 3;

    private Map<String, Object> extra = new LinkedHashMap<>();

    @JsonAnySetter
    public void setExtra(String key, Object value) {
        extra.put(key, value);
    }

    // Getters / Setters
    public String getProjectKey() { return projectKey; }
    public void setProjectKey(String projectKey) { this.projectKey = projectKey; }

    public String getIssueType() { return issueType; }
    public void setIssueType(String issueType) { this.issueType = issueType; }

    public Map<String, String> getFieldMappings() { return fieldMappings; }
    public void setFieldMappings(Map<String, String> fieldMappings) { this.fieldMappings = fieldMappings; }

    public Map<String, String> getStepMappings() { return stepMappings; }
    public void setStepMappings(Map<String, String> stepMappings) { this.stepMappings = stepMappings; }

    public String getTestCaseIdentifierColumn() { return testCaseIdentifierColumn; }
    public void setTestCaseIdentifierColumn(String c) { this.testCaseIdentifierColumn = c; }

    public Map<String, String> getPriorityMap() { return priorityMap; }
    public void setPriorityMap(Map<String, String> priorityMap) { this.priorityMap = priorityMap; }

    public boolean isDryRun() { return dryRun; }
    public void setDryRun(boolean dryRun) { this.dryRun = dryRun; }

    public int getThreadCount() { return threadCount; }
    public void setThreadCount(int threadCount) { this.threadCount = threadCount; }

    public int getMaxRetries() { return maxRetries; }
    public void setMaxRetries(int maxRetries) { this.maxRetries = maxRetries; }

    public Map<String, Object> getExtra() { return extra; }
}
```

-----

## File 5 — `src/main/java/com/estjira/model/ImportResult.java`

```java
package com.estjira.model;

import java.time.LocalDateTime;

public class ImportResult {
    private final String testCaseIdentifier;
    private final boolean success;
    private final String jiraKey;
    private final String errorMessage;
    private final boolean dryRun;
    private final LocalDateTime timestamp;

    public ImportResult(String testCaseIdentifier, boolean success, String jiraKey, String errorMessage, boolean dryRun) {
        this.testCaseIdentifier = testCaseIdentifier;
        this.success = success;
        this.jiraKey = jiraKey;
        this.errorMessage = errorMessage;
        this.dryRun = dryRun;
        this.timestamp = LocalDateTime.now();
    }

    public static ImportResult success(String id, String jiraKey, boolean dryRun) {
        return new ImportResult(id, true, jiraKey, null, dryRun);
    }

    public static ImportResult failure(String id, String error) {
        return new ImportResult(id, false, null, error, false);
    }

    public String getTestCaseIdentifier() { return testCaseIdentifier; }
    public boolean isSuccess() { return success; }
    public String getJiraKey() { return jiraKey; }
    public String getErrorMessage() { return errorMessage; }
    public boolean isDryRun() { return dryRun; }
    public LocalDateTime getTimestamp() { return timestamp; }

    @Override
    public String toString() {
        if (dryRun) return "[DRY-RUN] " + testCaseIdentifier + " → (not imported)";
        if (success) return "[OK] " + testCaseIdentifier + " → " + jiraKey;
        return "[FAIL] " + testCaseIdentifier + " → " + errorMessage;
    }
}
```

-----

## File 6 — `src/main/java/com/estjira/config/ConfigLoader.java`

```java
package com.estjira.config;

import com.estjira.model.ImportConfig;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.File;
import java.io.IOException;

public class ConfigLoader {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private ConfigLoader() {}

    /**
     * Loads an ImportConfig from a JSON file.
     */
    public static ImportConfig load(File file) throws IOException {
        if (file == null || !file.exists()) {
            throw new IOException("Config file not found: " + (file != null ? file.getAbsolutePath() : "null"));
        }
        return MAPPER.readValue(file, ImportConfig.class);
    }

    /**
     * Loads an ImportConfig from a JSON string.
     */
    public static ImportConfig loadFromString(String json) throws IOException {
        return MAPPER.readValue(json, ImportConfig.class);
    }

    /**
     * Saves an ImportConfig to a JSON file (pretty-printed).
     */
    public static void save(ImportConfig config, File file) throws IOException {
        MAPPER.writerWithDefaultPrettyPrinter().writeValue(file, config);
    }

    /**
     * Returns a sample config as a pretty-printed JSON string.
     */
    public static String sampleConfigJson() {
        return """
                {
                  "projectKey": "PROJ",
                  "issueType": "Test",
                  "testCaseIdentifierColumn": "Test Case Identifier",
                  "fieldMappings": {
                    "summary": "Test Case Name",
                    "description": "Description",
                    "assignee": "Assignee Employee ID",
                    "reporter": "Reporter Employee ID",
                    "priority": "Priority",
                    "labels": "Labels"
                  },
                  "stepMappings": {
                    "stepNo": "Step no.",
                    "action": "Action",
                    "data": "Test Data",
                    "expectedResult": "Expected Result"
                  },
                  "priorityMap": {
                    "High": "High",
                    "Medium": "Medium",
                    "Low": "Low",
                    "Critical": "Highest"
                  },
                  "dryRun": false,
                  "threadCount": 4,
                  "maxRetries": 3
                }
                """;
    }
}
```

-----

## File 7 — `src/main/java/com/estjira/util/AppLogger.java`

```java
package com.estjira.util;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.function.Consumer;

public class AppLogger {

    private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("HH:mm:ss");

    private final Consumer<String> logSink;

    public AppLogger(Consumer<String> logSink) {
        this.logSink = logSink;
    }

    public void info(String message) {
        log("INFO", message);
    }

    public void warn(String message) {
        log("WARN", message);
    }

    public void error(String message) {
        log("ERROR", message);
    }

    public void success(String message) {
        log("OK", message);
    }

    public void dryRun(String message) {
        log("DRY-RUN", message);
    }

    private void log(String level, String message) {
        String line = String.format("[%s] [%s] %s", LocalDateTime.now().format(FMT), level, message);
        if (logSink != null) {
            logSink.accept(line);
        } else {
            System.out.println(line);
        }
    }
}
```

-----

## File 8 — `src/main/java/com/estjira/util/RetryUtil.java`

```java
package com.estjira.util;

import java.util.concurrent.Callable;
import java.util.function.Consumer;

public class RetryUtil {

    private RetryUtil() {}

    /**
     * Executes a callable with exponential backoff retries.
     *
     * @param task       The task to execute
     * @param maxRetries Maximum number of retry attempts (not counting initial attempt)
     * @param baseDelayMs Initial delay in milliseconds (doubles each retry)
     * @param logger     Optional logger consumer for retry messages
     * @param <T>        Return type
     * @return Result of the task
     * @throws Exception If all retries are exhausted
     */
    public static <T> T withRetry(Callable<T> task, int maxRetries, long baseDelayMs, Consumer<String> logger) throws Exception {
        int attempt = 0;
        long delay = baseDelayMs;
        Exception lastException = null;

        while (attempt <= maxRetries) {
            try {
                return task.call();
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw ie;
            } catch (Exception e) {
                lastException = e;
                if (attempt < maxRetries) {
                    // Use toString() not getMessage() — getMessage() returns null for some exception types
                    String errDesc = e.getMessage() != null ? e.getMessage() : e.toString();
                    String msg = String.format("[Retry %d/%d] Error: %s. Waiting %dms before retry...",
                            attempt + 1, maxRetries, errDesc, delay);
                    if (logger != null) logger.accept(msg);
                    try {
                        Thread.sleep(delay);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw ie;
                    }
                    delay *= 2; // exponential backoff
                }
                attempt++;
            }
        }

        String lastMsg = (lastException != null)
                ? (lastException.getMessage() != null ? lastException.getMessage() : lastException.toString())
                : "unknown";
        throw new Exception("All " + maxRetries + " retries exhausted. Last error: " + lastMsg, lastException);
    }

    /**
     * Convenience overload with no logger.
     */
    public static <T> T withRetry(Callable<T> task, int maxRetries, long baseDelayMs) throws Exception {
        return withRetry(task, maxRetries, baseDelayMs, null);
    }
}
```

-----

## File 9 — `src/main/java/com/estjira/util/UserCache.java`

```java
package com.estjira.util;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.util.EntityUtils;

import java.io.IOException;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Resolves employee IDs (e.g. G01348533) to Jira Data Center usernames.
 *
 * DC differences vs Cloud:
 *   - Endpoint:  GET /rest/api/2/user/search?username={query}   (not api/3 + ?query=)
 *   - Returns:   user["name"]  (the Jira login name)            (not "accountId")
 *   - Auth:      Bearer {PAT}
 *
 * The resolved username is used in issue payloads as:
 *   {"assignee": {"name": "jsmith"}}
 *
 * ConcurrentHashMap does not permit null values, so we use a sentinel string
 * NOT_FOUND to represent a failed / negative lookup.
 */
public class UserCache {

    private static final ObjectMapper MAPPER     = new ObjectMapper();
    private static final String       NOT_FOUND  = "__NOT_FOUND__";

    private final Map<String, String>  cache = new ConcurrentHashMap<>();
    private final CloseableHttpClient  httpClient;
    private final String               jiraBaseUrl;
    private final String               authHeader;
    private final AppLogger            logger;

    public UserCache(CloseableHttpClient httpClient, String jiraBaseUrl, String pat, AppLogger logger) {
        this.httpClient  = httpClient;
        this.jiraBaseUrl = jiraBaseUrl.endsWith("/") ? jiraBaseUrl.substring(0, jiraBaseUrl.length() - 1) : jiraBaseUrl;
        this.authHeader  = "Bearer " + pat;
        this.logger      = logger;
    }

    /**
     * Resolve an employee ID to a Jira DC username.
     * Returns null if not found or employeeId is blank.
     */
    public String resolve(String employeeId) {
        if (employeeId == null || employeeId.isBlank()) return null;

        String cached = cache.get(employeeId);
        if (cached != null) {
            return NOT_FOUND.equals(cached) ? null : cached;
        }

        try {
            String encoded = URLEncoder.encode(employeeId, StandardCharsets.UTF_8);
            // DC user search: ?username= (not ?query= which is Cloud-only)
            String url = jiraBaseUrl + "/rest/api/2/user/search?username=" + encoded;
            HttpGet request = new HttpGet(url);
            request.setHeader("Authorization", authHeader);
            request.setHeader("Accept", "application/json");

            try (CloseableHttpResponse response = httpClient.execute(request)) {
                int    status = response.getStatusLine().getStatusCode();
                String body   = response.getEntity() != null
                        ? EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8) : "";

                if (status == 200) {
                    JsonNode arr = MAPPER.readTree(body);
                    if (arr.isArray() && arr.size() > 0) {
                        // DC user object field is "name", not "accountId"
                        String username = arr.get(0).path("name").asText(null);
                        if (username != null && !username.isBlank()) {
                            cache.put(employeeId, username);
                            logger.info("Resolved user " + employeeId + " → " + username);
                            return username;
                        }
                    }
                    logger.warn("No Jira user found for employee ID: " + employeeId);
                } else {
                    logger.warn("User search returned HTTP " + status + " for: " + employeeId);
                }
            }
        } catch (IOException e) {
            logger.error("Failed to resolve user " + employeeId + ": " + e.getMessage());
        }

        // Cache negative result using sentinel to avoid repeated failed calls
        cache.put(employeeId, NOT_FOUND);
        return null;
    }

    public void clearCache()  { cache.clear(); }
    public int  cacheSize()   { return cache.size(); }
}
```

-----

## File 10 — `src/main/java/com/estjira/csv/CsvParser.java`

```java
package com.estjira.csv;

import com.estjira.model.ImportConfig;
import com.estjira.model.TestCase;
import com.estjira.model.TestStep;
import com.opencsv.CSVReader;
import com.opencsv.CSVReaderBuilder;

import java.io.File;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.FileInputStream;
import java.nio.charset.StandardCharsets;
import java.util.*;

/**
 * Parses a CSV file into a list of TestCases.
 * Groups rows by "Test Case Identifier" and preserves step order by "Step no."
 */
public class CsvParser {

    private CsvParser() {}

    /**
     * Parse the CSV and return ordered list of TestCases.
     *
     * @param csvFile      The CSV file to parse
     * @param config       The import config providing column mappings
     * @return Ordered list of TestCase objects (order of first appearance in CSV)
     */
    public static List<TestCase> parse(File csvFile, ImportConfig config) throws IOException {
        String identifierCol = config.getTestCaseIdentifierColumn();

        // Field mappings
        Map<String, String> fieldMappings = config.getFieldMappings();
        Map<String, String> stepMappings = config.getStepMappings();

        // Ordered map: identifier → TestCase
        LinkedHashMap<String, TestCase> caseMap = new LinkedHashMap<>();

        try (FileInputStream fis = new FileInputStream(csvFile);
             InputStreamReader isr = new InputStreamReader(fis, StandardCharsets.UTF_8);
             CSVReader reader = new CSVReaderBuilder(isr).build()) {
            String[] headers = reader.readNext();
            if (headers == null) throw new IOException("CSV file is empty");

            // Trim headers
            for (int i = 0; i < headers.length; i++) {
                headers[i] = headers[i].trim();
            }

            String[] row;
            while ((row = reader.readNext()) != null) {
                if (row.length == 0) continue;

                // Build column → value map for this row
                Map<String, String> rowMap = new LinkedHashMap<>();
                for (int i = 0; i < headers.length && i < row.length; i++) {
                    rowMap.put(headers[i], row[i].trim());
                }

                String identifier = rowMap.getOrDefault(identifierCol, "").trim();
                if (identifier.isEmpty()) continue;

                // Create or retrieve TestCase
                TestCase tc = caseMap.computeIfAbsent(identifier, id -> {
                    TestCase newTc = new TestCase();
                    newTc.setIdentifier(id);
                    // Map scalar fields from first encountered row
                    mapFields(newTc, rowMap, fieldMappings, config);
                    // Store raw row for display
                    newTc.setRawCsvRow(new LinkedHashMap<>(rowMap));
                    return newTc;
                });

                // Build step
                TestStep step = buildStep(rowMap, stepMappings);
                if (step != null) {
                    tc.addStep(step);
                }
            }
        }

        // Sort steps by stepNo within each TestCase
        for (TestCase tc : caseMap.values()) {
            tc.getSteps().sort(Comparator.comparingInt(TestStep::getStepNo));
        }

        return new ArrayList<>(caseMap.values());
    }

    private static void mapFields(TestCase tc, Map<String, String> rowMap,
                                   Map<String, String> fieldMappings, ImportConfig config) {
        for (Map.Entry<String, String> mapping : fieldMappings.entrySet()) {
            String jiraField = mapping.getKey();
            String csvCol = mapping.getValue();
            String value = rowMap.getOrDefault(csvCol, "").trim();

            switch (jiraField) {
                case "summary" -> tc.setSummary(value);
                case "description" -> tc.setDescription(value);
                case "assignee" -> tc.setAssignee(value);
                case "reporter" -> tc.setReporter(value);
                case "priority" -> {
                    // Map through priorityMap if configured
                    String mapped = config.getPriorityMap().getOrDefault(value, value);
                    tc.setPriority(mapped.isEmpty() ? null : mapped);
                }
                case "labels" -> {
                    if (!value.isEmpty()) {
                        String[] parts = value.split("[,;|]");
                        for (String part : parts) {
                            String label = part.trim();
                            if (!label.isEmpty()) tc.getLabels().add(label);
                        }
                    }
                }
                default -> {
                    // customfield_xxxxx or other custom fields
                    if (!value.isEmpty()) {
                        tc.putExtraField(jiraField, value);
                    }
                }
            }
        }

        // Fallback summary from identifier if not set
        if (tc.getSummary() == null || tc.getSummary().isEmpty()) {
            tc.setSummary(tc.getIdentifier());
        }
    }

    private static TestStep buildStep(Map<String, String> rowMap, Map<String, String> stepMappings) {
        String stepNoCol = stepMappings.getOrDefault("stepNo", "Step no.");
        String actionCol = stepMappings.getOrDefault("action", "Action");
        String dataCol = stepMappings.getOrDefault("data", "Test Data");
        String expectedCol = stepMappings.getOrDefault("expectedResult", "Expected Result");

        String stepNoStr = rowMap.getOrDefault(stepNoCol, "").trim();
        String action = rowMap.getOrDefault(actionCol, "").trim();

        if (stepNoStr.isEmpty() && action.isEmpty()) return null;

        int stepNo = 0;
        try {
            stepNo = Integer.parseInt(stepNoStr);
        } catch (NumberFormatException e) {
            // default 0
        }

        return new TestStep(
                stepNo,
                action,
                rowMap.getOrDefault(dataCol, "").trim(),
                rowMap.getOrDefault(expectedCol, "").trim()
        );
    }
}
```

-----

## File 11 — `src/main/java/com/estjira/jira/JiraClient.java`

```java
package com.estjira.jira;

import com.estjira.util.AppLogger;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

/**
 * Jira Data Center REST API client.
 *
 * Auth: Personal Access Token → "Authorization: Bearer {PAT}"
 * API:  /rest/api/2/  (Data Center uses v2, not v3 which is Cloud-only)
 */
public class JiraClient implements AutoCloseable {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final String jiraBaseUrl;
    private final String authHeader;
    private final CloseableHttpClient httpClient;
    private final AppLogger logger;

    /**
     * @param jiraBaseUrl  e.g. "https://estjira.company.com"
     * @param pat          Personal Access Token
     * @param logger       Log sink
     */
    public JiraClient(String jiraBaseUrl, String pat, AppLogger logger) {
        this.jiraBaseUrl = normalizeUrl(jiraBaseUrl);
        this.authHeader  = "Bearer " + pat;
        this.httpClient  = HttpClients.createDefault();
        this.logger      = logger;
    }

    public String getJiraBaseUrl() { return jiraBaseUrl; }
    public String getAuthHeader()  { return authHeader; }
    public CloseableHttpClient getHttpClient() { return httpClient; }

    /**
     * Verifies the PAT and connectivity by calling GET /rest/api/2/myself.
     * Returns the display name of the authenticated user.
     */
    public String testConnection() throws IOException {
        HttpGet request = new HttpGet(jiraBaseUrl + "/rest/api/2/myself");
        request.setHeader("Authorization", authHeader);
        request.setHeader("Accept", "application/json");

        try (CloseableHttpResponse response = httpClient.execute(request)) {
            int status = response.getStatusLine().getStatusCode();
            HttpEntity entity = response.getEntity();
            String body = entity != null ? EntityUtils.toString(entity, StandardCharsets.UTF_8) : "";

            if (status == 200) {
                JsonNode node = MAPPER.readTree(body);
                String displayName = node.path("displayName").asText("Unknown");
                String name        = node.path("name").asText("");
                logger.success("Connected to Jira DC as: " + displayName + " (" + name + ")");
                return displayName;
            } else if (status == 401) {
                throw new IOException("Authentication failed (HTTP 401) — check your Personal Access Token.");
            } else {
                throw new IOException("Jira connection failed: HTTP " + status + " — " + truncate(body, 200));
            }
        }
    }

    private static String normalizeUrl(String url) {
        if (url == null) return "";
        return url.endsWith("/") ? url.substring(0, url.length() - 1) : url;
    }

    private static String truncate(String s, int max) {
        if (s == null) return "";
        return s.length() <= max ? s : s.substring(0, max) + "...";
    }

    @Override
    public void close() throws IOException {
        httpClient.close();
    }
}
```

-----

## File 12 — `src/main/java/com/estjira/xray/ImportMode.java`

```java
package com.estjira.xray;

/**
 * Controls what happens after Test issues are created in Jira.
 */
public enum ImportMode {

    /** Create Test issues only. No test execution is created or updated. */
    CREATE_ONLY,

    /** Create Test issues and add them to an existing Test Execution (key supplied by user). */
    ADD_TO_EXISTING,

    /** Create Test issues, then create a new Test Execution and add all tests to it. */
    CREATE_NEW_EXECUTION
}
```

-----

## File 13 — `src/main/java/com/estjira/xray/XrayDcClient.java`

```java
package com.estjira.xray;

import com.estjira.model.ImportConfig;
import com.estjira.model.ImportResult;
import com.estjira.model.TestCase;
import com.estjira.model.TestStep;
import com.estjira.util.AppLogger;
import com.estjira.util.RetryUtil;
import com.estjira.util.UserCache;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;
import java.util.function.Consumer;
import java.util.stream.Collectors;

/**
 * Xray for Jira Data Center REST API client.
 *
 * Auth: Personal Access Token → "Authorization: Bearer {PAT}"
 *
 * Flow per test case:
 *   1. POST /rest/api/2/issue                          → creates Jira "Test" issue, returns key
 *   2. POST /rest/raven/1.0/testcase/{key}/steps       → adds each step (sequential, in order)
 *
 * Post-import (based on ImportMode):
 *   ADD_TO_EXISTING  → POST /rest/raven/1.0/testexec/{execKey}/test
 *   CREATE_NEW       → POST /rest/api/2/issue (Test Execution)
 *                      POST /rest/raven/1.0/testexec/{newKey}/test
 */
public class XrayDcClient implements AutoCloseable {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final int FUTURE_TIMEOUT_SECONDS = 120;

    private final String jiraBaseUrl;
    private final String authHeader;
    private final AppLogger logger;
    private final CloseableHttpClient httpClient;

    public XrayDcClient(String jiraBaseUrl, String pat, AppLogger logger) {
        this.jiraBaseUrl = normalizeUrl(jiraBaseUrl);
        this.authHeader = "Bearer " + pat;
        this.logger = logger;
        this.httpClient = HttpClients.createDefault();
    }

    // -------------------------------------------------------------------------
    // Public API
    // -------------------------------------------------------------------------

    /**
     * Creates a Jira "Test" issue via /rest/api/2/issue.
     * Returns the created issue key (e.g. "PROJ-123").
     */
    public String createTestIssue(TestCase tc, ImportConfig config, UserCache userCache) throws Exception {
        return RetryUtil.withRetry(() -> {
            ObjectNode payload = buildIssuePayload(tc, config, userCache);
            HttpPost post = post("/rest/api/2/issue", payload);

            try (CloseableHttpResponse response = httpClient.execute(post)) {
                int status = response.getStatusLine().getStatusCode();
                String body = body(response);
                if (status == 200 || status == 201) {
                    JsonNode node = MAPPER.readTree(body);
                    String key = node.path("key").asText("");
                    if (key.isBlank()) {
                        throw new IOException("Create issue returned no key. Body: " + truncate(body, 200));
                    }
                    return key;
                } else {
                    throw new IOException("Create test failed: HTTP " + status + " — " + truncate(body, 300));
                }
            }
        }, config.getMaxRetries(), 1000L, logger::warn);
    }

    /**
     * Adds a single step to a Test issue via Xray DC REST.
     * POST /rest/raven/1.0/testcase/{testKey}/steps
     * Appends in call order — caller must call this in step order.
     */
    public void addStep(String testKey, TestStep step, ImportConfig config) throws Exception {
        RetryUtil.withRetry(() -> {
            ObjectNode payload = MAPPER.createObjectNode();
            payload.put("step",   step.getAction()         != null ? step.getAction()         : "");
            payload.put("data",   step.getData()           != null ? step.getData()           : "");
            payload.put("result", step.getExpectedResult() != null ? step.getExpectedResult() : "");

            HttpPost post = post("/rest/raven/1.0/testcase/" + testKey + "/steps", payload);

            try (CloseableHttpResponse response = httpClient.execute(post)) {
                int status = response.getStatusLine().getStatusCode();
                if (status == 200 || status == 201) {
                    return null; // Callable<Void>
                } else {
                    String b = body(response);
                    throw new IOException("Add step failed: HTTP " + status + " — " + truncate(b, 200));
                }
            }
        }, config.getMaxRetries(), 1000L, logger::warn);
    }

    /**
     * Creates a new Test Execution issue in Jira.
     * Returns the execution key (e.g. "PROJ-456").
     */
    public String createTestExecution(String projectKey, String summary) throws Exception {
        return RetryUtil.withRetry(() -> {
            ObjectNode fields = MAPPER.createObjectNode();
            ObjectNode project  = MAPPER.createObjectNode(); project.put("key", projectKey);
            ObjectNode execType = MAPPER.createObjectNode(); execType.put("name", "Test Execution");
            fields.set("project",   project);
            fields.set("issuetype", execType);
            fields.put("summary",   summary.isBlank() ? "Test Execution - Bulk Import" : summary);

            ObjectNode payload = MAPPER.createObjectNode();
            payload.set("fields", fields);

            HttpPost post = post("/rest/api/2/issue", payload);

            try (CloseableHttpResponse response = httpClient.execute(post)) {
                int status = response.getStatusLine().getStatusCode();
                String b = body(response);
                if (status == 200 || status == 201) {
                    String key = MAPPER.readTree(b).path("key").asText("");
                    if (key.isBlank()) throw new IOException("Create execution returned no key. Body: " + truncate(b, 200));
                    return key;
                } else {
                    throw new IOException("Create execution failed: HTTP " + status + " — " + truncate(b, 300));
                }
            }
        }, 3, 1000L, logger::warn);
    }

    /**
     * Adds test keys to an existing Test Execution.
     * POST /rest/raven/1.0/testexec/{execKey}/test   body: {"add": ["PROJ-1", ...]}
     */
    public void addTestsToExecution(String execKey, List<String> testKeys) throws Exception {
        if (testKeys == null || testKeys.isEmpty()) return;

        RetryUtil.withRetry(() -> {
            ObjectNode payload = MAPPER.createObjectNode();
            ArrayNode addArr   = MAPPER.createArrayNode();
            testKeys.forEach(addArr::add);
            payload.set("add", addArr);

            HttpPost post = post("/rest/raven/1.0/testexec/" + execKey + "/test", payload);

            try (CloseableHttpResponse response = httpClient.execute(post)) {
                int status = response.getStatusLine().getStatusCode();
                // 200/201/204 all indicate success
                if (status >= 200 && status < 300) {
                    return null;
                } else {
                    String b = body(response);
                    throw new IOException("Add to execution failed: HTTP " + status + " — " + truncate(b, 300));
                }
            }
        }, 3, 1000L, logger::warn);
    }

    // -------------------------------------------------------------------------
    // Bulk Import Orchestration
    // -------------------------------------------------------------------------

    /**
     * Full bulk import: create tests in parallel, add steps per test, then handle execution.
     *
     * @param testCases            Test cases to import
     * @param config               Config (projectKey, issueType, maxRetries, threadCount, dryRun)
     * @param mode                 CREATE_ONLY / ADD_TO_EXISTING / CREATE_NEW_EXECUTION
     * @param executionKeyOrSummary Execution key (ADD_TO_EXISTING) or new execution summary (CREATE_NEW_EXECUTION)
     * @param userCache            Resolves employee IDs to Jira usernames
     * @param progressCb           int[]{completed, total} progress callback
     * @param logCb                Log message callback
     * @return Results, one per test case
     */
    public List<ImportResult> bulkImport(List<TestCase> testCases,
                                          ImportConfig config,
                                          ImportMode mode,
                                          String executionKeyOrSummary,
                                          UserCache userCache,
                                          Consumer<int[]> progressCb,
                                          Consumer<String> logCb) {

        List<ImportResult> results = new CopyOnWriteArrayList<>();
        int total = testCases.size();
        int[] progress = {0};

        if (config.isDryRun()) {
            for (TestCase tc : testCases) {
                results.add(ImportResult.success(tc.getIdentifier(), null, true));
                logCb.accept("[DRY-RUN] Would import: " + tc.getIdentifier() +
                        " (" + tc.getSteps().size() + " steps)");
            }
            return results;
        }

        // Phase 1: create test issues + steps in parallel
        int threadCount = Math.max(1, config.getThreadCount());
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        List<Future<ImportResult>> futures = new ArrayList<>();

        for (TestCase tc : testCases) {
            futures.add(executor.submit(() -> {
                try {
                    // 1a. Create the Test issue
                    String testKey = createTestIssue(tc, config, userCache);

                    // 1b. Add steps in order (sequential within this thread — order preserved)
                    List<TestStep> steps = tc.getSteps();
                    for (TestStep step : steps) {
                        addStep(testKey, step, config);
                    }

                    logCb.accept("[OK] " + tc.getIdentifier() + " → " + testKey +
                            " (" + steps.size() + " steps)");
                    return ImportResult.success(tc.getIdentifier(), testKey, false);

                } catch (Exception e) {
                    String errMsg = e.getMessage() != null ? e.getMessage() : e.toString();
                    logCb.accept("[FAIL] " + tc.getIdentifier() + ": " + errMsg);
                    return ImportResult.failure(tc.getIdentifier(), errMsg);
                } finally {
                    synchronized (progress) {
                        progress[0]++;
                        progressCb.accept(new int[]{progress[0], total});
                    }
                }
            }));
        }

        executor.shutdown();
        for (Future<ImportResult> future : futures) {
            try {
                results.add(future.get(FUTURE_TIMEOUT_SECONDS, TimeUnit.SECONDS));
            } catch (TimeoutException e) {
                future.cancel(true);
                logger.error("A task timed out after " + FUTURE_TIMEOUT_SECONDS + "s — skipping.");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                executor.shutdownNow();
                logger.error("Import interrupted — remaining tasks cancelled.");
                break;
            } catch (ExecutionException e) {
                String cause = e.getCause() != null ? e.getCause().toString() : e.toString();
                logger.error("Task failed: " + cause);
            }
        }

        // Phase 2: add to execution (single-threaded, after all tests created)
        if (mode != ImportMode.CREATE_ONLY) {
            List<String> createdKeys = results.stream()
                    .filter(ImportResult::isSuccess)
                    .map(ImportResult::getJiraKey)
                    .filter(k -> k != null && !k.isBlank())
                    .collect(Collectors.toList());

            if (createdKeys.isEmpty()) {
                logCb.accept("[WARN] No tests were created successfully — skipping execution step.");
            } else {
                try {
                    String execKey;
                    if (mode == ImportMode.ADD_TO_EXISTING) {
                        execKey = executionKeyOrSummary.trim();
                        logCb.accept("[INFO] Adding " + createdKeys.size() + " tests to execution " + execKey + "...");
                    } else {
                        // CREATE_NEW_EXECUTION
                        String summary = executionKeyOrSummary.isBlank()
                                ? "Test Execution - Bulk Import"
                                : executionKeyOrSummary.trim();
                        logCb.accept("[INFO] Creating new Test Execution: \"" + summary + "\"...");
                        execKey = createTestExecution(config.getProjectKey(), summary);
                        logCb.accept("[OK] Created Test Execution: " + execKey);
                    }
                    addTestsToExecution(execKey, createdKeys);
                    logCb.accept("[OK] Added " + createdKeys.size() + " tests to execution " + execKey);
                } catch (Exception e) {
                    String errMsg = e.getMessage() != null ? e.getMessage() : e.toString();
                    logCb.accept("[ERROR] Execution step failed: " + errMsg);
                    logger.error("Execution step failed: " + errMsg);
                }
            }
        }

        return results;
    }

    // -------------------------------------------------------------------------
    // Payload Builders
    // -------------------------------------------------------------------------

    private ObjectNode buildIssuePayload(TestCase tc, ImportConfig config, UserCache userCache) {
        ObjectNode root   = MAPPER.createObjectNode();
        ObjectNode fields = MAPPER.createObjectNode();

        fields.put("summary", tc.getSummary() != null ? tc.getSummary() : tc.getIdentifier());

        ObjectNode project   = MAPPER.createObjectNode(); project.put("key", config.getProjectKey());
        ObjectNode issueType = MAPPER.createObjectNode(); issueType.put("name", config.getIssueType());
        fields.set("project",   project);
        fields.set("issuetype", issueType);

        if (tc.getDescription() != null && !tc.getDescription().isBlank()) {
            fields.put("description", tc.getDescription());
        }

        if (tc.getPriority() != null && !tc.getPriority().isBlank()) {
            ObjectNode priority = MAPPER.createObjectNode();
            priority.put("name", tc.getPriority());
            fields.set("priority", priority);
        }

        if (!tc.getLabels().isEmpty()) {
            ArrayNode labels = MAPPER.createArrayNode();
            tc.getLabels().forEach(labels::add);
            fields.set("labels", labels);
        }

        // DC uses {"name": "jira-username"} — NOT {"accountId": "..."} which is Cloud-only
        if (tc.getAssignee() != null && !tc.getAssignee().isBlank() && userCache != null) {
            String username = userCache.resolve(tc.getAssignee());
            if (username != null) {
                ObjectNode assignee = MAPPER.createObjectNode();
                assignee.put("name", username);
                fields.set("assignee", assignee);
            }
        }

        if (tc.getReporter() != null && !tc.getReporter().isBlank() && userCache != null) {
            String username = userCache.resolve(tc.getReporter());
            if (username != null) {
                ObjectNode reporter = MAPPER.createObjectNode();
                reporter.put("name", username);
                fields.set("reporter", reporter);
            }
        }

        for (var entry : tc.getExtraFields().entrySet()) {
            if (entry.getValue() instanceof String s) {
                fields.put(entry.getKey(), s);
            }
        }

        root.set("fields", fields);
        return root;
    }

    // -------------------------------------------------------------------------
    // HTTP Helpers
    // -------------------------------------------------------------------------

    /** Build an authorized JSON POST request. */
    private HttpPost post(String path, ObjectNode payload) throws IOException {
        HttpPost post = new HttpPost(jiraBaseUrl + path);
        post.setHeader("Authorization", authHeader);
        post.setHeader("Content-Type",  "application/json");
        post.setEntity(new StringEntity(MAPPER.writeValueAsString(payload), StandardCharsets.UTF_8));
        return post;
    }

    /** Safely read a response body, returning empty string if entity is null. */
    private static String body(CloseableHttpResponse response) throws IOException {
        return response.getEntity() != null
                ? EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8)
                : "";
    }

    private static String normalizeUrl(String url) {
        if (url == null) return "";
        return url.endsWith("/") ? url.substring(0, url.length() - 1) : url;
    }

    private static String truncate(String s, int max) {
        if (s == null) return "";
        return s.length() <= max ? s : s.substring(0, max) + "...";
    }

    @Override
    public void close() throws IOException {
        httpClient.close();
    }
}
```

-----

## File 14 — `src/main/java/com/estjira/ui/MainApp.java`

```java
package com.estjira.ui;

import javax.swing.*;

/**
 * Application entry point.
 */
public class MainApp {

    public static void main(String[] args) {
        // High-DPI support
        System.setProperty("sun.java2d.uiScale", "1.0");

        SwingUtilities.invokeLater(() -> {
            try {
                UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());
            } catch (Exception ignored) {}
            new MainWindow();
        });
    }
}
```

-----

## File 15 — `src/main/java/com/estjira/ui/MainWindow.java`

```java
package com.estjira.ui;

import com.estjira.config.ConfigLoader;
import com.estjira.csv.CsvParser;
import com.estjira.jira.JiraClient;
import com.estjira.model.ImportConfig;
import com.estjira.model.ImportResult;
import com.estjira.model.TestCase;
import com.estjira.util.AppLogger;
import com.estjira.util.UserCache;
import com.estjira.xray.ImportMode;
import com.estjira.xray.XrayDcClient;

import javax.swing.*;
import javax.swing.border.EmptyBorder;
import javax.swing.border.TitledBorder;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.io.File;
import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.ArrayList;
import java.util.List;

/**
 * Main application window — Jira Data Center edition.
 *
 * Auth:     Personal Access Token (PAT) → Bearer header
 * API:      /rest/api/2/  +  /rest/raven/1.0/
 *
 * Import modes (chosen at runtime):
 *   ● Create Tests Only
 *   ● Add to Existing Test Execution
 *   ● Create New Test Execution
 */
public class MainWindow extends JFrame {

    // ── Connection ──────────────────────────────────────────────────────────
    private final JTextField    txtJiraUrl = new JTextField(30);
    private final JPasswordField txtPat    = new JPasswordField(30);

    // ── Files ───────────────────────────────────────────────────────────────
    private final JTextField txtCsvFile    = new JTextField(30);
    private final JTextField txtConfigFile = new JTextField(30);

    // ── Import Mode ─────────────────────────────────────────────────────────
    private final ButtonGroup  modeGroup        = new ButtonGroup();
    private final JRadioButton rbCreateOnly     = new JRadioButton("Create Tests Only");
    private final JRadioButton rbAddToExisting  = new JRadioButton("Add to Existing Execution");
    private final JRadioButton rbCreateNew      = new JRadioButton("Create New Execution");

    /** Shown when rbAddToExisting is selected — user types the execution key. */
    private final JTextField txtExecKey     = new JTextField(15);

    /** Shown when rbCreateNew is selected — user types the execution summary. */
    private final JTextField txtExecSummary = new JTextField(30);

    /** Card panel that swaps between exec-key input and exec-summary input. */
    private final JPanel     pnlModeExtra   = new JPanel(new CardLayout());

    // ── Buttons ─────────────────────────────────────────────────────────────
    private final JButton btnTestConnection = new JButton("Test Connection");
    private final JButton btnPreview        = new JButton("Preview");
    private final JButton btnImport         = new JButton("Import to Jira");
    private final JButton btnClearLog       = new JButton("Clear Log");

    // ── Preview Table ───────────────────────────────────────────────────────
    private final DefaultTableModel previewTableModel = new DefaultTableModel();
    private final JTable            previewTable      = new JTable(previewTableModel);

    // ── Log / Progress ──────────────────────────────────────────────────────
    private final JTextArea    logArea     = new JTextArea(12, 60);
    private final JProgressBar progressBar = new JProgressBar(0, 100);

    // ── State ───────────────────────────────────────────────────────────────
    private volatile List<TestCase> previewedTestCases = new ArrayList<>();
    private AppLogger logger;

    // ── Palette ─────────────────────────────────────────────────────────────
    private static final Color BG_DARK     = new Color(30,  33,  40);
    private static final Color BG_PANEL    = new Color(42,  45,  54);
    private static final Color ACCENT      = new Color(99,  179, 237);
    private static final Color FG_TEXT     = new Color(220, 220, 220);
    private static final Color BORDER_COLOR = new Color(70, 73,  82);

    // CardLayout card names
    private static final String CARD_EXEC_KEY     = "execKey";
    private static final String CARD_EXEC_SUMMARY = "execSummary";
    private static final String CARD_EMPTY        = "empty";

    // =========================================================================
    // Constructor
    // =========================================================================

    public MainWindow() {
        super("Jira Test Case Importer — Data Center");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setMinimumSize(new Dimension(1100, 780));
        setPreferredSize(new Dimension(1200, 860));

        logger = new AppLogger(this::appendLog);

        // Build the content pane first, THEN call applyDarkTheme() so it
        // targets the real root panel (not the placeholder from JFrame init)
        setContentPane(buildContentPane());
        applyDarkTheme();

        wireListeners();
        pack();
        setLocationRelativeTo(null);
        setVisible(true);
    }

    // =========================================================================
    // UI Construction
    // =========================================================================

    private JPanel buildContentPane() {
        JPanel root = darkPanel(new BorderLayout(8, 8));
        root.setBorder(new EmptyBorder(10, 10, 10, 10));
        root.add(buildSettingsPanel(), BorderLayout.NORTH);

        JSplitPane center = new JSplitPane(JSplitPane.VERTICAL_SPLIT, buildPreviewPanel(), buildLogPanel());
        center.setResizeWeight(0.55);
        center.setDividerSize(6);
        center.setBackground(BG_DARK);
        root.add(center, BorderLayout.CENTER);
        root.add(buildBottomPanel(), BorderLayout.SOUTH);
        return root;
    }

    private JPanel buildSettingsPanel() {
        JPanel outer = darkPanel(new BorderLayout());
        outer.setBorder(titledBorder("Connection & Import Settings"));

        JPanel grid = darkPanel(new GridBagLayout());
        GridBagConstraints c = new GridBagConstraints();
        c.insets = new Insets(4, 6, 4, 6);
        c.fill   = GridBagConstraints.HORIZONTAL;

        // Row 0: Jira URL
        c.gridx = 0; c.gridy = 0; c.weightx = 0; grid.add(styledLabel("Jira URL:"), c);
        c.gridx = 1; c.gridy = 0; c.weightx = 1;
        txtJiraUrl.setText("https://estjira.company.com");
        grid.add(style(txtJiraUrl), c);

        // Row 1: Personal Access Token
        c.gridx = 0; c.gridy = 1; c.weightx = 0; grid.add(styledLabel("Personal Access Token:"), c);
        c.gridx = 1; c.gridy = 1; c.weightx = 1; grid.add(style(txtPat), c);

        // Row 2: CSV File
        c.gridx = 0; c.gridy = 2; c.weightx = 0; grid.add(styledLabel("CSV File:"), c);
        c.gridx = 1; c.gridy = 2; c.weightx = 1; grid.add(buildFileChooserRow(txtCsvFile, "Select CSV", "csv"), c);

        // Row 3: Config JSON
        c.gridx = 0; c.gridy = 3; c.weightx = 0; grid.add(styledLabel("Config JSON:"), c);
        c.gridx = 1; c.gridy = 3; c.weightx = 1; grid.add(buildFileChooserRow(txtConfigFile, "Select JSON", "json"), c);

        // Row 4: Import Mode
        c.gridx = 0; c.gridy = 4; c.weightx = 0; grid.add(styledLabel("Import Mode:"), c);
        c.gridx = 1; c.gridy = 4; c.weightx = 1; grid.add(buildImportModePanel(), c);

        // Row 5: Action buttons
        c.gridx = 0; c.gridy = 5; c.gridwidth = 2; c.weightx = 0;
        JPanel btnRow = darkPanel(new FlowLayout(FlowLayout.LEFT, 8, 0));
        styleButton(btnTestConnection, new Color(52, 152, 219));
        styleButton(btnPreview,        new Color(39, 174, 96));
        styleButton(btnImport,         new Color(155, 89, 182));
        styleButton(btnClearLog,       new Color(100, 100, 110));
        btnRow.add(btnTestConnection);
        btnRow.add(btnPreview);
        btnRow.add(btnImport);
        btnRow.add(btnClearLog);
        grid.add(btnRow, c);

        outer.add(grid, BorderLayout.CENTER);
        return outer;
    }

    /**
     * Builds the Import Mode radio panel with a CardLayout extra-input area:
     *   ● Create Tests Only        → [no extra input]
     *   ● Add to Existing Execution → [Execution Key field]
     *   ● Create New Execution     → [Execution Summary field]
     */
    private JPanel buildImportModePanel() {
        // Style radio buttons
        for (JRadioButton rb : new JRadioButton[]{rbCreateOnly, rbAddToExisting, rbCreateNew}) {
            modeGroup.add(rb);
            rb.setBackground(BG_PANEL);
            rb.setForeground(FG_TEXT);
            rb.setFont(new Font("Segoe UI", Font.PLAIN, 13));
            rb.setFocusPainted(false);
        }
        rbCreateOnly.setSelected(true); // default

        // Radio row
        JPanel radioRow = darkPanel(new FlowLayout(FlowLayout.LEFT, 12, 0));
        radioRow.add(rbCreateOnly);
        radioRow.add(rbAddToExisting);
        radioRow.add(rbCreateNew);

        // Card: empty (shown for CREATE_ONLY)
        JPanel cardEmpty = darkPanel(new FlowLayout(FlowLayout.LEFT, 0, 0));

        // Card: exec key (shown for ADD_TO_EXISTING)
        JPanel cardKey = darkPanel(new FlowLayout(FlowLayout.LEFT, 4, 0));
        style(txtExecKey);
        txtExecKey.setToolTipText("Existing Test Execution key, e.g. PROJ-123");
        cardKey.add(styledLabel("Execution Key:"));
        cardKey.add(txtExecKey);

        // Card: exec summary (shown for CREATE_NEW_EXECUTION)
        JPanel cardSummary = darkPanel(new FlowLayout(FlowLayout.LEFT, 4, 0));
        style(txtExecSummary);
        txtExecSummary.setToolTipText("Summary for the new Test Execution issue");
        txtExecSummary.setText("Test Execution - Bulk Import");
        cardSummary.add(styledLabel("Execution Summary:"));
        cardSummary.add(txtExecSummary);

        pnlModeExtra.setBackground(BG_PANEL);
        pnlModeExtra.add(cardEmpty,   CARD_EMPTY);
        pnlModeExtra.add(cardKey,     CARD_EXEC_KEY);
        pnlModeExtra.add(cardSummary, CARD_EXEC_SUMMARY);

        JPanel outer = darkPanel(new BorderLayout(0, 4));
        outer.add(radioRow,     BorderLayout.NORTH);
        outer.add(pnlModeExtra, BorderLayout.CENTER);
        return outer;
    }

    private JPanel buildFileChooserRow(JTextField field, String btnLabel, String ext) {
        JPanel row = darkPanel(new BorderLayout(4, 0));
        style(field);
        field.setEditable(false);
        JButton btn = new JButton(btnLabel);
        btn.setPreferredSize(new Dimension(110, 26));
        btn.setBackground(new Color(60, 63, 75));
        btn.setForeground(FG_TEXT);
        btn.setFocusPainted(false);
        btn.setBorderPainted(false);
        btn.addActionListener(e -> {
            JFileChooser chooser = new JFileChooser();
            chooser.setFileFilter(new javax.swing.filechooser.FileNameExtensionFilter(
                    ext.toUpperCase() + " Files", ext));
            if (chooser.showOpenDialog(this) == JFileChooser.APPROVE_OPTION) {
                field.setText(chooser.getSelectedFile().getAbsolutePath());
            }
        });
        row.add(field, BorderLayout.CENTER);
        row.add(btn,   BorderLayout.EAST);
        return row;
    }

    private JPanel buildPreviewPanel() {
        JPanel panel = darkPanel(new BorderLayout());
        panel.setBorder(titledBorder("Preview"));

        previewTable.setBackground(BG_PANEL);
        previewTable.setForeground(FG_TEXT);
        previewTable.setGridColor(BORDER_COLOR);
        previewTable.setSelectionBackground(ACCENT.darker());
        previewTable.setSelectionForeground(Color.WHITE);
        previewTable.setFont(new Font("Consolas", Font.PLAIN, 12));
        previewTable.getTableHeader().setBackground(BG_DARK);
        previewTable.getTableHeader().setForeground(ACCENT);
        previewTable.getTableHeader().setFont(new Font("Segoe UI", Font.BOLD, 12));
        previewTable.setRowHeight(22);
        previewTable.setAutoResizeMode(JTable.AUTO_RESIZE_OFF);

        JScrollPane scroll = new JScrollPane(previewTable);
        scroll.getViewport().setBackground(BG_PANEL);
        scroll.setBackground(BG_DARK);
        panel.add(scroll, BorderLayout.CENTER);
        return panel;
    }

    private JPanel buildLogPanel() {
        JPanel panel = darkPanel(new BorderLayout());
        panel.setBorder(titledBorder("Live Log"));

        logArea.setBackground(new Color(20, 22, 28));
        logArea.setForeground(FG_TEXT);
        logArea.setFont(new Font("Consolas", Font.PLAIN, 12));
        logArea.setEditable(false);
        logArea.setLineWrap(true);
        logArea.setWrapStyleWord(true);

        JScrollPane scroll = new JScrollPane(logArea);
        scroll.getViewport().setBackground(new Color(20, 22, 28));
        panel.add(scroll, BorderLayout.CENTER);
        return panel;
    }

    private JPanel buildBottomPanel() {
        JPanel panel = darkPanel(new BorderLayout(8, 0));
        panel.setBorder(new EmptyBorder(6, 0, 0, 0));

        progressBar.setStringPainted(true);
        progressBar.setString("Ready");
        progressBar.setBackground(BG_PANEL);
        progressBar.setForeground(ACCENT);
        progressBar.setFont(new Font("Segoe UI", Font.BOLD, 11));
        progressBar.setBorderPainted(false);
        progressBar.setPreferredSize(new Dimension(0, 22));
        panel.add(progressBar, BorderLayout.CENTER);

        JLabel versionLabel = new JLabel("  v2.0.0  |  Xray Importer — Data Center");
        versionLabel.setForeground(new Color(120, 120, 130));
        versionLabel.setFont(new Font("Segoe UI", Font.ITALIC, 11));
        panel.add(versionLabel, BorderLayout.EAST);
        return panel;
    }

    // =========================================================================
    // Event Listeners
    // =========================================================================

    private void wireListeners() {
        // Radio buttons → swap CardLayout panel
        rbCreateOnly.addActionListener(e    -> showCard(CARD_EMPTY));
        rbAddToExisting.addActionListener(e -> showCard(CARD_EXEC_KEY));
        rbCreateNew.addActionListener(e     -> showCard(CARD_EXEC_SUMMARY));

        btnClearLog.addActionListener(e -> {
            logArea.setText("");
            progressBar.setValue(0);
            progressBar.setString("Ready");
        });

        btnTestConnection.addActionListener(e -> runAsync(this::doTestConnection));
        btnPreview.addActionListener(e        -> runAsync(this::doPreview));
        btnImport.addActionListener(e -> {
            int confirm = JOptionPane.showConfirmDialog(this,
                    "Start import to Jira / Xray DC?\nThis cannot be undone.",
                    "Confirm Import", JOptionPane.YES_NO_OPTION);
            if (confirm == JOptionPane.YES_OPTION) runAsync(this::doImport);
        });
    }

    private void showCard(String card) {
        ((CardLayout) pnlModeExtra.getLayout()).show(pnlModeExtra, card);
    }

    // =========================================================================
    // Actions
    // =========================================================================

    private void doTestConnection() {
        setUiBusy(true, "Testing connection...");
        try {
            String jiraUrl = getJiraUrl();
            validateField(jiraUrl, "Jira URL");
            String pat = getPat();
            validateField(pat, "Personal Access Token");

            try (JiraClient jira = new JiraClient(jiraUrl, pat, logger)) {
                String name = jira.testConnection();
                appendLog("[OK] Connected to Jira DC as: " + name);
            }
            setProgress(100, "Connection OK");
        } catch (Exception ex) {
            String msg = ex.getMessage() != null ? ex.getMessage() : ex.toString();
            appendLog("[ERROR] " + msg);
            showError("Connection Failed", msg);
        } finally {
            setUiBusy(false, null);
        }
    }

    private void doPreview() {
        setUiBusy(true, "Parsing CSV...");
        try {
            File csvFile    = validateFileField(txtCsvFile,    "CSV File");
            File configFile = validateFileField(txtConfigFile, "Config JSON");

            ImportConfig config = ConfigLoader.load(configFile);
            List<TestCase> cases = CsvParser.parse(csvFile, config);
            previewedTestCases = cases;

            SwingUtilities.invokeLater(() -> populatePreviewTable(cases));
            appendLog("[OK] Loaded " + cases.size() + " test cases from CSV.");
            int totalSteps = cases.stream().mapToInt(tc -> tc.getSteps().size()).sum();
            appendLog("     Total steps: " + totalSteps);
            if (config.isDryRun()) appendLog("[DRY-RUN] Config has dryRun=true — import will be simulated.");
            setProgress(100, "Preview complete — " + cases.size() + " test cases");
        } catch (Exception ex) {
            String msg = ex.getMessage() != null ? ex.getMessage() : ex.toString();
            appendLog("[ERROR] Preview failed: " + msg);
            showError("Preview Error", msg);
        } finally {
            setUiBusy(false, null);
        }
    }

    private void doImport() {
        setUiBusy(true, "Importing...");
        try {
            String jiraUrl = getJiraUrl();
            validateField(jiraUrl, "Jira URL");
            String pat = getPat();
            validateField(pat, "Personal Access Token");

            File csvFile    = validateFileField(txtCsvFile,    "CSV File");
            File configFile = validateFileField(txtConfigFile, "Config JSON");
            ImportConfig config = ConfigLoader.load(configFile);

            // Re-parse if preview wasn't run
            List<TestCase> snapshot = previewedTestCases;
            if (snapshot.isEmpty()) {
                snapshot = CsvParser.parse(csvFile, config);
                previewedTestCases = snapshot;
                appendLog("[INFO] Parsed " + snapshot.size() + " test cases.");
            }
            if (snapshot.isEmpty()) {
                appendLog("[WARN] No test cases to import.");
                return;
            }

            // Determine import mode and extra parameter
            ImportMode mode;
            String executionKeyOrSummary;
            if (rbAddToExisting.isSelected()) {
                mode = ImportMode.ADD_TO_EXISTING;
                executionKeyOrSummary = txtExecKey.getText().trim();
                validateField(executionKeyOrSummary, "Execution Key");
            } else if (rbCreateNew.isSelected()) {
                mode = ImportMode.CREATE_NEW_EXECUTION;
                executionKeyOrSummary = txtExecSummary.getText().trim();
            } else {
                mode = ImportMode.CREATE_ONLY;
                executionKeyOrSummary = "";
            }

            AppLogger log = new AppLogger(this::appendLog);
            UserCache userCache = new UserCache(
                    org.apache.http.impl.client.HttpClients.createDefault(),
                    jiraUrl, pat, log);

            final List<TestCase> testCases = snapshot;
            try (XrayDcClient xray = new XrayDcClient(jiraUrl, pat, log)) {

                int total = testCases.size();
                appendLog("[INFO] Starting import of " + total + " test cases (mode: " + mode + ")...");

                List<ImportResult> results = xray.bulkImport(
                        testCases, config, mode, executionKeyOrSummary, userCache,
                        progress -> SwingUtilities.invokeLater(() -> {
                            int pct = (int) ((progress[0] / (double) total) * 100);
                            progressBar.setValue(pct);
                            progressBar.setString(progress[0] + "/" + total + " tests created");
                        }),
                        this::appendLog
                );

                long ok   = results.stream().filter(ImportResult::isSuccess).count();
                long fail = results.stream().filter(r -> !r.isSuccess()).count();
                appendLog("──────────────────────────────");
                appendLog("Import complete: " + ok + " succeeded, " + fail + " failed.");
                setProgress(100, "Done: " + ok + "/" + total + " imported");

                SwingUtilities.invokeLater(() -> updateTableWithResults(results));
            }
        } catch (Exception ex) {
            String msg = ex.getMessage() != null ? ex.getMessage() : ex.toString();
            appendLog("[ERROR] Import failed: " + msg);
            StringWriter sw = new StringWriter();
            ex.printStackTrace(new PrintWriter(sw));
            appendLog(sw.toString());
            showError("Import Error", msg);
        } finally {
            setUiBusy(false, null);
        }
    }

    // =========================================================================
    // Table Helpers
    // =========================================================================

    private void populatePreviewTable(List<TestCase> cases) {
        previewTableModel.setColumnCount(0);
        previewTableModel.setRowCount(0);

        String[] cols = {"#", "Identifier", "Summary", "Priority", "Labels", "Assignee", "Steps", "Status"};
        for (String col : cols) previewTableModel.addColumn(col);

        int i = 1;
        for (TestCase tc : cases) {
            previewTableModel.addRow(new Object[]{
                    i++,
                    tc.getIdentifier(),
                    tc.getSummary(),
                    tc.getPriority()  != null ? tc.getPriority()  : "",
                    String.join(", ", tc.getLabels()),
                    tc.getAssignee()  != null ? tc.getAssignee()  : "",
                    tc.getSteps().size(),
                    "Pending"
            });
        }

        for (int col = 0; col < previewTable.getColumnCount(); col++) {
            int maxWidth = 80;
            for (int row = 0; row < previewTable.getRowCount(); row++) {
                Object val = previewTable.getValueAt(row, col);
                if (val != null) {
                    int w = previewTable.getFontMetrics(previewTable.getFont())
                            .stringWidth(val.toString()) + 20;
                    maxWidth = Math.max(maxWidth, w);
                }
            }
            previewTable.getColumnModel().getColumn(col).setPreferredWidth(Math.min(maxWidth, 300));
        }
    }

    private void updateTableWithResults(List<ImportResult> results) {
        int statusCol = previewTableModel.findColumn("Status");
        if (statusCol < 0) return;
        for (int row = 0; row < previewTableModel.getRowCount() && row < results.size(); row++) {
            ImportResult r = results.get(row);
            if (r.isDryRun()) {
                previewTableModel.setValueAt("DRY-RUN", row, statusCol);
            } else if (r.isSuccess()) {
                previewTableModel.setValueAt("✓ " + (r.getJiraKey() != null ? r.getJiraKey() : "OK"), row, statusCol);
            } else {
                previewTableModel.setValueAt("✗ FAILED", row, statusCol);
            }
        }
    }

    // =========================================================================
    // UI Utilities
    // =========================================================================

    private void appendLog(String message) {
        SwingUtilities.invokeLater(() -> {
            logArea.append(message + "\n");
            logArea.setCaretPosition(logArea.getDocument().getLength());
        });
    }

    private void setUiBusy(boolean busy, String progressMsg) {
        SwingUtilities.invokeLater(() -> {
            btnTestConnection.setEnabled(!busy);
            btnPreview.setEnabled(!busy);
            btnImport.setEnabled(!busy);
            rbCreateOnly.setEnabled(!busy);
            rbAddToExisting.setEnabled(!busy);
            rbCreateNew.setEnabled(!busy);
            if (progressMsg != null) {
                progressBar.setString(progressMsg);
                progressBar.setIndeterminate(busy);
            }
        });
    }

    private void setProgress(int value, String text) {
        SwingUtilities.invokeLater(() -> {
            progressBar.setIndeterminate(false);
            progressBar.setValue(value);
            progressBar.setString(text);
        });
    }

    private void showError(String title, String message) {
        SwingUtilities.invokeLater(() ->
                JOptionPane.showMessageDialog(this, message, title, JOptionPane.ERROR_MESSAGE));
    }

    private String getJiraUrl() {
        String url = txtJiraUrl.getText().trim();
        return url.endsWith("/") ? url.substring(0, url.length() - 1) : url;
    }

    private String getPat() {
        return new String(txtPat.getPassword()).trim();
    }

    private void validateField(String value, String fieldName) {
        if (value == null || value.isBlank())
            throw new IllegalArgumentException(fieldName + " is required.");
    }

    private File validateFileField(JTextField field, String fieldName) {
        String path = field.getText().trim();
        if (path.isBlank()) throw new IllegalArgumentException(fieldName + " is required.");
        File f = new File(path);
        if (!f.exists()) throw new IllegalArgumentException(fieldName + " does not exist: " + path);
        return f;
    }

    private void runAsync(Runnable task) {
        Thread thread = new Thread(task, "importer-worker");
        thread.setDaemon(true);
        thread.start();
    }

    // =========================================================================
    // Dark Theme / Styling
    // =========================================================================

    private void applyDarkTheme() {
        try { UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName()); }
        catch (Exception ignored) {}
        // Called after setContentPane() so it targets the real root panel
        getContentPane().setBackground(BG_DARK);
    }

    private JPanel darkPanel(LayoutManager layout) {
        JPanel p = new JPanel(layout);
        p.setBackground(BG_PANEL);
        return p;
    }

    private JLabel styledLabel(String text) {
        JLabel lbl = new JLabel(text);
        lbl.setForeground(FG_TEXT);
        lbl.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        return lbl;
    }

    private <T extends JTextField> T style(T field) {
        field.setBackground(new Color(55, 58, 68));
        field.setForeground(FG_TEXT);
        field.setCaretColor(FG_TEXT);
        field.setBorder(BorderFactory.createCompoundBorder(
                BorderFactory.createLineBorder(BORDER_COLOR),
                new EmptyBorder(3, 6, 3, 6)));
        field.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        return field;
    }

    private void styleButton(JButton btn, Color bg) {
        btn.setBackground(bg);
        btn.setForeground(Color.WHITE);
        btn.setFocusPainted(false);
        btn.setBorderPainted(false);
        btn.setFont(new Font("Segoe UI", Font.BOLD, 13));
        btn.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        btn.setPreferredSize(new Dimension(btn.getPreferredSize().width + 20, 32));
    }

    private TitledBorder titledBorder(String title) {
        TitledBorder border = BorderFactory.createTitledBorder(
                BorderFactory.createLineBorder(BORDER_COLOR), title);
        border.setTitleColor(ACCENT);
        border.setTitleFont(new Font("Segoe UI", Font.BOLD, 12));
        return border;
    }
}
```

-----

## File 16 — `src/main/resources/sample-config.json`

```json
{
  "projectKey": "PROJ",
  "issueType": "Test",
  "fieldMappings": {
    "summary":     "Test Case Name",
    "description": "Description",
    "assignee":    "Assignee Employee ID",
    "reporter":    "Reporter Employee ID",
    "priority":    "Priority",
    "labels":      "Labels"
  },
  "stepMappings": {
    "stepNo":         "Step no.",
    "action":         "Action",
    "data":           "Test Data",
    "expectedResult": "Expected Result"
  },
  "testCaseIdentifierColumn": "Test Case Identifier",
  "priorityMap": {
    "High":   "High",
    "Medium": "Medium",
    "Low":    "Low",
    "Critical": "Critical"
  },
  "dryRun":      false,
  "threadCount": 4,
  "maxRetries":  3
}
```

-----

## File 17 — `src/main/resources/sample-test-cases.csv`

```
Test Case Identifier,Test Case Name,Description,Priority,Labels,Assignee Employee ID,Reporter Employee ID,Custom Field,Step no.,Action,Test Data,Expected Result
TC-001,Login with valid credentials,Verify user can log in with valid credentials,High,regression;smoke,G01348533,G01348534,AUTH,1,Navigate to login page,https://app.example.com/login,Login page is displayed
TC-001,Login with valid credentials,Verify user can log in with valid credentials,High,regression;smoke,G01348533,G01348534,AUTH,2,Enter username,testuser@example.com,Username field populated
TC-001,Login with valid credentials,Verify user can log in with valid credentials,High,regression;smoke,G01348533,G01348534,AUTH,3,Enter password,P@ssw0rd!,Password field populated (masked)
TC-001,Login with valid credentials,Verify user can log in with valid credentials,High,regression;smoke,G01348533,G01348534,AUTH,4,Click Login button,,Dashboard page is displayed
TC-002,Login with invalid password,Verify error shown for wrong password,Medium,regression,G01348533,G01348534,AUTH,1,Navigate to login page,https://app.example.com/login,Login page is displayed
TC-002,Login with invalid password,Verify error shown for wrong password,Medium,regression,G01348533,G01348534,AUTH,2,Enter username,testuser@example.com,Username field populated
TC-002,Login with invalid password,Verify error shown for wrong password,Medium,regression,G01348533,G01348534,AUTH,3,Enter wrong password,WrongPass123,Password field populated
TC-002,Login with invalid password,Verify error shown for wrong password,Medium,regression,G01348533,G01348534,AUTH,4,Click Login button,,Error message "Invalid credentials" is shown
TC-003,Logout flow,Verify user can log out successfully,Low,smoke,G01348533,G01348534,AUTH,1,Log in as valid user,testuser@example.com / P@ssw0rd!,User is on Dashboard
TC-003,Logout flow,Verify user can log out successfully,Low,smoke,G01348533,G01348534,AUTH,2,Click user menu,,Dropdown menu appears
TC-003,Logout flow,Verify user can log out successfully,Low,smoke,G01348533,G01348534,AUTH,3,Click Logout,,User is redirected to login page
```

-----

## File 18 — `README.md`

```markdown
# Xray Importer Pro — Jira Data Center Edition

Import test cases from CSV directly into **Xray for Jira Data Center** via REST API.

## Requirements

- Java 17+
- Maven 3.6+ (to build)
- Jira Data Center with Xray plugin installed
- A **Personal Access Token (PAT)** — generate one in Jira → Profile → Personal Access Tokens

## Build

```bash
mvn clean package
java -jar target/xray-importer-pro-1.0.0.jar
```

## UI Fields

|Field                |Example                      |Notes                     |
|---------------------|-----------------------------|--------------------------|
|Jira URL             |`https://estjira.company.com`|No trailing slash         |
|Personal Access Token|`NjQ4ODc1...`                |From Jira → Profile → PATs|
|CSV File             |`test-cases.csv`             |See format below          |
|Config JSON          |`sample-config.json`         |Column mappings           |

## Import Modes (chosen at runtime)

|Mode                         |What happens                                                                 |
|-----------------------------|-----------------------------------------------------------------------------|
|**Create Tests Only**        |Creates a Jira “Test” issue per row group. No execution created.             |
|**Add to Existing Execution**|Creates tests, then adds all to the execution key you enter (e.g. `PROJ-456`)|
|**Create New Execution**     |Creates tests, creates a new “Test Execution” issue, adds all tests to it    |

## How it works (DC API calls)

For each test case in the CSV:

```
POST /rest/api/2/issue                          → creates the Test issue → returns PROJ-123
POST /rest/raven/1.0/testcase/PROJ-123/steps    → one call per step, in order
```

If mode is **Add to Existing**:

```
POST /rest/raven/1.0/testexec/{execKey}/test    → { "add": ["PROJ-123", ...] }
```

If mode is **Create New Execution**:

```
POST /rest/api/2/issue                          → creates Test Execution issue → PROJ-456
POST /rest/raven/1.0/testexec/PROJ-456/test     → { "add": ["PROJ-123", ...] }
```

## CSV Format

First row is the header. Each test case groups all its rows by **Test Case Identifier**.
Steps are sorted by **Step no.** within each group.

```
Test Case Identifier,Test Case Name,Description,Priority,Labels,Assignee Employee ID,Reporter Employee ID,Step no.,Action,Test Data,Expected Result
TC-001,Login Test,Verify login flow,High,smoke,G01348533,G09876543,1,Navigate to login page,,Login page is displayed
TC-001,Login Test,Verify login flow,High,smoke,G01348533,G09876543,2,Enter username and password,user@company.com / pass123,Credentials accepted
TC-001,Login Test,Verify login flow,High,smoke,G01348533,G09876543,3,Click Login,,Dashboard is shown
TC-002,Logout Test,Verify logout,Medium,,G01348533,,1,Click the user menu,,Dropdown appears
TC-002,Logout Test,Verify logout,Medium,,G01348533,,2,Click Sign Out,,Redirected to login page
```

Column order doesn’t matter — mappings are by name via `config.json`.

## Config JSON

```json
{
  "projectKey":  "PROJ",
  "issueType":   "Test",
  "fieldMappings": {
    "summary":     "Test Case Name",
    "description": "Description",
    "assignee":    "Assignee Employee ID",
    "reporter":    "Reporter Employee ID",
    "priority":    "Priority",
    "labels":      "Labels"
  },
  "stepMappings": {
    "stepNo":         "Step no.",
    "action":         "Action",
    "data":           "Test Data",
    "expectedResult": "Expected Result"
  },
  "testCaseIdentifierColumn": "Test Case Identifier",
  "priorityMap": {
    "High": "High", "Medium": "Medium", "Low": "Low"
  },
  "dryRun":      false,
  "threadCount": 4,
  "maxRetries":  3
}
```

Set `"dryRun": true` to simulate the import without creating any Jira issues.

```
---

## After creating all files

```bash
mvn clean package
java -jar target/xray-importer-pro-1.0.0.jar
```

Enter `https://estjira.company.com` as the Jira URL. Paste your Personal Access Token.
