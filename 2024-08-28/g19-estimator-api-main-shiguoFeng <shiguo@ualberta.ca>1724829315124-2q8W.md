# OpenAi Code Review.
### 😄 Code Score: 85
#### 😄 Code Logic and Purpose:
The modifications to the code add new fields to the `QuotationVO` class and enhance the `ItemService` to generate and use `QuotationVO` objects instead of raw `Quotation` entities. Additionally, the observer pattern has been updated to deal with `QuotationVO` objects. These changes contribute towards making the code more scalable and maintainable by centralizing more logic within the VO.
#### ✅ Code Strengths:
- Improved encapsulation by using `QuotationVO`.
- Clear separation of concerns: services should not be directly manipulating raw entities.
- Usage of `BeanCopyUtils` for copying properties, aiding in code reuse and maintainability.
- Attempt to refactor to make future enhancements easier.
#### 🤔 Issues:
1. **Variable Naming:**
   - Inconsistent camel case for `VehicleName`.
2. **Potential for NullPointerException:**
   - `materialRepository.findById(quotation.getMaterialId())` assumes `quotation.getMaterialId()` is non-null.
   - `entry -> itemRepository.findById(entry.getKey()).map(Item::getItemName).orElse("Unknown Item")` can result in all "Unknown Item" if any ID is invalid.
3. **Lack of Exception Handling:**
   - Parsing JSON and finding items in the repository assumes success, which isn’t always guaranteed.
4. **Hard to Trace Logic:**
   - Nested lambda functions within `Function<String, Map<String, Integer>> parseAndMapNames` make understanding the flow difficult.
5. **Magic Strings:**
   - `"ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"` should be declared as a constant.
#### 🎯 Suggestions:
1. **Consistent Naming Conventions:**
   - Change `VehicleName` to `vehicleName`.
2. **Null Checks:**
   - Add null checks for `materialId` and `vehicleTypeId`.
   - Handle possible null returns from `findById` more gracefully.
3. **Exception Handling:**
   - Add exception handling for the JSON parsing logic and repository fetches.
4. **Code Readability:**
   - Break down complex lambda functions into more readable methods.
5. **Constants Declaration:**
   - Declare character strings as constants.
