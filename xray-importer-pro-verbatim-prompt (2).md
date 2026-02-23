# Copilot Workspace Task — Recreate Project: xray-importer-pro

Create a Maven Java 17 project by writing each file below **exactly as shown**, character for character. The project must build with `mvn clean package` and run with `java -jar target/xray-importer-pro-1.0.0.jar`.

---

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
    <n>Xray Importer Pro</n>

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

---

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

---

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

---

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

    @JsonProperty("xrayProjectKey")
    private String xrayProjectKey;

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

    public String getXrayProjectKey() { return xrayProjectKey; }
    public void setXrayProjectKey(String xrayProjectKey) { this.xrayProjectKey = xrayProjectKey; }

    public Map<String, Object> getExtra() { return extra; }
}
```

---

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

---

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
                  "xrayProjectKey": "PROJ",
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

---

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

---

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

---

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
 * Resolves Jira user accountId from an employee ID (e.g., G01348533).
 * Results are cached in memory.
 *
 * NOTE: ConcurrentHashMap does not permit null values, so we use the sentinel
 * value NOT_FOUND to represent a failed/negative lookup result.
 */
public class UserCache {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    /** Sentinel stored in the cache when a lookup returns no result. */
    private static final String NOT_FOUND = "__NOT_FOUND__";

    private final Map<String, String> cache = new ConcurrentHashMap<>();
    private final CloseableHttpClient httpClient;
    private final String jiraBaseUrl;
    private final String authHeader;
    private final AppLogger logger;

    public UserCache(CloseableHttpClient httpClient, String jiraBaseUrl, String authHeader, AppLogger logger) {
        this.httpClient = httpClient;
        this.jiraBaseUrl = jiraBaseUrl;
        this.authHeader = authHeader;
        this.logger = logger;
    }

    /**
     * Resolve an employee ID to a Jira accountId.
     * Returns null if not found or if employee ID is blank.
     */
    public String resolve(String employeeId) {
        if (employeeId == null || employeeId.isBlank()) return null;

        // Check cache first (including negative results stored as sentinel)
        String cached = cache.get(employeeId);
        if (cached != null) {
            return NOT_FOUND.equals(cached) ? null : cached;
        }

        try {
            String encoded = URLEncoder.encode(employeeId, StandardCharsets.UTF_8);
            String url = jiraBaseUrl + "/rest/api/3/user/search?query=" + encoded;
            HttpGet request = new HttpGet(url);
            request.setHeader("Authorization", authHeader);
            request.setHeader("Accept", "application/json");

            try (CloseableHttpResponse response = httpClient.execute(request)) {
                int status = response.getStatusLine().getStatusCode();
                String body = EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);

                if (status == 200) {
                    JsonNode arr = MAPPER.readTree(body);
                    if (arr.isArray() && arr.size() > 0) {
                        String accountId = arr.get(0).path("accountId").asText(null);
                        if (accountId != null && !accountId.isBlank()) {
                            cache.put(employeeId, accountId);
                            logger.info("Resolved user " + employeeId + " → " + accountId);
                            return accountId;
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

        // Cache negative result using sentinel to avoid repeated failed API calls
        cache.put(employeeId, NOT_FOUND);
        return null;
    }

    /** Clear the cache (useful for re-running imports). */
    public void clearCache() {
        cache.clear();
    }

    public int cacheSize() {
        return cache.size();
    }
}
```

---

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

---

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
import java.util.Base64;

/**
 * Client for Jira REST API interactions.
 */
public class JiraClient implements AutoCloseable {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final String jiraBaseUrl;
    private final String authHeader;
    private final CloseableHttpClient httpClient;
    private final AppLogger logger;

    public JiraClient(String jiraBaseUrl, String email, String apiToken, AppLogger logger) {
        this.jiraBaseUrl = normalizeUrl(jiraBaseUrl);
        String credentials = email + ":" + apiToken;
        this.authHeader = "Basic " + Base64.getEncoder().encodeToString(credentials.getBytes(StandardCharsets.UTF_8));
        this.httpClient = HttpClients.createDefault();
        this.logger = logger;
    }

