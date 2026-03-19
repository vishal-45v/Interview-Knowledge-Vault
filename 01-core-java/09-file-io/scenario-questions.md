# File I/O — Scenario Questions

> 15 real-world scenarios on file processing, streaming, configuration management, and I/O patterns in Java backend applications.

---

## Scenario 1: Processing Large CSV Import

**Situation:** A client uploads a 500MB CSV file with 5 million orders. Process it without running out of memory.

```java
@Service
public class OrderImportService {

    private static final int BATCH_SIZE = 500;

    @Async
    public CompletableFuture<ImportResult> importOrders(Path csvFile) {
        long imported = 0, failed = 0;
        List<OrderRecord> batch = new ArrayList<>(BATCH_SIZE);

        try (Stream<String> lines = Files.lines(csvFile, StandardCharsets.UTF_8)) {
            // Skip header, process in batches
            Iterator<String> iter = lines.skip(1).iterator();

            while (iter.hasNext()) {
                String line = iter.next();
                try {
                    OrderRecord record = parseLine(line);
                    batch.add(record);

                    if (batch.size() >= BATCH_SIZE) {
                        bulkInsert(batch);
                        imported += batch.size();
                        batch.clear();
                        log.info("Imported {} orders so far", imported);
                    }
                } catch (ParseException e) {
                    failed++;
                    log.warn("Skipping malformed line: {}", line);
                }
            }

            // Process remaining
            if (!batch.isEmpty()) {
                bulkInsert(batch);
                imported += batch.size();
            }

        } catch (IOException e) {
            throw new ImportException("Failed to read CSV: " + csvFile, e);
        }

        return CompletableFuture.completedFuture(new ImportResult(imported, failed));
    }

    @Transactional
    public void bulkInsert(List<OrderRecord> records) {
        orderRepository.saveAll(records.stream()
            .map(this::toOrder)
            .collect(Collectors.toList()));
    }

    private OrderRecord parseLine(String line) throws ParseException {
        String[] fields = line.split(",", -1);  // -1 to keep trailing empty fields
        if (fields.length < 5) throw new ParseException("Expected 5+ fields", 0);
        return new OrderRecord(
            fields[0].trim(),
            fields[1].trim(),
            new BigDecimal(fields[2].trim()),
            LocalDate.parse(fields[3].trim()),
            fields[4].trim()
        );
    }
}
```

---

## Scenario 2: Configuration File Hot-Reload

**Situation:** Your application reads configuration from a YAML file. The ops team needs to update settings without restarting the service.

```java
@Component
public class HotReloadableConfig implements Closeable {

    private final Path configPath;
    private volatile ApplicationConfig config;  // Volatile for visibility across threads
    private final WatchService watcher;
    private final ScheduledExecutorService scheduler;

    public HotReloadableConfig(@Value("${app.config.path}") String configPath) throws IOException {
        this.configPath = Path.of(configPath);
        this.config = loadConfig();

        this.watcher = FileSystems.getDefault().newWatchService();
        this.configPath.getParent().register(watcher, StandardWatchEventKinds.ENTRY_MODIFY);

        this.scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "config-watcher");
            t.setDaemon(true);
            return t;
        });

        // Poll for changes
        scheduler.scheduleWithFixedDelay(this::checkForChanges, 1, 1, TimeUnit.SECONDS);
        log.info("Config hot-reload enabled for: {}", configPath);
    }

    private void checkForChanges() {
        WatchKey key = watcher.poll();
        if (key == null) return;

        boolean configChanged = key.pollEvents().stream()
            .filter(e -> e.kind() != StandardWatchEventKinds.OVERFLOW)
            .anyMatch(e -> configPath.getFileName()
                .equals(((WatchEvent<Path>) e).context()));

        key.reset();

        if (configChanged) {
            try {
                ApplicationConfig newConfig = loadConfig();
                this.config = newConfig;  // Atomic reference update (volatile)
                log.info("Config reloaded: {}", configPath);
            } catch (Exception e) {
                log.error("Failed to reload config — keeping previous version", e);
            }
        }
    }

    private ApplicationConfig loadConfig() throws IOException {
        String yaml = Files.readString(configPath, StandardCharsets.UTF_8);
        return yamlMapper.readValue(yaml, ApplicationConfig.class);
    }

    public ApplicationConfig getConfig() { return config; }

    @Override
    public void close() throws IOException {
        scheduler.shutdownNow();
        watcher.close();
    }
}
```

