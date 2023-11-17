## Enum Validator


**1. Enum Validator** 
```java
@Documented
@Constraint(validatedBy = EnumValidatorImpl.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.PARAMETER, ElementType.METHOD, ElementType.FIELD})
@NotNull(message = "Value cannot be null")
@ReportAsSingleViolation
public @interface EnumValidator {

    Class<? extends Enum<?>> enumClazz();

    String message() default "Value is not valid";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};


}
```

**2. EnumValidator Implementation**
```java
public class EnumValidatorImpl implements ConstraintValidator<EnumValidator, String> {

    List<String> valueList = null;


    public boolean isValid(String value, ConstraintValidatorContext context) {
        if(!valueList.contains(value.toUpperCase())) {
            return false;
        }
        return true;
    }


    public void initialize(EnumValidator constraintAnnotation) {
        valueList = new ArrayList<String>();
        Class<? extends Enum<?>> enumClass = constraintAnnotation.enumClazz();

        @SuppressWarnings("rawtypes")
        Enum[] enumValArr = enumClass.getEnumConstants();

        for(@SuppressWarnings("rawtypes")
                Enum enumVal : enumValArr) {
            valueList.add(enumVal.toString().toUpperCase());
        }

    }

}
```

**3. EnumValidator Interface**
```java
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {OffersEngineCodeStringValidatorImp.class, OffersEngineCodeListOfStringValidatorImp.class})
public @interface OffersEngineCodeValidator {

    OffersEngineCodeEnum codeEnum() default OffersEngineCodeEnum.OFFER_CODE;

    MetadataVerification metadataVerification() default MetadataVerification.VALUE;

    String message() default "Value is not valid";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

**4. EnumValidator Implementation**
```java
public class OffersEngineCodeStringValidatorImp implements ConstraintValidator<OffersEngineCodeValidator, String> {

    @Autowired
    private InternalOperationsClient internalOperationsClient;

    private String key = null;

    private MetadataVerification metadataVerification;

    @Override
    public void initialize(OffersEngineCodeValidator constraint) {
        key = constraint.codeEnum().getKey();
        metadataVerification = constraint.metadataVerification();
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        // check metadata code
        if (metadataVerification == MetadataVerification.CODE) {
            return !StringUtils.isEmpty(value)
                    && internalOperationsClient
                    .getApplicationMetaDataDTOByMetaDataKey(key)
                    .keySet()
                    .stream()
                    .anyMatch(v -> v.equalsIgnoreCase(value));
        }

        // check metadata value
        return !StringUtils.isEmpty(value)
                && internalOperationsClient
                .getApplicationMetaDataDTOByMetaDataKey(key)
                .values()
                .stream()
                .anyMatch(v -> v.equalsIgnoreCase(value));
    }
}
```

**5. EnumValidator Implementation**
```java
public class OffersEngineCodeListOfStringValidatorImp implements ConstraintValidator<OffersEngineCodeValidator, List<String>> {
    @Autowired
    private InternalOperationsClient internalOperationsClient;

    private String key = null;

    private MetadataVerification metadataVerification;

    @Override
    public void initialize(OffersEngineCodeValidator constraint) {
        key = constraint.codeEnum().getKey();
        metadataVerification = constraint.metadataVerification();
     }

    @Override
    public boolean isValid(List<String> values, ConstraintValidatorContext context) {
        // check metadata codes
        if (metadataVerification == MetadataVerification.CODE) {
            return !ObjectUtils.isEmpty(values)
                    && internalOperationsClient
                    .getApplicationMetaDataDTOByMetaDataKey(key)
                    .keySet()
                    .containsAll(values);
        }

        // check metadata values
        return !ObjectUtils.isEmpty(values)
                && internalOperationsClient
                .getApplicationMetaDataDTOByMetaDataKey(key)
                .values()
                .containsAll(values);
    }
}
```

**6. These annotations can be added to a field and we can pass any enum class.**
```java
public class Customer {
    @EnumValidator(enumClass = CustomerType.class)
    private String customerTypeString;

    @NotNull
    @CustomerTypeSubset(anyOf = {CustomerType.NEW, CustomerType.OLD})
    private CustomerType customerTypeOfSubset;
    
    @OffersEngineCodeValidator(enumClass = OffersEngineCode.class)
    private OffersEngineCode offersEngineCode;

    @OffersEngineCodeValidator(enumClass = OffersEngineCode.class)
    private List<OffersEngineCode> offersEngineCodeList;

    @EnumNamePattern(regexp = "NEW|DEFAULT")
    private CustomerType customerTypeMatchesPattern;

    // constructor, getters etc.
}
```
