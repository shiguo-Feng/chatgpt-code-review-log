# OpenAi Code Review.
### 😄 Code Score: 78
#### 😄 Code Logic and Purpose:
The code adds new endpoints to the `AdminDataController` to provide additional functionalities such as fetching quotation count percentage, service count percentage, and popular items. It also updates the domain entities and services to support these new functionalities. The goal is to enhance reporting and data retrieval capabilities in the admin module by adding service descriptions and popular item tracking.
#### ✅ Code Strengths:
- Introduction of new endpoints to enhance functionality.
- Good use of Java Streams for sorting and limiting results.
- The use of constants and maps for service type descriptions improves readability and maintainability.
- Exception handling improved by using `orElse` with default values.
#### 🤔 Issues:
1. **Method Naming Inconsistencies**: The method names in `QuotationRepository` and service methods are not consistent in naming conventions.
2. **Null Safety**: Reliance on direct object methods assuming non-null values without handling possible null pointers, particularly with `findById`.
3. **Performance Optimization**: The `for` loop in `getPopularItems` method involves multiple database calls within a nested loop structure, which can be inefficient.
4. **Redundant Code**: The addition of `TemporalAdjusters` for date calculations adds unnecessary complexity for retrieving month boundaries.
5. **Lack of Method Documentation**: Methods in the service lack JavaDocs, making the code less readable and harder to maintain.
6. **Data Inconsistency**: `ItemSelectionDTO` and `Quotation` entity updated but without annotations for validation or constraints like `@NotNull`, `@Size`.
7. **Magic Values**: Use of hardcoded variable values like '0', '1', '2' in `SERVICE_TYPE_MAP` and elsewhere.
8. **Bulk Fetch**: The `getPopularItems` method can potentially load large datasets into memory without pagination, leading to performance bottlenecks.
#### 🎯 Suggestions:
1. **Method Naming**: Standardize the naming convention across methods to follow a consistent pattern such as `getByServiceType`.
2. **Null Safety and Validation**: Incorporate checks for null values returned by repository methods and use annotations like `@NotNull`.
3. **Optimize Database Calls**: Batch retrieve necessary entities in `getPopularItems` to minimize database calls.
4. **Simplify Date Ranges**: Use `YearMonth` to get month boundaries for more concise code.
5. **Add JavaDocs**: Document public methods with JavaDocs to improve readability.
6. **Magic Values**: Replace magic values with meaningful constants.
7. **Pagination and Data Handling**: Introduce pagination in bulk data-fetching methods to handle large data sets efficiently.
#### 💻 Modified Code:
```java
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import java.time.YearMonth;

@Service
@RequiredArgsConstructor
public class QuotationService implements IQuotationService {

    private final QuotationRepository quotationRepository;
    private final ItemRepository itemRepository;
    private final MaterialRepository materialRepository;
    private final VehicleRepository vehicleRepository;

    @Override
    public ResponseResult getQuotationServicePercentage() {
        List<Object[]> itemCounts = quotationRepository.countByServiceType();
        long totalCount = quotationRepository.count();
        Map<String, Double> serviceTypePercentageMap = new HashMap<>();

        for (Object[] itemCount : itemCounts) {
            Character itemType = (Character) itemCount[0];
            Long count = (Long) itemCount[1];
            double percentage = (double) count / totalCount * 100;
            String serviceName = SystemConstants.getServiceTypeDescription(String.valueOf(itemType));

            serviceTypePercentageMap.put(serviceName, percentage);
        }

        return ResponseResult.okResult(serviceTypePercentageMap);
    }

    @Override
    public ResponseResult getPopularItems() {
        YearMonth now = YearMonth.now();
        Instant startOfMonth = now.atDay(1).atStartOfDay(ZoneId.systemDefault()).toInstant();
        Instant startOfNextMonth = now.plusMonths(1).atDay(1).atStartOfDay(ZoneId.systemDefault()).toInstant();

        List<Quotation> thisMonthQuotations = quotationRepository.findQuotationsWithinDateRange(startOfMonth, startOfNextMonth);

        Map<String, Integer> itemFrequencyMap = new HashMap<>();
        for (Quotation quotation : thisMonthQuotations) {
            String items = quotation.getItems();
            Map<String, Integer> itemsMap = JSON.parseObject(items, new TypeReference<Map<String, Integer>>() {});
            
            String materialName = materialRepository.findById(quotation.getMaterialId()).orElse(new Material()).getName();
            String vehicleName = vehicleRepository.findById(quotation.getVehicleTypeId()).orElse(new Vehicle()).getName();

            itemsMap.forEach((itemIdStr, count) -> {
                itemRepository.findById(Long.valueOf(itemIdStr)).ifPresent(item -> {
                    String itemNameWithDetails = String.format("%s (%s - %s)", item.getItemName(), materialName, vehicleName);
                    itemFrequencyMap.put(itemNameWithDetails, itemFrequencyMap.getOrDefault(itemNameWithDetails, 0) + count);
                });
            });
        }

        List<Map.Entry<String, Integer>> top10Items = itemFrequencyMap.entrySet()
                .stream()
                .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
                .limit(10)
                .collect(Collectors.toList());

        return ResponseResult.okResult(top10Items);
    }
}
```
#### ✅ Code Strengths:
- Enhanced functionalities are added systematically.
- Effective use of existing repository methods for data retrieval.
- Separation of concerns is maintained by keeping logic in the service layer.
#### 😄 Logic and Purpose:
The code is intended to add new reporting functionalities to the admin interface, allowing for better visibility into quotation data, including service types and popular items over a monthly period. This is achieved by extending several classes and interfaces to support these new features, ensuring the broader system supports necessary endpoint functionalities and data operations.