---

## Scenario 3: Atomic File Write for Critical Data

**Situation:** Your billing service writes daily reports to files. If the write fails midway, you must not have a partially-written file (another process reads it).

```java
@Service
public class AtomicReportWriter {

    public void writeReport(Path targetPath, ReportData data) throws IOException {
        // Write to a temp file in the same directory (same filesystem = atomic move)
        Path tempFile = Files.createTempFile(
            targetPath.getParent(),
            ".report-tmp-",
            ".csv"
        );

        try {
            // Write complete content to temp file
            try (BufferedWriter writer = Files.newBufferedWriter(
                    tempFile, StandardCharsets.UTF_8,
                    StandardOpenOption.WRITE)) {

                writer.write("orderId,total,status,date\n");
                for (OrderSummary order : data.getOrders()) {
                    writer.write(String.format("%s,%.2f,%s,%s\n",
                        order.getId(), order.getTotal(),
                        order.getStatus(), order.getDate()));
                }
                writer.flush();
            }

            // Atomically replace the target file
            Files.move(tempFile, targetPath,
                StandardCopyOption.ATOMIC_MOVE,
                StandardCopyOption.REPLACE_EXISTING);

            log.info("Report written atomically to {}", targetPath);

        } catch (Exception e) {
            Files.deleteIfExists(tempFile);  // Clean up temp on failure
            throw new ReportWriteException("Failed to write report", e);
        }
    }
}
```

---

## Scenario 4: File Upload to Object Storage via Streaming

**Situation:** Handle multipart file uploads that may be up to 5GB. Stream directly to S3 without loading into memory.

```java
@RestController
public class FileUploadController {

    @Autowired private S3StreamingService s3Service;

    @PostMapping("/upload")
    public ResponseEntity<UploadResult> uploadFile(
            @RequestParam("file") MultipartFile multipartFile,
            @RequestParam("category") String category) throws IOException {

        // DO NOT use: multipartFile.getBytes() — loads entire file into heap!
        // DO use: streaming via InputStream
        String key = category + "/" + UUID.randomUUID() + "-" + multipartFile.getOriginalFilename();

        try (InputStream inputStream = multipartFile.getInputStream()) {
            UploadResult result = s3Service.uploadStream(
                "my-bucket",
                key,
                inputStream,
                multipartFile.getSize(),
                multipartFile.getContentType()
            );
            return ResponseEntity.ok(result);
        }
    }
}

@Service
public class S3StreamingService {

    @Autowired private AmazonS3 s3Client;

    public UploadResult uploadStream(String bucket, String key,
                                      InputStream inputStream,
                                      long contentLength,
                                      String contentType) {

        // Use multipart upload for files > 5MB
        if (contentLength > 5 * 1024 * 1024) {
            return multipartUpload(bucket, key, inputStream, contentLength, contentType);
        }

        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentLength(contentLength);
        metadata.setContentType(contentType);

        PutObjectResult result = s3Client.putObject(
            new PutObjectRequest(bucket, key, inputStream, metadata));

        return new UploadResult(key, result.getETag(), contentLength);
    }

    private UploadResult multipartUpload(String bucket, String key,
                                          InputStream stream,
                                          long totalSize,
                                          String contentType) {
        // 5MB part size
        int partSize = 5 * 1024 * 1024;
        InitiateMultipartUploadResult initResponse =
            s3Client.initiateMultipartUpload(new InitiateMultipartUploadRequest(bucket, key));
        String uploadId = initResponse.getUploadId();

        List<PartETag> partETags = new ArrayList<>();
        int partNumber = 1;
        long remaining = totalSize;

        try {
            byte[] buffer = new byte[partSize];
            while (remaining > 0) {
                int bytesRead = readExactly(stream, buffer, (int) Math.min(partSize, remaining));
                if (bytesRead <= 0) break;

                UploadPartResult result = s3Client.uploadPart(
                    new UploadPartRequest()
                        .withBucketName(bucket).withKey(key).withUploadId(uploadId)
                        .withPartNumber(partNumber++)
                        .withInputStream(new ByteArrayInputStream(buffer, 0, bytesRead))
                        .withPartSize(bytesRead));

                partETags.add(result.getPartETag());
                remaining -= bytesRead;
            }

            s3Client.completeMultipartUpload(
                new CompleteMultipartUploadRequest(bucket, key, uploadId, partETags));

        } catch (Exception e) {
            s3Client.abortMultipartUpload(
                new AbortMultipartUploadRequest(bucket, key, uploadId));
            throw new StorageException("Multipart upload failed", e);
        }

        return new UploadResult(key, null, totalSize);
    }
}
```

