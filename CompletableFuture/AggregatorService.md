## Aggregator Service using Completable Future

**1. Async Service Interface**
```java
public interface AsyncService {
    
    CompletableFuture<List<UserAccountDTO>> getAsyncUserManagedAccounts(Long coUserId);
    
    CompletableFuture<List<UserManagedAccountPortfolioDTO>> getAsyncUserPortfolioChangesBetweenDates(String contractId, String startDate, String endDate);
    
    CompletableFuture<WrapperDTO<GoalModificationProposalDTO>> asyncProposalUpdate(ProposalUpdateDTO proposalUpdateDTO);

    CompletableFuture<AggregatorResponseDTO> asyncAggregatorRequest(AggregatorRequestDTO aggregatorRequestDTO, String authorizationToken);
}
```

**2. Async Service Implementation**
```java
@Async
@Service
public class AsyncServiceImpl implements AsyncService {
    @Override
    public CompletableFuture<AggregatorResponseDTO> asyncAggregatorRequest(AggregatorRequestDTO aggregatorRequestDTO, String authorizationToken) {
        logger.info("Task start thread: " + Thread.currentThread().getName());
        long start = System.nanoTime();

        // create aggregatorResponseDTO and set its service name
        AggregatorResponseDTO aggregatorResponseDTO = new AggregatorResponseDTO();
        aggregatorResponseDTO.setServiceName(aggregatorRequestDTO.getServiceName());

        // check the service path and httpmethod first
        if (StringUtils.isEmpty(aggregatorRequestDTO.getServicePath())
                || StringUtils.isEmpty(aggregatorRequestDTO.getHttpMethod())) {

            return CompletableFuture.completedFuture(aggregatorResponseDTO);
        }

        ResponseEntity<String> responseEntity = null;
        int statusCodeValue = HttpStatus.NO_CONTENT.value();
        HttpStatus httpStatusCode = HttpStatus.NO_CONTENT;
        String responseBody = null;
        String url = generateUrl(aggregatorRequestDTO.getServicePath());
        HttpHeaders headers = generateHeaders(aggregatorRequestDTO.getHttpHeaders(), authorizationToken);

        // for each request to make a call
        try {
            responseEntity
                    = httpClient.requestAndReturnResponseEntity(
                    url,
                    aggregatorRequestDTO.getHttpMethod(),
                    headers,
                    aggregatorRequestDTO.getPayload(),
                    String.class
            );

            statusCodeValue = responseEntity.getStatusCodeValue();
            httpStatusCode = responseEntity.getStatusCode();
            responseBody = responseEntity.getBody();
        }
        catch (HttpClientErrorException | HttpServerErrorException e) {
            logger.warn("Cannot make service call, aggregateRequestDTO = " + aggregatorRequestDTO, e);
            statusCodeValue = e.getRawStatusCode();
            httpStatusCode = e.getStatusCode();
            responseBody = e.getResponseBodyAsString();
        }
        catch (Exception e) {
            logger.warn("Cannot make service call, aggregateRequestDTO = " + aggregatorRequestDTO, e);
            aggregatorResponseDTO.setResponse(e.getMessage());
        }
        finally {
            aggregatorResponseDTO.setHttpStatusCodeValue(statusCodeValue);
            aggregatorResponseDTO.setHttpStatusCode(httpStatusCode);
            aggregatorResponseDTO.setResponse(responseBody);
        }

        long duration = (System.nanoTime() - start) / 1_000_000;
        logger.info("Task end thread: " + Thread.currentThread().getName() + ", duration: " + duration);

        return CompletableFuture.completedFuture(aggregatorResponseDTO);
    }
}
```

**3. Aggregator Service Implementation using Async Service and Completable Future**
```java
@Service
public class AggregatorServiceImpl implements AggregatorService {
    private static final CLogger logger = LogFactory.getCLogger(AggregatorServiceImpl.class.getName());

    private final AsyncService asyncService;

    public AggregatorServiceImpl(AsyncService asyncService) {
        this.asyncService = asyncService;
    }

    @Override
    public AggregatorResponseListDTO getAggregatorResponseList(List<AggregatorRequestDTO> aggregatorRequestDTOList, String authorizationToken) {
        if (ObjectUtils.isEmpty(aggregatorRequestDTOList)) {
            logger.info("AggregatorRequestListDTO is empty or null");
            return null;
        }

        logger.info("Main start thread: " + Thread.currentThread().getName());
        long start = System.nanoTime();

        List<CompletableFuture<AggregatorResponseDTO>> aggregatorResponseDTOFutures
                = aggregatorRequestDTOList
                    .stream()
                    .map(i -> asyncService.asyncAggregatorRequest(i, authorizationToken))
                    .collect(Collectors.toList());

        CompletableFuture<Void> allAggregatorResponseDTOFutures
                = CompletableFuture.allOf(
                        aggregatorResponseDTOFutures.toArray(new CompletableFuture[aggregatorResponseDTOFutures.size()])
        );

        CompletableFuture<AggregatorResponseListDTO> aggregatorResponseListDTOFuture
                = allAggregatorResponseDTOFutures.thenApply(v ->
                        aggregatorResponseDTOFutures
                                .stream()
                                .map(CompletableFuture::join)
                                .collect(Collectors.toList())
                )
                .thenApply(v -> {
                    AggregatorResponseListDTO aggregatorResponseListDTO = new AggregatorResponseListDTO();
                    aggregatorResponseListDTO.setAggregatorResponseDTOList(v);

                    return aggregatorResponseListDTO;
        });

        AggregatorResponseListDTO aggregatorResponseListDTO = aggregatorResponseListDTOFuture.join();

        long duration = (System.nanoTime() - start) / 1_000_000;
        logger.info("Main end thread: " + Thread.currentThread().getName() + ", Execution time in milliseconds nanoTime: " + duration);

        return aggregatorResponseListDTO;
    }
}
```