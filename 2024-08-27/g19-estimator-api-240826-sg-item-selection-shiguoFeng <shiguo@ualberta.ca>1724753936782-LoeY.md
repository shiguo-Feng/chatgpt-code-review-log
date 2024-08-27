# OpenAi Code Review.
### 😄 Code Score: 80
#### 😄 Code Logic and Purpose:
The changes across the provided files mainly focus on enhancing the application's ability to handle data types, improving type accuracy, and optimizing handling of item and package operations. Specifically, it modifies IDs from `Integer` to `Long`, adjusts DTOs, repository methods, and utilities to reflect these type changes. Additionally, the test code is updated to reflect these improvements. These changes are likely to provide better scalability and prevent potential overflow issues when handling identifiers.

#### ✅ Code Strengths:
1. **Type Safety Improvements:** Changing from `Integer` to `Long` ensures better support for larger datasets.
2. **Enhanced API Usability:** The update from `@GetMapping` to `@PostMapping` and the addition of `@RequestBody` annotation improve adherence to RESTful principles.
3. **Reflection of Real-World Scenarios:** Improved mapping of item and package data, enhancing the logical structure of data interactions.
4. **Streamlined Test Cases:** Comprehensive test cases ensure new logic's robustness.

#### 🤔 Issues:
1. **Potential Code Clutter:** Commenting out the `packageItemCache.loadPackage();` and leaving it commented can create confusion.
2. **Lack of Exception Handling:** Methods like `findById` need proper exception handling to cater to cases where the item is not found.
3. **Hard-coded Values:** File path in `dataService.importData` is hard-coded.
4. **Comments Usage:** Some comments add little value or are redundant.
5. **Incomplete Methods:** `itemList` method has a `TODO` which is not addressed, indicating incomplete implementation.
6. **Inefficient Stream Usage:** While streams in `PackageItemCache` and `ItemService` are generally well-applied, they could benefit from intermediate results caching or simplification for clarity.

#### 🎯 Suggestions:
1. **Remove Commented Code:** Eliminate or address the commented `packageItemCache.loadPackage();`.
2. **Exception Handling:** Implement custom exceptions or handling mechanisms for cases where data is not found.
3. **Configuration File Paths:** Replace hard-coded file paths with configuration entries to improve maintainability.
4. **Improve Comments:** Update comments to be more descriptive or remove redundant ones.
5. **Complete TODOs:** Ensure all `TODO` items are completed before committing.
6. **Streamline Stream Operations:** Cache intermediate results in streams where beneficial and ensure clarity of stream logic.

