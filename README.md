
# Mohsen Masoudi Moghadam - Bussiness Intelligence & Data Analysis Specialist

📧 Email: [mohsenmoghadam277@gmail.com](mailto:mohsenmoghadam277@gmail.com)  
💼 Position: Bussiness Intelligence & Data Analysis Specialist  
🌐 GitHub: [github.com/mohsenmoghadam277](https://github.com/mohsenmoghadam277)

---

## 🛠️ Skills

- Data analysis and business intelligence dashboard design  
- Advanced SQL for reporting and analytics  
- Financial and sales report optimization  
- Power BI / Tableau (interactive dashboards)

---

##📊 Sample BI Dashboard Query

# ShimaShoesOverallStoreSales 
#Actual Product Sales per Store
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



###Actual Product Sales per Store



SELECT
    invoiceitem.inventoryref AS "StoreID",
    store.name AS "StoreName",
    product.number AS "ProductCode",
    product.name AS "ProductName",
    SUM(invoiceitem.quantity) - COALESCE(SUM(returninvoiceitem.quantity),0) AS "ActualSoldQuantity"
FROM rahkaran.invoiceitem
JOIN rahkaran.invoice
    ON invoiceitem.documentref = invoice.invoiceid
   AND (invoice.status IS NULL OR invoice.status NOT IN (2,3,4))
JOIN rahkaran.store
    ON invoiceitem.inventoryref = store.storeid
JOIN rahkaran.product
    ON invoiceitem.productref = product.productid
LEFT JOIN rahkaran.returninvoiceitem
    ON returninvoiceitem.productref = product.productid
   AND returninvoiceitem.inventoryref = invoiceitem.inventoryref
LEFT JOIN rahkaran.returninvoice
    ON returninvoiceitem.documentref = returninvoice.returninvoiceid
   AND (returninvoice.status IS NULL OR returninvoice.status NOT IN (2,3,4))
WHERE {{store}} IS NULL OR store.name ILIKE '%' || {{store}} || '%'
GROUP BY invoiceitem.inventoryref, store.name, product.number, product.name
ORDER BY invoiceitem.inventoryref, product.number;


###Store surplus inventory



WITH sales_data AS (
    SELECT
        invoiceitem.inventoryref AS storeid,
        store.name AS storename,
        product.number AS partcode,
        SUM(invoiceitem.quantity)::numeric - COALESCE(SUM(returninvoiceitem.quantity),0)::numeric AS real_sales_quantity
    FROM rahkaran.invoiceitem
    JOIN rahkaran.invoice
        ON invoiceitem.documentref = invoice.invoiceid
       AND (invoice.status IS NULL OR invoice.status NOT IN (2,3,4))
       [[AND {{the_day_invoice}}]]
    JOIN rahkaran.store
        ON invoiceitem.inventoryref = store.storeid
    JOIN rahkaran.product
        ON invoiceitem.productref = product.productid
    LEFT JOIN rahkaran.returninvoiceitem
        ON returninvoiceitem.productref = product.productid
       AND returninvoiceitem.inventoryref = invoiceitem.inventoryref
    LEFT JOIN rahkaran.returninvoice
        ON returninvoiceitem.documentref = returninvoice.returninvoiceid
       AND (returninvoice.status IS NULL OR returninvoice.status NOT IN (2,3,4))
       [[AND {{the_day_returninvoice}}]]
    WHERE {{store}} IS NULL OR store.name ILIKE '%' || {{store}} || '%'
    GROUP BY invoiceitem.inventoryref, store.name, product.number
),
total_mojodi_per_product AS (
    SELECT
        storename,
        LEFT(partcode, LENGTH(partcode)-2) AS base_code,
        SUM(mojodi)::numeric AS total_mojodi
    FROM rahkaran.partmojodi
    GROUP BY storename, LEFT(partcode, LENGTH(partcode)-2)
),
size_distribution_per_product AS (
    SELECT
        storename,
        LEFT(partcode, LENGTH(partcode)-2) AS base_code,
        STRING_AGG(CONCAT(RIGHT(partcode,2), ':', CAST(mojodi AS INTEGER)), ' | ') AS distribution_text,
        COUNT(*) AS size_count
    FROM rahkaran.partmojodi
    GROUP BY storename, LEFT(partcode, LENGTH(partcode)-2)
),
voucher_data AS (
    SELECT
        storeref,
        partref,
        SUM(CASE WHEN inventoryvoucherspecificationref = 17 THEN quantity ELSE 0 END) AS open_quantity,
        SUM(CASE WHEN inventoryvoucherspecificationref = 68 THEN quantity ELSE 0 END) AS received_quantity
    FROM rahkaran.inventoryvoucheritem
    GROUP BY storeref, partref
),
computed AS (
    SELECT
        pm.storename,
        pm.partcode,
        pm.mojodi,
        pm.partname AS part_name,
        COALESCE(sd.real_sales_quantity,0) AS real_sales,
        COALESCE(tmp.total_mojodi,0) AS total_mojodi_numeric,
        sdp.distribution_text,
        sdp.size_count,
        GREATEST(pm.mojodi - COALESCE(sd.real_sales_quantity,0), 0) AS remaining_stock_safe,
        CEIL(COALESCE(sd.real_sales_quantity,0)/NULLIF(sdp.size_count,0)) AS min_per_size,
        COALESCE(vd.open_quantity,0) AS open_vouchers,
        COALESCE(vd.received_quantity,0) AS received_vouchers
    FROM rahkaran.partmojodi pm
    JOIN rahkaran.part p
        ON pm.partcode = p.code
    LEFT JOIN sales_data sd
        ON TRIM(LOWER(pm.storename)) = TRIM(LOWER(sd.storename))
       AND pm.partcode = sd.partcode
    LEFT JOIN total_mojodi_per_product tmp
        ON pm.storename = tmp.storename
       AND LEFT(pm.partcode, LENGTH(pm.partcode)-2) = tmp.base_code
    LEFT JOIN size_distribution_per_product sdp
        ON pm.storename = sdp.storename
       AND LEFT(pm.partcode, LENGTH(pm.partcode)-2) = sdp.base_code
    LEFT JOIN voucher_data vd
        ON pm.storename = (SELECT name FROM rahkaran.store WHERE storeid = vd.storeref LIMIT 1)
       AND pm.partcode = (SELECT number FROM rahkaran.product WHERE productid = vd.partref LIMIT 1)
    WHERE pm.mojodi > 0
      AND ({{store}} IS NULL OR pm.storename ILIKE '%' || {{store}} || '%')
      AND (
          ({{mazaad_level}} = 'زیاد' AND GREATEST(pm.mojodi - COALESCE(sd.real_sales_quantity,0), 0) > 30)
          OR ({{mazaad_level}} = 'متوسط' AND GREATEST(pm.mojodi - COALESCE(sd.real_sales_quantity,0), 0) BETWEEN 10 AND 30)
          OR ({{mazaad_level}} = 'کم' AND GREATEST(pm.mojodi - COALESCE(sd.real_sales_quantity,0), 0) < 10)
          OR {{mazaad_level}} IS NULL
      )
)
SELECT
    storename AS "نام فروشگاه",
    REGEXP_REPLACE(part_name, '\s*سایز\s*\d+$','', 'g') AS "عنوان کالا بدون سایز",
    part_name AS "عنوان کالا با سایز",
    LEFT(partcode, LENGTH(partcode)-2) AS "کد پایه کالا",
    partcode AS "کد کامل کالا",
    mojodi AS "موجودی سایز فعلی",
    real_sales AS "تعداد فروش این سایز",
    total_mojodi_numeric AS "موجودی کلی کالا بدون تفکیک سایز",
    distribution_text AS "توزیع موجودی به تفکیک سایز",
    remaining_stock_safe AS "اختلاف موجودی از سایز فعلی",
    GREATEST(total_mojodi_numeric - real_sales,0) AS "اختلاف موجودی از فروش",
    size_count AS "تعداد سایز موجود",
    min_per_size AS "حداقل موجودی مورد انتظار برای هر سایز",
    GREATEST(remaining_stock_safe - min_per_size,0) AS "مازاد مرحله‌ای سایز",
    open_vouchers AS "حواله باز",
    received_vouchers AS "سند رسیده انبار",
    CASE
        WHEN remaining_stock_safe > 30 THEN ' مازاد زیاد (' || CAST(remaining_stock_safe AS INTEGER) || ')'
        WHEN remaining_stock_safe BETWEEN 10 AND 30 THEN ' مازاد متوسط (' || CAST(remaining_stock_safe AS INTEGER) || ')'
        ELSE ' مازاد کم (' || CAST(remaining_stock_safe AS INTEGER) || ')'
    END AS "وضعیت مازاد موجودی"
FROM computed
ORDER BY storename, part_name;




