# EasyExcel写入Excel函数

1. **基本实现方案**([1](https://easyexcel.opensource.alibaba.com/docs/current/))
```java
@Test
public void writeWithFormula() {
    String fileName = "测试公式.xlsx";
    
    try (ExcelWriter excelWriter = EasyExcel.write(fileName, DemoData.class).build()) {
        WriteSheet writeSheet = EasyExcel.writerSheet("模板").build();
        
        // 1. 写入正常数据
        List<DemoData> dataList = getData();
        excelWriter.write(dataList, writeSheet);
        
        // 2. 写入公式行
        // 假设金额在第3列，数据从第2行开始(第1行是表头)
        int lastRow = dataList.size() + 1;  // 最后一行的位置
        List<List<Object>> formulaRowList = new ArrayList<>();
        List<Object> formulaRow = new ArrayList<>();
        formulaRow.add("合计");  // 第1列
        formulaRow.add(null);   // 第2列
        // 第3列写入SUM函数，计算从第2行到最后一行的和
        formulaRow.add(new FormulaData("SUM(C2:C" + lastRow + ")"));
        formulaRowList.add(formulaRow);
        
        excelWriter.write(formulaRowList, writeSheet);
    }
}

// FormulaData类用于包装公式
@Data
public class FormulaData {
    private String formula;
    
    public FormulaData(String formula) {
        this.formula = formula;
    }
}
```

2. **使用CellWriteHandler实现**
```java
public class FormulaCellWriteHandler implements CellWriteHandler {
    @Override
    public void afterCellDispose(CellWriteHandlerContext context) {
        Cell cell = context.getCell();
        // 判断是否是最后一行的第三列
        if (cell.getRowIndex() == context.getWriteSheetHolder().getSheetNo() 
            && cell.getColumnIndex() == 2) {
            // 设置公式
            cell.setCellFormula("SUM(C2:C" + (cell.getRowIndex()) + ")");
        }
    }
}
```

3. **支持的常用Excel函数**
```java
// 求和函数
"SUM(C2:C10)"

// 平均值函数
"AVERAGE(C2:C10)"

// 最大值函数
"MAX(C2:C10)"

// 最小值函数
"MIN(C2:C10)"

// 计数函数
"COUNT(C2:C10)"

// IF条件函数
"IF(C2>100,\"高\",\"低\")"
```

4. **注意事项**
- 函数单元格需要使用FormulaData包装
- 函数引用的单元格范围要准确
- 注意行列的索引从0开始
- 可以使用相对引用和绝对引用
- 复杂函数建议使用CellWriteHandler实现 

# EasyExcel写入公式完整示例

```java
@Test
public void writeWithFormula() {
    String fileName = "测试公式.xlsx";
    
    // 关键点：在这里注册CellWriteHandler
    try (ExcelWriter excelWriter = EasyExcel.write(fileName, DemoData.class)
            .registerWriteHandler(new FormulaCellWriteHandler())  // 注册处理器
            .build()) {
            
        WriteSheet writeSheet = EasyExcel.writerSheet("模板").build();
        
        // 写入数据
        List<DemoData> dataList = getData();
        excelWriter.write(dataList, writeSheet);
    }
}

// 处理器实现
public class FormulaCellWriteHandler implements CellWriteHandler {
    @Override
    public void afterCellDispose(CellWriteHandlerContext context) {
        Cell cell = context.getCell();
        // 判断是否是最后一行的第三列
        if (cell.getRowIndex() == context.getWriteSheetHolder().getSheetNo() 
            && cell.getColumnIndex() == 2) {
            // 设置公式
            cell.setCellFormula("SUM(C2:C" + (cell.getRowIndex()) + ")");
        }
    }
}

// 数据对象
@Data
public class DemoData {
    private String name;
    private String date;
    private BigDecimal amount;
}
```

关键点说明：
1. 使用`.registerWriteHandler()`注册处理器
2. 处理器会在每个单元格写入时被调用
3. 不需要单独写入最后一行，处理器会自动处理
4. 相比之前的方案更加简洁和自动化 