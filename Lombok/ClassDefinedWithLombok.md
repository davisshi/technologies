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