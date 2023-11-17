## Customized Enum with Customized Methods

```java
public enum TrexFileType {
    QUALIFIED(QUALIFIED_ACCOUNT, QUALIFIED_TEMPLATE_FILENAME, AccountIRSCode.QUALIFIED) {
        @Override
        public void setCountOfAccountForRFQ (TrexFileReportDataModel trexFileReportDataModel, int countOfAccountForRFQ){
            trexFileReportDataModel.setCountOfQualifiedAccountForRFQ(countOfAccountForRFQ);
        }

        @Override
        public void setEmailSentSuccessfullyForAccounts(TrexFileReportDataModel trexFileReportDataModel, boolean emailSentStatus) {
            trexFileReportDataModel.setEmailSentSuccessfullyForQualifiedAccounts(emailSentStatus);
        }
    },
    NON_QUALIFIED(NON_QUALIFIED_ACCOUNT, NON_QUALIFIED_TEMPLATE_FILENAME, AccountIRSCode.NON_QUALIFIED) {
        @Override
        public void setCountOfAccountForRFQ(TrexFileReportDataModel trexFileReportDataModel, int countOfAccountForRFQ) {
            trexFileReportDataModel.setCountOfNonQualifiedAccountForRFQ(countOfAccountForRFQ);
        }

        @Override
        public void setEmailSentSuccessfullyForAccounts(TrexFileReportDataModel trexFileReportDataModel, boolean emailSentStatus) {
            trexFileReportDataModel.setEmailSentSuccessfullyForNonQualifiedAccounts(emailSentStatus);
        }
    };

    private String fileType;
    private String templateFileName;
    private AccountIRSCode accountIRSCode;
    private String emailToAddress;
    private String emailCcAddress;

    TrexFileType(String fileType, String templateFileName, AccountIRSCode accountIRSCode) {
        this.fileType = fileType;
        this.templateFileName = templateFileName;
        this.accountIRSCode = accountIRSCode;
    }

    public String getFileType() {
        return fileType;
    }

    public String getTemplateFileName() {
        return templateFileName;
    }

    public String getOutputFileName() {
        return "./" + templateFileName.replace("YYYYMMDD", LocalDate.now().toString().replace("-",""));
    }

    public AccountIRSCode getAccountIRSCode() {
        return accountIRSCode;
    }

    public String getEmailToAddress() {
        return emailToAddress;
    }

    public String getEmailCcAddress() {
        return emailCcAddress;
    }

    public abstract void setCountOfAccountForRFQ(TrexFileReportDataModel trexFileReportDataModel, int countOfAccountForRFQ);

    public abstract void setEmailSentSuccessfullyForAccounts(TrexFileReportDataModel trexFileReportDataModel, boolean emailSentStatus);

    @Component
    public static class EmailAddressInjector {
        @Autowired
        private EmailSMTPProperties emailSMTPProperties;

        @PostConstruct
        public void postConstruct() {
            QUALIFIED.emailToAddress = emailSMTPProperties.getEmailQualifiedToAddress();
            QUALIFIED.emailCcAddress = emailSMTPProperties.getEmailQualifiedCcAddress();
            NON_QUALIFIED.emailToAddress = emailSMTPProperties.getEmailNonQualifiedToAddress();
        }
    }
}
```