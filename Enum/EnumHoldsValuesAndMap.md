## Enum holds values and Map

```java
public enum PruBrandedProductCodeEnum {
    DIGITAL_BASIC_TERM("Simply Term", "DIGITAL_BASIC_TERM"),
    SIFE("SIFE", "SIFE"),
    TERM_ESSENTIAL("Term Essential", "TERM_ESSENTIAL"),
    TERM_ESSENTIAL_PILOT("Term Essential Pilot", "TERM_ESSENTIAL_PILOT");

    private static List<String> allProductCodes = Arrays.stream(PruBrandedProductCodeEnum.values())
            .map(PruBrandedProductCodeEnum::getValue)
            .collect(Collectors.toList());

    private static Map<String,PruBrandedProductCodeEnum> pruBrandedProductCodeEnumMap
            = Arrays.stream(PruBrandedProductCodeEnum.values())
            .collect(Collectors.toMap(PruBrandedProductCodeEnum::getUpperCaseValue, Function.identity()));

    private final String name;
    private final String value;

    PruBrandedProductCodeEnum(String name, String value) {
        this.name = name;
        this.value = value;
    }

    public String getName() {
        return name;
    }

    public String getValue() {
        return value;
    }

    public String getUpperCaseValue() {
        return value.toUpperCase();
    }

    public static List<String> getAllProductCodes() {
        return allProductCodes;
    }

    public static PruBrandedProductCodeEnum getPruBrandedProductCodeEnum(String key) {
        return pruBrandedProductCodeEnumMap.get(key.toUpperCase());
    }
}
```