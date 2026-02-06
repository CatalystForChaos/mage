# Building XMage on Windows 11

This guide walks you through compiling XMage from source on Windows 11.

## Prerequisites

You need the following tools installed:

| Tool | Version | Purpose |
|------|---------|---------|
| Java JDK | 17 or 21 (recommended) | Compile and run Java code |
| Apache Maven | 3.8+ | Build system |
| Git | Latest | Clone the repository |

## Step 1: Install Java JDK

### Option A: Eclipse Temurin (Recommended)

1. Download from https://adoptium.net/
2. Select **Windows x64** and **JDK 21** (or JDK 17)
3. Download the `.msi` installer
4. Run the installer with these options checked:
   - **Set JAVA_HOME variable**
   - **Add to PATH**
5. Restart your terminal

### Option B: Oracle JDK

1. Download from https://www.oracle.com/java/technologies/downloads/
2. Select **Windows x64 Installer**
3. Run the installer
4. Manually set environment variables (see below)

### Option C: Microsoft Build of OpenJDK

```powershell
winget install Microsoft.OpenJDK.21
```

### Verify Java Installation

Open a new PowerShell or Command Prompt window:

```powershell
java -version
javac -version
```

Expected output (version numbers may vary):
```
openjdk version "21.0.2" 2024-01-16 LTS
javac 21.0.2
```

## Step 2: Install Apache Maven

### Option A: Manual Installation

1. Download from https://maven.apache.org/download.cgi
2. Download the **Binary zip archive** (e.g., `apache-maven-3.9.6-bin.zip`)
3. Extract to `C:\Program Files\Apache\maven`
4. Add to PATH (see Environment Variables section below)

### Option B: Using Chocolatey

```powershell
choco install maven
```

### Option C: Using Scoop

```powershell
scoop install maven
```

### Verify Maven Installation

```powershell
mvn -version
```

Expected output:
```
Apache Maven 3.9.6
Maven home: C:\Program Files\Apache\maven
Java version: 21.0.2, vendor: Eclipse Adoptium
```

## Step 3: Install Git

### Option A: Git for Windows

1. Download from https://git-scm.com/download/win
2. Run the installer with default options
3. Restart your terminal

### Option B: Using winget

```powershell
winget install Git.Git
```

### Verify Git Installation

```powershell
git --version
```

## Step 4: Set Environment Variables (If Needed)

If Java or Maven aren't recognized, set the environment variables manually:

1. Press `Win + X` → **System**
2. Click **Advanced system settings**
3. Click **Environment Variables**
4. Under **System variables**, add or edit:

| Variable | Value |
|----------|-------|
| `JAVA_HOME` | `C:\Program Files\Eclipse Adoptium\jdk-21.0.2.13-hotspot` (adjust to your path) |
| `MAVEN_HOME` | `C:\Program Files\Apache\maven` |

5. Edit the **Path** variable and add:
   - `%JAVA_HOME%\bin`
   - `%MAVEN_HOME%\bin`

6. Click **OK** to save
7. **Restart your terminal** for changes to take effect

## Step 5: Clone the Repository

Open PowerShell or Git Bash:

```powershell
# Clone the main repository
git clone https://github.com/magefree/mage.git
cd mage

# Or clone a specific fork/branch
git clone https://github.com/CatalystForChaos/mage.git
cd mage
git checkout claude/add-claude-documentation-pekli
```

## Step 6: Build XMage

### Full Build (Recommended for First Time)

```powershell
mvn clean install package -DskipTests
```

This will:
- Download all dependencies (first run takes several minutes)
- Compile all modules
- Package the client and server ZIPs

### Quick Rebuild (After Code Changes)

```powershell
mvn install package -DskipTests
```

### Build with Tests

```powershell
mvn clean install package
```

### Build Output Locations

After a successful build:

| Component | Location |
|-----------|----------|
| Client ZIP | `Mage.Client\target\mage-client.zip` |
| Server ZIP | `Mage.Server\target\mage-server.zip` |

## Step 7: Run XMage

### Extract and Run the Client

