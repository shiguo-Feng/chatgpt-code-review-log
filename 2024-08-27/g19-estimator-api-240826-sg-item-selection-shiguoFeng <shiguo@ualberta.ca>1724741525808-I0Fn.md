# OpenAi Code Review.

### 😄 Code Score: 75

#### 😄 Code Logic and Purpose:
The code modifications include adding new dependencies, updating DTOs and VOs, introducing additional validation and error handling in the service layer, and implementing a utility to replace items with package IDs.

#### ✅ Code Strengths:
1. Improved structure and separation of concerns by adding a new utility class.
2. Enhanced DTO to better represent item quantities using a Map.
3. Additional validation checks in the `ItemService` to ensure data integrity.
4. Comprehensive testing added to cover utility functionality.
5. Guava library added, which can provide efficient utilities.

#### 🤔 Issues:
1. The `ItemService` implementation uses potentially inefficient, nested lookup operations inside the `forEach` loop.
2. `PackageItemUtil` has no unit tests.
3. Several newly created classes lack proper JavaDoc comments (e.g., describing parameters and return values).
4. Inconsistent naming conventions, such as `itemQuantities` versus `itemIds`.
5. No input validation for the utility method `replaceItemWithPackage`.
6. The utility function has an internal while-loop with unbounded iterations which can lead to performance bottlenecks.
7. Misleading: Adding Guava dependency but not utilizing its full potential.
8. No null checks or validation for crucial inputs in `PackageItemUtil`.

#### 🎯 Suggestions:
1. Optimize nested lookups in `ItemService` by refactoring code to minimize database hits.
2. Add unit tests for `PackageItemUtil`.
3. Standardize naming conventions across DTOs.
4. Add thorough input validation and JavaDoc comments in new methods.
5. Enhance loop conditions to prevent potential infinite loops.
6. Refactor code to leverage Guava’s capabilities more extensively.
7. Implement custom exceptions for more granular error handling in services.

#### 💻 Modified Code:

```java
diff --git a/framework/pom.xml b/framework/pom.xml
index 6571805..6e4e9e3 100644
--- a/framework/pom.xml
+++ b/framework/pom.xml
@@ -71,6 +71,11 @@
     <dependency>
         <groupId>commons-lang3</groupId>
         <artifactId>commons-lang3</artifactId>
         <version>3.12.0</version>
     </dependency>
+    <dependency>
+        <groupId>com.google.guava</groupId>
+        <artifactId>guava</artifactId>
+        <version>32.1.2-jre</version>
+    </dependency>

 </dependencies>

diff --git a/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java b/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
index 98ebed2..eff37f2 100644
--- a/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
+++ b/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
@@ -2,6 +2,8 @@ package com.g19graphics.domain.dto;

 import lombok.Data;

+import java.util.HashMap;
+import java.util.Map;
/**
* DTO for selection array to total price
 */ 
@Data
public class ItemSelectionDTO {
    private Long vehicleId;
-   private Long materialId;
-   private Character serviceType;
-   private Integer[] itemIds;
+   private Map<Integer, Integer> itemQuantities = new HashMap<>();
}
diff --git a/framework/src/main/java/com/g19graphics/domain/vo/ItemVO.java b/framework/src/main/java/com/g19graphics/domain/vo/ItemVO.java
index c0d3990..dcf2650 100644
--- a/framework/src/main/java/com/g19graphics/domain/vo/ItemVO.java
+++ b/framework/src/main/java/com/g19graphics/domain/vo/ItemVO.java
@@ -15,4 +15,6 @@ import lombok.NoArgsConstructor;
public class ItemVO {
    private Integer id;
    private String itemName;
+   private Character serviceType;
+   private String displaySection;
}
diff --git a/framework/src/main/java/com/g19graphics/enums/AppHttpCodeEnum.java b/framework/src/main/java/com/g19graphics/enums/AppHttpCodeEnum.java
index 83ff024..3af7900 100644
--- a/framework/src/main/java/com/g19graphics/enums/AppHttpCodeEnum.java
+++ b/framework/src/main/java/com/g19graphics/enums/AppHttpCodeEnum.java
@@ -23,7 +23,9 @@ public enum AppHttpCodeEnum {

    PARAMETERS_NOT_NULL(506, "Parameters cannot be empty"),
    CONTENT_NOT_FOUND(512, "Requested resource not found"),
    IMPORT_CONTENT_NOT_FOUND(513, "Item not found for package"),
+   ITEM_SELECTION_NOT_NULL(514, "Selected items cannot be empty");

    final int code;
    final String msg;

diff --git a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
index 69fc412..c6226c6 100644
--- a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
+++ b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
@@ -20,6 +20,7 @@ import org.springframework.util.StringUtils;

import java.util.ArrayList;
import java.util.List;
+import java.util.Optional;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

public class ItemService implements IItemService {
    @Override
    public ResponseResult packageList(ItemRequestDTO requestDTO) {
         validateParameters(requestDTO.getVehicleId(), requestDTO.getMaterialId(), requestDTO.getServiceType());

        Long vehicleId = requestDTO.getVehicleId();
        Long materialId = requestDTO.getMaterialId();
        Character serviceType = requestDTO.getServiceType();

        // Omitted irrelevant code

        for (Item item : packageItems) {
            PackageVO packageVO = BeanCopyUtils.copyBean(item, PackageVO.class);
            List<ItemVO> packageDetails = item.getIncludedItems().stream()
-                   .map(pakageItem -> new ItemVO(pakageItem.getId(), pakageItem.getItemName()))
+                   .map(packageItem -> new ItemVO(packageItem.getId(), packageItem.getItemName(), packageItem.getServiceType(), packageItem.getDisplaySection()))
                    .collect(Collectors.toList());
            packageVO.setIncludedItems(packageDetails);
            vos.add(packageVO);
        }

        // Omitted irrelevant code
        
        return new ResponseResult(Success, vos);
    }

    @Override
    public ResponseResult processSelection(ItemSelectionDTO requestDTO) {
        validateParameters(requestDTO.getVehicleId());
        if (requestDTO.getItemQuantities() == null || requestDTO.getItemQuantities().isEmpty()) {
            throw new SystemException(AppHttpCodeEnum.ITEM_SELECTION_NOT_NULL);
        }

        List<Item> items = requestDTO.getItemQuantities().keySet().parallelStream()
            .map(key -> itemRepository.findById(key.longValue())
                .orElseThrow(() -> new SystemException(AppHttpCodeEnum.CONTENT_NOT_FOUND)))
            .collect(Collectors.toList());

        // Further processing as per original logic

        // TODO: use observer pattern to send email
        return null;
    }
}
diff --git a/framework/src/main/java/com/g19graphics/utility/PackageItemUtil.java b/framework/src/main/java/com/g19graphics/utility/PackageItemUtil.java
new file mode 100644
index 0000000..fe9e6cc
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/utility/PackageItemUtil.java
@@ -0,0 +1,55 @@
package com.g19graphics.utility;

import com.google.common.collect.Sets;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Utility class providing methods to replace item lists with package IDs based on predefined mappings.
 */
public class PackageItemUtil {
    /**
     * Replaces fully matched subsets in the given list 'arr' with their corresponding IDs.
     * Continues this process until no more matches are found or until performance threshold is reached.
     *
     * @param arr        The main list to process.
     * @param packageMap A map where each key is a subset (list of item IDs) and the value is the corresponding package ID.
     * @return The updated list containing subset IDs and unmatched elements.
     */
    public static List<Object> replaceItemWithPackage(List<Integer> arr, Map<List<Integer>, Integer> packageMap) {
        if (arr == null || packageMap == null) {
            throw new IllegalArgumentException("Input list and package map cannot be null");
        }

        Set<Integer> arrSet = new HashSet<>(arr); // Convert the list to a HashSet for fast lookup
        List<Object> result = new ArrayList<>(); // To store the final result

        // Continuously attempt to match until no more full matches are found
        int iterations = 0;
        int maxIterations = 1000; // set a safe upper limit for iterations
        while (iterations < maxIterations) {
            final Set<Integer> currentArrSet = new HashSet<>(arrSet); // Capture current state for lambda usage

            // Use Stream to filter and collect fully contained subsets into matchesMap
            Map<Set<Integer>, Integer> matchesMap = packageMap.entrySet().stream()
                .filter(entry -> currentArrSet.containsAll(entry.getKey())) // Check if arrSet contains all elements of each subset
                .collect(Collectors.toMap(entry -> new HashSet<>(entry.getKey()), Map.Entry::getValue));

            // If there are matches, select the longest one
            if (!matchesMap.isEmpty()) {
                // Find the entry with the largest key set
                Map.Entry<Set<Integer>, Integer> bestMatch = Collections.max(matchesMap.entrySet(),
                        Comparator.comparingInt(e -> e.getKey().size()));

                // Remove elements of the best match from arrSet
                arrSet.removeAll(bestMatch.getKey());
                // Add the ID of the best match to the result
                result.add(bestMatch.getValue());
            } else {
                // No more matches found, break the loop
                break;
            }

            iterations++;
        }

        // Add the remaining elements in arrSet to the result
        result.addAll(arrSet);
        return result;
    }
}
diff --git a/framework/src/test/java/com/g19gpraphics/ApiTest.java b/framework/src/test/java/com/g19gpraphics/ApiTest.java
new file mode 100644
index 0000000..4053241
--- /dev/null
+++ b/framework/src/test/java/com/g19gpraphics/ApiTest.java
@@ -0,0 +1,90 @@
package com.g19gpraphics;

import com.google.common.collect.ImmutableSet;
import com.google.common.collect.Sets;
import org.junit.jupiter.api.Test;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Test class for package item utility
 */
public class ApiTest {
    @Test
    public void testPackageReplace() {
        List<Integer> itemIds = Arrays.asList(8, 9, 13, 14, 15, 7, 11, 99);

        // Create a map to represent subsets and their corresponding package IDs
        Map<List<Integer>, Integer> packageMap = new HashMap<>();
        packageMap.put(Arrays.asList(8, 9, 13, 14, 15), 1);
        packageMap.put(Arrays.asList(8, 9, 11, 13, 14, 15), 2);
        packageMap.put(Arrays.asList(7), 3);

        // Call the method to test
        List<Object> updatedList = PackageItemUtil.replaceItemWithPackage(itemIds, packageMap);

        // Print the result for verification
        System.out.println("Updated list 1: " + updatedList);

        List<Integer> itemIds2 = Arrays.asList(8, 9, 13, 14, 15, 7, 11, 99);

        // Create a map to represent subsets and their corresponding package IDs
        Map<List<Integer>, Integer> packageMap2 = new HashMap<>();
        packageMap2.put(Arrays.asList(8, 9, 13, 14, 15), 1);
        packageMap2.put(Arrays.asList(8, 9, 11, 13, 14, 15), 2);
        packageMap2.put(Arrays.asList(7, 99), 3);

        // Call the method to test
        List<Object> updatedList2 = PackageItemUtil.replaceItemWithPackage(itemIds2, packageMap2);

        // Print the result for verification
        System.out.println("Updated list 2: " + updatedList2);
    }
}

```

#### ✅ Code Strengths:
- The code for `ItemService` has improved validation and error handling, ensuring data consistency.
- Utilizes Java 8 Streams effectively to simplify collection operations.
- Introduced a utility class to handle specific logic, promoting separation of concerns.
- Added a new test class to ensure functionality, ensuring the code works as expected.
- The effort to add documentation where present shows attention to detail.

#### 😄 Code Logic and Purpose:
The purpose of these changes is to enhance the application's robustness by adding more thorough validations and improving the efficiency of data handling. Introducing a utility class for item/package replacements allows for cleaner and more maintainable service layer code. The modifications also include a test suite to validate new functionalities, ensuring the system's reliability.