#### 💻 Modified Code:
```java
diff --git a/framework/src/main/java/com/g19graphics/domain/vo/QuotationVO.java b/framework/src/main/java/com/g19graphics/domain/vo/QuotationVO.java
index 364de5e..82e6607 100644
--- a/framework/src/main/java/com/g19graphics/domain/vo/QuotationVO.java
+++ b/framework/src/main/java/com/g19graphics/domain/vo/QuotationVO.java
@@ -4,6 +4,9 @@ import lombok.AllArgsConstructor;
 import lombok.Data;
 import lombok.NoArgsConstructor;
 
+import java.math.BigDecimal;
+import java.util.Map;
+
 /**
  * @description This class represents a Quotation View Object which is a more inclusive 
  * representation of Quotation entity for various business logic purposes.
  */
 @Data
 @NoArgsConstructor
 @AllArgsConstructor
 public class QuotationVO {
     private String identifier;
     private BigDecimal totalPrice;
     private String email;
     private String materialName;
     private String vehicleName; // Renamed for consistency
     // key: Name, value: quantity
     Map<String, Integer> itemQuantities;
     Map<String, Integer> replacedItemQuantities;
 }
diff --git a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
index 8d994bd..083f0d2 100644
--- a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
+++ b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
@@ -10,10 +10,13 @@ import com.g19graphics.domain.entity.Item;
 import com.g19graphics.domain.entity.Quotation;
 import com.g19graphics.domain.vo.ItemVO;
 import com.g19graphics.domain.vo.PackageVO;
+import com.g19graphics.domain.vo.QuotationVO;
 import com.g19graphics.enums.AppHttpCodeEnum;
 import com.g19graphics.exception.SystemException;
 import com.g19graphics.repositories.ItemRepository;
+import com.g19graphics.repositories.MaterialRepository;
 import com.g19graphics.repositories.QuotationRepository;
+import com.g19graphics.repositories.VehicleRepository;
 import com.g19graphics.service.IItemService;
 import com.g19graphics.service.notification.QuotationObserverManager;
 import com.g19graphics.utility.BeanCopyUtils;
@@ -27,6 +30,7 @@ import java.math.BigDecimal;
 import java.util.ArrayList;
 import java.util.List;
 import java.util.Map;
+import java.util.function.Function;
 import java.util.stream.Collectors;
 
 /**
@@ -40,6 +44,8 @@ public class ItemService implements IItemService {
 
     private final ItemRepository itemRepository;
     private final QuotationRepository quotationRepository;
     private final MaterialRepository materialRepository;
     private final VehicleRepository vehicleRepository;
     private final RedisCache redisCache;
     private final QuotationObserverManager quotationObserverManager;
 
@@ -115,11 +121,12 @@ public class ItemService implements IItemService {
         // 4. create && save quotation
         Quotation quotation = createAndSaveQuotation(requestDTO, finalQuantities, total);
 
         // 5. convert to quotation vo
         QuotationVO vo = createQuotationVO(quotation);
         // 6. Notify observers about the new quotation
         quotationObserverManager.notifyQuotationCreated(vo);
 
         // 6. return quotation vo
         return ResponseResult.okResult(vo);
     }
 
@@ -142,4 +149,37 @@ public class ItemService implements IItemService {
         quotationRepository.save(quotation);
         return quotation;
     }
 
     private QuotationVO createQuotationVO(Quotation quotation) {
         QuotationVO vo = BeanCopyUtils.copyBean(quotation, QuotationVO.class);
         materialRepository.findById(quotation.getMaterialId())
             .ifPresent(material -> vo.setMaterialName(material.getName()));
         vehicleRepository.findById(quotation.getVehicleTypeId())
             .ifPresent(vehicle -> vo.setVehicleName(vehicle.getName()));
 
         // Parse JSON and fetch item names
         Function<String, Map<String, Integer>> parseAndMapNames = json -> {
             try {
                 return JSON.parseObject(json, new TypeReference<Map<Long, Integer>>() {})
                     .entrySet().stream().collect(Collectors.toMap(
                         entry -> itemRepository.findById(entry.getKey())
                             .map(Item::getItemName)
                             .orElse("Unknown Item"),
                         Map.Entry::getValue
                     ));
             } catch (JSONException e) {
                 throw new SystemException("Failed to parse JSON", e);
             }
         };
 
         // Set the item names in the VO
         vo.setItemQuantities(parseAndMapNames.apply(quotation.getItems()));
         vo.setReplacedItemQuantities(parseAndMapNames.apply(quotation.getReplacedWith()));
 
         return vo;
     }
 }
diff --git a/framework/src/main/java/com/g19graphics/utility/RandomStringUtils.java b/framework/src/main/java/com/g19graphics/utility/RandomStringUtils.java
index 5a2d7be..2897f84 100644
--- a/framework/src/main/java/com/g19graphics/utility/RandomStringUtils.java
+++ b/framework/src/main/java/com/g19graphics/utility/RandomStringUtils.java
@@ -9,7 +9,11 @@ import java.util.Random;
  */
 public class RandomStringUtils {
     private static final String CHARACTERS = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"; // Declared as constant
     
     public static String randomNumeric(int length) {
-        String characters = "abcdefghijklmnopqrstuvwxyz0123456789";
+        String characters = CHARACTERS; // Updated to use the declared constant
         Random random = new Random();
         StringBuilder sb = new StringBuilder(length);
         for (int i = 0; i < length; i++) {
```
#### ✅ Code Strengths:
- The refactor to use `QuotationVO` is a good step towards better design.
- Ensuring that observers receive a more detailed object (`QuotationVO`) rather than an entity is commendable.
- The refactor adheres to the single responsibility principle by delegating entity to VO transformations.
#### 😄 Code Logic and Purpose:
The modified code aims to improve scalability and internal structure by employing the `QuotationVO` object for notifications and interaction within the service layer. It encapsulates more data relevant to a quotation beyond just the database entity, allowing for easier adjustments and extensions in the future.