---

## Scenario 5: Log File Rotation and Archiving

**Situation:** Implement log file rotation: rename the current log, create a new one, compress the old one.

```java
@Component
@Slf4j
public class LogRotationService {

    @Value("${log.dir:/var/log/app}")
    private String logDirectory;

    @Scheduled(cron = "0 0 0 * * *")  // Midnight daily
    public void rotateLogs() {
        Path logDir = Path.of(logDirectory);
        Path currentLog = logDir.resolve("app.log");

        if (!Files.exists(currentLog)) {
            log.debug("No log file to rotate");
            return;
        }

        String timestamp = LocalDate.now().minusDays(1)
            .format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
        Path archivedLog = logDir.resolve("app." + timestamp + ".log");

        try {
            // 1. Rename current log (atomic on same filesystem)
            Files.move(currentLog, archivedLog, StandardCopyOption.ATOMIC_MOVE);

            // 2. Application continues writing — new file created on next write
            // (If using file descriptor, app holds reference to old file; rotation still works)

            // 3. Compress the archived log asynchronously
            CompletableFuture.runAsync(() -> compressLog(archivedLog));

            // 4. Delete logs older than 30 days
            cleanupOldLogs(logDir);

        } catch (IOException e) {
            log.error("Log rotation failed", e);
        }
    }

    private void compressLog(Path logFile) {
        Path compressed = Path.of(logFile + ".gz");
        try (InputStream in = new BufferedInputStream(Files.newInputStream(logFile));
             OutputStream out = new GZIPOutputStream(
                 new BufferedOutputStream(Files.newOutputStream(compressed)))) {

            byte[] buffer = new byte[8192];
            int len;
            while ((len = in.read(buffer)) > 0) {
                out.write(buffer, 0, len);
            }

            // Only delete original if compression succeeded
            Files.delete(logFile);
            log.info("Compressed {} → {}", logFile.getFileName(), compressed.getFileName());

        } catch (IOException e) {
            log.error("Failed to compress log: {}", logFile, e);
            Files.deleteIfExists(compressed);  // Remove incomplete compressed file
        }
    }

    private void cleanupOldLogs(Path logDir) throws IOException {
        Instant cutoff = Instant.now().minus(30, ChronoUnit.DAYS);

        try (Stream<Path> files = Files.list(logDir)) {
            files.filter(f -> f.toString().endsWith(".log.gz"))
                 .filter(f -> {
                     try {
                         return Files.getLastModifiedTime(f).toInstant().isBefore(cutoff);
                     } catch (IOException e) { return false; }
                 })
                 .forEach(f -> {
                     try {
                         Files.delete(f);
                         log.info("Deleted old log: {}", f.getFileName());
                     } catch (IOException e) {
                         log.warn("Could not delete old log: {}", f, e);
                     }
                 });
        }
    }
}
```

---

## Scenario 6: Read/Write Properties File Safely

**Situation:** Your application reads API keys from a properties file that may be updated by ops while the service is running.

