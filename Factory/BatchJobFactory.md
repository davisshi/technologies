## Batch Job Factory

**1. Batch Job Interface**
```java
@Component
public interface BatchJob {
    
    void doWork();
}
```

**2. Concrete Batch Job**
```Java
@Component
public class GIFilesLoadBatchJobImpl implements BatchJob {
    
    @Override
    public void doWork() throws Exception {
        logger.info("Batch Job gi-files-load is started");

        try {
            giFileService
                    .getGIFilesFromLandingServer()
                    .validateNumOfRecords()
                    .createLeadList(giFileService.loadLeads())
                    .archiveAndDeleteGIFiles();

        } catch (Exception e) {
            logger.error(e.toString());

        } finally {
            // send email giFilesLoadMetricReport
            giFilesLoadMetricReportService.sendEmail(giFilesLoadMetricReport);
        }

        logger.info("Batch Job gi-files-load is done");
    }
}
```

**3. Batch Job Factory Implementation**
``java
@Component
public class BatchJobFactoryImpl implements BatchJobFactory {

    @Autowired
    private DoNotCallBatchJob doNotCallBatchJob;

    @Autowired
    private DoNotEmailBatchJob doNotEmailBatchJob;

    @Autowired
    private GIFilesLoadBatchJobImpl giFilesLoadBatchJob

    @Override
    public BatchJob getBatchJob(BatchJobName batchJobName) {

        switch (batchJobName) {
            case DONOTCALL:
                return doNotCallBatchJob;

            case DONOTEMAIL:
                return doNotEmailBatchJob;

            case GIFILELOADBATCH:
                return giFilesLoadBatchJob;

            default:
                throw new RuntimeException("Non supported batch job");
        }
    }
}
``

**4. Load and Run Batch**
```java
@Override
    public void run(String... args) throws Exception {
        logger.info("The batch job for consent management is started");

        String batchName = System.getProperty(BATCH_NAME);

        if (StringUtils.isEmpty(batchName)) {
            throw new IllegalArgumentException("Toggle Batch Job Name is required to start a toggle batch job");
        }

        BatchJob batchJob = batchJobFactory.getBatchJob(BatchJobName.valueOf(batchName.toUpperCase()));
        batchJob.doWork();

        logger.info("The batch job for consent management is done");
    }
```