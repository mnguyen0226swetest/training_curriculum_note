# Assignment 2: Jasper Report — Intern Learning Guide

> **Audience:** Java student with coding background
> **Goal:** Understand Jasper Report, integrate it with Spring Boot, build two report APIs
> **Estimated time:** 2–3 days

---

## Overview: What is This Assignment Actually Asking?

You need to:
1. **Understand** what Jasper Report is and how it generates reports
2. **Draw** a diagram showing how Jasper integrates with Spring Boot
3. **Build** a sample report with 2 data categories, exportable as PDF and XLSX
4. **Build** an API that queries PostgreSQL directly inside Jasper
5. **Write** sample code for the full Spring Boot + Jasper setup

---

## Day-by-Day Learning Plan

---

### Day 1 — Understand Jasper Core Concepts + Jasper Studio

**What to learn:**

#### 1. What is Jasper Report?
- JasperReports is a **Java-based open-source reporting library**
- It takes a **template file** (`.jrxml`) + **data** → outputs a formatted report (PDF, XLSX, HTML, etc.)
- Think of it like a mail-merge tool: you design the layout once, feed it data, and it fills in the blanks

#### 2. The Core Workflow

```
You design template (.jrxml)
        +
You provide data (Java objects or SQL query)
        |
        v
JasperReports Engine compiles + fills the template
        |
        v
Output: PDF / XLSX / HTML / CSV
```

#### 3. Key Terms You Must Know

| Term | Simple Explanation |
|---|---|
| **`.jrxml`** | The report template file — XML format, designed in Jasper Studio |
| **`.jasper`** | The compiled binary version of `.jrxml` — faster at runtime |
| **JasperCompileManager** | Java class that compiles `.jrxml` → `.jasper` |
| **JasperFillManager** | Java class that fills the template with data |
| **JasperExportManager** | Java class that exports the filled report to PDF/XLSX/etc. |
| **JasperPrint** | The in-memory object representing a filled report |
| **Parameters** | Variables you pass into the report from Java (e.g., title, date range) |
| **Fields** | Columns mapped from your datasource (e.g., `$F{name}`, `$F{price}`) |
| **Band** | A horizontal section in the report (Title, Page Header, Column Header, Detail, Summary) |
| **Datasource** | Where the data comes from — Java collection or direct SQL/JDBC |
| **Subreport** | A report embedded inside another report (used for nested data) |

#### 4. What is Jasper Studio?
- Jasper Studio is a **visual drag-and-drop designer** for `.jrxml` templates
- It's a free Eclipse-based desktop app
- You design tables, headers, charts visually — no need to hand-write XML

**Download:** https://community.jaspersoft.com/download-jaspersoft/

**Why learn this first?** Every API you write depends on `.jrxml` templates. Understanding what the template contains tells you what data your Java code needs to provide.

---

### Day 2 — Spring Boot Integration + Build the Two Report APIs

**What to learn:**

#### 1. Add JasperReports to Spring Boot

Add to `pom.xml`:
```xml
<dependency>
    <groupId>net.sf.jasperreports</groupId>
    <artifactId>jasperreports</artifactId>
    <version>6.21.0</version>
</dependency>

<!-- For XLSX export -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.2.3</version>
</dependency>
```

Place your `.jrxml` files in:
```
src/main/resources/reports/my_report.jrxml
```

#### 2. How the Integration Works

```
HTTP Request (GET /report/pdf)
        |
        v
Spring Boot Controller
        |
        v
Load .jrxml from resources/
        |
        v
Compile → JasperReport object
        |
        v
Fill with data (Java List or JDBC Connection)
        |
        v
Export to PDF or XLSX bytes
        |
        v
Return as HTTP response with correct Content-Type
```

#### 3. The Core Java Code Pattern

This is the pattern you will reuse for both APIs:

```java
@Service
public class ReportService {

    public byte[] generatePdf(List<MyData> dataList) throws Exception {
        // Step 1: Load template
        InputStream templateStream = getClass().getResourceAsStream("/reports/my_report.jrxml");

        // Step 2: Compile template
        JasperReport jasperReport = JasperCompileManager.compileReport(templateStream);

        // Step 3: Prepare data
        JRBeanCollectionDataSource dataSource = new JRBeanCollectionDataSource(dataList);

        // Step 4: Pass parameters (optional, e.g. title, date)
        Map<String, Object> parameters = new HashMap<>();
        parameters.put("ReportTitle", "My Report");

        // Step 5: Fill the report
        JasperPrint jasperPrint = JasperFillManager.fillReport(jasperReport, parameters, dataSource);

        // Step 6: Export to PDF bytes
        return JasperExportManager.exportReportToPdf(jasperPrint);
    }
}
```

