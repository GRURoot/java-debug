# System Integration Guide - java-debug Bugfixes

## Overview

This document describes how to integrate the root cause bug fixes into your build system and CI/CD pipeline.

---

## 1. Build System Integration

### Maven (pom.xml)

```xml
<!-- Add system property configuration -->
<properties>
    <!-- VM connection timeout in milliseconds (default: 30 seconds) -->
    <java.debug.vm.connect.timeout>30000</java.debug.vm.connect.timeout>
</properties>

<!-- Pass to test execution -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.0.0</version>
            <configuration>
                <systemPropertyVariables>
                    <java.debug.vm.connect.timeout>${java.debug.vm.connect.timeout}</java.debug.vm.connect.timeout>
                </systemPropertyVariables>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Gradle (build.gradle)

```gradle
test {
    systemProperties = [
        'java.debug.vm.connect.timeout': '30000'
    ]
}
```

---

## 2. CI/CD Pipeline Integration

### GitHub Actions (.github/workflows/test.yml)

```yaml
name: Test with Root Cause Fixes

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        java-version: [11, 17, 21]
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Java
        uses: actions/setup-java@v3
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
      
      # Test with default timeout
      - name: Test (Default 30s timeout)
        run: mvn test
      
      # Test with extended timeout for slow runners
      - name: Test (Extended 60s timeout)
        if: matrix.os == 'windows-latest'
        run: mvn test -Djava.debug.vm.connect.timeout=60000
      
      # Test error handling specifically
      - name: Test Protocol Error Handling
        run: mvn test -Dtest=AbstractProtocolServerTest
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    environment {
        // Set VM connection timeout
        JAVA_DEBUG_VM_CONNECT_TIMEOUT = '30000'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    mvn test \
                        -Djava.debug.vm.connect.timeout=${JAVA_DEBUG_VM_CONNECT_TIMEOUT}
                '''
            }
        }
        
        stage('Integration Test') {
            steps {
                sh '''
                    # Test with extended timeout
                    mvn verify \
                        -Djava.debug.vm.connect.timeout=60000
                '''
            }
        }
    }
    
    post {
        always {
            junit 'target/surefire-reports/**/*.xml'
        }
    }
}
```

---

## 3. Runtime Configuration

### Application Startup

#### Java Command Line
```bash
java -Djava.debug.vm.connect.timeout=30000 -jar myapp.jar
```

#### Environment Variable (Gradle wrapper)
```bash
export JAVA_OPTS="-Djava.debug.vm.connect.timeout=30000"
./gradlew test
```

#### Docker Container
```dockerfile
FROM openjdk:17-jdk

ENV JAVA_DEBUG_VM_CONNECT_TIMEOUT=30000
ENV JAVA_OPTS="-Djava.debug.vm.connect.timeout=${JAVA_DEBUG_VM_CONNECT_TIMEOUT}"

WORKDIR /app
COPY . .
RUN mvn clean package
CMD ["java", "-jar", "target/java-debug.jar"]
```

---

## 4. Monitoring & Logging

### Logging Configuration (logback.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>debug-protocol.log</file>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- Protocol error logging -->
    <logger name="com.microsoft.java.debug.core.protocol" level="DEBUG">
        <appender-ref ref="FILE"/>
    </logger>
    
    <!-- VM connector logging -->
    <logger name="com.microsoft.java.debug.plugin.internal" level="INFO">
        <appender-ref ref="FILE"/>
    </logger>
    
    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### Error Codes Reference

| Code | Error | Meaning |
|------|-------|----------|
| 1000 | UNKNOWN_FAILURE | Generic error |
| 1001 | UNRECOGNIZED_REQUEST_FAILURE | Invalid request |
| 1002 | LAUNCH_FAILURE | VM launch failed |
| 1003 | ATTACH_FAILURE | Debugger attach failed |
| 1004 | ARGUMENT_MISSING | Missing required argument |

---

## 5. Performance Tuning

### Recommended Timeout Values

```properties
# Fast networks (local development)
java.debug.vm.connect.timeout=10000

# Standard networks (most deployments)
java.debug.vm.connect.timeout=30000

# Slow networks (CI/CD runners)
java.debug.vm.connect.timeout=60000

# Very slow environments (embedded/ARM)
java.debug.vm.connect.timeout=120000
```

---

## 6. Troubleshooting

### Timeout Errors

```
VM did not connect within 30000 ms. Verify:
1) Java installation
2) Debugger port availability
3) Firewall settings
4) JVM options
```

**Solutions:**

1. **Check Java installation:**
   ```bash
   java -version
   echo $JAVA_HOME
   ```

2. **Verify debugger port:**
   ```bash
   netstat -an | grep 5005  # Default JDWP port
   ```

3. **Check firewall:**
   ```bash
   # Linux
   sudo ufw allow 5005/tcp
   
   # macOS
   sudo pfctl -s nat | grep 5005
   ```

4. **Increase timeout:**
   ```bash
   mvn test -Djava.debug.vm.connect.timeout=60000
   ```

### Memory Leaks

Monitor with:
```bash
jps -l  # List Java processes
jconsole  # Visual monitoring
jmap -heap <pid>  # Heap dump
```

---

## 7. Version Compatibility

- **Java:** 8+
- **Maven:** 3.6+
- **Gradle:** 6.0+
- **JDI:** 1.4+

---

## 8. Deployment Checklist

- [ ] All tests passing
- [ ] Performance benchmarks acceptable
- [ ] Logging configured
- [ ] Timeout values tuned for environment
- [ ] Error handling verified
- [ ] Resource cleanup tested
- [ ] Documentation updated
- [ ] Team trained on new error codes

---

## Support

For issues or questions:
- GitHub Issues: https://github.com/GRURoot/java-debug/issues
- Upstream: https://github.com/microsoft/java-debug/issues