```java
@Component
public class PropertiesService {

    private final Path propertiesFile;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    public String getProperty(String key) {
        lock.readLock().lock();
        try {
            Properties props = loadProperties();
            return props.getProperty(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void setProperty(String key, String value) {
        lock.writeLock().lock();
        try {
            Properties props = loadProperties();
            props.setProperty(key, value);
            saveProperties(props);
        } finally {
            lock.writeLock().unlock();
        }
    }

    private Properties loadProperties() {
        Properties props = new Properties();
        if (Files.exists(propertiesFile)) {
            try (InputStream is = new BufferedInputStream(Files.newInputStream(propertiesFile))) {
                props.load(is);
            } catch (IOException e) {
                throw new RuntimeException("Failed to load properties", e);
            }
        }
        return props;
    }

    private void saveProperties(Properties props) {
        // Atomic write via temp file
        Path temp = null;
        try {
            temp = Files.createTempFile(propertiesFile.getParent(), ".props-tmp", null);
            try (OutputStream os = new BufferedOutputStream(Files.newOutputStream(temp))) {
                props.store(os, "Application configuration — last updated: " + Instant.now());
            }
            Files.move(temp, propertiesFile, StandardCopyOption.ATOMIC_MOVE,
                                              StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            if (temp != null) Files.deleteIfExists(temp);
            throw new RuntimeException("Failed to save properties", e);
        }
    }
}
```

---

## Scenario 7: Checksum Verification for File Integrity

**Situation:** After downloading files from an external source, verify their integrity using SHA-256 checksums.

```java
@Service
public class FileIntegrityService {

    public String computeChecksum(Path file) throws IOException {
        MessageDigest digest;
        try {
            digest = MessageDigest.getInstance("SHA-256");
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("SHA-256 not available", e);
        }

        // Stream file in chunks — works for any file size
        try (InputStream is = new BufferedInputStream(Files.newInputStream(file))) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                digest.update(buffer, 0, bytesRead);
            }
        }

        return bytesToHex(digest.digest());
    }

    public void verifyChecksum(Path file, String expectedChecksum) throws IOException {
        String actual = computeChecksum(file);
        if (!MessageDigest.isEqual(expectedChecksum.getBytes(), actual.getBytes())) {
            // MessageDigest.isEqual is timing-safe
            throw new ChecksumMismatchException(
                "Checksum mismatch for " + file.getFileName() +
                ": expected " + expectedChecksum + ", got " + actual);
        }
    }

    // Verify against a checksum file (one file per line: "hash  filename")
    public Map<Path, Boolean> verifyChecksumFile(Path directory, Path checksumFile)
            throws IOException {
        Map<Path, Boolean> results = new LinkedHashMap<>();
        List<String> lines = Files.readAllLines(checksumFile, StandardCharsets.UTF_8);

        for (String line : lines) {
            if (line.isBlank() || line.startsWith("#")) continue;
            String[] parts = line.split("\\s+", 2);
            if (parts.length != 2) continue;

            String expectedHash = parts[0];
            Path file = directory.resolve(parts[1].trim());

            try {
                verifyChecksum(file, expectedHash);
                results.put(file, true);
            } catch (ChecksumMismatchException | IOException e) {
                log.error("Verification failed for {}: {}", file, e.getMessage());
                results.put(file, false);
            }
        }

        return results;
    }

    private static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder(bytes.length * 2);
        for (byte b : bytes) sb.append(String.format("%02x", b));
        return sb.toString();
    }
}
```

---

## Scenario 8: File-Based Event Sourcing / Audit Log

**Situation:** Append all order events to an append-only file for audit. Multiple threads generate events.