**Why understand this pattern?** Both APIs (PDF/XLSX export and PostgreSQL datasource) use this same flow — only Step 3 and Step 6 differ.

---

#### API 1: Export Report as PDF and XLSX

**Two categories means your report has two sections/groups of data.**
Example: Category A = Products, Category B = Orders

**Controller:**
```java
@RestController
@RequestMapping("/api/report")
public class ReportController {

    @Autowired
    private ReportService reportService;

    // Export as PDF
    @GetMapping(value = "/pdf", produces = MediaType.APPLICATION_PDF_VALUE)
    public ResponseEntity<byte[]> exportPdf() throws Exception {
        byte[] pdfBytes = reportService.generatePdf();
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=report.pdf")
                .body(pdfBytes);
    }

    // Export as XLSX
    @GetMapping(value = "/xlsx", produces = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")
    public ResponseEntity<byte[]> exportXlsx() throws Exception {
        byte[] xlsxBytes = reportService.generateXlsx();
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=report.xlsx")
                .body(xlsxBytes);
    }
}
```

**XLSX export (different exporter class):**
```java
public byte[] generateXlsx(List<MyData> dataList) throws Exception {
    // Steps 1–5 same as PDF...

    // Step 6: Export to XLSX instead
    ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
    JRXlsxExporter exporter = new JRXlsxExporter();
    exporter.setExporterInput(new SimpleExporterInput(jasperPrint));
    exporter.setExporterOutput(new SimpleOutputStreamExporterOutput(outputStream));
    exporter.exportReport();
    return outputStream.toByteArray();
}
```

**Key difference:** PDF uses `JasperExportManager`, XLSX uses `JRXlsxExporter`. Same data, different output class.

---

#### API 2: Report with Direct PostgreSQL Query (JDBC Datasource)

**What it means:** Instead of passing Java objects, you let Jasper run the SQL query itself using a database connection.

**Why use this?** Simpler for pure reporting — no need to map DB rows to Java objects. Jasper handles the query + display directly.

**The `.jrxml` template defines the SQL query inside it:**
```xml
<queryString language="SQL">
    <![CDATA[SELECT id, name, amount FROM orders WHERE status = $P{status}]]>
</queryString>
```

**Java code — pass a JDBC connection instead of a datasource:**
```java
public byte[] generateReportFromDb(String status) throws Exception {
    // Step 1-2: Load and compile (same as before)
    InputStream templateStream = getClass().getResourceAsStream("/reports/db_report.jrxml");
    JasperReport jasperReport = JasperCompileManager.compileReport(templateStream);

    // Step 3: Pass parameters (used inside the SQL query)
    Map<String, Object> parameters = new HashMap<>();
    parameters.put("status", status);

    // Step 4: Use JDBC connection directly — Jasper runs the SQL
    Connection connection = dataSource.getConnection();
    JasperPrint jasperPrint = JasperFillManager.fillReport(jasperReport, parameters, connection);
    connection.close();

    // Step 5: Export
    return JasperExportManager.exportReportToPdf(jasperPrint);
}
```

**Spring Boot datasource config (`application.yml`):**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/your_db
    username: postgres
    password: your_password
    driver-class-name: org.postgresql.Driver
```

**Add PostgreSQL driver to `pom.xml`:**
```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

**Why learn both datasource approaches?**
- `JRBeanCollectionDataSource` = data already in memory (Java list) → API 1
- `Connection` (JDBC) = data fetched from DB by Jasper itself → API 2
These are the two most common patterns in real projects.

---

### Day 3 — Design the `.jrxml` Template + Document + Polish

**What to learn:**

#### 1. Design Your Report Template in Jasper Studio

A report has bands (sections). For a 2-category report:

```
┌────────────────────────────────┐
│           TITLE BAND           │  ← Report title, logo, date
├────────────────────────────────┤
│         PAGE HEADER BAND       │  ← Printed on every page
├────────────────────────────────┤
│        COLUMN HEADER BAND      │  ← Table column names (ID, Name, Amount)
├────────────────────────────────┤
│           DETAIL BAND          │  ← Repeats for each data row  ← $F{name}
├────────────────────────────────┤
│          GROUP FOOTER          │  ← Subtotal per category
├────────────────────────────────┤
│           SUMMARY BAND         │  ← Grand total at bottom
└────────────────────────────────┘
```

