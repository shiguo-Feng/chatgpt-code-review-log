# OpenAi Code Review.
### 😄 Code Score: 75
#### 😄 Code Logic and Purpose:
The primary purpose of this code update is to enhance the existing item selection and quotation management functionality. This update involves adding email fields, refactoring utility functions, creating new entity classes, and implementing an observer pattern to allow notification of quotation creation.

#### ✅ Code Strengths:
- Introduces an observer pattern for handling quotation notifications.
- Keeps entity and data transfer object (DTO) classes concise.
- Refactors utility functions for better readability and consistency.
- Applies consistent naming conventions.
- Makes good use of Lombok annotations to reduce boilerplate code.

#### 🤔 Issues:
1. **Potential Naming Conflicts**: The `randomNumeric` function in `RandomStringUtils` generates an alphanumeric string, which is misleading due to its name.
2. **No Input Validation**: The `randomNumeric` method does not validate the input length.
3. **Inadequate Error Handling**: The newly added methods do not include sufficient error handling.
4. **Inefficient and Verbose Import Statements**: Using wildcard imports can lead to issues in the future and should be avoided.
5. **Missing Unit Tests**: The new functionality, especially regarding the observer pattern and the email sending mechanism, lacks unit tests.
6. **Code Duplication**: Similar comments and logic structures are duplicated across different files, particularly in the observer classes.
7. **No Newline at EOF**: The Quotation entity class is missing a newline at the end of the file.
8. **Hard-Coded Values**: The `randomNumeric` method uses hard-coded strings.
9. **Commented Out TODOs**: Comments such as `// TODO: use observer pattern to send email` need to be removed as this functionality has now been implemented.

#### 🎯 Suggestions:
1. **Rename `randomNumeric`** to something more appropriate like `randomAlphanumeric`.
2. **Add Input Validation** to the `randomAlphanumeric` method to ensure the input length is non-negative.
3. **Introduce Error Handling** in methods that involve external operations like database saves and email sending.
4. **Avoid Wildcard Imports** by specifying the exact classes being imported.
5. **Implement Unit Tests** for new functionalities such as observer pattern and `QuotationObserverManager`.
6. **Remove Duplication and Incorporate Abstract Classes** or base methods for observer functionalities to reduce redundancy.
7. **Ensure EOF Line**: Ensure all files end with a newline to adhere to standard file formatting practices.
8. **Externalize Hard-Coded Values**: Move hard-coded strings to configuration files or constants.
9. **Remove Redundant Comments** to keep the code clean.