```java
@Component
public class AuditLogWriter implements Closeable {

    private final FileChannel channel;
    private final ObjectMapper objectMapper;

    public AuditLogWriter(@Value("${audit.log.path}") String logPath) throws IOException {
        Path path = Path.of(logPath);
        this.channel = FileChannel.open(path,
            StandardOpenOption.WRITE,
            StandardOpenOption.APPEND,
            StandardOpenOption.CREATE);
        this.objectMapper = new ObjectMapper();
        this.objectMapper.registerModule(new JavaTimeModule());
    }

    // Thread-safe append — FileChannel.write() is atomic for small writes
    public synchronized void append(AuditEvent event) throws IOException {
        String json = objectMapper.writeValueAsString(event) + "\n";
        ByteBuffer buffer = ByteBuffer.wrap(json.getBytes(StandardCharsets.UTF_8));

        // Write atomically (FileChannel supports concurrent writes with synchronization)
        while (buffer.hasRemaining()) {
            channel.write(buffer);
        }
    }

    @Override
    public void close() throws IOException {
        channel.close();
    }
}

// Event model
public record AuditEvent(
    String eventType,
    String entityType,
    String entityId,
    String userId,
    Instant timestamp,
    Map<String, Object> changes
) {}

// Reader — replay events for audit
@Service
public class AuditLogReader {

    public Stream<AuditEvent> readAll(Path logFile) throws IOException {
        ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());
        return Files.lines(logFile, StandardCharsets.UTF_8)
            .filter(line -> !line.isBlank())
            .map(line -> {
                try {
                    return mapper.readValue(line, AuditEvent.class);
                } catch (JsonProcessingException e) {
                    log.warn("Malformed audit entry: {}", line);
                    return null;
                }
            })
            .filter(Objects::nonNull);
    }

    public List<AuditEvent> getEventsForEntity(Path logFile, String entityId)
            throws IOException {
        try (Stream<AuditEvent> events = readAll(logFile)) {
            return events.filter(e -> entityId.equals(e.entityId()))
                         .sorted(Comparator.comparing(AuditEvent::timestamp))
                         .collect(Collectors.toList());
        }
    }
}
```

---

## Scenario 9: ZIP File Processing

**Situation:** Process a ZIP archive containing multiple CSV files. Extract and process them without extracting to disk.

```java
@Service
public class ZipProcessor {

    public ImportSummary processZip(InputStream zipStream) throws IOException {
        ImportSummary summary = new ImportSummary();

        try (ZipInputStream zis = new ZipInputStream(
                new BufferedInputStream(zipStream), StandardCharsets.UTF_8)) {

            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                try {
                    if (entry.isDirectory()) continue;
                    if (!entry.getName().endsWith(".csv")) {
                        log.warn("Skipping non-CSV entry: {}", entry.getName());
                        continue;
                    }

                    log.info("Processing entry: {} ({} bytes)",
                             entry.getName(), entry.getSize());

                    // Process entry without closing the ZipInputStream
                    processEntry(entry.getName(),
                                 new NonClosingInputStream(zis),  // Don't close ZipInputStream
                                 summary);

                } finally {
                    zis.closeEntry();  // Move to next entry
                }
            }
        }

        return summary;
    }

    // Create a ZIP with multiple files
    public void createZipReport(Path zipPath, Map<String, byte[]> files) throws IOException {
        try (ZipOutputStream zos = new ZipOutputStream(
                new BufferedOutputStream(Files.newOutputStream(zipPath)),
                StandardCharsets.UTF_8)) {

            zos.setLevel(Deflater.BEST_COMPRESSION);

            for (Map.Entry<String, byte[]> file : files.entrySet()) {
                ZipEntry entry = new ZipEntry(file.getKey());
                entry.setLastModifiedTime(FileTime.from(Instant.now()));
                zos.putNextEntry(entry);
                zos.write(file.getValue());
                zos.closeEntry();
            }
        }
    }

    // Wrapper to prevent closing ZipInputStream when closing inner stream
    private static class NonClosingInputStream extends FilterInputStream {
        NonClosingInputStream(InputStream in) { super(in); }
        @Override
        public void close() { /* Don't close underlying ZipInputStream */ }
    }
}
```

---

## Scenario 10: File Lock for Distributed Single-Writer

**Situation:** Multiple instances of your batch job might run simultaneously. Use a file lock to ensure only one runs at a time.

