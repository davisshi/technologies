## File Data Processing with Lambda Functions

**1. File Service Interface**
```java
public interface FileService {

    void load(ConcurrentMap<String, LeadInfoDTO> leadInfoDTOMap) throws Exception;
}
```

**2. Abstract File Base Service for common lambda functions defined and common methods defined**
```java
public abstract class GIFileBaseService {
    private static final CLogger logger = LogFactory.getCLogger(GIFileBaseService.class.getName());

    protected final GIFilesLoadMetricReport giFilesLoadMetricReport;


    protected GIFileBaseService(MetricReportRegistry metricReportRegistry) {
        this.giFilesLoadMetricReport = metricReportRegistry.metricReport(GI_FILES_LOAD_METRIC_REPORT);
    }

    protected Function<String, String[]> processLine = line ->
            line.replace("\"", "").split(GI_FILE_COLUMN_SEPARATOR, -1);

    protected Predicate<String[]> pickOnlyLineOfPruPassagesSourceCode = line ->
            PruPassagesSourceCode.contains(line[GI_FILE_SOURCECODE_COLUMN_INEX]);


    protected LeadInfoDTO getLeadInfoDTO(ConcurrentMap<String, LeadInfoDTO> leadInfoDTOMap,
                                         String employeeId,
                                         GIFile giFile) {
        LeadInfoDTO leadInfoDTO = leadInfoDTOMap.get(employeeId);

        if (ObjectUtils.isEmpty(leadInfoDTO)) {
            if (giFile == GIFile.CUSTOMER) {
                leadInfoDTO = LeadInfoDTO.builder()
                        .externalIdType(GI_EXTERNAL_ID_TYPE)
                        .externalIdValue(employeeId)
                        .leadSource(LeadSourceEnum.PRUPASSAGES.getValue())
                        .build();
                giFilesLoadMetricReport.increaseETSCOMAndETSJETRecord();
            }
            else {
                String msg = String.format("Not processed: Employee Id %s in GI file %s, but not in %s file",
                        employeeId, giFile.name(), GIFile.CUSTOMER.name());
                logger.info(msg);
                giFilesLoadMetricReport.getGiFilesLoadMessages().add(msg);
            }
        }

        return leadInfoDTO;
    }

    protected void addPhoneDTO(LeadInfoDTO leadInfoDTO, String phoneType, String phoneValue) {
        if (ObjectUtils.isEmpty(leadInfoDTO)) {
            logger.warn("leadInfoDTO is empty");
            return;
        }
        if (StringUtils.isEmpty(phoneValue)) {
            logger.info("phoneValue is empty");
            return;
        }

        PhoneDTO phoneDTO = PhoneDTO.builder()
                .phoneType(PhoneTypeEnum.fromValue(phoneType).getValue())
                .phoneValue(StringUtils.right(phoneValue, PHONE_LENGTH))
                .build();

        if (leadInfoDTO.getPhone() == null)
            leadInfoDTO.setPhone(new ArrayList<>());

        leadInfoDTO.getPhone().add(phoneDTO);
    }

    protected void addEmailAddressDTO(LeadInfoDTO leadInfoDTO, String emailType, String emailValue) {
        if (ObjectUtils.isEmpty(leadInfoDTO)) {
            logger.warn("leadInfoDTO is empty");
            return;
        }
        if (ObjectUtils.isEmpty(emailValue)) {
            logger.info("emailValue is empty");
            return;
        }

        EmailAddressDTO emailAddressDTO = EmailAddressDTO.builder()
                .emailType(EmailTypeEnum.fromValue(emailType).getValue())
                .addrLine(emailValue)
                .build();

        if (leadInfoDTO.getEmail() == null)
            leadInfoDTO.setEmail(new ArrayList<>());

        leadInfoDTO.getEmail().add(emailAddressDTO);
    }
}
```

**3. Concrete Customer Service for loading and processing customer's file**
```java
@Service
public class CustomerService extends GIFileBaseService implements FileService {
    private static final CLogger logger = LogFactory.getCLogger(CustomerService.class.getName());

    public CustomerService(MetricReportRegistry metricReportRegistry) {
        super(metricReportRegistry);
    }

    @Override
    public void load(ConcurrentMap<String, LeadInfoDTO> leadInfoDTOMap) throws Exception {
        String filename = getFileWithDate(GI_CUSTOMER_DATA_FILE);

        logger.info("GI Customer file: " + filename + " loading is started");
        AtomicInteger numOfRecords = new AtomicInteger();

        Consumer<String[]> customerConsumer = line -> {
            LeadInfoDTO leadInfoDTO = getLeadInfoDTO(
                    leadInfoDTOMap,
                    line[Customer.EMPLOYEEID.getIndex()],
                    GIFile.CUSTOMER);

            if (ObjectUtils.isEmpty(leadInfoDTO.getEmployment()))
                leadInfoDTO.setEmployment(EmploymentDTO.builder().build());

            leadInfoDTO.getEmployment().setEmployeeId(line[Customer.EMPLOYEEID.getIndex()]);
            leadInfoDTO.getEmployment().setAnnualSalary(line[Customer.ANNUALINCOME.getIndex()]);
            leadInfoDTO.setFirstName(line[Customer.FIRSTNAME.getIndex()]);
            leadInfoDTO.setLastName(line[Customer.LASTNAME.getIndex()]);
            leadInfoDTO.setBirthDate(line[Customer.DATEOFBIRTH.getIndex()]);
            leadInfoDTO.setGender(Gender.fromValue(line[Customer.GENDER.getIndex()]).getValue());
            leadInfoDTO.setAnnualHouseholdIncome(line[Customer.ANNUALINCOME.getIndex()]);
            leadInfoDTO.setAnnualIndividualIncome(line[Customer.ANNUALINCOME.getIndex()]);

            leadInfoDTOMap.put(line[Customer.EMPLOYEEID.getIndex()], leadInfoDTO);
            numOfRecords.getAndIncrement();

            if (PruPassagesSourceCode.isETSCOM(line[GI_FILE_SOURCECODE_COLUMN_INEX]))
                giFilesLoadMetricReport.increaseETSCOMRecord();

            if (PruPassagesSourceCode.isETSJET(line[GI_FILE_SOURCECODE_COLUMN_INEX]))
                giFilesLoadMetricReport.increaseETSJETRecord();
        };

        // One line Lambda function to load, filter, and process the data file
        Files.lines(Paths.get(filename))
                .map(processLine)
                .filter(pickOnlyLineOfPruPassagesSourceCode)
                .forEach(customerConsumer);

        logger.info("GI Customer file: " + filename + " records loaded are: " + numOfRecords);
        logger.info("leadInfoDTOMap size: "+leadInfoDTOMap.size());
        logger.info("GI Customer file: " + filename + " loading is done");
        
        //Delete file from container once processing finished
        boolean fileDeleted =  Files.deleteIfExists(Paths.get(filename));
        logger.info("GI Customer file: " + filename + " deleted:  "+fileDeleted);
        
        String filenameCTL = getFileWithDate(GI_CUSTOMER_CONTROL_FILE);        
        boolean fileDeletedCTL =  Files.deleteIfExists(Paths.get(filenameCTL));        
        logger.info("GI Customer file: " + filenameCTL + " deleted:  "+fileDeletedCTL);
    }
}
```