#### 💻 Modified Code:
```java
diff --git a/framework/src/main/java/com/g19graphics/constants/SystemConstants.java b/framework/src/main/java/com/g19graphics/constants/SystemConstants.java
index 192221d..43bb97f 100644
--- a/framework/src/main/java/com/g19graphics/constants/SystemConstants.java
+++ b/framework/src/main/java/com/g19graphics/constants/SystemConstants.java
@@ -6,7 +6,7 @@ package com.g19graphics.constants;
  * @create 2024-08-24
  */
 public class SystemConstants {
-    public static final Integer MATERIAL_TYPE_ID_1 = 1;
+    public static final Integer IDENTIFIER_LENGTH = 4;
     public static final Character ITEM_TYPE_PACKAGE = '1';
     public static final Character ITEM_TYPE_ITEM = '0';
     public static final Character SERVICE_TYPE_PPF = '0';
diff --git a/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java b/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
index 501cedc..c171c04 100644
--- a/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
+++ b/framework/src/main/java/com/g19graphics/domain/dto/ItemSelectionDTO.java
@@ -15,4 +15,5 @@ public class ItemSelectionDTO {
     private Long materialId;
     // key: id, value: quantity
     private Map<Long, Integer> itemQuantities;
+    private String email;
 }
diff --git a/framework/src/main/java/com/g19graphics/domain/entity/Quotation.java b/framework/src/main/java/com/g19graphics/domain/entity/Quotation.java
index 0000000..591f003 100644
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/domain/entity/Quotation.java
@@ -0,0 +1,45 @@
+package com.g19graphics.domain.entity;
+
+import com.g19graphics.domain.entity.base.Auditable;
+import lombok.Getter;
+import lombok.Setter;
+
+import javax.persistence.*;
+import java.math.BigDecimal;
+import java.time.Instant;
+
+@Getter
+@Setter
+@Entity
+@Table(name = "quotation", schema = "g19_estimator")
+public class Quotation extends Auditable {
+    @Id
+    @GeneratedValue(strategy = GenerationType.IDENTITY)
+    @Column(name = "id", nullable = false)
+    private Long id;
+
+    @Column(name = "identifier", nullable = false, length = 20)
+    private String identifier;
+
+    @Lob
+    @Column(name = "items", nullable = false)
+    private String items;
+
+    @Lob
+    @Column(name = "replaced_with")
+    private String replacedWith;
+
+    @Column(name = "material_id")
+    private Long materialId;
+
+    @Column(name = "vehicle_type_id")
+    private Long vehicleTypeId;
+
+    @Column(name = "total_price", precision = 10, scale = 2)
+    private BigDecimal totalPrice;
+
+    @Column(name = "email", length = 64)
+    private String email;
+}
diff --git a/framework/src/main/java/com/g19graphics/domain/vo/QuotationVO.java b/framework/src/main/java/com/g19graphics/domain/vo/QuotationVO.java
new file mode 100644
index 0000000..364de5e
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/domain/vo/QuotationVO.java
@@ -0,0 +1,16 @@
+package com.g19graphics.domain.vo;
+
+import lombok.AllArgsConstructor;
+import lombok.Data;
+import lombok.NoArgsConstructor;
+
+/**
+ * @description
+ * @create 2024-08-28
+ */
+@Data
+@NoArgsConstructor
+@AllArgsConstructor
+public class QuotationVO {
+}
diff --git a/framework/src/main/java/com/g19graphics/repositories/QuotationRepository.java b/framework/src/main/java/com/g19graphics/repositories/QuotationRepository.java
new file mode 100644
index 0000000..374d6a4
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/repositories/QuotationRepository.java
@@ -0,0 +1,14 @@
+package com.g19graphics.repositories;
+
+import com.g19graphics.domain.entity.Quotation;
+import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.stereotype.Repository;
+
+/**
+ * @description
+ * @create 2024-08-27
+ */
+@Repository
+public interface QuotationRepository extends JpaRepository<Quotation, Long> {
+}
diff --git a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
index 212a2c3..8d994bd 100644
--- a/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
+++ b/framework/src/main/java/com/g19graphics/service/impl/ItemService.java
@@ -2,31 +2,31 @@ package com.g19graphics.service.impl;
 
 import com.alibaba.fastjson.JSON;
 import com.alibaba.fastjson.TypeReference;
+import com.g19graphics.constants.SystemConstants;
 import com.g19graphics.domain.ResponseResult;
 import com.g19graphics.domain.dto.ItemRequestDTO;
 import com.g19graphics.domain.dto.ItemSelectionDTO;
 import com.g19graphics.domain.entity.Item;
-import com.g19graphics.constants.SystemConstants;
+import com.g19graphics.domain.entity.Quotation;
 import com.g19graphics.domain.vo.ItemVO;
 import com.g19graphics.domain.vo.PackageVO;
 import com.g19graphics.enums.AppHttpCodeEnum;
 import com.g19graphics.exception.SystemException;
 import com.g19graphics.repositories.ItemRepository;
+import com.g19graphics.repositories.QuotationRepository;
 import com.g19graphics.service.IItemService;
-import com.g19graphics.service.cache.PackageItemCache;
+import com.g19graphics.service.notification.QuotationObserverManager;
 import com.g19graphics.utility.BeanCopyUtils;
-import com.g19graphics.utility.PackageItemUtil;
+import com.g19graphics.utility.PackageItemUtils;
+import com.g19graphics.utility.RandomStringUtils;
 import com.g19graphics.utility.RedisCache;
 import lombok.RequiredArgsConstructor;
 import org.springframework.stereotype.Service;
-import org.springframework.util.StringUtils;
 
 import java.math.BigDecimal;
 import java.util.ArrayList;
 import java.util.List;
 import java.util.Map;
-import java.util.Optional;
-import java.util.concurrent.TimeUnit;
 import java.util.stream.Collectors;
 
 /**
@@ -39,8 +39,9 @@ import java.util.stream.Collectors;
 public class ItemService implements IItemService {
 
     private final ItemRepository itemRepository;
-
+    private final QuotationRepository quotationRepository;
     private final RedisCache redisCache;
+    private final QuotationObserverManager quotationObserverManager;
 
     @Override
     public ResponseResult packageList(ItemRequestDTO requestDTO) {
@@ -83,7 +84,7 @@ public class ItemService implements IItemService {
 
     @Override
     public ResponseResult processSelection(ItemSelectionDTO requestDTO) {
-        validateParameters(requestDTO.getVehicleId(), requestDTO.getMaterialId());
+        validateParameters(requestDTO.getVehicleId(), requestDTO.getMaterialId(), requestDTO.getEmail());
         if (requestDTO.getItemQuantities() == null || requestDTO.getItemQuantities().isEmpty()) {
             throw new SystemException(AppHttpCodeEnum.ITEM_SELECTION_NOT_NULL);
         }
@@ -94,20 +95,29 @@ public class ItemService implements IItemService {
         });
 
         // 2. Replace itemIds with package if applicable
-        List<Long> replaced = PackageItemUtil.replaceItemWithPackage(requestDTO.getItemQuantities(), packageMap);
+        List<Long> replaced = PackageItemUtils.replaceItemWithPackage(requestDTO.getItemQuantities(), packageMap);
+
+        // 3. Calculate total price using Stream
         Map<Long, Integer> finalQuantities = replaced.stream()
                 .collect(Collectors.toMap(
-                        itemId -> itemId, // Key: The item ID (or package ID)
-                        itemId -> requestDTO.getItemQuantities().getOrDefault(itemId, 1) // Value: Original quantity or 1 if it got replaced
+                        // Key: The item ID (or package ID)
+                        itemId -> itemId,
+                        // Value: Original quantity or 1 if it got replaced
+                        itemId -> requestDTO.getItemQuantities().getOrDefault(itemId, 1)
                 ));
-        // 3. Calculate total price using Stream
+
         BigDecimal total = finalQuantities.entrySet().stream()
-                .map(entry -> itemRepository.findById(entry.getKey()) // Find the item by ID
-                        .map(item -> item.getPrice().multiply(BigDecimal.valueOf(entry.getValue()))) // Multiply price with quantity
-                        .orElse(BigDecimal.ZERO)) // Default to BigDecimal.ZERO if the item is not found
-                .reduce(BigDecimal.ZERO, BigDecimal::add); // Sum up all the prices
-        // 5. create && save quotation
-        // TODO: use observer pattern to send email
+                .map(entry -> itemRepository.findById(entry.getKey())
+                        .map(item -> item.getPrice().multiply(BigDecimal.valueOf(entry.getValue())))
+                        .orElse(BigDecimal.ZERO))
+                .reduce(BigDecimal.ZERO, BigDecimal::add);
+
+        // 4. create && save quotation
+        Quotation quotation = createAndSaveQuotation(requestDTO, finalQuantities, total);
+
+        // 5. Notify observers about the new quotation
+        quotationObserverManager.notifyQuotationCreated(quotation);
+
         // 6. return quotation vo
         return ResponseResult.okResult(total);
     }
@@ -120,4 +130,16 @@ public class ItemService implements IItemService {
         }
     }
 
+    private Quotation createAndSaveQuotation(ItemSelectionDTO requestDTO, Map<Long, Integer> finalQuantities, BigDecimal total) {
+        Quotation quotation = new Quotation();
+        quotation.setIdentifier(RandomStringUtils.randomAlphanumeric(SystemConstants.IDENTIFIER_LENGTH));
+        quotation.setItems(JSON.toJSONString(requestDTO.getItemQuantities()));
+        quotation.setReplacedWith(JSON.toJSONString(finalQuantities));
+        quotation.setVehicleTypeId(requestDTO.getVehicleId());
+        quotation.setMaterialId(requestDTO.getMaterialId());
+        quotation.setTotalPrice(total);
+        quotation.setEmail(requestDTO.getEmail());
+        quotationRepository.save(quotation);
+        return quotation;
+    }
 }
diff --git a/framework/src/main/java/com/g19graphics/service/notification/QuotationObserver.java b/framework/src/main/java/com/g19graphics/service/notification/QuotationObserver.java
new file mode 100644
index 0000000..9f5b36b
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/service/notification/QuotationObserver.java
@@ -0,0 +1,12 @@
+package com.g19graphics.service.notification;
+
+import com.g19graphics.domain.entity.Quotation;
+
+/**
+ * @description interface for observers
+ * @create 2024-08-27
+ */
+public interface QuotationObserver {
+    void handleQuotationCreated(Quotation quotation);
+}
diff --git a/framework/src/main/java/com/g19graphics/service/notification/QuotationObserverManager.java b/framework/src/main/java/com/g19graphics/service/notification/QuotationObserverManager.java
new file mode 100644
index 0000000..d8b5028
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/service/notification/QuotationObserverManager.java
@@ -0,0 +1,27 @@
+package com.g19graphics.service.notification;
+
+import com.g19graphics.domain.entity.Quotation;
+import lombok.RequiredArgsConstructor;
+import org.springframework.stereotype.Component;
+
+import java.util.List;
+
+/**
+ * @description
+ * @create 2024-08-27
+ */
+@Component
+@RequiredArgsConstructor
+public class QuotationObserverManager {
+
+    // Injects all beans that implement QuotationObserver interface
+    private final List<QuotationObserver> observers;
+
+    public void notifyQuotationCreated(Quotation quotation) {
+        observers.forEach(observer -> {
+            try {
+                observer.handleQuotationCreated(quotation);
+            } catch (Exception e) {
+                // add proper logging and handling
+            }
+        });
+    }
 }
diff --git a/framework/src/main/java/com/g19graphics/service/notification/observer/CustomerEmailObserver.java b/framework/src/main/java/com/g19graphics/service/notification/observer/CustomerEmailObserver.java
new file mode 100644
index 0000000..ec19902
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/service/notification/observer/CustomerEmailObserver.java
@@ -0,0 +1,23 @@
+package com.g19graphics.service.notification.observer;
+
+import com.g19graphics.domain.entity.Quotation;
+import com.g19graphics.service.notification.QuotationObserver;
+import org.springframework.stereotype.Component;
+
+/**
+ * @description send quotation summary to customer
+ * @create 2024-08-27
+ */
+@Component
+public class CustomerEmailObserver implements QuotationObserver {
+    @Override
+    public void handleQuotationCreated(Quotation quotation) {
+        sendEmailToCustomer(quotation);
+    }
+
+    private void sendEmailToCustomer(Quotation quotation) {
+        System.out.println("Sending email to customer: " + quotation.getEmail());
+        // add actual implementation
+    }
+}
diff --git a/framework/src/main/java/com/g19graphics/service/notification/observer/InternalSupportObserver.java b/framework/src/main/java/com/g19graphics/service/notification/observer/InternalSupportObserver.java
new file mode 100644
index 0000000..c5ecad5
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/service/notification/observer/InternalSupportObserver.java
@@ -0,0 +1,23 @@
+package com.g19graphics.service.notification.observer;
+
+import com.g19graphics.domain.entity.Quotation;
+import com.g19graphics.service.notification.QuotationObserver;
+import org.springframework.stereotype.Component;
+
+/**
+ * @description send notification to internal team
+ * @create 2024-08-27
+ */
+@Component
+public class InternalSupportObserver implements QuotationObserver {
+    @Override
+    public void handleQuotationCreated(Quotation quotation) {
+        notifyInternalSupport(quotation);
+    }
+
+    private void notifyInternalSupport(Quotation quotation) {
+        System.out.println("Notifying internal support team for quotation: " + quotation.getId());
+        // add actual implementation
+    }
+}
```

#### 💪 Code Strengths:
- The addition of features such as observer pattern for notification.
- Utilization of modern Java and Spring Boot practices.
- Improved data handling with DTOs and VOs.
- Usage of Lombok to reduce boilerplate.
- New classes and methods are well-composed and maintain Single Responsibility Principle.

#### 📌 Logic and Purpose:
Enhancing the item selection and quotation creation functionality by adding email fields, refactoring utility functions, and introducing an observer pattern for improved notification management. The changes aim to improve readability, maintainability, and scalability of the codebase.