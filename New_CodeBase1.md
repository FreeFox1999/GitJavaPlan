@echo off
setlocal EnableExtensions DisableDelayedExpansion
title Drafter Portable Runner (Run existing JAR)
REM ================================================================
REM Portable Runner (Run pre-built JAR)
REM - Uses portable JDK from:    tools\jdk-17\
REM - Uses portable Maven from:  tools\maven\
REM - Runs existing JAR:         app\drafter-tool.jar
REM - Clears injected envs, then sets local envs for THIS SESSION:
REM       JAVA_HOME, MAVEN_HOME, PATH, MAVEN_OPTS
REM - Sets MAVEN_OPTS trustStore to dynamic absolute path:
REM       "<bundle_root>\certs\java.cacerts" (quoted to handle spaces)
REM - No JAVA_TOOL_OPTIONS
REM - Output: app\out\<Association>_<yyyyMMdd_HHmmss>.xlsx
REM - Logs:   run.log (bundle root)
REM ================================================================

REM ---- Resolve bundle root (folder where this .cmd exists) ----
set "BUNDLE_ROOT=%~dp0"
if "%BUNDLE_ROOT:~-1%"=="\" set "BUNDLE_ROOT=%BUNDLE_ROOT:~0,-1%"

set "APP_DIR=%BUNDLE_ROOT%\app"
set "TOOLS_DIR=%BUNDLE_ROOT%\tools"
set "CERTS_DIR=%BUNDLE_ROOT%\certs"
set "LOG=%BUNDLE_ROOT%\run.log"

REM ---- Portable tool locations ----
set "JAVA_HOME=%TOOLS_DIR%\jdk-17"
set "MAVEN_HOME=%TOOLS_DIR%\maven"
set "JAVA_EXE=%JAVA_HOME%\bin\java.exe"
set "MVN_CMD=%MAVEN_HOME%\bin\mvn.cmd"

REM ---- Fixed inputs (relative -> resolved dynamically) ----
set "CERT_FILE=%CERTS_DIR%\java.cacerts"
set "JAR=%APP_DIR%\drafter-tool.jar"

REM ---- Start log ----
echo ===== RUN START: %DATE% %TIME% ===== > "%LOG%"
echo ===== Drafter Portable Runner (Run existing JAR) =====
echo Bundle : %BUNDLE_ROOT%
echo App    : %APP_DIR%
echo Tools  : %TOOLS_DIR%
echo Certs  : %CERTS_DIR%
echo Log    : %LOG%
echo.

REM ================================================================
REM 0) Clear potentially injected env overrides (session-only)
REM ================================================================
echo --- Pre-existing env overrides (before clearing) --- >> "%LOG%"
echo MAVEN_OPTS=%MAVEN_OPTS% >> "%LOG%"
echo MAVEN_ARGS=%MAVEN_ARGS% >> "%LOG%"
echo JAVA_TOOL_OPTIONS=%JAVA_TOOL_OPTIONS% >> "%LOG%"
echo _JAVA_OPTIONS=%_JAVA_OPTIONS% >> "%LOG%"
echo JDK_JAVA_OPTIONS=%JDK_JAVA_OPTIONS% >> "%LOG%"

set "MAVEN_OPTS="
set "MAVEN_ARGS="
set "JAVA_TOOL_OPTIONS="
set "_JAVA_OPTIONS="
set "JDK_JAVA_OPTIONS="

REM ---- Show env after clearing ----
echo --- Env after clearing (this session) ---
echo MAVEN_OPTS=%MAVEN_OPTS%
echo MAVEN_ARGS=%MAVEN_ARGS%
echo JAVA_TOOL_OPTIONS=%JAVA_TOOL_OPTIONS%
echo _JAVA_OPTIONS=%_JAVA_OPTIONS%
echo JDK_JAVA_OPTIONS=%JDK_JAVA_OPTIONS%
echo --- Env after clearing (this session) --- >> "%LOG%"
echo MAVEN_OPTS=%MAVEN_OPTS% >> "%LOG%"
echo MAVEN_ARGS=%MAVEN_ARGS% >> "%LOG%"
echo JAVA_TOOL_OPTIONS=%JAVA_TOOL_OPTIONS% >> "%LOG%"
echo _JAVA_OPTIONS=%_JAVA_OPTIONS% >> "%LOG%"
echo JDK_JAVA_OPTIONS=%JDK_JAVA_OPTIONS% >> "%LOG%"

