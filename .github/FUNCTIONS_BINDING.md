# Function Bindings & System Hooks

## Overview

This document describes how to bind the root cause bug fixes to system functions and hooks for automatic integration.

---

## 1. JVM Hooks

### Runtime.addShutdownHook()

```java
// In AbstractProtocolServer constructor
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    logger.info("Debug protocol server shutting down...");
    // Flush logs with proper error codes
    System.out.flush();
    System.err.flush();
}));
```

### System Properties Listener

```java
// Monitor timeout changes at runtime
class TimeoutPropertyListener extends PropertyChangeListener {
    private static final String TIMEOUT_PROPERTY = "java.debug.vm.connect.timeout";
    
    @Override
    public void propertyChange(PropertyChangeEvent evt) {
        if (evt.getPropertyName().equals(TIMEOUT_PROPERTY)) {
            int newTimeout = Integer.parseInt((String) evt.getNewValue());
            logger.info("VM timeout changed to: " + newTimeout + "ms");
        }
    }
}
```

---

## 2. Exception Handlers

### Global Exception Handler

```java
// Register global exception handler
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    int errorCode = ErrorCode.UNKNOWN_FAILURE.getId();
    
    if (throwable instanceof LaunchException) {
        errorCode = ErrorCode.LAUNCH_FAILURE.getId();
    } else if (throwable instanceof IOException) {
        errorCode = ErrorCode.VM_TERMINATED.getId();
    }
    
    logger.log(Level.SEVERE, 
        String.format("Uncaught exception [%d] in thread %s: %s",
            errorCode, thread.getName(), throwable.getMessage()),
        throwable);
});
```

### Error Code Mapping Function

```java
public static int mapExceptionToErrorCode(Exception e) {
    if (e instanceof IllegalArgumentException) {
        return ErrorCode.ARGUMENT_MISSING.getId();
    } else if (e instanceof UnsupportedOperationException) {
        return ErrorCode.UNRECOGNIZED_REQUEST_FAILURE.getId();
    } else if (e instanceof IOException) {
        return ErrorCode.VM_TERMINATED.getId();
    } else if (e instanceof LaunchException) {
        return ErrorCode.LAUNCH_FAILURE.getId();
    } else if (e instanceof VMStartException) {
        return ErrorCode.LAUNCH_FAILURE.getId();
    }
    return ErrorCode.UNKNOWN_FAILURE.getId();
}
```

---

## 3. Resource Management

### AutoCloseable Implementation

```java
public class DebugProtocolServer implements AutoCloseable {
    private Reader reader;
    private Writer writer;
    
    @Override
    public void close() {
        // Properly close resources
        try {
            if (writer != null) writer.close();
        } catch (IOException e) {
            logger.log(Level.WARNING, "Error closing writer", e);
        }
        
        try {
            if (reader != null) reader.close();
        } catch (IOException e) {
            logger.log(Level.WARNING, "Error closing reader", e);
        }
    }
}

// Usage with try-with-resources
try (DebugProtocolServer server = new DebugProtocolServer(input, output)) {
    server.run();
} catch (IOException e) {
    logger.log(Level.SEVERE, "Protocol server error", e);
}
```

### Timeout Management

```java
public class TimeoutManager {
    private static final ScheduledExecutorService executor = 
        Executors.newScheduledThreadPool(1);
    
    public static <T> T withTimeout(Callable<T> task, long timeoutMs) 
            throws TimeoutException, Exception {
        Future<T> future = executor.submit(task);
        try {
            return future.get(timeoutMs, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);
            throw e;
        }
    }
}
```

---

## 4. Monitoring Functions

### Performance Metrics

```java
public class DebugMetrics {
    private static final AtomicLong protocolErrors = new AtomicLong(0);
    private static final AtomicLong vmTimeouts = new AtomicLong(0);
    private static final AtomicLong resourceLeaks = new AtomicLong(0);
    
    public static void recordProtocolError() {
        protocolErrors.incrementAndGet();
    }
    
    public static void recordVMTimeout() {
        vmTimeouts.incrementAndGet();
    }
    
    public static void recordResourceLeak() {
        resourceLeaks.incrementAndGet();
    }
    
    public static Map<String, Long> getMetrics() {
        return Map.of(
            "protocol_errors", protocolErrors.get(),
            "vm_timeouts", vmTimeouts.get(),
            "resource_leaks", resourceLeaks.get()
        );
    }
}
```

### Health Check Function

```java
public class HealthCheck {
    public static boolean isHealthy() {
        try {
            // Check protocol server
            if (protocolServer == null || protocolServer.terminateSession) {
                return false;
            }
            
            // Check resource utilization
            Runtime runtime = Runtime.getRuntime();
            long maxMemory = runtime.maxMemory();
            long usedMemory = runtime.totalMemory() - runtime.freeMemory();
            double usage = (double) usedMemory / maxMemory;
            
            if (usage > 0.90) {
                logger.warning("Memory usage high: " + (usage * 100) + "%");
                return false;
            }
            
            return true;
        } catch (Exception e) {
            logger.log(Level.SEVERE, "Health check failed", e);
            return false;
        }
    }
}
```

---

## 5. Integration Test Functions

### Test Fixtures

