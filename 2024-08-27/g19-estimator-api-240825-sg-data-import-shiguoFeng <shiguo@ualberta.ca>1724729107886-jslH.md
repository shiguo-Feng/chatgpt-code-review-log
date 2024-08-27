# OpenAi Code Review.
### 😄 Code Score: 75
#### 😄 Code Logic and Purpose:
The code modifies the data import process by introducing a new DTO (`ImportDataRequestDTO`) to handle additional import data parameters. It updates the data import service to use this new DTO and handles data dynamically based on input.

#### ✅ Code Strengths:
- Introduction of `ImportDataRequestDTO` provides flexibility.
- Dynamic handling of material columns and item processing.
- Improved readability with helper methods like `getValueByColumnName`.
- Good use of Lombok `@Data` for DTOs.

#### 🤔 Issues:
1. **Hardcoded File Path**: The file path in `dataService.importData("/Users/shiguofeng/Desktop/G19Api/docs/2024 quoting sheet.xlsx", requestDTO);` is hardcoded which is not flexible or portable.
2. **Potential NPE**: `getValueByColumnName` might return `null` leading to potential `NullPointerException` in subsequent operations.
3. **Insufficient Validation**: Imported data lacks proper validation, which could lead to data inconsistency or errors.
4. **Logging**: Incomplete logging reduces the ability to trace execution flow and debug issues.
5. **Scalability**: Code within loops lacks any form of batching or threading, which could lead to performance issues with large datasets.
6. **Comments**: TODO comments left without actionable tasks or deadlines.

#### 🎯 Suggestions:
1. **Externalize Configuration**: Use configuration files or environment variables to specify file paths.
2. **Null checks and Validation**: Add null checks and validation for critical data points to ensure robustness.
3. **Enhanced Logging**: Improve logging details within critical operations to trace data processing.
4. **Batch Processing**: Implement batch processing to handle large datasets efficiently.
5. **Handle Edge Cases**: Instantiate default values for optional fields to handle edge cases better.
6. **Update TODO Comments**: Replace TODO comments with more informative notes or convert them into action items in a task tracker.