REM ================================================================
REM 0c) Set local envs for THIS SESSION
REM ================================================================
set "PATH=%JAVA_HOME%\bin;%MAVEN_HOME%\bin;%PATH%"

REM [FIX 1] Quote the path in MAVEN_OPTS so spaces in the path don't
REM         cause JVM to interpret the second token as the main class.
REM         Inner quotes become part of the env var value, which Maven
REM         passes correctly to the JVM as a single token.
set "MAVEN_OPTS="
set "TRUSTSTORE_ARG="
if exist "%CERT_FILE%" (
  for %%I in ("%CERT_FILE%") do (
    set "MAVEN_OPTS=-Djavax.net.ssl.trustStore="%%~fI""
    set "TRUSTSTORE_ARG=-Djavax.net.ssl.trustStore=%%~fI"
  )
) else (
  echo [WARN] Certificate file not found: "%CERT_FILE%"
  echo [WARN] Certificate file not found: "%CERT_FILE%" >> "%LOG%"
)

REM ---- Show local envs after setting ----
echo.
echo --- Local env set (this session) ---
echo JAVA_HOME=%JAVA_HOME%
echo MAVEN_HOME=%MAVEN_HOME%
echo MAVEN_OPTS=%MAVEN_OPTS%
echo TRUSTSTORE_ARG=%TRUSTSTORE_ARG%
echo --- Local env set (this session) --- >> "%LOG%"
echo JAVA_HOME=%JAVA_HOME% >> "%LOG%"
echo MAVEN_HOME=%MAVEN_HOME% >> "%LOG%"
echo MAVEN_OPTS=%MAVEN_OPTS% >> "%LOG%"
echo TRUSTSTORE_ARG=%TRUSTSTORE_ARG% >> "%LOG%"

REM ================================================================
REM 1) Validate required paths
REM ================================================================
if not exist "%APP_DIR%" (
  echo [ERROR] Missing app folder: "%APP_DIR%"
  echo [ERROR] Missing app folder: "%APP_DIR%" >> "%LOG%"
  goto :fail
)
if not exist "%JAVA_EXE%" (
  echo [ERROR] Missing portable Java: "%JAVA_EXE%"
  echo Fix: tools\jdk-17\bin\java.exe must exist.
  echo [ERROR] Missing portable Java: "%JAVA_EXE%" >> "%LOG%"
  goto :fail
)
if not exist "%MVN_CMD%" (
  echo [ERROR] Missing portable Maven: "%MVN_CMD%"
  echo Fix: tools\maven\bin\mvn.cmd must exist.
  echo [ERROR] Missing portable Maven: "%MVN_CMD%" >> "%LOG%"
  goto :fail
)
if not exist "%JAR%" (
  echo [ERROR] Missing jar: "%JAR%"
  echo [ERROR] Missing jar: "%JAR%" >> "%LOG%"
  echo [INFO] Jars under app\: >> "%LOG%"
  dir /b /a:-d "%APP_DIR%\*.jar" >> "%LOG%" 2>&1
  goto :fail
)

REM ================================================================
REM 2) Print versions (for supportability)
REM ================================================================
echo.
echo --- Java Version ---
"%JAVA_EXE%" -version
"%JAVA_EXE%" -version >> "%LOG%" 2>&1
if errorlevel 1 goto :fail

echo.
echo --- Maven Version ---
call "%MVN_CMD%" -v
call "%MVN_CMD%" -v >> "%LOG%" 2>&1
if errorlevel 1 goto :fail

REM ================================================================
REM 3) Parse CLI args (defaults preserved)
REM ================================================================
set "ASSOC=Mastercard"
set "P1=5"
set "P2=2"
set "P3=1"
set "P4=1"
set "P5=1"
set "OUT="

