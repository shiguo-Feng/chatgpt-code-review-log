# OpenAi Code Review.
### 😄 Code Score: 80
#### 😄 Code Logic and Purpose:
The modifications primarily introduce a new endpoint in the `AdminDataController` for retrieving quotation count percentage statistics. This includes adding necessary services and repository methods to support the functionality.
#### ✅ Code Strengths:
- The addition of the `IQuotationService` interface and its implementation is clean and adheres to good design principles.
- The use of Spring Data JPA and custom query to count quotations within a date range is efficient.
- The logic for calculating the percentage change is straightforward.
#### 🤔 Issues:
1. **Exception Handling:** Lack of exception handling in the new endpoint may lead to unhandled exceptions.
2. **Magic Numbers:** The values '7' and '1' used in date calculations should be defined as constants for better readability and maintainability.
3. **Formatting:** Extra newlines in `AdminVehicle.java` can be removed to keep the code tidy.
4. **Zero Division Risk:** Division by zero can occur if `lastWeekCount` is zero, which should be handled explicitly.
5. **Inline Comments:** Important sections like date calculations can benefit from inline comments.
#### 🎯 Suggestions:
1. Add exception handling in `getQuotationCountPercentage` to manage possible runtime issues.
2. Define constants for magic numbers used in date calculations.
3. Remove unnecessary newlines in `AdminVehicle.java` to maintain clean code.
4. Explicitly handle the zero division risk in calculating percentage change.
5. Add comments to clarify the date calculations for future maintainability.

