## Builder

### Using Builder and Template Method to generate customized excel report file

**1. Report Builder interface**
```java
public interface ReportBuilder<T> {

    ReportBuilder setReportData(T reportData);

    ReportBuilder setHeaderColumns(String[] headerColumns);

    ReportBuilder setOutputFilePath(String outputFilePath);

    ReportBuilder setTitle(String title);

    void build() throws Exception;
}
```

**2. Abstract Excel Report Builder to implement the set data, build header, and build excel file** 
```java
public abstract class ExcelReportBuilder<T> implements ReportBuilder<T> {
    private static final CLogger logger = LogFactory.getCLogger(ExcelReportBuilder.class.getName());

    protected T reportData;
    private String[] headerColumns;
    private String outputFilePath;
    private String title;

    @Override
    public ReportBuilder setReportData(T reportData) {
        this.reportData = reportData;
        return this;
    }

    @Override
    public ReportBuilder setHeaderColumns(String[] headerColumns) {
        this.headerColumns = headerColumns;
        return this;
    }

    @Override
    public ReportBuilder setOutputFilePath(String outputFilePath) {
        this.outputFilePath = outputFilePath;
        return this;
    }

    @Override
    public ReportBuilder setTitle(String title) {
        this.title = title;
        return this;
    }

    private int buildHeader(Workbook workbook, Sheet sheet) {
        Font titleFont = workbook.createFont();
        titleFont.setBold(true);
        titleFont.setFontHeightInPoints((short) 16);

        CellStyle headerCellStyle = workbook.createCellStyle();
        headerCellStyle.setFont(titleFont);

        int rowNum = 0;
        Row titleRow = sheet.createRow(rowNum++);

        Cell cell = titleRow.createCell(1);
        cell.setCellValue(title);
        cell.setCellStyle(headerCellStyle);

        Font subTitleFont = workbook.createFont();
        subTitleFont.setBold(true);
        subTitleFont.setFontHeightInPoints((short) 12);

        headerCellStyle = workbook.createCellStyle();
        headerCellStyle.setFont(subTitleFont);

        Row subTitleRow = sheet.createRow(rowNum++);
        Cell subTitleCell = subTitleRow.createCell(1);
        StringBuilder builder = new StringBuilder("Run Time/Date:  ");
        builder.append(getZoneDateTime(PATTERN_MMddYYYYHHMM));
        subTitleCell.setCellValue(builder.toString());
        subTitleCell.setCellStyle(headerCellStyle);
        Row headerRow = sheet.createRow(rowNum++);
        Font headerFont=workbook.createFont();
        headerFont.setBold(true);
        headerFont.setFontHeightInPoints((short) 12);
        headerCellStyle = workbook.createCellStyle();
        headerCellStyle.setFont(headerFont);

        for (int i=0; i < headerColumns.length; i++) {
            Cell headerCell=headerRow.createCell(i);
            headerCell.setCellValue(headerColumns[i]);
            headerCell.setCellStyle(headerCellStyle);
        }

        return rowNum;
    }

    protected abstract int buildBody(Workbook workbook, Sheet sheet, CellStyle dataCellStyle, int rowNum);

    private void writeToFile(String outputFilePath, Workbook workbook) throws IOException {
        FileOutputStream fileOut=new FileOutputStream(outputFilePath);
        workbook.write(fileOut);
        fileOut.close();
        workbook.close();
    }

    @Override
    public void build() throws Exception {
        Workbook workbook = new XSSFWorkbook();
        CreationHelper createHelper  = workbook.getCreationHelper();
        Sheet sheet = workbook.createSheet(title);

        int rowNum = buildHeader(workbook, sheet);

        Font dataFont=workbook.createFont();
        dataFont.setFontHeightInPoints((short) 12);
        CellStyle dataCellStyle = workbook.createCellStyle();
        dataCellStyle.setFont(dataFont);

        buildBody(workbook, sheet, dataCellStyle, rowNum);

        // Write the output to a file
        writeToFile(outputFilePath, workbook);
        logger.debug("Done with excel");
    }

    protected void createCell(Row row, int column, CellStyle cellStyle, String value) {
        Cell dataCell=row.createCell(column);
        dataCell.setCellStyle(cellStyle);
        dataCell.setCellValue(value);
    }

    protected void createCell(Row row, int column, CellStyle cellStyle, double value) {
        Cell dataCell=row.createCell(column);
        dataCell.setCellStyle(cellStyle);
        dataCell.setCellValue(value);
    }

    protected void createCell(Row row, int column, CellStyle cellStyle, Date value) {
        Cell dataCell=row.createCell(column);
        dataCell.setCellStyle(cellStyle);
        dataCell.setCellValue(value);
    }

    protected void createCell(Row row, int column, CellStyle cellStyle, Calendar value) {
        Cell dataCell=row.createCell(column);
        dataCell.setCellStyle(cellStyle);
        dataCell.setCellValue(value);
    }

    protected void createCell(Row row, int column, CellStyle cellStyle, RichTextString value) {
        Cell dataCell=row.createCell(column);
        dataCell.setCellStyle(cellStyle);
        dataCell.setCellValue(value);
    }
}
```