:parseArgs
if "%~1"=="" goto :argsDone
if /I "%~1"=="--association" ( set "ASSOC=%~2" & shift & shift & goto :parseArgs )
if /I "%~1"=="--p1"          ( set "P1=%~2"   & shift & shift & goto :parseArgs )
if /I "%~1"=="--p2"          ( set "P2=%~2"   & shift & shift & goto :parseArgs )
if /I "%~1"=="--p3"          ( set "P3=%~2"   & shift & shift & goto :parseArgs )
if /I "%~1"=="--p4"          ( set "P4=%~2"   & shift & shift & goto :parseArgs )
if /I "%~1"=="--p5"          ( set "P5=%~2"   & shift & shift & goto :parseArgs )
if /I "%~1"=="--out"         ( set "OUT=%~2"  & shift & shift & goto :parseArgs )
shift
goto :parseArgs

:argsDone
if /I "%ASSOC%"=="visa"       set "ASSOC=Visa"
if /I "%ASSOC%"=="mastercard" set "ASSOC=Mastercard"

REM ---- Timestamp yyyyMMdd_HHmmss ----
set "TS="
for /f %%i in ('powershell -NoProfile -Command "Get-Date -Format yyyyMMdd_HHmmss" 2^>nul') do set "TS=%%i"
if not defined TS (
  for /f "tokens=2 delims==" %%i in ('wmic os get localdatetime /value 2^>nul ^| find "="') do set "ldt=%%i"
  if defined ldt set "TS=%ldt:~0,8%_%ldt:~8,6%"
)

if not defined OUT set "OUT=out\%ASSOC%_%TS%.xlsx"
if not exist "%APP_DIR%\out" mkdir "%APP_DIR%\out" >nul 2>&1

echo.
echo Association : %ASSOC%
echo Output file : %APP_DIR%\%OUT%
echo Params      : p1=%P1% p2=%P2% p3=%P3% p4=%P4% p5=%P5%
echo.

REM ================================================================
REM 4) Run jar (live output + log)
REM [FIX 2] MAVEN_OPTS does NOT affect java -jar, so we must pass
REM         the trustStore flag explicitly via TRUSTSTORE_ARG.
REM         The path is stored unquoted in TRUSTSTORE_ARG; PowerShell
REM         receives it via an expandable here-string so spaces are safe.
REM ================================================================
echo ===== Running jar =====
echo "%JAVA_EXE%" "%TRUSTSTORE_ARG%" -jar "%JAR%" --association "%ASSOC%" --p1 %P1% --p2 %P2% --p3 %P3% --p4 %P4% --p5 %P5% --out "%OUT%"
echo.

REM Build the trustStore argument safely for PowerShell (handle spaces in path)
set "PS_TRUST="
if defined TRUSTSTORE_ARG set "PS_TRUST=%TRUSTSTORE_ARG%"

powershell -NoProfile -Command ^
  "& { $javaArgs = @(); if ($env:PS_TRUST) { $javaArgs += $env:PS_TRUST }; $javaArgs += '-jar', '%JAR%', '--association', '%ASSOC%', '--p1', '%P1%', '--p2', '%P2%', '--p3', '%P3%', '--p4', '%P4%', '--p5', '%P5%', '--out', '%OUT%'; & '%JAVA_EXE%' @javaArgs 2>&1 | Tee-Object -FilePath '%LOG%' -Append; exit $LASTEXITCODE }"

if errorlevel 1 (
  echo [ERROR] Application failed. Last 60 lines:
  call :tailLog 60
  goto :fail
)

echo.
echo All done successfully!
echo Output file: "%APP_DIR%\%OUT%"
echo Log file   : "%LOG%"
echo.
pause
exit /b 0

:fail
echo.
echo FAILED. Log: "%LOG%"
echo --- Last 80 lines of log ---
call :tailLog 80
echo.
pause
exit /b 1

:tailLog
set "N=%~1"
powershell -NoProfile -Command "if (Test-Path '%LOG%') { Get-Content -Path '%LOG%' -Tail %N% }" 2>nul
if errorlevel 1 (
  if exist "%LOG%" type "%LOG%"
)
exit /b 0