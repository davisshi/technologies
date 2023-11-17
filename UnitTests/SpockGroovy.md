## Using Spock Groovy to do the unit tests

### Test each business logic with different use cases in each unit test 

```java
def "SaveOrUpdateUserAgent with Exception"() {
        given:
        UserAgent resultUserAgent
        userAgentRepository.countCoUser(_, _) >> countCoUser
        userAgentRepository.getAgentInfo(_, _) >> {throw new Exception()}

        when:
        resultUserAgent = userAgentService.saveOrUpdateUserAgent(userAgentDTO)

        then:
        expectedWarn1 * logger.warn(expectedMessage)
        expectedWarn2 * logger.warn(expectedMessage2, _)
        def e = thrown(expectedException)
        e.message == expectedMessage

        where:
        countCoUser << [0, 0, 1]
        userAgentDTO << [new UserAgentDTO(), getUserAgentDTOWithcoUserId(new UserAgentDTO()), getUserAgentDTOWithcoUserId(new UserAgentDTO())]
        expectedMessage2 = "Cannot get agent couserid and type based on agent employeeid"

        expectedException            | expectedWarn1 | expectedWarn2 | expectedMessage
        InternalOperationsException  | 1             | 0             | "coUserId cannot be empty"
        InternalOperationsException  | 1             | 0             | "No this coUserId: 11111"
        InternalOperationsException  | 1             | 1             | "Invalid Employee Id"
}
```

```java
def "CreateUserBYCStatus Successfully"() {
        given:
        UserActivity userActivity

        when:
        userActivity = userBYCStatusService.createUserBYCStatus(userBYCStatusDTO)

        then:
        1 * userActivityRepository.getCountCoUserId(userBYCStatusDTO.getCoUserId(), NOT_DELETED) >> 1
        0 * bycStepRepository.getBYCStep(USERACTIVITY_ACTIVITYTYPECODE_BYCSTATUS, stepCode, NOT_DELETED) >> bycStepDTO
        1 * userActivityRepository.findByCoUserIdAndActivityTypeCodeAndActivityMilestoneCodeAndDeleted(
        coUserId,
        USERACTIVITY_ACTIVITYTYPECODE_BYCSTATUS,
        stepCode,
        NOT_DELETED
        ) >> null

        1 * userActivityRepository.save(_) >> {args -> userActivity = args[0]}
        userActivity instanceof UserActivity
        userActivity.coUserId == coUserId
        userActivity.activityTypeCode == USERACTIVITY_ACTIVITYTYPECODE_BYCSTATUS
        userActivity.activityMilestoneCode == stepCode
        userActivity.activityStatusCode == StepStatus.COMPLETED.name()
        userActivity.activityMessage == description
        userActivity.createdDate != null
        userActivity.createdBy == coUserId.toString()
        userActivity.updatedDate != null

        where:
        stepDate = new LocalDate(2017, 12, 28)

        coUserId | step   | stepCode   | description    | status                | contractId | managedAccountId | engagementSource | engagementPath | goalType
        11111    | 1      | "STEP1"    | "description1" | StepStatus.COMPLETED  | "8CP1234"  | "123456"         | "EP"             | "SingleGoal"   | "Retirement"
        22222    | 5      | "STEP5"    | "description5" | StepStatus.COMPLETED  | "8CP1234"  | "123456"         | "EP"             | "SingleGoal"   | "Retirement"

        userBYCStatusDTO = new UserBYCStatusDTO(coUserId, stepDate, stepCode, description, status, contractId, managedAccountId, engagementSource, engagementPath, goalType)

        bycStepDTO = new BYCStepDTO(step, stepCode, description)
}
```