```java
public class DebugTestFixtures {
    @BeforeAll
    static void setupTimeouts() {
        System.setProperty("java.debug.vm.connect.timeout", "30000");
    }
    
    @AfterEach
    void cleanupResources() {
        // Verify no resource leaks
        assertNoOpenStreams();
    }
    
    private void assertNoOpenStreams() {
        long openStreams = ManagementFactory.getOperatingSystemMXBean()
            .getOpenFileDescriptorCount();
        assertTrue(openStreams < 100, "Too many open file descriptors: " + openStreams);
    }
}
```

### Error Handling Tests

```java
@Test
void testProtocolErrorMapping() {
    IllegalArgumentException ex = new IllegalArgumentException("Missing arg");
    int code = mapExceptionToErrorCode(ex);
    assertEquals(ErrorCode.ARGUMENT_MISSING.getId(), code);
}

@Test
void testStreamClosing() {
    ByteArrayInputStream stream = new ByteArrayInputStream("test".getBytes());
    String result = streamToString(stream);
    
    // Verify stream was closed
    assertThrows(IOException.class, () -> stream.read());
}

@Test
void testTimeoutConfiguration() {
    int timeout = Integer.parseInt(
        System.getProperty("java.debug.vm.connect.timeout", "30000"));
    assertTrue(timeout >= 30000, "Timeout too low: " + timeout);
}
```

---

## 6. Event Listeners

### Custom Event System

```java
public interface DebugEventListener {
    void onProtocolError(int errorCode, String message);
    void onVMTimeout();
    void onResourceLeak();
}

public class DebugEventManager {
    private static final List<DebugEventListener> listeners = new ArrayList<>();
    
    public static void addEventListener(DebugEventListener listener) {
        listeners.add(listener);
    }
    
    public static void fireProtocolError(int errorCode, String message) {
        listeners.forEach(l -> l.onProtocolError(errorCode, message));
    }
    
    public static void fireVMTimeout() {
        listeners.forEach(DebugEventListener::onVMTimeout);
    }
}
```

---

## 7. Configuration Functions

### Configuration Loader

```java
public class DebugConfiguration {
    private Properties config = new Properties();
    
    public DebugConfiguration load(String configFile) throws IOException {
        try (InputStream is = new FileInputStream(configFile)) {
            config.load(is);
        }
        return this;
    }
    
    public int getVMTimeout() {
        return Integer.parseInt(
            config.getProperty("vm.connect.timeout", "30000"));
    }
    
    public String getLogLevel() {
        return config.getProperty("log.level", "INFO");
    }
    
    public boolean enableDetailedErrors() {
        return Boolean.parseBoolean(
            config.getProperty("errors.detailed", "true"));
    }
}
```

---

## 8. Deployment Functions

### Initialization Function

```java
public class DebugSystemInitializer {
    public static void initialize() {
        // Set up error handling
        Thread.setDefaultUncaughtExceptionHandler(
            (thread, throwable) -> handleUncaughtException(thread, throwable));
        
        // Load configuration
        try {
            config = new DebugConfiguration().load("debug.properties");
        } catch (IOException e) {
            logger.log(Level.WARNING, "Could not load debug config", e);
        }
        
        // Set system properties
        System.setProperty("java.debug.vm.connect.timeout", 
            String.valueOf(config.getVMTimeout()));
        
        // Start health check
        startHealthCheckTimer();
        
        logger.info("Debug system initialized");
    }
    
    private static void startHealthCheckTimer() {
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
        executor.scheduleAtFixedRate(() -> {
            if (!HealthCheck.isHealthy()) {
                logger.warning("Health check failed");
            }
        }, 0, 1, TimeUnit.MINUTES);
    }
}
```

---

## 9. Usage Example

```java
public class DebugServerApplication {
    public static void main(String[] args) {
        // Initialize system
        DebugSystemInitializer.initialize();
        
        // Add event listeners
        DebugEventManager.addEventListener(new DebugEventListener() {
            @Override
            public void onProtocolError(int errorCode, String message) {
                logger.severe("Protocol error [" + errorCode + "]: " + message);
                DebugMetrics.recordProtocolError();
            }
            
            @Override
            public void onVMTimeout() {
                logger.warning("VM connection timeout");
                DebugMetrics.recordVMTimeout();
            }
            
            @Override
            public void onResourceLeak() {
                logger.severe("Resource leak detected");
                DebugMetrics.recordResourceLeak();
            }
        });
        
        // Start server
        try (AbstractProtocolServer server = createProtocolServer()) {
            server.run();
        } catch (Exception e) {
            int errorCode = mapExceptionToErrorCode(e);
            logger.log(Level.SEVERE, "Server error [" + errorCode + "]", e);
        }
    }
}
```

---

## Summary

| Function | Purpose | File |
|----------|---------|------|
| `mapExceptionToErrorCode()` | Map exceptions to error codes | AbstractProtocolServer.java |
| `streamToString()` | Safely convert streams with cleanup | AdvancedLaunchingConnector.java |
| `HealthCheck.isHealthy()` | Monitor system health | New utility |
| `DebugMetrics.recordXxx()` | Track debug metrics | New utility |
| `DebugSystemInitializer.initialize()` | Initialize on startup | New utility |