```java
@Component
public class BatchJobLock {

    private final Path lockFile;

    public BatchJobLock(@Value("${batch.lock.file:/tmp/batch.lock}") String lockFilePath) {
        this.lockFile = Path.of(lockFilePath);
    }

    public boolean tryRun(Callable<Void> job) throws Exception {
        try (FileChannel channel = FileChannel.open(lockFile,
                StandardOpenOption.WRITE, StandardOpenOption.CREATE);
             FileLock lock = channel.tryLock()) {  // Non-blocking

            if (lock == null) {
                log.info("Another instance is running batch job — skipping");
                return false;
            }

            // Write PID to lock file for debugging
            String pid = String.valueOf(ProcessHandle.current().pid());
            channel.write(ByteBuffer.wrap((pid + "\n").getBytes()));

            log.info("Acquired batch lock (PID: {}), starting job", pid);
            job.call();
            log.info("Batch job completed, releasing lock");
            return true;
        }
        // Lock file released when channel is closed (even on exception)
    }

    // Blocking version — wait up to timeout
    public boolean tryRunWithTimeout(Callable<Void> job, Duration timeout)
            throws Exception, TimeoutException {

        Instant deadline = Instant.now().plus(timeout);

        try (FileChannel channel = FileChannel.open(lockFile,
                StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {

            FileLock lock = null;
            while (lock == null) {
                lock = channel.tryLock();
                if (lock == null) {
                    if (Instant.now().isAfter(deadline)) {
                        throw new TimeoutException("Could not acquire batch lock within " + timeout);
                    }
                    Thread.sleep(5000);  // Wait 5s before retry
                }
            }

            try {
                job.call();
                return true;
            } finally {
                lock.release();
            }
        }
    }
}
```

---

## Scenario 11: Generate and Stream Large PDF/Report

**Situation:** Generate a monthly report from database for 100K rows and stream it to the HTTP response without loading all data into memory.

```java
@GetMapping("/reports/monthly")
public void downloadMonthlyReport(
        @RequestParam int year,
        @RequestParam int month,
        HttpServletResponse response) throws IOException {

    response.setContentType("text/csv");
    response.setHeader("Content-Disposition",
        "attachment; filename=\"report-" + year + "-" + month + ".csv\"");

    // Stream directly to response OutputStream — no temp file, no memory buffer
    try (PrintWriter writer = response.getWriter()) {
        // Write CSV header
        writer.println("OrderId,CustomerId,Total,Status,CreatedAt");

        // Stream orders from database in chunks
        orderRepository.streamByYearAndMonth(year, month)
            .forEach(order -> {
                writer.printf("%s,%d,%.2f,%s,%s%n",
                    order.getId(),
                    order.getCustomerId(),
                    order.getTotal().doubleValue(),
                    order.getStatus(),
                    order.getCreatedAt().toString());
                // Flush periodically to prevent buffering too much
                writer.flush();
            });
    }
}

// Repository with streaming query
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o WHERE YEAR(o.createdAt) = :year AND MONTH(o.createdAt) = :month")
    @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "500"))
    Stream<Order> streamByYearAndMonth(@Param("year") int year, @Param("month") int month);
}
```

---

## Scenario 12: File Deduplication Service

**Situation:** Users upload files but the same file is uploaded multiple times. Deduplicate using content hash.

```java
@Service
public class FileDeduplicationService {

    private final Path storageRoot;
    private final Map<String, Path> hashToPath = new ConcurrentHashMap<>();  // In-memory index

    public StoredFile store(InputStream inputStream, String originalFilename) throws IOException {
        // Write to temp file first to compute hash
        Path tempFile = Files.createTempFile(storageRoot, "upload-", ".tmp");

        try {
            // Compute hash while writing to temp file
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            try (OutputStream out = new BufferedOutputStream(Files.newOutputStream(tempFile))) {
                byte[] buffer = new byte[8192];
                int len;
                while ((len = inputStream.read(buffer)) != -1) {
                    out.write(buffer, 0, len);
                    digest.update(buffer, 0, len);
                }
            }

            String hash = bytesToHex(digest.digest());
            long size = Files.size(tempFile);

            // Check if we already have this file
            Path existingPath = hashToPath.get(hash);
            if (existingPath != null && Files.exists(existingPath)) {
                Files.delete(tempFile);  // Discard duplicate
                log.info("Deduplicated upload {} → {}", originalFilename, hash);
                return new StoredFile(hash, existingPath, size, true);
            }

            // New file — move to permanent location
            String extension = getExtension(originalFilename);
            Path permanentPath = storageRoot.resolve(hash.substring(0, 2))  // Sharding
                                            .resolve(hash + extension);
            Files.createDirectories(permanentPath.getParent());
            Files.move(tempFile, permanentPath, StandardCopyOption.ATOMIC_MOVE);

            hashToPath.put(hash, permanentPath);
            log.info("Stored new file {} → {}", originalFilename, permanentPath);
            return new StoredFile(hash, permanentPath, size, false);

        } catch (Exception e) {
            Files.deleteIfExists(tempFile);
            throw new StorageException("Failed to store file", e);
        }
    }
}
```