#### 💻 Modified Code:
```java
diff --git a/estimator/src/main/java/com/g19graphics/controller/ItemController.java b/estimator/src/main/java/com/g19graphics/controller/ItemController.java
index daa2e05..3cf66c0 100644
--- a/estimator/src/main/java/com/g19graphics/controller/ItemController.java
+++ b/estimator/src/main/java/com/g19graphics/controller/ItemController.java
@@ -22,8 +22,10 @@ public class ItemController {
     private final IDataImportService dataService;
     private final IItemService itemService;
 
+    @Value("${data.import.file.path}")
+    private String importFilePath;
+
     @PostMapping("/import")
     public ResponseResult testImport(@RequestBody ImportDataRequestDTO requestDTO) {
         //TODO: migrate this to use S3
-        dataService.importData("/Users/shiguofeng/Desktop/G19Api/docs/2024 quoting sheet.xlsx", requestDTO);
+        dataService.importData(importFilePath, requestDTO);
         return ResponseResult.okResult();
     }
 }
diff --git a/estimator/src/main/java/com/g19graphics/runner/CacheRunner.java b/estimator/src/main/java/com/g19graphics/runner/CacheRunner.java
index 972cac6..ae7b9d6 100644
--- a/estimator/src/main/java/com/g19graphics/runner/CacheRunner.java
+++ b/estimator/src/main/java/com/g19graphics/runner/CacheRunner.java
@@ -18,6 +18,8 @@ public class CacheRunner implements CommandLineRunner {
 
     @Override
     public void run(String... args) throws Exception {
         packageItemCache.loadPackage();
     }
 }
diff --git a/framework/src/main/java/com/g19graphics/domain/dto/ImportDataRequestDTO.java b/framework/src/main/java/com/g19graphics/domain/dto/ImportDataRequestDTO.java
index fc4436a..e173f1a 100644
--- a/framework/src/main/java/com/g19graphics/domain/dto/ImportDataRequestDTO.java
+++ b/framework/src/main/java/com/g19graphics/domain/dto/ImportDataRequestDTO.java
@@ -10,6 +10,6 @@ import lombok.Data;
 
 @Data
 public class ImportDataRequestDTO {
     private Integer materialStartIndex;
     private Integer materialEndIndex;
 }
diff --git a/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java b/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
index 72eafc4..501cedc 100644
--- a/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
+++ b/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
@@ -13,5 +13,6 @@ import java.util.Map;
 public class ItemSelectionDTO {
     private Long vehicleId;
     private Long materialId;
     // key: id, value: quantity
     private Map<Long, Integer> itemQuantities;
 }
diff --git a/framework/src/main/java/com/g19graphics/domain/entity/Item.java b/framework/src/main/java/com/g19graphics/domain/entity/Item.java
index 8512022..a09bfbc 100644
--- a/framework/src/main/java/com/g19graphics/domain/entity/Item.java
+++ b/framework/src/main/java/com/g19graphics/domain/entity/Item.java
@@ -21,7 +21,7 @@ public class Item extends Auditable {
     @Id
     @GeneratedValue(strategy = GenerationType.IDENTITY)
     @Column(name = "id", nullable = false)
     private Long id;
 
     @Column(name = "item_name", nullable = false)
     private String itemName;
 
@@ -36,13 +36,13 @@ public class Item extends Auditable {
     private Character serviceType;
 
     @Column(name = "vehicle_type_id")
     private Long vehicleTypeId;
 
     @Column(name = "display_section")
     private String displaySection;
 
     @Column(name = "material_id")
     private Long materialId;
 
     @Column(name = "price", precision = 10, scale = 2)
diff --git a/framework/src/main/java/com/g19graphics/domain/entity/Material.java b/framework/src/main/java/com/g19graphics/domain/entity/Material.java
index 5d477b4..c410e01 100644
--- a/framework/src/main/java/com/g19graphics/domain/entity/Material.java
+++ b/framework/src/main/java/com/g19graphics/domain/entity/Material.java
@@ -5,7 +5,6 @@ import lombok.Getter;
 import lombok.Setter;
 
 import javax.persistence.*;
 
 @Getter
 @Setter
 public class Material extends Auditable {
     @Id
     @GeneratedValue(strategy = GenerationType.IDENTITY)
     @Column(name = "id", nullable = false)
     private Long id;
 
     @Column(name = "name", nullable = false)
     private String name;
diff --git a/framework/src/main/java/com/g19graphics/domain/entity/Vehicle.java b/framework/src/main/java/com/g19graphics/domain/entity/Vehicle.java
index 9eda938..1476fc8 100644
--- a/framework/src/main/java/com/g19graphics/domain/entity/Vehicle.java
+++ b/framework/src/main/java/com/g19graphics/domain/entity/Vehicle.java
@@ -15,7 +15,7 @@ public class Vehicle extends Auditable {
     @Id
     @GeneratedValue(strategy = GenerationType.IDENTITY)
     @Column(name = "id", nullable = false)
     private Long id;
 
     @Column(name = "name", nullable = false)
     private String name;
diff --git a/framework/src/main/java/com/g19graphics/repositories/ItemRepository.java b/framework/src/main/java/com/g19graphics/repositories/ItemRepository.java
index d3b0d72..915064d 100644
--- a/framework/src/main/java/com/g19graphics/repositories/ItemRepository.java
+++ b/framework/src/main/java/com/g19graphics/repositories/ItemRepository.java
@@ -14,11 +14,11 @@ import java.util.List;
 @Repository
 public interface ItemRepository extends JpaRepository<Item, Long> {
 
     Item findByItemNameAndMaterialIdAndVehicleTypeId(String itemName, Long materialId, Long vehicleId);
 
     List<Item> findByVehicleTypeIdAndMaterialIdAndItemTypeAndServiceType(
             Long vehicleId,
             Long materialId,
             Character itemType,
             Character serviceType
     );
diff --git a/framework/src/main/java/com/g19graphics/service/cache/PackageItemCache.java b/framework/src/main/java/com/g19graphics/service/cache/PackageItemCache.java
index 1f08f0d..03da739 100644
--- a/framework/src/main/java/com/g19graphics/service/cache/PackageItemCache.java
+++ b/framework/src/main/java/com/g19graphics/service/cache/PackageItemCache.java
@@ -47,7 +47,7 @@ public class PackageItemCache {
                         SystemConstants.ITEM_TYPE_PACKAGE,
                         SystemConstants.SERVICE_TYPE_PPF
                 );
                 Map<Long, List<Long>> packageMap = packages.stream()
                         .collect(Collectors.toMap(
                                 // Key: The package ID
                                 Item::getId,
@@ -62,7 +62,7 @@ public class PackageItemCache {
         }
     }
 
     private String generateRedisKey(Long vehicleId, Long materialId) {
         return SystemConstants.REDIS_PACKAGE_KEY + vehicleId + ":" + materialId;
     }
 }
diff --git a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
index bdc8d54..d990ee4 100644
--- a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
+++ b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
@@ -13,12 +13,15 @@ import com.g19graphics.enums.AppHttpCodeEnum;
 import com.g19graphics.exception.SystemException;
 import com.g19graphics.repositories.ItemRepository;
 import com.g19graphics.service.IItemService;
 import com.g19graphics.utility.BeanCopyUtils;
 import com.g19graphics.utility.PackageItemUtil;
 import com.g19graphics.utility.RedisCache;
 import lombok.RequiredArgsConstructor;
 import org.springframework.stereotype.Service;
 import org.springframework.util.StringUtils;
 
 import java.math.BigDecimal;
 import java.util.ArrayList;
 import java.util.List;
 import java.util.Map;
 import java.util.stream.Collectors;
 
 @Service
 @RequiredArgsConstructor
 public class ItemService implements IItemService {
 
     private final ItemRepository itemRepository;
     private final RedisCache redisCache;
 
     @Override
     public List<PackageVO> loadPackageList(Long vehicleId, Long materialId, Character serviceType) {
         List<PackageVO> vos = new ArrayList<>();
 
         // 1. Attempt to get package details from the Redis cache
         String cacheKey = generateRedisKey(vehicleId, materialId);
         String cachedData = redisCache.getCacheObject(cacheKey);
         if (StringUtils.hasText(cachedData)) {
             // Parse the cached data and convert it to a List of ItemVOs
             vos = JSON.parseObject(cachedData, new TypeReference<>() {});
         }
 
         // 2. Query from DB if cache is empty or not available
         List<Item> packageItems = itemRepository.findByVehicleTypeIdAndMaterialIdAndItemTypeAndServiceType(
                 vehicleId,
                 materialId,
                 SystemConstants.ITEM_TYPE_PACKAGE,
                 serviceType
         );
 
         for (Item item : packageItems) {
             PackageVO packageVO = BeanCopyUtils.copyBean(item, PackageVO.class);
             List<ItemVO> packageDetails = item.getIncludedItems().stream()
                     .map(pakageItem -> new ItemVO(pakageItem.getId().intValue(), pakageItem.getItemName(), pakageItem.getServiceType(), pakageItem.getDisplaySection()))
                     .collect(Collectors.toList());
             packageVO.setIncludedItems(packageDetails);
             vos.add(packageVO);
         }
         return vos;
     }
 
     @Override
     public ResponseResult itemList(ItemRequestDTO requestDTO) {
         validateParameters(requestDTO.getVehicleId(), requestDTO.getMaterialId(), requestDTO.getServiceType());
         List<Item> itemList = itemRepository.findByVehicleTypeIdAndMaterialIdAndItemTypeAndServiceType(
                 requestDTO.getVehicleId(),
                 requestDTO.getMaterialId(),
                 SystemConstants.ITEM_TYPE_ITEM,
                 requestDTO.getServiceType()
         );
         return ResponseResult.okResult(itemList);
     }
 
     @Override
     public ResponseResult createQuotation(QuotationRequestDTO requestDTO) {
         // 1. Get package list from redis
         Object cachedData = redisCache.getCacheObject(SystemConstants.REDIS_PACKAGE_KEY + requestDTO.getVehicleId() + ":" + requestDTO.getMaterialId());
         Map<Long, List<Long>> packageMap = JSON.parseObject((String) cachedData, new TypeReference<>() {
         });
 
         // 2. Replace itemIds with package if applicable
         List<Long> replaced = PackageItemUtil.replaceItemWithPackage(requestDTO.getItemQuantities(), packageMap);
         Map<Long, Integer> finalQuantities = replaced.stream()
                 .collect(Collectors.toMap(
                         itemId -> itemId, // Key: The item ID (or package ID)
                         itemId -> requestDTO.getItemQuantities().getOrDefault(itemId, 1) // Value: Original quantity or 1 if not found
                 ));
         // 3. Calculate total price using Stream
         BigDecimal total = finalQuantities.entrySet().stream()
                 .map(entry -> itemRepository.findById(entry.getKey().longValue()) // Find the item by ID
                         .map(item -> item.getPrice().multiply(BigDecimal.valueOf(entry.getValue()))) // Multiply price with quantity
                         .orElse(BigDecimal.ZERO)) // Default to BigDecimal.ZERO if the item is not found
                 .reduce(BigDecimal.ZERO, BigDecimal::add); // Sum up all the prices
         // 5. create && save quotation
         // TODO: use observer pattern to send email
         // 6. return quotation vo
         return ResponseResult.okResult(total);
     }
 
     private void validateParameters(Object... params) {
        for(Object param : params){
            if (param == null) {
                throw new IllegalArgumentException("Parameter cannot be null");
            }
        }
     }
     
     private String generateRedisKey(Long vehicleId, Long materialId) {
         return SystemConstants.REDIS_PACKAGE_KEY + vehicleId + ":" + materialId;
     }
 }
diff --git a/framework/src/main/java/com/g19graphics/utility/PackageItemUtil.java b/framework/src/main/java/com/g19graphics/utility/PackageItemUtil.java
index 9a6f0a5..554ae41 100644
--- a/framework/src/main/java/com/g19graphics/utility/PackageItemUtil.java
+++ b/framework/src/main/java/com/g19graphics/utility/PackageItemUtil.java
@@ -12,23 +12,26 @@ import java.util.stream.Collectors;
  */
 public class PackageItemUtil {
     /**
      * Replaces fully matched subsets in the given item map with their corresponding package IDs.
      * Continues this process until no more complete matches are found.
      *
      * @param itemQuantities A map where each key is an item ID and the value is its quantity.
      * @param packageMap     A map where each key is a package ID and the value is a list of item IDs
      *                       that constitute the package.
      * @return The updated list containing the package IDs for fully matched subsets and
      * any unmatched item IDs that remain after processing.
      */
     public static List<Long> replaceItemWithPackage(Map<Long, Integer> itemQuantities, Map<Long, List<Long>> packageMap) {
         Set<Long> arrSet = new HashSet<>(itemQuantities.keySet());
         List<Long> result = new ArrayList<>();
 
         // Continuously attempt to match until no more full matches are found
         while (true) {
             final Set<Long> currentArrSet = arrSet; // Capture current state for lambda usage
 
             // Use Stream to filter and collect fully contained subsets into matchesMap
             Map<Long, Set<Long>> matchesMap = packageMap.entrySet().stream()
                     .filter(entry -> !entry.getValue().isEmpty()) // Exclude empty subsets
                     .filter(entry -> currentArrSet.containsAll(entry.getValue())) // Check if arrSet contains all elements of each subset
                     .collect(Collectors.toMap(
                             Map.Entry::getKey, // Key: Package ID
                             e -> new HashSet<>(e.getValue()) // Value: Copy of the subset values as a Set
                     ));
 
             // If there are matches, select the longest one
             if (!matchesMap.isEmpty()) {
                 // Find the entry with the largest value set
                 Map.Entry<Long, Set<Long>> bestMatch = Collections.max(matchesMap.entrySet(),
                         Comparator.comparingInt(e -> e.getValue().size()));
 
                 // Remove elements of the best match from arrSet
                 arrSet.removeAll(bestMatch.getValue());
                 // Add the package ID to the result
                 result.add(bestMatch.getKey());
             } else {
diff --git a/framework/src/test/java/com/g19gpraphics/ApiTest.java b/framework/src/test/java/com/g19gpraphics/ApiTest.java
index 62cbc07..a103fb1 100644
--- a/framework/src/test/java/com/g19gpraphics/ApiTest.java
+++ b/framework/src/test/java/com/g19gpraphics/ApiTest.java
@@ -3,10 +3,7 @@ package com.g19gpraphics;
 import com.g19graphics.utility.PackageItemUtil;
 import org.junit.jupiter.api.Test;
 
 import java.util.*;
 
 public class ApiTest {
     @Test
     public void testPackageReplace() {
         Map<Long, Integer> itemQuantities1 = new HashMap<>();
         itemQuantities1.put(13L, 1);
         itemQuantities1.put(16L, 1);
         itemQuantities1.put(28L, 1);
         itemQuantities1.put(31L, 1);
         itemQuantities1.put(40L, 1);
         itemQuantities1.put(19L, 1);
         itemQuantities1.put(22L, 1);
         itemQuantities1.put(37L, 1);
         itemQuantities1.put(34L, 1);
 
         Map<Long, List<Long>> packageMap1 = new HashMap<>();
         packageMap1.put(1L, Arrays.asList(13L, 16L, 28L, 31L, 40L)); // Package ID 1 with included items
         package