**3. The concrete Excel Report Body builder**
```java
@Component
public class FeaturesDiscrepancyExcelReportBuilder extends ExcelReportBuilder<FeaturesDiscrepancyDTO> {
    private static final CLogger logger = LogFactory.getCLogger(FeaturesDiscrepancyExcelReportBuilder.class.getName());

    private void addDataForFeatures(Row row, CellStyle dataCellStyle, String type, Features features) {
        if (ObjectUtils.isEmpty(features))
            return;

        if (ObjectUtils.isEmpty(row))
            return;

        int column = 0;
        createCell(row, column++, dataCellStyle, type);
        createCell(row, column++, dataCellStyle, features.getFeatureId());
        createCell(row, column++, dataCellStyle, features.getFeatureName());
        createCell(row, column++, dataCellStyle, features.getFeatureStatus());
        createCell(row, column++, dataCellStyle, features.getAllEnabledFlag());
        createCell(row, column++, dataCellStyle, features.getStartDateStringValue());
        createCell(row, column++, dataCellStyle, features.getEndDateStringValue());
        createCell(row, column++, dataCellStyle, features.getFeatureOrg());
        createCell(row, column++, dataCellStyle, features.getFeatureOwner());
        createCell(row, column++, dataCellStyle, features.getApplication());
        createCell(row, column++, dataCellStyle, features.getFeatureDescription());
        createCell(row, column++, dataCellStyle, features.getCreatedDate());
        createCell(row, column++, dataCellStyle, features.getCreatedBy());
        createCell(row, column++, dataCellStyle, features.getUpdatedDate());
        createCell(row, column++, dataCellStyle, features.getUpdatedBy());
        createCell(row, column++, dataCellStyle, features.getDeleted());
    }

    private int buildFeatures(Workbook workbook, Sheet sheet, CellStyle dataCellStyle, int rowNum, String type, List<Features> featuresList) {
        if (ObjectUtils.isEmpty(featuresList))
            return rowNum;

        if (ObjectUtils.isEmpty(workbook) || ObjectUtils.isEmpty(sheet))
            return rowNum;

        for (Features features : featuresList) {
            Row row = sheet.createRow(rowNum++);

            addDataForFeatures(row, dataCellStyle, type, features);
        }

        return rowNum;
    }

    @Override
    public int buildBody(Workbook workbook, Sheet sheet, CellStyle dataCellStyle, int rowNum) {
        logger.debug("Start to build body for report data: " + reportData);

        FeaturesDiscrepancyDTO featuresDiscrepancyDTO = reportData;

        if (ObjectUtils.isEmpty(featuresDiscrepancyDTO))
            return rowNum;

        if (workbook == null || sheet == null)
            return rowNum;

        rowNum = buildFeatures(workbook, sheet, dataCellStyle, rowNum,"Features In Cache Not InDB", featuresDiscrepancyDTO.getFeaturesInCacheNotInDB());

        rowNum = buildFeatures(workbook, sheet, dataCellStyle, rowNum,"Features In DB Not In Cache", featuresDiscrepancyDTO.getFeaturesInDBNotInCache());

        return rowNum;
    }
}
```
**4. Call this Report Builder**
```java
@Autowired
private ReportBuilder<FeaturesDiscrepancyDTO> reportBuilder;

FeaturesDiscrepancyDTO featuresDiscrepancyDTO = featureProvisionService.getFeaturesCacheDiffDB();

String outputFilePath = EXCEL_FILE_PATH + FEATURES_DISCREPANCY_REPORT_OUTPUT;

String[] headerColumns = {"Type", "Feature Id", "Feature Name", "Feature Status", "All Enabled Flag",
        "Start Date", "End Date", "Feature Org", "Feature Owner", "Application", "Feature Description",
        "Created Date", "Created By", "Updated Date", "Updated By", "Deleted"};

reportBuilder
        .setReportData(featuresDiscrepancyDTO)
        .setHeaderColumns(headerColumns)
        .setOutputFilePath(outputFilePath)
        .setTitle(FEATURES_DISCREPANCY_REPORT_TITLE)
        .build();
```