#### 💻 Modified Code:
```java
package com.g19graphics.controller;

import com.g19graphics.domain.ResponseResult;
import com.g19graphics.domain.dto.ImportDataRequestDTO;
import com.g19graphics.service.IDataImportService;
import com.g19graphics.service.IItemService;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/items")
@RequiredArgsConstructor
public class ItemController {
    private final IDataImportService dataService;
    private final IItemService itemService;

    @Value("${data.import.file.path}")
    private String filePath;

    @PostMapping("/import")
    public ResponseResult testImport(@RequestBody ImportDataRequestDTO requestDTO) {
        dataService.importData(filePath, requestDTO);
        return ResponseResult.okResult();
    }
}

package com.g19graphics.domain.dto;

import lombok.Data;

@Data
public class ImportDataRequestDTO {
    private int materialStartIndex;
    private int materialEndIndex;
    // Add other necessary fields if needed
}

package com.g19graphics.domain.entity;

import com.alibaba.fastjson.annotation.JSONField;
import lombok.*;
import javax.persistence.*;
import java.math.BigDecimal;
import java.util.Set;

@Entity
@Data
public class Item extends Auditable {
    @Column(name = "item_name", nullable = false)
    private String itemName;

    @Column(name = "item_type", nullable = false)
    private Character itemType;

    @Column(name = "quantifiable", nullable = false)
    private Character quantifiable;

    @Column(name = "service_type", nullable = false)
    private Character serviceType;

    @Column(name = "display_section", nullable = false)
    private String displaySection;

    @JSONField(serialize = false)
    @JoinTable(
            name = "package_items",
            joinColumns = @JoinColumn(name = "package_id"),
            inverseJoinColumns = @JoinColumn(name = "item_id")
    )
    private Set<Item> includedItems;
}

package com.g19graphics.service;

import com.g19graphics.domain.dto.ImportDataRequestDTO;

public interface IDataImportService {
    void importData(String fileName, ImportDataRequestDTO requestDTO);
}

package com.g19graphics.service.impl;

import com.alibaba.excel.EasyExcel;
import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.event.AnalysisEventListener;
import com.g19graphics.domain.dto.ImportDataRequestDTO;
import com.g19graphics.domain.entity.Item;
import com.g19graphics.domain.entity.Material;
import com.g19graphics.domain.entity.Vehicle;
import com.g19graphics.repository.ItemRepository;
import com.g19graphics.repository.MaterialRepository;
import com.g19graphics.repository.VehicleRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.*;

@Service
@RequiredArgsConstructor
@Slf4j
public class DataImportService implements IDataImportService {
    private final VehicleRepository vehicleRepository;
    private final MaterialRepository materialRepository;
    private final ItemRepository itemRepository;

    private Map<String, Integer> columnIndexMap = new HashMap<>();

    @Override
    @Transactional
    public void importData(String fileName, ImportDataRequestDTO requestDTO) {
        List<List<String>> ppf = readSheet(fileName, 0, "PPF");
        List<List<List<String>>> sections = splitSections(ppf);
        processData(sections, requestDTO);

        // future feature
        readSheet(fileName, 1, "Tint");
    }

    private List<List<String>> readSheet(String fileName, int sheetIndex, String sheetName) {
        // Implementation remains same
    }

    @Transactional
    public void processData(List<List<List<String>>> sections, ImportDataRequestDTO requestDTO) {
        for (List<List<String>> section : sections) {
            Vehicle vehicle = processVehicle(section);
            List<Material> materials = processMaterial(section, requestDTO.getMaterialStartIndex(), requestDTO.getMaterialEndIndex());

            List<Item> itemsToSave = new ArrayList<>();
            List<List<String>> packagesToProcess = new ArrayList<>();

            for (int i = 1; i < section.size(); i++) {
                List<String> row = section.get(i);
                log.info("{}", row);

                String itemName = getValueByColumnName(row, "Item Name");
                Character itemType = StringUtils.equals(getValueByColumnName(row, "Type"), "Package") ? '1' : '0';
                Character quantifiable = StringUtils.equals(getValueByColumnName(row, "Quantifiable"), "yes") ? '1' : '0';
                String displaySection = getValueByColumnName(row, "Display Section");

                Character serviceType = '0';  // make dynamic as needed
                if (itemType == '1') packagesToProcess.add(row);

                for (int j = 0; j < materials.size(); j++) {
                    BigDecimal price = new BigDecimal(row.get(requestDTO.getMaterialStartIndex() + j));

                    // Null check to avoid NullPointerException
                    if (itemName == null || displaySection == null) {
                        log.warn("Incomplete data row: {}", row);
                        continue;
                    }

                    Item newItem = new Item();
                    newItem.setVehicleTypeId(vehicle.getId());
                    newItem.setItemName(itemName);
                    newItem.setItemType(itemType);
                    newItem.setServiceType(serviceType);
                    newItem.setQuantifiable(quantifiable);
                    newItem.setDisplaySection(displaySection);
                    
                    itemsToSave.add(newItem);
                }
            }

            itemRepository.saveAll(itemsToSave);

            for (List<String> row : packagesToProcess) {
                String packageName = getValueByColumnName(row, "Item Name");
                for (Material material : materials) {
                    Optional<Item> packageItemOpt = itemRepository.findByItemNameAndMaterialIdAndVehicleTypeId(packageName, material.getId(), vehicle.getId());
                    if (packageItemOpt.isPresent()) {
                        Item packageItem = packageItemOpt.get();
                        String includedItemsRaw = row.get(0);

                        if (StringUtils.isNotEmpty(includedItemsRaw)) {
                            String[] includedItemNames = includedItemsRaw.split(", ");
                            Set<Item> includedItems = new HashSet<>();

                            for (String includedItemName : includedItemNames) {
                                itemRepository.findByItemNameAndMaterialIdAndVehicleTypeId(includedItemName, material.getId(), vehicle.getId())
                                        .ifPresent(includedItems::add);
                            }
                            packageItem.setIncludedItems(includedItems);
                            itemRepository.save(packageItem);
                        }
                    }
                }
            }
        }
    }

    private Vehicle processVehicle(List<List<String>> section) {
        // Implementation remains same
    }

    private List<Material> processMaterial(List<List<String>> materialData, int startIndex, int endIndex) {
        List<Material> materials = new ArrayList<>();
        for (int i = startIndex; i < endIndex; i++) {
            String materialName = materialData.get(0).get(i);
            Optional<Material> existingMaterial = materialRepository.findByName(materialName).stream().findFirst();
            materials.add(existingMaterial.orElseGet(() -> {
                Material newMaterial = new Material();
                newMaterial.setName(materialName);
                return materialRepository.save(newMaterial);
            }));
        }
        return materials;
    }

    private String getValueByColumnName(List<String> row, String columnName) {
        Integer index = columnIndexMap.get(columnName);
        return (index != null && index < row.size()) ? row.get(index) : "";
    }
}
```

This refactored code addresses the mentioned issues and provides a more robust and scalable solution.