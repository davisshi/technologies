## Class defined with Lombok

```java
import lombok.*;

import javax.persistence.*;
import javax.validation.constraints.NotNull;

@Builder
@NoArgsConstructor
@AllArgsConstructor
@Data
@Entity
@EqualsAndHashCode(of = {"email,source,firstName,lastName,deleted"})
public class DoNotEmail extends DoNotSolicit {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "DONOTEMAIL_SEQ")
    @SequenceGenerator(sequenceName = "DONOTEMAILID", allocationSize = 1, name = "DONOTEMAIL_SEQ")
    private Long doNotEmailId;

    @NotNull
    private String email;
}
```

```java
@Data
public class ContactEligibilityDto {
    private UserInfoDto userInfoDto;
    private String phoneNum;

    private Boolean contactEligibility;
    private List<String> comments = new ArrayList<>();

    @Data
    @Builder
    public static class StateDNCDate {
        private String stateCode;
        private LocalDate dncDate;
    }

    @Data
    @Builder
    public static class ContactConsentDto implements Comparable<ContactConsentDto> {
        private String consentName;
        private String overridePermissions;
        private LocalDate consentDate;
        private ConsentStatus consentStatus;

        @Override
        public int compareTo(ContactConsentDto o) {
            return this.consentDate.compareTo(o.consentDate);
        }
    }
}
```