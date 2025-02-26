# CellWriteHandlerContext上下文信息获取

1. **获取总行数**
```java
public class FormulaCellWriteHandler implements CellWriteHandler {
    @Override
    public void afterCellDispose(CellWriteHandlerContext context) {
        // 获取当前sheet
        WriteSheetHolder writeSheetHolder = context.getWriteSheetHolder();
        // 获取当前sheet的总行数
        int totalRows = writeSheetHolder.getSheet().getLastRowNum();
        
        Cell cell = context.getCell();
        // 判断是否是最后一行的第三列(C列)
        if (cell.getRowIndex() == totalRows && cell.getColumnIndex() == 2) {
            // 设置求和公式，从第2行开始到最后一行
            cell.setCellFormula("SUM(C2:C" + totalRows + ")");
        }
    }
}
```

2. **上下文中可获取的信息**
```java
// 行相关
context.getRow();                    // 当前行
context.getRowIndex();              // 当前行号
context.getRelativeRowIndex();      // 相对行号(不包括表头)

// 列相关
context.getCell();                  // 当前单元格
context.getColumnIndex();           // 当前列号
context.getHead();                  // 表头信息

// Sheet相关
context.getWriteSheetHolder();      // Sheet相关信息
context.getWriteTableHolder();      // Table相关信息

// 工作簿相关
context.getWriteWorkbookHolder();   // 工作簿相关信息
```

3. **实际应用示例**
```java
public class FormulaCellWriteHandler implements CellWriteHandler {
    @Override
    public void afterCellDispose(CellWriteHandlerContext context) {
        WriteSheetHolder writeSheetHolder = context.getWriteSheetHolder();
        Sheet sheet = writeSheetHolder.getSheet();
        
        // 当前行号
        int currentRowIndex = context.getRowIndex();
        // 总行数
        int totalRows = sheet.getLastRowNum();
        // 当前列号
        int columnIndex = context.getColumnIndex();
        
        // 判断是否是最后一行的金额列
        if (currentRowIndex == totalRows && columnIndex == 2) {
            Cell cell = context.getCell();
            // 设置公式
            cell.setCellFormula("SUM(C2:C" + totalRows + ")");
            
            // 设置样式
            CellStyle style = context.getWriteWorkbookHolder().getWorkbook().createCellStyle();
            style.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
            style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
            cell.setCellStyle(style);
        }
    }
}
```

4. **注意事项**
- getLastRowNum()返回的是最后一行的索引（从0开始）
- 需要考虑表头行的影响
- 相对行号不包括表头行
- 可以通过上下文获取完整的写入信息 