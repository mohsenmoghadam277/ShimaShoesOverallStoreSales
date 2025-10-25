# ShimaShoesOverallStoreSales 
# Mohsen Masoudi Moghadam - Business Intelligence & Data Analysis Specialist

📧 Email: [mohsenmoghadam277@gmail.com](mailto:mohsenmoghadam277@gmail.com)  
💼 Position: Business Intelligence & Data Analysis Specialist  
🌐 GitHub: [github.com/mohsenmoghadam277](https://github.com/mohsenmoghadam277)

---

## 🛠️ Skills

- Data analysis and business intelligence dashboard design  
- Advanced SQL for reporting and analytics  
- Financial and sales report optimization  
- Power BI / Tableau (interactive dashboards)

---

## 📊 Sample BI Dashboard Query

Example SQL query to calculate net sales for each store:

```sql
SELECT
    "rahkaran"."store"."storeid",
    "rahkaran"."store"."name" AS "StoreName",
    COALESCE(SUM("invoice_summary"."InvoiceNetPrice"), 0)
      - COALESCE(SUM("return_summary"."ReturnNetPrice"), 0) AS "NetSales",
    (
        COALESCE(SUM("invoice_summary"."InvoiceNetPrice"), 0)
        - COALESCE(SUM("return_summary"."ReturnNetPrice"), 0)
    ) / 10 AS "NetSalesDividedBy10"
FROM "rahkaran"."store"
LEFT JOIN (
    SELECT
        "inventoryref",
        SUM("netprice") AS "InvoiceNetPrice"
    FROM "rahkaran"."invoice"
    WHERE "status" = 1
      [[AND {{the_day_invoice}}]]
    GROUP BY "inventoryref"
) AS "invoice_summary"
    ON "invoice_summary"."inventoryref" = "rahkaran"."store"."storeid"
LEFT JOIN (
    SELECT
        "inventoryref",
        SUM("netprice") AS "ReturnNetPrice"
    FROM "rahkaran"."returninvoice"
    WHERE "status" = 1
      [[AND {{the_day_returninvoice}}]]
    GROUP BY "inventoryref"
) AS "return_summary"
    ON "return_summary"."inventoryref" = "rahkaran"."store"."storeid"
WHERE 1 = 1
  [[AND "rahkaran"."store"."name" ILIKE '%' || {{store}} || '%']]
  AND "rahkaran"."store"."name" NOT ILIKE '%(بلا)%'
  AND "rahkaran"."store"."name" NOT ILIKE '%اتکا%'
GROUP BY "rahkaran"."store"."storeid", "rahkaran"."store"."name";