---

## Scenario 13: Classpath Resource Loading

**Situation:** Your application needs to read template files, SQL scripts, and configuration bundled in the JAR.

```java
@Service
public class ResourceLoader {

    // Spring's ResourceLoader — handles classpath:, file:, http: URLs
    @Autowired
    private org.springframework.core.io.ResourceLoader resourceLoader;

    public String loadTemplate(String templateName) throws IOException {
        // Works from classpath (inside JAR) or file system
        Resource resource = resourceLoader.getResource(
            "classpath:templates/" + templateName);

        if (!resource.exists()) {
            throw new ResourceNotFoundException("Template not found: " + templateName);
        }

        try (InputStream is = resource.getInputStream()) {
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        }
    }

    // Load multiple resources matching a pattern
    @Autowired
    private ResourcePatternResolver patternResolver;

    public List<String> loadSqlMigrations() throws IOException {
        Resource[] resources = patternResolver.getResources(
            "classpath:db/migration/*.sql");

        return Arrays.stream(resources)
            .sorted(Comparator.comparing(Resource::getFilename))
            .map(r -> {
                try (InputStream is = r.getInputStream()) {
                    return new String(is.readAllBytes(), StandardCharsets.UTF_8);
                } catch (IOException e) {
                    throw new RuntimeException("Failed to load: " + r.getFilename(), e);
                }
            })
            .collect(Collectors.toList());
    }

    // Without Spring — pure Java
    public static String loadClasspathResource(String path) throws IOException {
        InputStream is = ResourceLoader.class.getClassLoader()
            .getResourceAsStream(path);

        if (is == null) {
            throw new FileNotFoundException("Classpath resource not found: " + path);
        }

        try (is) {
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        }
    }
}
```

---

## Scenario 14: File Scanning for Security

**Situation:** Scan uploaded files for dangerous content (zip bombs, embedded executables).

```java
@Component
public class FileSecurityScanner {

    private static final long MAX_FILE_SIZE = 100 * 1024 * 1024;  // 100MB
    private static final long MAX_UNCOMPRESSED_SIZE = 500 * 1024 * 1024;  // 500MB

    // Magic byte signatures for common dangerous types
    private static final Map<byte[], String> MAGIC_BYTES = Map.of(
        new byte[]{0x4D, 0x5A}, "Windows PE Executable",        // MZ
        new byte[]{0x7F, 0x45, 0x4C, 0x46}, "Linux ELF",       // \x7fELF
        new byte[]{(byte)0xCA, (byte)0xFE, (byte)0xBA, (byte)0xBE}, "Java Class"  // CAFEBABE
    );

    public ScanResult scan(Path file, String contentType) throws IOException {
        List<String> issues = new ArrayList<>();

        // 1. Check file size
        long size = Files.size(file);
        if (size > MAX_FILE_SIZE) {
            issues.add("File too large: " + size + " bytes");
        }

        // 2. Check magic bytes (actual type vs declared type)
        String detectedType = detectFileType(file);
        if (!isContentTypeAcceptable(contentType, detectedType)) {
            issues.add("Content type mismatch: declared=" + contentType + " actual=" + detectedType);
        }

        // 3. Check for zip bomb if it's an archive
        if (isArchive(detectedType)) {
            long uncompressed = measureUncompressedSize(file);
            if (uncompressed > MAX_UNCOMPRESSED_SIZE) {
                issues.add("Potential zip bomb: uncompressed size " + uncompressed + " bytes");
            }
            double compressionRatio = (double) uncompressed / size;
            if (compressionRatio > 1000) {
                issues.add("Suspicious compression ratio: " + compressionRatio);
            }
        }

        return new ScanResult(issues.isEmpty(), issues);
    }

    private String detectFileType(Path file) throws IOException {
        try (InputStream is = new BufferedInputStream(Files.newInputStream(file))) {
            byte[] header = is.readNBytes(16);
            for (Map.Entry<byte[], String> entry : MAGIC_BYTES.entrySet()) {
                if (startsWith(header, entry.getKey())) return entry.getValue();
            }
        }
        return "unknown";
    }

    private long measureUncompressedSize(Path file) throws IOException {
        long total = 0;
        try (ZipFile zipFile = new ZipFile(file.toFile())) {
            Enumeration<? extends ZipEntry> entries = zipFile.entries();
            while (entries.hasMoreElements()) {
                total += entries.nextElement().getSize();
                if (total > MAX_UNCOMPRESSED_SIZE) return total;  // Short-circuit
            }
        }
        return total;
    }
}
```

