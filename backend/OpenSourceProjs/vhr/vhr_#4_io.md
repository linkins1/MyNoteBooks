## #4.POI-文件上传与下载

### #4.1 上传

> 前端发来`/employee/basic/import`请求

#### #4.1.1 Controller

```java
@PostMapping("/import")
public RespBean importData(MultipartFile file) throws IOException {
    List<Employee> list = POIUtils.excel2Employee(file, nationService.getAllNations(), politicsstatusService.getAllPoliticsstatus(), departmentService.getAllDepartmentsWithOutChildren(), positionService.getAllPositions(), jobLevelService.getAllJobLevels());
    if (employeeService.addEmps(list) == list.size()) {
        return RespBean.ok("上传成功");
    }
    return RespBean.error("上传失败");
}
```

这里SpringMVC会传来已经解析好的MultipartFile对象，包含了HTTP报文实体（包含实体首部和实体主体）

#### #4.1.2 POIUtils

```java
public static List<Employee> excel2Employee(MultipartFile file, List<Nation> allNations, List<Politicsstatus> allPoliticsstatus, List<Department> allDepartments, List<Position> allPositions, List<JobLevel> allJobLevels) {
    List<Employee> list = new ArrayList<>();
    Employee employee = null;
    try {
        //1. 创建一个 workbook 对象
        HSSFWorkbook workbook = new HSSFWorkbook(file.getInputStream());
        //2. 获取 workbook 中表单的数量
        int numberOfSheets = workbook.getNumberOfSheets();
        for (int i = 0; i < numberOfSheets; i++) {
            //3. 获取表单
            HSSFSheet sheet = workbook.getSheetAt(i);
            //4. 获取表单中的行数
            int physicalNumberOfRows = sheet.getPhysicalNumberOfRows();
            for (int j = 0; j < physicalNumberOfRows; j++) {
                //5. 跳过标题行
                if (j == 0) {
                    continue;//跳过标题行
                }
                //6. 获取行
                HSSFRow row = sheet.getRow(j);
                if (row == null) {
                    continue;//防止数据中间有空行
                }
                //7. 获取列数
                int physicalNumberOfCells = row.getPhysicalNumberOfCells();
                employee = new Employee();
                int k;
                for (k = 1; k < physicalNumberOfCells; k++) {
                    //从左至右获取单元格的值，且跳过0列，0列是序号列，序号在插入数据库时主键自增
                    HSSFCell cell = row.getCell(k);
                    if (cell == null) {
                        logger.error("表格中第" + k + 1 + "列的员工信息未填写，不予添加");
                        break;
                    }
                    switch (cell.getCellType()) {
                        case STRING:
                            String cellValue = cell.getStringCellValue();
                            switch (k) {
                                case 1:
                                    /**
                                       此处省略为emp设定值的部分
                                    */
                            }
                        break;
                    }
                }
                if (k == physicalNumberOfCells)
                    list.add(employee);
            }
        }

    } catch (IOException e) {
        e.printStackTrace();
    }
    return list;
}
```

步骤如下

- 通过apache提供的HSSFWorkbook构造函数来传入MultipartFile的字节流，构造出一个表格对象
- getNumberOfSheets获取表格中的表单数
  - getPhysicalNumberOfRows获取表单中的行数（包含空行--只要行中有一个单元格不是空就不是空行）
  - getRow获取对应行
    - getPhysicalNumberOfCells获取单元格数量，也即列数

每次创建一个空的Employee对象并设定数值，之后添加到list中，返回的list调用添加方法插入到数据库中

此外，在遇到空单元格时，会跳过构造且不会添加到list中

> 要注意添加时没有使用发送邮件的方法，而是直接插入到数据库中
>
> ```java
> public Integer addEmps(List<Employee> list) {
>     return employeeMapper.addEmps(list);
> }
> ```

### #4.2 下载

>前端发来`/employee/basic/export`请求

#### #4.2.1 Controller

