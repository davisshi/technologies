## Use Method Chaining and Service Layering

### 1. Following is a part of an example coding from most developers including senior developers
Put the most business logic in a big long method, which is not good for maintain

The purpose of this doWork logic here is:
1. load a file from a landing server
2. validate the records
3. load leads and into a lead list
4. archive and delete this loaded file

![](../resources/images/CommonCodingForProcessingFileData.png)

### 2. And here is my code for doing the same business logic just for processing a different file data

```Java
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
```