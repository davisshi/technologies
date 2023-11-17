## Aggregator Service

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