    /**
     * Build a Basic auth header from Xray clientId:clientSecret (or any pre-built header).
     */
    public JiraClient(String jiraBaseUrl, String basicAuthHeader, AppLogger logger) {
        this.jiraBaseUrl = normalizeUrl(jiraBaseUrl);
        this.authHeader = basicAuthHeader;
        this.httpClient = HttpClients.createDefault();
        this.logger = logger;
    }

    /**
     * PAT (Personal Access Token) constructor — uses Bearer token scheme.
     * For Jira Data Center / Server instances configured with PAT support.
     */
    public static JiraClient withPat(String jiraBaseUrl, String pat, AppLogger logger) {
        return new JiraClient(jiraBaseUrl, "Bearer " + pat, logger);
    }

    public String getJiraBaseUrl() { return jiraBaseUrl; }
    public String getAuthHeader() { return authHeader; }
    public CloseableHttpClient getHttpClient() { return httpClient; }

    /**
     * Test connection to Jira by calling /rest/api/3/myself
     * Returns display name if successful, throws exception otherwise.
     */
    public String testConnection() throws IOException {
        String url = jiraBaseUrl + "/rest/api/3/myself";
        HttpGet request = new HttpGet(url);
        request.setHeader("Authorization", authHeader);
        request.setHeader("Accept", "application/json");

        try (CloseableHttpResponse response = httpClient.execute(request)) {
            int status = response.getStatusLine().getStatusCode();
            HttpEntity entity = response.getEntity();
            String body = entity != null ? EntityUtils.toString(entity, StandardCharsets.UTF_8) : "";

            if (status == 200) {
                JsonNode node = MAPPER.readTree(body);
                String displayName = node.path("displayName").asText("Unknown");
                String accountId = node.path("accountId").asText("");
                logger.success("Connected to Jira as: " + displayName + " (" + accountId + ")");
                return displayName;
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

---

## File 12 — `src/main/java/com/estjira/xray/XrayClient.java`

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

/**
 * Xray Cloud API client.
 * Handles authentication and bulk import of test cases via POST /api/v2/import/execution.
 */
public class XrayClient implements AutoCloseable {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final String XRAY_BASE = "https://xray.cloud.getxray.app";

    /** Timeout per individual test case import (seconds). Prevents indefinite hangs. */
    private static final int FUTURE_TIMEOUT_SECONDS = 120;

    private final String clientId;
    private final String clientSecret;
    private final AppLogger logger;
    private final CloseableHttpClient httpClient;

    private String xrayToken; // cached bearer token

    public XrayClient(String clientId, String clientSecret, String jiraBaseUrl, String jiraAuthHeader, AppLogger logger) {
        this.clientId = clientId;
        this.clientSecret = clientSecret;
        this.logger = logger;
        this.httpClient = HttpClients.createDefault();
        // jiraBaseUrl and jiraAuthHeader retained as constructor params for API compatibility
        // but Jira calls are handled externally by JiraClient / UserCache
    }

    // -------------------------------------------------------------------------
    // Authentication
    // -------------------------------------------------------------------------

    /**
     * Authenticates with Xray Cloud and caches the bearer token.
     */
    public String authenticate() throws IOException {
        HttpPost post = new HttpPost(XRAY_BASE + "/api/v2/authenticate");
        post.setHeader("Content-Type", "application/json");

        ObjectNode body = MAPPER.createObjectNode();
        body.put("client_id", clientId);
        body.put("client_secret", clientSecret);
        post.setEntity(new StringEntity(MAPPER.writeValueAsString(body), StandardCharsets.UTF_8));

        try (CloseableHttpResponse response = httpClient.execute(post)) {
            int status = response.getStatusLine().getStatusCode();
            String responseBody = EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);
            if (status == 200) {
                // Xray returns a bare JSON-quoted string token; strip the quotes
                xrayToken = responseBody.replace("\"", "").trim();
                logger.success("Xray authentication successful.");
                return xrayToken;
            } else {
                throw new IOException("Xray auth failed: HTTP " + status + " — " + responseBody);
            }
        }
    }

    private String getToken() throws IOException {
        if (xrayToken == null || xrayToken.isBlank()) {
            authenticate();
        }
        return xrayToken;
    }

    // -------------------------------------------------------------------------
    // Bulk Multithreaded Import
    // -------------------------------------------------------------------------

    /**
     * Import test cases in parallel, one Xray execution call per test case.
     *
     * @param testCases    All test cases to import
     * @param config       Import config (threadCount, maxRetries, dryRun, etc.)
     * @param executionKey Existing execution key to attach to (or blank to create new)
     * @param userCache    Resolves employee IDs to Jira accountIds
     * @param progressCb   Callback invoked with int[]{completed, total} after each task
     * @param logCb        Log message callback (called from worker threads)
     * @return List of ImportResult, one per test case, in submission order
     */
    public List<ImportResult> bulkImport(List<TestCase> testCases, ImportConfig config,
                                          String executionKey, UserCache userCache,
                                          Consumer<int[]> progressCb, Consumer<String> logCb) {
        // Use a size-matching list so we can index results correctly; CopyOnWriteArrayList for thread safety
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

        int threadCount = Math.max(1, config.getThreadCount());
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);

        List<Future<ImportResult>> futures = new ArrayList<>();
        for (TestCase tc : testCases) {
            futures.add(executor.submit(() -> {
                try {
                    String key = importSingleTest(tc, config, executionKey, userCache);
                    ImportResult result = ImportResult.success(tc.getIdentifier(), key, false);
                    logCb.accept("[OK] " + tc.getIdentifier() + " → " + key);
                    return result;
                } catch (Exception e) {
                    // Use toString() not getMessage() — getMessage() can return null for some exception types
                    String errMsg = e.getMessage() != null ? e.getMessage() : e.toString();
                    ImportResult result = ImportResult.failure(tc.getIdentifier(), errMsg);
                    logCb.accept("[FAIL] " + tc.getIdentifier() + ": " + errMsg);
                    return result;
                } finally {
                    synchronized (progress) {
                        progress[0]++;
                        progressCb.accept(new int[]{progress[0], total});
                    }
                }
            }));
        }

        // Stop accepting new tasks, then wait for all submitted tasks to finish
        executor.shutdown();

        for (Future<ImportResult> future : futures) {
            try {
                // Use timeout to prevent indefinite hangs if an HTTP call never returns
                results.add(future.get(FUTURE_TIMEOUT_SECONDS, TimeUnit.SECONDS));
            } catch (TimeoutException e) {
                future.cancel(true);
                logger.error("A task timed out after " + FUTURE_TIMEOUT_SECONDS + "s — skipping.");
            } catch (InterruptedException e) {
                // Restore the interrupt flag so the caller and JVM shutdown hooks are aware
                Thread.currentThread().interrupt();
                executor.shutdownNow();
                logger.error("Import interrupted — remaining tasks cancelled.");
                break;
            } catch (ExecutionException e) {
                String cause = e.getCause() != null ? e.getCause().toString() : e.toString();
                logger.error("Task execution failed: " + cause);
            }
        }

        return results;
    }

    /**
     * Import a single test case as its own Xray execution (called from worker threads in bulkImport).
     */
    private String importSingleTest(TestCase tc, ImportConfig config, String executionKey, UserCache userCache) throws Exception {
        String token = getToken();
        ObjectNode payload = buildSingleTestPayload(tc, config, executionKey, userCache);
        String payloadStr = MAPPER.writeValueAsString(payload);

        return RetryUtil.withRetry(() -> {
            HttpPost post = new HttpPost(XRAY_BASE + "/api/v2/import/execution");
            post.setHeader("Authorization", "Bearer " + token);
            post.setHeader("Content-Type", "application/json");
            post.setEntity(new StringEntity(payloadStr, StandardCharsets.UTF_8));

            try (CloseableHttpResponse response = httpClient.execute(post)) {
                int status = response.getStatusLine().getStatusCode();
                String body = EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);
                if (status == 200 || status == 201) {
                    JsonNode node = MAPPER.readTree(body);
                    return node.path("key").asText(
                            node.path("testExecIssue").path("key").asText("IMPORTED"));
                } else {
                    throw new IOException("HTTP " + status + ": " + truncate(body, 200));
                }
            }
        }, config.getMaxRetries(), 1000L, logger::warn);
    }

    // -------------------------------------------------------------------------
    // Payload Builders
    // -------------------------------------------------------------------------

    private ObjectNode buildSingleTestPayload(TestCase tc, ImportConfig config,
                                               String executionKey, UserCache userCache) {
        ObjectNode root = MAPPER.createObjectNode();
        ObjectNode execInfo = MAPPER.createObjectNode();
        if (executionKey != null && !executionKey.isBlank()) {
            execInfo.put("testExecutionKey", executionKey);
        } else {
            execInfo.put("project", config.getXrayProjectKey() != null ?
                    config.getXrayProjectKey() : config.getProjectKey());
            execInfo.put("summary", "Test Execution - " + tc.getIdentifier());
        }
        root.set("testExecutionInfo", execInfo);

        ArrayNode testsArr = MAPPER.createArrayNode();
        testsArr.add(buildTestNode(tc, config, userCache));
        root.set("tests", testsArr);

        return root;
    }

    private ObjectNode buildTestNode(TestCase tc, ImportConfig config, UserCache userCache) {
        ObjectNode test = MAPPER.createObjectNode();
        ObjectNode fields = MAPPER.createObjectNode();

        fields.put("summary", tc.getSummary() != null ? tc.getSummary() : tc.getIdentifier());

        ObjectNode issueType = MAPPER.createObjectNode();
        issueType.put("name", config.getIssueType());
        fields.set("issuetype", issueType);

        ObjectNode project = MAPPER.createObjectNode();
        project.put("key", config.getProjectKey());
        fields.set("project", project);

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

        if (tc.getAssignee() != null && !tc.getAssignee().isBlank() && userCache != null) {
            String accountId = userCache.resolve(tc.getAssignee());
            if (accountId != null) {
                ObjectNode assignee = MAPPER.createObjectNode();
                assignee.put("accountId", accountId);
                fields.set("assignee", assignee);
            }
        }

        if (tc.getReporter() != null && !tc.getReporter().isBlank() && userCache != null) {
            String accountId = userCache.resolve(tc.getReporter());
            if (accountId != null) {
                ObjectNode reporter = MAPPER.createObjectNode();
                reporter.put("accountId", accountId);
                fields.set("reporter", reporter);
            }
        }

        // Extra/custom fields (e.g. customfield_10200)
        for (var entry : tc.getExtraFields().entrySet()) {
            if (entry.getValue() instanceof String s) {
                fields.put(entry.getKey(), s);
            }
        }

        test.set("fields", fields);

        // Xray test metadata: type + steps
        ObjectNode testInfo = MAPPER.createObjectNode();
        testInfo.put("type", "Manual");

        if (!tc.getSteps().isEmpty()) {
            ArrayNode stepsArr = MAPPER.createArrayNode();
            for (TestStep step : tc.getSteps()) {
                ObjectNode s = MAPPER.createObjectNode();
                s.put("action", step.getAction() != null ? step.getAction() : "");
                s.put("data", step.getData() != null ? step.getData() : "");
                s.put("result", step.getExpectedResult() != null ? step.getExpectedResult() : "");
                stepsArr.add(s);
            }
            testInfo.set("steps", stepsArr);
        }
        test.set("testInfo", testInfo);

        return test;
    }

    // -------------------------------------------------------------------------
    // Utility
    // -------------------------------------------------------------------------

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

---

## File 13 — `src/main/java/com/estjira/ui/MainApp.java`

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

---

## File 14 — `src/main/java/com/estjira/ui/MainWindow.java`

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
import com.estjira.xray.XrayClient;

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
 * Main application window for the Jira Test Case Importer Pro.
 */
public class MainWindow extends JFrame {

    // ── Connection Fields ───────────────────────────────────────────────────
    private final JTextField txtJiraUrl = new JTextField(30);
    private final JPasswordField txtPat = new JPasswordField(20);
    private final JTextField txtXrayClientId = new JTextField(20);
    private final JPasswordField txtXrayClientSecret = new JPasswordField(20);

    // ── File Choosers ───────────────────────────────────────────────────────
    private final JTextField txtCsvFile = new JTextField(30);
    private final JTextField txtConfigFile = new JTextField(30);

    // ── Execution Options ───────────────────────────────────────────────────
    private final JCheckBox chkUseExistingExecution = new JCheckBox("Use Existing Test Execution");
    private final JTextField txtExecutionKey = new JTextField(15);

    // ── Buttons ─────────────────────────────────────────────────────────────
    private final JButton btnTestConnection = new JButton("Test Connection");
    private final JButton btnPreview = new JButton("Preview");
    private final JButton btnImport = new JButton("Import to Jira");
    private final JButton btnClearLog = new JButton("Clear Log");

    // ── Preview Table ───────────────────────────────────────────────────────
    private final DefaultTableModel previewTableModel = new DefaultTableModel();
    private final JTable previewTable = new JTable(previewTableModel);

    // ── Log Area ────────────────────────────────────────────────────────────
    private final JTextArea logArea = new JTextArea(12, 60);

    // ── Progress Bar ────────────────────────────────────────────────────────
    private final JProgressBar progressBar = new JProgressBar(0, 100);

    // ── State ───────────────────────────────────────────────────────────────
    // volatile: written by background thread, read by EDT and other background threads
    private volatile List<TestCase> previewedTestCases = new ArrayList<>();
    private AppLogger logger;

    // ── Colours ─────────────────────────────────────────────────────────────
    private static final Color BG_DARK    = new Color(30, 33, 40);
    private static final Color BG_PANEL   = new Color(42, 45, 54);
    private static final Color ACCENT     = new Color(99, 179, 237);
    private static final Color FG_TEXT    = new Color(220, 220, 220);
    private static final Color BORDER_COLOR = new Color(70, 73, 82);

    public MainWindow() {
        super("Jira Test Case Importer — Pro");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setMinimumSize(new Dimension(1100, 750));
        setPreferredSize(new Dimension(1200, 820));

        // Logger that appends to the log area
        logger = new AppLogger(this::appendLog);

        // Build and set content pane FIRST, then apply theme on the real pane
        setContentPane(buildContentPane());
        applyDarkTheme();

        // Wire listeners
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
        c.fill = GridBagConstraints.HORIZONTAL;

        c.gridx = 0; c.gridy = 0; c.weightx = 0; grid.add(styledLabel("Jira URL:"), c);
        c.gridx = 1; c.gridy = 0; c.weightx = 1; grid.add(style(txtJiraUrl), c);

        c.gridx = 0; c.gridy = 1; c.weightx = 0; grid.add(styledLabel("Jira PAT:"), c);
        style(txtPat);
        txtPat.setToolTipText("Personal Access Token — if set, overrides Xray Client ID/Secret for Jira auth");
        c.gridx = 1; c.gridy = 1; c.weightx = 1; grid.add(txtPat, c);

        c.gridx = 0; c.gridy = 2; c.weightx = 0; grid.add(styledLabel("Xray Client ID:"), c);
        c.gridx = 1; c.gridy = 2; c.weightx = 1; grid.add(style(txtXrayClientId), c);

        c.gridx = 0; c.gridy = 3; c.weightx = 0; grid.add(styledLabel("Xray Client Secret:"), c);
        c.gridx = 1; c.gridy = 3; c.weightx = 1; grid.add(style(txtXrayClientSecret), c);

        c.gridx = 0; c.gridy = 4; c.weightx = 0; grid.add(styledLabel("CSV File:"), c);
        c.gridx = 1; c.gridy = 4; c.weightx = 1; grid.add(buildFileChooserRow(txtCsvFile, "Select CSV", "csv"), c);

        c.gridx = 0; c.gridy = 5; c.weightx = 0; grid.add(styledLabel("Config JSON:"), c);
        c.gridx = 1; c.gridy = 5; c.weightx = 1; grid.add(buildFileChooserRow(txtConfigFile, "Select JSON", "json"), c);

        c.gridx = 0; c.gridy = 6; c.weightx = 0; grid.add(styledLabel("Execution:"), c);
        JPanel execRow = darkPanel(new FlowLayout(FlowLayout.LEFT, 4, 0));
        chkUseExistingExecution.setBackground(BG_PANEL);
        chkUseExistingExecution.setForeground(FG_TEXT);
        chkUseExistingExecution.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        style(txtExecutionKey);
        txtExecutionKey.setEnabled(false);
        txtExecutionKey.setToolTipText("Enter existing execution key (e.g. PROJ-123)");
        execRow.add(chkUseExistingExecution);
        execRow.add(txtExecutionKey);
        c.gridx = 1; c.gridy = 6; c.weightx = 1; grid.add(execRow, c);

        c.gridx = 0; c.gridy = 7; c.gridwidth = 2; c.weightx = 0;
        JPanel btnRow = darkPanel(new FlowLayout(FlowLayout.LEFT, 8, 0));
        styleButton(btnTestConnection, new Color(52, 152, 219));
        styleButton(btnPreview, new Color(39, 174, 96));
        styleButton(btnImport, new Color(155, 89, 182));
        styleButton(btnClearLog, new Color(100, 100, 110));
        btnRow.add(btnTestConnection);
        btnRow.add(btnPreview);
        btnRow.add(btnImport);
        btnRow.add(btnClearLog);
        grid.add(btnRow, c);

        outer.add(grid, BorderLayout.CENTER);
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
        row.add(btn, BorderLayout.EAST);
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

        JLabel versionLabel = new JLabel("  v1.0.0  |  Xray Importer Pro");
        versionLabel.setForeground(new Color(120, 120, 130));
        versionLabel.setFont(new Font("Segoe UI", Font.ITALIC, 11));
        panel.add(versionLabel, BorderLayout.EAST);

        return panel;
    }

    // =========================================================================
    // Event Listeners
    // =========================================================================

    private void wireListeners() {
        chkUseExistingExecution.addActionListener(e ->
                txtExecutionKey.setEnabled(chkUseExistingExecution.isSelected()));

        btnClearLog.addActionListener(e -> {
            logArea.setText("");
            progressBar.setValue(0);
            progressBar.setString("Ready");
        });

        btnTestConnection.addActionListener(e -> runAsync(this::doTestConnection));
        btnPreview.addActionListener(e -> runAsync(this::doPreview));
        btnImport.addActionListener(e -> {
            int confirm = JOptionPane.showConfirmDialog(this,
                    "Start import to Jira / Xray?\nThis cannot be undone.",
                    "Confirm Import", JOptionPane.YES_NO_OPTION);
            if (confirm == JOptionPane.YES_OPTION) runAsync(this::doImport);
        });
    }

    // =========================================================================
    // Actions
    // =========================================================================

    private void doTestConnection() {
        setUiBusy(true, "Testing connection...");
        try {
            String jiraUrl = getJiraUrl();
            validateField(jiraUrl, "Jira URL");

            String clientId = txtXrayClientId.getText().trim();
            String clientSecret = new String(txtXrayClientSecret.getPassword()).trim();
            String pat = new String(txtPat.getPassword()).trim();

            JiraClient jira = pat.isBlank()
                    ? new JiraClient(jiraUrl, clientId, clientSecret, logger)
                    : JiraClient.withPat(jiraUrl, pat, logger);

            try (JiraClient j = jira) {
                String name = j.testConnection();
                appendLog("[OK] Jira connection successful — logged in as: " + name);
                if (!pat.isBlank()) {
                    appendLog("[INFO] Authenticated via PAT (Bearer token).");
                }
            }

            if (!pat.isBlank()) {
                appendLog("[WARN] PAT mode — skipping Xray OAuth test (Xray Client ID/Secret still needed for import).");
            } else if (!clientId.isBlank() && !clientSecret.isBlank()) {
                try (XrayClient xray = new XrayClient(clientId, clientSecret, jiraUrl, "", logger)) {
                    xray.authenticate();
                }
            } else {
                appendLog("[WARN] Xray Client ID/Secret not provided — skipping Xray auth test.");
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
            File csvFile = validateFileField(txtCsvFile, "CSV File");
            File configFile = validateFileField(txtConfigFile, "Config JSON");

            ImportConfig config = ConfigLoader.load(configFile);
            List<TestCase> cases = CsvParser.parse(csvFile, config);
            previewedTestCases = cases; // volatile write — visible to all threads

            SwingUtilities.invokeLater(() -> populatePreviewTable(cases));
            appendLog("[OK] Loaded " + cases.size() + " test cases from CSV.");
            int totalSteps = cases.stream().mapToInt(tc -> tc.getSteps().size()).sum();
            appendLog("     Total steps: " + totalSteps);
            if (config.isDryRun()) appendLog("[DRY-RUN] Config has dryRun=true. Import will be simulated.");
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
            String clientId = txtXrayClientId.getText().trim();
            String clientSecret = new String(txtXrayClientSecret.getPassword()).trim();
            String pat = new String(txtPat.getPassword()).trim();
            File csvFile = validateFileField(txtCsvFile, "CSV File");
            File configFile = validateFileField(txtConfigFile, "Config JSON");

            ImportConfig config = ConfigLoader.load(configFile);

            // Volatile read — safe even if doPreview ran on a different thread
            List<TestCase> snapshot = previewedTestCases;
            if (snapshot.isEmpty()) {
                snapshot = CsvParser.parse(csvFile, config);
                previewedTestCases = snapshot; // volatile write
                appendLog("[INFO] Parsed " + snapshot.size() + " test cases.");
            }

            if (snapshot.isEmpty()) {
                appendLog("[WARN] No test cases to import.");
                return;
            }

            String executionKey = chkUseExistingExecution.isSelected() ? txtExecutionKey.getText().trim() : "";

            // Build Jira auth header: PAT takes priority over Basic Auth
            String jiraAuth;
            if (!pat.isBlank()) {
                jiraAuth = "Bearer " + pat;
                appendLog("[INFO] Using PAT (Bearer) for Jira authentication.");
            } else {
                jiraAuth = "Basic " + java.util.Base64.getEncoder().encodeToString(
                        (clientId + ":" + clientSecret).getBytes(java.nio.charset.StandardCharsets.UTF_8));
            }

            AppLogger log = new AppLogger(this::appendLog);
            UserCache userCache = new UserCache(
                    org.apache.http.impl.client.HttpClients.createDefault(),
                    jiraUrl, jiraAuth, log);

            final List<TestCase> testCases = snapshot;
            try (XrayClient xray = new XrayClient(clientId, clientSecret, jiraUrl, jiraAuth, log)) {
                xray.authenticate();

                int total = testCases.size();
                List<ImportResult> results = xray.bulkImport(
                        testCases, config, executionKey, userCache,
                        progress -> SwingUtilities.invokeLater(() -> {
                            int pct = (int) ((progress[0] / (double) total) * 100);
                            progressBar.setValue(pct);
                            progressBar.setString(progress[0] + "/" + total + " imported");
                        }),
                        this::appendLog
                );

                long ok = results.stream().filter(ImportResult::isSuccess).count();
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
                    tc.getPriority() != null ? tc.getPriority() : "",
                    String.join(", ", tc.getLabels()),
                    tc.getAssignee() != null ? tc.getAssignee() : "",
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
    // UI Utility Methods
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
        if (url.endsWith("/")) url = url.substring(0, url.length() - 1);
        return url;
    }

    private void validateField(String value, String fieldName) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException(fieldName + " is required.");
        }
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
        try {
            UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());
        } catch (Exception ignored) {}
        // Applied after setContentPane() so it affects the actual root panel
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

---

## File 15 — `src/main/resources/sample-config.json`

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
  "priorityMap": {
    "High": "High",
    "Medium": "Medium",
    "Low": "Low",
    "Critical": "Highest",
    "Blocker": "Highest"
  },
  "dryRun": false,
  "threadCount": 4,
  "maxRetries": 3
}
```

---

## File 16 — `src/main/resources/sample-test-cases.csv`

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

---

## File 17 — `README.md`

```markdown
# Xray Importer Pro

A Java 17 Swing desktop application for importing manual test cases from CSV into Jira/Xray Cloud.

## Features

- **Dark-themed Swing UI** with live log, preview table, and progress bar
- **CSV parsing** — groups rows by "Test Case Identifier", preserves step order by "Step no."
- **JSON config mapping** — dynamically maps CSV columns → Jira fields (labels, priority, customfield_xxxxx)
- **User resolution** — resolves employee IDs (e.g. G01348533) to Jira accountIds via `/rest/api/3/user/search`
- **Xray Cloud import** — `POST /api/v2/import/execution` with manual test steps
- **Existing execution support** — attach tests to an existing Test Execution key
- **Multithreaded bulk import** — configurable thread count
- **Retry with exponential backoff** — configurable max retries
- **Dry-run mode** — set `"dryRun": true` in config JSON to simulate without importing
- **PAT authentication** — supports Jira Personal Access Tokens (Bearer) in addition to Basic Auth

## Quick Start

### 1. Build the fat JAR

```bash
mvn clean package -q
```

This produces `target/xray-importer-pro-1.0.0.jar`.

### 2. Run

```bash
java -jar target/xray-importer-pro-1.0.0.jar
```

### 3. Fill in the UI

| Field | Value |
|-------|-------|
| Jira URL | `https://estjira.company.com` |
| Jira PAT | Your Personal Access Token (optional — overrides Basic Auth for Jira) |
| Xray Client ID | Your Xray Cloud Client ID |
| Xray Client Secret | Your Xray Cloud Client Secret |
| CSV File | Select your test case CSV |
| Config JSON | Select your mapping config JSON |

> **PAT vs Basic Auth:** If you fill in **Jira PAT**, it is used as a `Bearer` token for all Jira REST calls (`/rest/api/3/myself`, user resolution, etc.). The Xray Client ID and Secret are always required for Xray Cloud OAuth regardless of which Jira auth method you choose.

### 4. Workflow

1. Click **Test Connection** — validates Jira and Xray credentials
2. Click **Preview** — parses CSV, shows test cases in the table
3. *(Optional)* Check **Use Existing Test Execution** and enter the execution key
4. Click **Import to Jira** — runs the bulk import

## CSV Format

The CSV must have a header row. Required columns (configurable via `stepMappings`):

| Column | Purpose |
|--------|---------|
| `Test Case Identifier` | Groups rows into test cases |
| `Test Case Name` | Maps to Jira `summary` |
| `Step no.` | Step ordering (integer) |
| `Action` | Step action |
| `Test Data` | Step test data |
| `Expected Result` | Step expected result |

See `src/main/resources/sample-test-cases.csv` for a working example.

## Config JSON

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
    "customfield_10200": "My Custom Column"
  },
  "stepMappings": {
    "stepNo": "Step no.",
    "action": "Action",
    "data": "Test Data",
    "expectedResult": "Expected Result"
  },
  "priorityMap": {
    "High": "High",
    "Critical": "Highest"
  },
  "dryRun": false,
  "threadCount": 4,
  "maxRetries": 3
}
```

See `src/main/resources/sample-config.json` for the full sample.

## Package Structure

```
com.estjira
├── ui        — Swing UI (MainApp, MainWindow)
├── csv       — CSV parsing (CsvParser)
├── xray      — Xray Cloud API client (XrayClient)
├── jira      — Jira REST client (JiraClient)
├── config    — Config JSON loader (ConfigLoader)
├── model     — Data models (TestCase, TestStep, ImportConfig, ImportResult)
└── util      — Utilities (AppLogger, RetryUtil, UserCache)
```

## Dependencies

- **Jackson** 2.16.1 — JSON parsing
- **OpenCSV** 5.9 — CSV parsing
- **Apache HttpClient** 4.5.14 — HTTP requests

## Requirements

- Java 17+
- Maven 3.6+
```

---

## After creating all files

Build:
```
mvn clean package
```
Run:
```
java -jar target/xray-importer-pro-1.0.0.jar
```

Enter `https://estjira.company.com` in the Jira URL field when the application launches.
Fill in your **Jira PAT** in the "Jira PAT" field to use Bearer token authentication instead of Basic Auth.
