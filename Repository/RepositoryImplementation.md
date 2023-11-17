## Repository Implementation

**1. Repository Interface**
```java
public interface OfferEnrollmentSummaryRepository {

    Integer getOfferEnrollmentSumamryTotalNumberOfItems(Long offerId, Long coUserId, String enrollmentStatus, IncludeFinishedEnrollmentStatus includeFinishedEnrollmentStatus);

    List<OfferEnrollmentSummaryDTO> getOfferEnrollmentSummaryList(Long offerId, Long coUserId, String enrollmentStatus, OfferEnrollmentSummarySortBy sortByEnum, SortDirection sortDirection, PaginationDTO paginationDTO, IncludeFinishedEnrollmentStatus includeFinishedEnrollmentStatus);
}
```

**2. Repository Implementation**
```java
@Repository
public class OfferEnrollmentSummaryRepositoryImpl implements OfferEnrollmentSummaryRepository {
    private static final CLogger logger = LogFactory.getCLogger(OfferEnrollmentSummaryRepositoryImpl.class.getName());

    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public Integer getOfferEnrollmentSumamryTotalNumberOfItems(Long offerId, Long coUserId, String enrollmentStatus, IncludeFinishedEnrollmentStatus includeFinishedEnrollmentStatus) {

        Object totalNumberOfItems = entityManager
                .createNativeQuery(COUNT_OFFERENROLLMENT_BY_CONDITIONS)
                .setParameter("offerid", offerId)
                .setParameter("couserid", coUserId)
                .setParameter("enrollmentstatus", enrollmentStatus)
                .setParameter("includefinishedenrollmentstatus", includeFinishedEnrollmentStatus.name())
                .getSingleResult();

        return ((Number) totalNumberOfItems).intValue();
    }

    @SuppressWarnings("unchecked")
    @Override
    public List<OfferEnrollmentSummaryDTO> getOfferEnrollmentSummaryList(Long offerId, Long coUserId, String enrollmentStatus, OfferEnrollmentSummarySortBy sortByEnum, SortDirection sortDirection, PaginationDTO paginationDTO, IncludeFinishedEnrollmentStatus includeFinishedEnrollmentStatus) {

        return (List<OfferEnrollmentSummaryDTO>) entityManager
                .createNativeQuery(FIND_OFFERENROLLMENT_BY_CONDITIONS, "OfferEnrollmentSummaryDTOMapping")
                .setParameter("offerid", offerId)
                .setParameter("couserid", coUserId)
                .setParameter("enrollmentstatus", enrollmentStatus)
                .setParameter("sortby", sortByEnum.name())
                .setParameter("sortdirecion", sortDirection.name())
                .setParameter("min", paginationDTO.getStart())
                .setParameter("max", paginationDTO.getEnd())
                .setParameter("includefinishedenrollmentstatus", includeFinishedEnrollmentStatus.name())
                .getResultList();
    }
}
```