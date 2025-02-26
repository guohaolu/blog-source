# EasyExcel导出并在最后插入汇总行

1. **实现方案**([1](https://easyexcel.opensource.alibaba.com/docs/current/))
```java
@Test
public void writeWithLastRow() {
    String fileName = "测试导出.xlsx";
    
    // 使用ExcelWriter可以实现更灵活的写入
    try (ExcelWriter excelWriter = EasyExcel.write(fileName, DemoData.class).build()) {
        WriteSheet writeSheet = EasyExcel.writerSheet("sheet1").build();
        
        // 1. 先写入正常的数据列表
        List<DemoData> dataList = getData();
        excelWriter.write(dataList, writeSheet);
        
        // 2. 创建最后一行的汇总数据
        List<List<String>> lastRowList = new ArrayList<>();
        List<String> lastRow = new ArrayList<>();
        lastRow.add("合计");  // 第一列
        lastRow.add("100");   // 第二列
        lastRow.add("200");   // 第三列
        lastRowList.add(lastRow);
        
        // 3. 写入最后一行
        excelWriter.write(lastRowList, writeSheet);
    }
}
```

2. **进阶实现：添加样式**
```java
public class LastRowWriteHandler implements RowWriteHandler {
    private int totalRows;
    
    public LastRowWriteHandler(int totalRows) {
        this.totalRows = totalRows;
    }

    @Override
    public void afterRowDispose(RowWriteHandlerContext context) {
        // 判断是否是最后一行
        if (context.getRow().getRowNum() == totalRows) {
            Row row = context.getRow();
            CellStyle style = context.getWriteWorkbookHolder().getWorkbook().createCellStyle();
            // 设置背景色为灰色
            style.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
            style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
            // 设置字体加粗
            Font font = context.getWriteWorkbookHolder().getWorkbook().createFont();
            font.setBold(true);
            style.setFont(font);
            
            // 为最后一行的每个单元格设置样式
            for (Cell cell : row) {
                cell.setCellStyle(style);
            }
        }
    }
}
```

3. **完整示例**
```java
@Test
public void writeWithStyledLastRow() {
    String fileName = "测试导出.xlsx";
    List<DemoData> dataList = getData();
    
    try (ExcelWriter excelWriter = EasyExcel.write(fileName, DemoData.class)
            .registerWriteHandler(new LastRowWriteHandler(dataList.size() + 1))  // +1是因为有表头
            .build()) {
        WriteSheet writeSheet = EasyExcel.writerSheet("sheet1").build();
        
        // 写入数据
        excelWriter.write(dataList, writeSheet);
        
        // 写入汇总行
        List<List<String>> lastRowList = new ArrayList<>();
        List<String> lastRow = new ArrayList<>();
        lastRow.add("合计");
        lastRow.add(calculateTotal(dataList));  // 计算汇总值
        lastRowList.add(lastRow);
        
        excelWriter.write(lastRowList, writeSheet);
    }
}

// 计算汇总值的辅助方法
private String calculateTotal(List<DemoData> dataList) {
    return dataList.stream()
            .map(DemoData::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add)
            .toString();
}
```

4. **注意事项**
- 需要使用ExcelWriter而不是简单的doWrite方法
- 最后一行数据需要单独构造List<List<String>>格式
- 可以通过RowWriteHandler自定义最后一行样式
- 注意关闭ExcelWriter资源
- 如果有合并单元格需求，可以使用CellWriteHandler 