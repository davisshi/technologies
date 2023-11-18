## Map holds Values, Map, and BiConsumers

```java
public enum LeadReportTimeLineEnum {

    CURRENT_DATE(Constants.CURRENT_DATE, (leadItemCountDTO, count) -> leadItemCountDTO.setCountOfToday(leadItemCountDTO.getCountOfToday() + count)),
    THIS_WEEK_TO_DATE(Constants.THIS_WEEK_TO_DATE, (leadItemCountDTO, count) -> leadItemCountDTO.setCountOfThisWeekToDate(leadItemCountDTO.getCountOfThisWeekToDate() + count)),
    LAST_WEEK(Constants.LAST_WEEK, (leadItemCountDTO, count) -> leadItemCountDTO.setCountOfLastWeek(leadItemCountDTO.getCountOfLastWeek() + count)),
    THIS_MONTH_TO_DATE(Constants.THIS_MONTH_TO_DATE, (leadItemCountDTO, count) -> leadItemCountDTO.setCountOfThisMonthToDate(leadItemCountDTO.getCountOfThisMonthToDate() + count)),
    LAST_MONTH(Constants.LAST_MONTH, (leadItemCountDTO, count) -> leadItemCountDTO.setCountOfLastMonth(leadItemCountDTO.getCountOfLastMonth() + count)),
    ALL_TIME(Constants.ALL_TIME, (leadItemCountDTO, count) -> leadItemCountDTO.setCountOfAllTime(leadItemCountDTO.getCountOfAllTime() + count));

    private static Map<String, LeadReportTimeLineEnum> leadReportTimeLineEnumMap
            = Arrays.stream(LeadReportTimeLineEnum.values())
            .collect(Collectors.toMap(LeadReportTimeLineEnum::getUpperCaseValue, Function.identity()));

    private final String value;

    private final BiConsumer<LeadItemCountDTO, Integer> setCount;

    LeadReportTimeLineEnum(String value, BiConsumer<LeadItemCountDTO, Integer> setCount) {
        this.value = value;
        this.setCount = setCount;
    }

    public String getValue() {
        return value;
    }

    public String getUpperCaseValue() {
        return value.toUpperCase();
    }

    public BiConsumer<LeadItemCountDTO, Integer> getSetCount() {
        return setCount;
    }

    public static LeadReportTimeLineEnum getLeadReportTimeLineEnum(String key) {
        return leadReportTimeLineEnumMap.get(key.toUpperCase());
    }
}
```