```java
@GetMapping("/export")
public ResponseEntity<byte[]> exportData() {
    List<Employee> list = (List<Employee>) employeeService.getEmployeeByPage(null, null, new Employee(),null).getData();
    return POIUtils.employee2Excel(list);
}
```

首先从数据库中查出所有的员工信息存为list，之后调用POIUtils来形成ResponseEntity对象

> ResponseEntity继承于HttpEntity，添加了HttpStatus status code属性，可以用于RestTemplate和Controller方法中
>
> 使用构造器时会调用HttpEntity构造器来构造对象，其中报文实体的类型可以通过泛型来给定，这里设置为字节数组
>
> 使用ResponseEntity必须给定状态码

#### #4.2.2 POIUtils

```java
public static ResponseEntity<byte[]> employee2Excel(List<Employee> list) {
    //1. 创建一个 Excel 文档
    HSSFWorkbook workbook = new HSSFWorkbook();
    //2. 创建文档摘要
    workbook.createInformationProperties();
    //3. 获取并配置文档信息
    DocumentSummaryInformation docInfo = workbook.getDocumentSummaryInformation();
    //文档类别
    docInfo.setCategory("员工信息");
    //文档管理员
    docInfo.setManager("javaboy");
    //设置公司信息
    docInfo.setCompany("www.javaboy.org");
    //4. 获取文档摘要信息
    SummaryInformation summInfo = workbook.getSummaryInformation();
    //文档标题
    summInfo.setTitle("员工信息表");
    //文档作者
    summInfo.setAuthor("javaboy");
    // 文档备注
    summInfo.setComments("本文档由 javaboy 提供");
    //5. 创建样式
    //创建标题行的样式
    HSSFCellStyle headerStyle = workbook.createCellStyle();
    headerStyle.setFillForegroundColor(IndexedColors.YELLOW.index);
    headerStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);
    HSSFCellStyle dateCellStyle = workbook.createCellStyle();
    dateCellStyle.setDataFormat(HSSFDataFormat.getBuiltinFormat("m/d/yy"));
    HSSFSheet sheet = workbook.createSheet("员工信息表");
    //设置列的宽度
    sheet.setColumnWidth(0, 5 * 256);
    sheet.setColumnWidth(1, 12 * 256);
    /**
    省略
    */
    //6. 创建标题行
    HSSFRow r0 = sheet.createRow(0);
    HSSFCell c0 = r0.createCell(0);
    c0.setCellValue("编号");
    c0.setCellStyle(headerStyle);
    HSSFCell c1 = r0.createCell(1);
    c1.setCellStyle(headerStyle);
    c1.setCellValue("姓名");
        /**
        省略...
        */
    for (int i = 0; i < list.size(); i++) {
        Employee emp = list.get(i);
        HSSFRow row = sheet.createRow(i + 1);
        row.createCell(0).setCellValue(emp.getId());
        row.createCell(1).setCellValue(emp.getName());
        row.createCell(2).setCellValue(emp.getWorkID());
        row.createCell(3).setCellValue(emp.getGender());
        HSSFCell cell4 = row.createCell(4);
        cell4.setCellStyle(dateCellStyle);
        cell4.setCellValue(emp.getBirthday());
        /**
        省略...
        */
    }

    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    HttpHeaders headers = new HttpHeaders();
    try {
        headers.setContentDispositionFormData("attachment", new String("员工表.xls".getBytes("UTF-8"), "ISO-8859-1"));
        headers.setContentType(MediaType.APPLICATION_OCTET_STREAM);
        workbook.write(baos);
    } catch (IOException e) {
        e.printStackTrace();
    }
    return new ResponseEntity<byte[]>(baos.toByteArray(), headers, HttpStatus.CREATED);
}
```

这里首先会创建一个HSSFWorkbook对象，先创建好表格的表头，之后遍历list创建每一条员工记录。

由于要在头部中添加`content-disposition=attachment;filename=xxx.xls`字段来表示内容的类型是附件且以弹窗形式给出。所以需要调用`headers.setContentDispositionFormData`方法来设定内容类型，此外为了避免文件名乱码，在给出文件名的时候需要指定编码方式来构造字符串作为文件名