#### 💻 Modified Code:
```java
diff --git a/admin/src/main/java/com/g19graphics/controller/AdminDataController.java b/admin/src/main/java/com/g19graphics/controller/AdminDataController.java
index 85d02d3..295b77a 100644
--- a/admin/src/main/java/com/g19graphics/controller/AdminDataController.java
+++ b/admin/src/main/java/com/g19graphics/controller/AdminDataController.java
@@ -3,6 +3,7 @@ package com.g19graphics.controller;
 import com.g19graphics.domain.ResponseResult;
 import com.g19graphics.domain.dto.ImportDataRequestDTO;
 import com.g19graphics.service.IDataImportService;
+import com.g19graphics.service.IQuotationService;
 import lombok.RequiredArgsConstructor;
 import org.springframework.web.bind.annotation.*;
 import org.springframework.web.multipart.MultipartFile;
@@ -20,6 +21,7 @@ import java.io.IOException;
 public class AdminDataController {
 
     private final IDataImportService dataService;
+    private final IQuotationService quotationService;
 
     @PostMapping("/import")
     public ResponseResult importData(
@@ -32,5 +34,12 @@ public class AdminDataController {
         return ResponseResult.okResult();
     }
 
+    @GetMapping("/quotation/percentage")
+    public ResponseResult getQuotationCountPercentage() {
+        try {
+            return quotationService.getQuotationCountPercentage();
+        } catch (Exception e) {
+            return ResponseResult.errorResult();
+        }
+    }
 
 }
diff --git a/admin/src/main/java/com/g19graphics/controller/AdminVehicle.java b/admin/src/main/java/com/g19graphics/controller/AdminVehicle.java
index d604e63..75c1ef6 100644
--- a/admin/src/main/java/com/g19graphics/controller/AdminVehicle.java
+++ b/admin/src/main/java/com/g19graphics/controller/AdminVehicle.java
@@ -28,6 +28,5 @@ public class AdminVehicle {
     public ResponseResult updateVehicle(@RequestBody Vehicle vehicle) {
         return vehicleService.updateVehicle(vehicle);
     }
+
-    
 }
diff --git a/framework/src/main/java/com/g19graphics/repositories/QuotationRepository.java b/framework/src/main/java/com/g19graphics/repositories/QuotationRepository.java
index 23ea255..00175be 100644
--- a/framework/src/main/java/com/g19graphics/repositories/QuotationRepository.java
+++ b/framework/src/main/java/com/g19graphics/repositories/QuotationRepository.java
@@ -2,8 +2,10 @@ package com.g19graphics.repositories;
 
 import com.g19graphics.domain.entity.Quotation;
 import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.Query;
 import org.springframework.stereotype.Repository;
 
+import java.time.Instant;
 import java.util.Optional;
 
 /**
@@ -14,4 +16,7 @@ import java.util.Optional;
 @Repository
 public interface QuotationRepository extends JpaRepository<Quotation, Long> {
     Optional<Quotation> findByIdentifier(String identifier);
+
+    @Query("SELECT COUNT(q) FROM Quotation q WHERE q.createTime BETWEEN :startDate AND :endDate")
+    Long countQuotationsWithinDateRange(Instant startDate, Instant endDate);
 }
diff --git a/framework/src/main/java/com/g19graphics/service/IQuotationService.java b/framework/src/main/java/com/g19graphics/service/IQuotationService.java
new file mode 100644
index 0000000..ef09846
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/service/IQuotationService.java
@@ -0,0 +1,12 @@
+package com.g19graphics.service;
+
+import com.g19graphics.domain.ResponseResult;
+
+/**
+ * @description
+ * @create 2024-09-05
+ */
+public interface IQuotationService {
+    ResponseResult getQuotationCountPercentage();
+}
diff --git a/framework/src/main/java/com/g19graphics/service/impl/QuotationService.java b/framework/src/main/java/com/g19graphics/service/impl/QuotationService.java
new file mode 100644
index 0000000..83bb4b1
--- /dev/null
+++ b/framework/src/main/java/com/g19graphics/service/impl/QuotationService.java
@@ -0,0 +1,54 @@
+package com.g19graphics.service.impl;
+
+import com.g19graphics.domain.ResponseResult;
+import com.g19graphics.repositories.QuotationRepository;
+import com.g19graphics.service.IQuotationService;
+import lombok.RequiredArgsConstructor;
+import org.springframework.stereotype.Service;
+
+import java.time.Instant;
+import java.time.ZoneId;
+import java.time.temporal.ChronoField;
+import java.time.temporal.ChronoUnit;
+import java.util.HashMap;
+import java.util.Map;
+
+@Service
+@RequiredArgsConstructor
+public class QuotationService implements IQuotationService {
+
+    private final QuotationRepository quotationRepository;
+
+    private static final int DAYS_IN_WEEK = 7;
+    private static final long ONE_SECOND = 1L;
+
+    public ResponseResult getQuotationCountPercentage() {
+        Instant now = Instant.now();
+        Instant startOfThisWeek = now.atZone(ZoneId.systemDefault()).with(ChronoField.DAY_OF_WEEK, 1).toInstant();
+        Instant startOfLastWeek = startOfThisWeek.minus(DAYS_IN_WEEK, ChronoUnit.DAYS);
+        Instant endOfLastWeek = startOfThisWeek.minus(ONE_SECOND, ChronoUnit.SECONDS);
+
+        Long thisWeekCount = quotationRepository.countQuotationsWithinDateRange(startOfThisWeek, now);
+        Long lastWeekCount = quotationRepository.countQuotationsWithinDateRange(startOfLastWeek, endOfLastWeek);
+
+        double percentageChange = 0;
+        if (lastWeekCount != 0) {
+            percentageChange = ((double) (thisWeekCount - lastWeekCount) / lastWeekCount) * 100;
+        }
+
+        Map<String, Object> statistics = new HashMap<>();
+        statistics.put("thisWeekCount", thisWeekCount);
+        statistics.put("lastWeekCount", lastWeekCount);
+        statistics.put("percentageChange", percentageChange);
+
+        return ResponseResult.okResult(statistics);
+    }
+}
```

#### ✅ Code Strengths:
- Proper use of Spring Boot annotations and dependency injection.
- Effective use of Spring Data JPA for querying the database.
- The logic for date range calculations is logically sound.
#### 😄 Code Logic and Purpose:
The new code adds functionality for calculating the percentage change in quotation counts week over week, integrates it into the `AdminDataController`, and handles database operations efficiently using Spring Data JPA.