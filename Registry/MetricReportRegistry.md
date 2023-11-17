## Metric Report Registry

**1. Metric Report Interface**
```java
public interface MetricReport {
}
```

**2. Metric Report Set Interface**
```java
public interface MetricReportSet extends MetricReport {

    Map<String, MetricReport> getMetricReports();
}
```

**3. Metric Report Registry Interface**
```java
public interface MetricReportRegistry {

    <T extends MetricReport> T register(String name, T metricReport) throws IllegalArgumentException;

    void registerAll(MetricReportSet metricReports) throws IllegalArgumentException;

    void registerAll(String prefix, MetricReportSet metricReports) throws IllegalArgumentException;

    <T extends MetricReport> T metricReport(String name);
}
```

**4. Metric Report Registry Implementation with Report Builder**
```java
@Component
public class MetricReportRegistryImpl implements MetricReportSet, MetricReportRegistry {

    public static String name(String name, String... names) {
        final StringBuilder builder = new StringBuilder();
        append(builder, name);
        if (names != null) {
            for (String s : names) {
                append(builder, s);
            }
        }
        return builder.toString();
    }

    public static String name(Class<?> klass, String... names) {
        return name(klass.getName(), names);
    }

    private static void append(StringBuilder builder, String part) {
        if (part != null && !part.isEmpty()) {
            if (builder.length() > 0) {
                builder.append('.');
            }
            builder.append(part);
        }
    }

    private final ConcurrentMap<String, MetricReport> metricReports;

    public MetricReportRegistryImpl() {
        this.metricReports = buildMap();
    }

    protected ConcurrentMap<String, MetricReport> buildMap() {
            return new ConcurrentHashMap<>();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends MetricReport> T register(String name, T metricReport) throws IllegalArgumentException {
        if (metricReport instanceof MetricReportSet) {
            registerAll(name, (MetricReportSet) metricReport);
        } else {
            final MetricReport existing = metricReports.putIfAbsent(name, metricReport);
            if (existing != null) {
                throw new IllegalArgumentException("A metric named " + name + " already exists");
            }
        }
        return metricReport;
    }

    @Override
    public void registerAll(MetricReportSet metricReports) throws IllegalArgumentException {
        registerAll(null, metricReports);
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends MetricReport> T metricReport(String name) {
        switch (name) {
            case FULFILLMENT_METRICREPORT:
                return (T) getOrAdd(name, MetricReportBuilder.FULFILLMENTMETRICREPORT);

            default:
                throw new IllegalArgumentException(name + " is not supported metric report");
        }
    }

    @SuppressWarnings("unchecked")
    private <T extends MetricReport> T getOrAdd(String name, MetricReportBuilder<T> builder) {
        final MetricReport metricReport = metricReports.get(name);
        if (builder.isInstance(metricReport)) {
            return (T) metricReport;
        } else if (metricReport == null) {
            try {
                return register(name, builder.newMetricReport());
            } catch (IllegalArgumentException e) {
                final MetricReport added = metricReports.get(name);
                if (builder.isInstance(added)) {
                    return (T) added;
                }
            }
        }
        throw new IllegalArgumentException(name + " is already used for a different type of metric report");
    }

    @Override
    public void registerAll(String prefix, MetricReportSet metricReports) throws IllegalArgumentException {
        metricReports.getMetricReports().forEach((k, v) -> {
            if (v instanceof MetricReportSet) {
                registerAll(name(prefix, k), (MetricReportSet) v);
            } else {
                register(name(prefix, k), v);
            }
        });
    }

    @Override
    public Map<String, MetricReport> getMetricReports() {
        return Collections.unmodifiableMap(metricReports);
    }

    private interface MetricReportBuilder<T extends MetricReport> {
        MetricReportBuilder<FulfillmentMetricReport> FULFILLMENTMETRICREPORT
                = new MetricReportBuilder<FulfillmentMetricReport>() {

            @Override
            public FulfillmentMetricReport newMetricReport() {
                return new FulfillmentMetricReport();
            }

            @Override
            public boolean isInstance(MetricReport metricReport) {
                return FulfillmentMetricReport.class.isInstance(metricReport);
            }
        };

        T newMetricReport();

        boolean isInstance(MetricReport metricReport);
    }
}
```