```powershell
cd Mage.Client\target
Expand-Archive mage-client.zip -DestinationPath mage-client
cd mage-client
.\startClient.bat
```

### Extract and Run the Server

```powershell
cd Mage.Server\target
Expand-Archive mage-server.zip -DestinationPath mage-server
cd mage-server
.\startServer.bat
```

## Build Options

### Speed Up Builds

Skip tests for faster builds:
```powershell
mvn install package -DskipTests
```

Build only the client module:
```powershell
mvn install package -DskipTests -pl Mage.Client -am
```

### Increase Memory for Large Builds

If you get OutOfMemoryError:

```powershell
$env:MAVEN_OPTS = "-Xmx4g"
mvn clean install package -DskipTests
```

Or set permanently in environment variables:
- Variable: `MAVEN_OPTS`
- Value: `-Xmx4g`

### Parallel Builds

Use multiple CPU cores:
```powershell
mvn install package -DskipTests -T 1C
```

(`1C` = 1 thread per CPU core)

## Troubleshooting

### "java is not recognized"

Java is not in your PATH. Either:
- Reinstall Java with "Add to PATH" checked
- Manually add `%JAVA_HOME%\bin` to your PATH

### "mvn is not recognized"

Maven is not in your PATH. Either:
- Reinstall via Chocolatey/Scoop
- Manually add `%MAVEN_HOME%\bin` to your PATH

### "JAVA_HOME is not set"

Set the `JAVA_HOME` environment variable to your JDK installation directory (not the `bin` folder).

### Build Fails with OutOfMemoryError

Increase Maven's heap size:
```powershell
$env:MAVEN_OPTS = "-Xmx4g"
mvn clean install package -DskipTests
```

### Build Fails Downloading Dependencies

Check your internet connection. If behind a proxy:

1. Create/edit `%USERPROFILE%\.m2\settings.xml`:
```xml
<settings>
  <proxies>
    <proxy>
      <active>true</active>
      <protocol>http</protocol>
      <host>your-proxy-host</host>
      <port>8080</port>
    </proxy>
  </proxies>
</settings>
```

### Tests Fail

Some tests may fail due to timing or environment issues. Use `-DskipTests` for building:
```powershell
mvn install package -DskipTests
```

### Antivirus Blocking Compilation

Some antivirus software (Windows Defender, Norton, etc.) may slow down or block Maven:
- Add exclusions for:
  - Your repository folder (e.g., `C:\Users\YourName\mage`)
  - Maven cache folder (`%USERPROFILE%\.m2`)
  - Java installation folder

### "Corrupted ZIP" or Incomplete Build

Clean and rebuild:
```powershell
mvn clean
mvn install package -DskipTests
```

### Port Already in Use (Server)

If the server won't start:
1. Check for other XMage instances: `netstat -ano | findstr :17171`
2. Kill the process: `taskkill /PID <pid> /F`

## IDE Setup (Optional)

### IntelliJ IDEA

1. **File → Open** → Select the `mage` folder
2. IntelliJ will detect it as a Maven project
3. Wait for indexing to complete
4. **Build → Build Project** or use the Maven tool window

### Eclipse

1. **File → Import → Maven → Existing Maven Projects**
2. Select the `mage` folder
3. Select all modules
4. Click **Finish**

### VS Code

1. Install the **Extension Pack for Java**
2. Open the `mage` folder
3. VS Code will detect Maven and offer to import

## Quick Reference

```powershell
# Full clean build
mvn clean install package -DskipTests

# Quick rebuild
mvn install package -DskipTests

# Run tests
mvn test

# Run specific test
mvn test -pl Mage.Tests -Dtest=ExtraTurnsTest

# Build only client
mvn install package -DskipTests -pl Mage.Client -am

# Check for dependency updates
mvn versions:display-dependency-updates
```

## Getting Help

- **GitHub Issues**: https://github.com/magefree/mage/issues
- **XMage Discord**: https://discord.gg/Pqf42yn
- **Build Documentation**: See `CLAUDE.md` in repository root