---

## Scenario 15: File-Based Job Queue

**Situation:** Create a lightweight job queue using the file system — jobs are written as files, workers claim them.

```java
@Component
public class FileBasedJobQueue {

    private final Path pendingDir;
    private final Path processingDir;
    private final Path completedDir;
    private final Path failedDir;

    public FileBasedJobQueue(@Value("${jobs.dir:/data/jobs}") String baseDir) throws IOException {
        Path base = Path.of(baseDir);
        this.pendingDir    = Files.createDirectories(base.resolve("pending"));
        this.processingDir = Files.createDirectories(base.resolve("processing"));
        this.completedDir  = Files.createDirectories(base.resolve("completed"));
        this.failedDir     = Files.createDirectories(base.resolve("failed"));
    }

    // Submit a job
    public Path submit(JobPayload payload) throws IOException {
        String jobId = UUID.randomUUID().toString();
        Path jobFile = pendingDir.resolve(jobId + ".json");
        Files.writeString(jobFile, objectMapper.writeValueAsString(payload),
            StandardCharsets.UTF_8, StandardOpenOption.CREATE_NEW);
        return jobFile;
    }

    // Claim next available job (atomic via ATOMIC_MOVE)
    public Optional<JobFile> claimNext() throws IOException {
        try (Stream<Path> pending = Files.list(pendingDir)) {
            Optional<Path> nextJob = pending
                .filter(p -> p.toString().endsWith(".json"))
                .sorted()  // FIFO by filename (UUID is random; use timestamp prefix for ordering)
                .findFirst();

            if (nextJob.isEmpty()) return Optional.empty();

            Path jobFile = nextJob.get();
            Path processingFile = processingDir.resolve(jobFile.getFileName());

            try {
                // Atomic claim — fails if another worker already claimed it
                Files.move(jobFile, processingFile, StandardCopyOption.ATOMIC_MOVE);
                JobPayload payload = objectMapper.readValue(
                    Files.readString(processingFile), JobPayload.class);
                return Optional.of(new JobFile(processingFile, payload));
            } catch (AtomicMoveNotSupportedException | java.nio.file.NoSuchFileException e) {
                return Optional.empty();  // Another worker took it
            }
        }
    }

    public void complete(Path processingFile) throws IOException {
        Files.move(processingFile, completedDir.resolve(processingFile.getFileName()),
            StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
    }

    public void fail(Path processingFile, String errorMessage) throws IOException {
        Path failedFile = failedDir.resolve(processingFile.getFileName());
        Files.move(processingFile, failedFile, StandardCopyOption.ATOMIC_MOVE);
        Files.writeString(failedFile.resolveSibling(
            processingFile.getFileName() + ".error"), errorMessage);
    }

    public record JobFile(Path path, JobPayload payload) {}
}
```