```java
def "GetOfferEnrollmentEligibility"() {
        given:
        OfferEnrollmentEligibilityDTO result
        1 * offerEnrollmentSummaryRepository.countValidOffer(_, _) >> countValidOffer
        numCallGetOfferEnrollmentStatusAndComments * offerEnrollmentSummaryRepository.getOfferEnrollmentStatusAndComments(offerSourceCode, coUserId) >> offerEnrollmentStatusAndCommentsDTOS
        numCallCountOfferEnrollmentProcessing * offerEnrollmentSummaryRepository.countOfferEnrollmentProcessing(coUserId) >> countOfferEnrollmentProcessing
        numCallRetrieveUserAccounts * internalOperationsClient.retrieveUserAccounts(coUserId) >> userAccounts
        numCallGetApplicationMetaDataDTOByMetaDataKey * internalOperationsClient.getApplicationMetaDataDTOByMetaDataKey(OffersEngineCodeEnum.OFFERS_PROMOTION_START_DATE.getKey()) >> mapOfferPromotionStartDate

        when:
        result = offerEnrollmentSummaryService.getOfferEnrollmentEligibility(offerSourceCode, coUserId, checkPMA)

        then:
        result.offerSourceCode == offerSourceCode
        result.coUserId == coUserId
        result.getOfferEnrollmentStatusAndCommentsDTOS() == offerEnrollmentStatusAndCommentsDTOS
        result.hasPMA == hasPMA
        result.getOfferEnrollmentEligibility() == Boolean.FALSE
        result.getComments() == comments

        where:
        offerSourceCode = "sourcecode2"
        coUserId = 200095390870
        offerEnrollmentStatus = "Enrolled"

        comments << [
                ["There is no valid offer"],
                ["There is more than one valid offer"],
                ["Only one active promotion can be enrolled in at a time"],
                ["There is an offer enrollment of either Enrolled, Waiting For Qualification, or Qualified"],
                ["Check PMA account before launch date " + DateUtil.currentDatePlusDaysToString(1, DEFAULT_BUSINESS_ZONEID),
                 "This user has no PMA account",
                 "There is an offer enrollment of either Enrolled, Waiting For Qualification, or Qualified"],
                ["Check PMA account before launch date " + DateUtil.currentDatePlusDaysToString(1, DEFAULT_BUSINESS_ZONEID),
                 "This user has PMA account"],
                ["Don't check PMA account on or after launch date " + DateUtil.currentDateToString(DEFAULT_BUSINESS_ZONEID),
                 "There is an offer enrollment of either Enrolled, Waiting For Qualification, or Qualified"],
                ["Don't check PMA account on or after launch date " + DateUtil.currentDateMinusDaysToString(1, DEFAULT_BUSINESS_ZONEID),
                 "There is an offer enrollment of either Enrolled, Waiting For Qualification, or Qualified"],
                ["Don't check PMA account on or after launch date " + DateUtil.currentDateMinusDaysToString(1, DEFAULT_BUSINESS_ZONEID),
                 "There is a disqualified offer enrollment with comments SSN Has Been Used for Promo"]
        ]

        countValidOffer << [0, 2, 1, 1, 1, 1, 1, 1, 1]

        numCallGetOfferEnrollmentStatusAndComments << [0, 0, 1, 1, 1, 1, 1, 1, 1]
        numCallCountOfferEnrollmentProcessing << [0, 0, 1, 1, 1, 1, 1, 1, 1]
        numCallRetrieveUserAccounts << [0, 0, 0, 0, 1, 1, 0, 0, 0]
        numCallGetApplicationMetaDataDTOByMetaDataKey << [0, 0, 0, 0, 1, 1, 1, 1, 1]

        countOfferEnrollmentProcessing << [0, 0, 1, 0, 0, 0, 0, 0, 0]

        mapOfferPromotionStartDate << [
                ['startdate': DateUtil.currentDateMinusDaysToString(1, DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDateMinusDaysToString(1, DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDateMinusDaysToString(1, DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDatePlusDaysToString(1, DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDatePlusDaysToString(1, DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDatePlusDaysToString(1, DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDateToString(DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDateMinusDaysToString(1, DEFAULT_BUSINESS_ZONEID)],
                ['startdate': DateUtil.currentDateMinusDaysToString(1, DEFAULT_BUSINESS_ZONEID)]
        ]

        checkPMA << [Boolean.TRUE, Boolean.TRUE, Boolean.TRUE, Boolean.FALSE, Boolean.TRUE, Boolean.TRUE, Boolean.TRUE, Boolean.TRUE, Boolean.TRUE]

        offerEnrollmentStatusAndCommentsDTOS << [
                null,
                null,
                {
                    List<OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO> offerEnrollmentStatusAndCommentsDTOS = new ArrayList<>()

                    OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO offerEnrollmentStatusAndCommentsDTO = new OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO()
                    offerEnrollmentStatusAndCommentsDTO.setStatus("Enrolled")

                    offerEnrollmentStatusAndCommentsDTOS.add(offerEnrollmentStatusAndCommentsDTO)

                    offerEnrollmentStatusAndCommentsDTOS
                }(),
                {
                    List<OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO> offerEnrollmentStatusAndCommentsDTOS = new ArrayList<>()

                    OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO offerEnrollmentStatusAndCommentsDTO = new OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO()
                    offerEnrollmentStatusAndCommentsDTO.setStatus("Enrolled")

                    offerEnrollmentStatusAndCommentsDTOS.add(offerEnrollmentStatusAndCommentsDTO)

                    offerEnrollmentStatusAndCommentsDTOS
                }(),
                {
                    List<OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO> offerEnrollmentStatusAndCommentsDTOS = new ArrayList<>()

                    OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO offerEnrollmentStatusAndCommentsDTO = new OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO()
                    offerEnrollmentStatusAndCommentsDTO.setStatus("Enrolled")

                    offerEnrollmentStatusAndCommentsDTOS.add(offerEnrollmentStatusAndCommentsDTO)

                    offerEnrollmentStatusAndCommentsDTOS
                }(),
                {
                    List<OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO> offerEnrollmentStatusAndCommentsDTOS = new ArrayList<>()

                    OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO offerEnrollmentStatusAndCommentsDTO = new OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO()
                    offerEnrollmentStatusAndCommentsDTO.setStatus("Enrolled")

                    offerEnrollmentStatusAndCommentsDTOS.add(offerEnrollmentStatusAndCommentsDTO)

                    offerEnrollmentStatusAndCommentsDTOS
                }(),
                {
                    List<OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO> offerEnrollmentStatusAndCommentsDTOS = new ArrayList<>()

                    OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO offerEnrollmentStatusAndCommentsDTO = new OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO()
                    offerEnrollmentStatusAndCommentsDTO.setStatus("Enrolled")

                    offerEnrollmentStatusAndCommentsDTOS.add(offerEnrollmentStatusAndCommentsDTO)

                    offerEnrollmentStatusAndCommentsDTOS
                }(),
                {
                    List<OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO> offerEnrollmentStatusAndCommentsDTOS = new ArrayList<>()

                    OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO offerEnrollmentStatusAndCommentsDTO = new OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO()
                    offerEnrollmentStatusAndCommentsDTO.setStatus("Enrolled")

                    offerEnrollmentStatusAndCommentsDTOS.add(offerEnrollmentStatusAndCommentsDTO)

                    offerEnrollmentStatusAndCommentsDTOS
                }(),
                {
                    List<OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO> offerEnrollmentStatusAndCommentsDTOS = new ArrayList<>()

                    OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO offerEnrollmentStatusAndCommentsDTO = new OfferEnrollmentEligibilityDTO.OfferEnrollmentStatusAndCommentsDTO()
                    offerEnrollmentStatusAndCommentsDTO.setStatus("Disqualified")
                    offerEnrollmentStatusAndCommentsDTO.setComments(DisqualificationComment.SSN_HAS_BEEN_USED_FOR_PROMO.getValue())

                    offerEnrollmentStatusAndCommentsDTOS.add(offerEnrollmentStatusAndCommentsDTO)

                    offerEnrollmentStatusAndCommentsDTOS
                }()
        ]

        hasPMA << [null, null, null, null, Boolean.FALSE, Boolean.TRUE, null, null, null]

        userAccounts << [
                null,
                null,
                null,
                null,
                {
                    List userAccounts = new ArrayList()
                    userAccounts
                }(),
                {
                    List userAccounts = new ArrayList()
                    Body body = new Body()
                    userAccounts.add(body)

                    userAccounts
                }(),
                {
                    List userAccounts = new ArrayList()
                    Body body = new Body()
                    userAccounts.add(body)

                    userAccounts
                }(),
                {
                    List userAccounts = new ArrayList()
                    Body body = new Body()
                    userAccounts.add(body)

                    userAccounts
                }(),
                {
                    List userAccounts = new ArrayList()
                    Body body = new Body()
                    userAccounts.add(body)

                    userAccounts
                }()
        ]
}
```