**To make 2 categories (groups) in Jasper Studio:**
1. Right-click your report → Add Group
2. Group by a field (e.g., `category`)
3. Jasper automatically splits the Detail band per group

#### 2. Minimal Working `.jrxml` (hand-written example)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jasperReport xmlns="http://jasperreports.sourceforge.net/jasperreports"
              name="SimpleReport" pageWidth="595" pageHeight="842"
              columnWidth="535" leftMargin="20" rightMargin="20"
              topMargin="20" bottomMargin="20">

    <!-- Fields from your datasource -->
    <field name="name" class="java.lang.String"/>
    <field name="amount" class="java.lang.Double"/>
    <field name="category" class="java.lang.String"/>

    <!-- Parameters from Java -->
    <parameter name="ReportTitle" class="java.lang.String"/>

    <!-- Title Section -->
    <title>
        <band height="50">
            <textField>
                <reportElement x="0" y="0" width="535" height="30"/>
                <textFieldExpression><![CDATA[$P{ReportTitle}]]></textFieldExpression>
            </textField>
        </band>
    </title>

    <!-- Data Row -->
    <detail>
        <band height="20">
            <textField>
                <reportElement x="0" y="0" width="200" height="20"/>
                <textFieldExpression><![CDATA[$F{name}]]></textFieldExpression>
            </textField>
            <textField>
                <reportElement x="200" y="0" width="100" height="20"/>
                <textFieldExpression><![CDATA[$F{amount}]]></textFieldExpression>
            </textField>
        </band>
    </detail>
</jasperReport>
```

#### 3. Document Structure for Your Word/PDF Deliverable

```
1. Introduction — What is JasperReports and what problems it solves
2. Architecture Diagram — Spring Boot + Jasper + PostgreSQL
3. Jasper Studio Setup — How to install and create a .jrxml template
4. API 1: Export PDF/XLSX — Code + explanation + screenshots
5. API 2: PostgreSQL Datasource — Code + explanation + screenshots
6. Sample Report Screenshots
7. References
```

#### 4. Architecture Diagram to Draw (Draw.io)

```
Angular / Postman
      |
      | HTTP GET /api/report/pdf
      v
Spring Boot Controller
      |
      v
ReportService
      |
      |── loads .jrxml template (from resources/)
      |── compiles to JasperReport
      |── fills with data ──► JRBeanCollectionDataSource (API 1)
      |                   └─► JDBC Connection → PostgreSQL (API 2)
      |── exports to PDF / XLSX
      |
      v
Returns byte[] as HTTP response
      |
      v
Browser downloads file
```

---

## Summary Checklist

- [ ] Can explain what JasperReports does and why it's used
- [ ] Know the 10 key terms: `.jrxml`, `.jasper`, Fill, Compile, Export, Band, Field, Parameter, Datasource, JasperPrint
- [ ] Jasper Studio installed and can create a basic template visually
- [ ] Spring Boot project set up with `jasperreports` + `poi-ooxml` dependencies
- [ ] `.jrxml` template designed with 2 data categories (groups)
- [ ] API 1 working: Returns PDF and XLSX from Java list data
- [ ] API 2 working: Returns PDF using direct PostgreSQL JDBC connection
- [ ] Architecture diagram drawn in Draw.io
- [ ] Document written (Word or PDF)
- [ ] Video demo recorded (optional but recommended)

---

## Quick Tips

- **Design the template in Jasper Studio first** — trying to hand-write `.jrxml` XML is painful
- **Test with hardcoded Java data first** (`JRBeanCollectionDataSource`) before connecting to PostgreSQL
- **`$F{fieldName}`** = data from datasource, **`$P{paramName}`** = parameter from Java, **`$V{variableName}`** = computed value (e.g. sum) — memorize these 3 expression types
- **PDF needs no extra dependency**, XLSX needs `poi-ooxml` — if you get a class not found error on XLSX export, that's why
- **Compile once, fill many times** — in production you cache the `.jasper` object, don't recompile on every request
- **The `.jrxml` file goes in `src/main/resources/`** so Spring Boot packages it into the JAR automatically
