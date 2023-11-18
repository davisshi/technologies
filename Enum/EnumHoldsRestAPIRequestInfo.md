## Enum holds Rest API Request Info

```java
public enum AccountReportService {

    PROFILE(PROFILE_SERVICENAME, PROFILE_SERVICEPATH, HttpMethod.GET, PROFILE_HTTPHEADERS, null, true),
    ABOUTME(ABOUTME_SERVICENAME, ABOUTME_SERVICEPATH, HttpMethod.GET, ABOUTME_HTTPHEADERS, null, false),
    GOALS(GOALS_SERVICENAME, GOALS_SERVICEPATH, HttpMethod.GET, GOALS_HTTPHEADERS, null, false),
    ACCOUNTSUMMARY(ACCOUNTSUMMARY_SERVICENAME, ACCOUNTSUMMARY_SERVICEPATH, HttpMethod.POST, ACCOUNTSUMMARY_HTTPHEADERS, ACCOUNTSUMMARY_PAYLOAD, false),
    ACCOUNTINFO (ACCOUNTINFO_SERVICENAME, ACCOUNTINFO_SERVICEPATH, HttpMethod.GET, ACCOUNTINFO_HTTPHEADERS, null, false),
    QUESTIONS(QUESTIONS_SERVICENAME, QUESTIONS_SERVICEPATH, HttpMethod.GET, QUESTIONS_HTTPHEADERS, null, false),
    ANSWERS(ANSWERS_SERVICENAME, ANSWERS_SERVICEPATH, HttpMethod.GET, ANSWERS_HTTPHEADERS, null, false);

    private final String serviceName;
    private final String servicePath;
    private final HttpMethod httpMethod;
    private final String httpHeaders;
    private final String payload;
    private final boolean isProfileService;

    AccountReportService(String serviceName, String servicePath, HttpMethod httpMethod, String httpHeaders, String payload, boolean isProfileService) {
        this.serviceName = serviceName;
        this.servicePath = servicePath;
        this.httpMethod = httpMethod;
        this.httpHeaders = httpHeaders;
        this.payload = payload;
        this.isProfileService = isProfileService;
    }

    public String getServiceName() {
        return serviceName;
    }

    public String getServicePath() {
        return servicePath;
    }

    public HttpMethod getHttpMethod() {
        return httpMethod;
    }

    public String getHttpHeaders() {
        return httpHeaders;
    }

    public String getPayload() {
        return payload;
    }

    public boolean isProfileService() {
        return isProfileService;
    }
}
```