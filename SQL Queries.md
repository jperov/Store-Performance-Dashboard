# SQL Queries


## Schema
Queries were performed on the table YTD2025:


<img src="images/dashboard_schema.PNG" width="295" height="320" />


<br>


## Queries
The following queries were used to aggregate data used in the dashboard

<br>

### Total Sales & Tickets Per Store & Day



```SQL
--Gathers total ticket count per location & day. Ignores tickets from online orders.
tickets AS (
  SELECT
    date, location,
    COUNT(DISTINCT Ticket_Id) AS ticketcount
  FROM YTD2025
  WHERE LOWER(payment_methods) <> 'ecommerce payment' OR payment_methods IS NULL
  GROUP BY date, location
)


--Gathers total sales and tickets per location & day.
SELECT
  location,
  date,
  MAX(ticketcount) as ticketcount,
  ROUND(SUM(Net_sales),2) as sales
FROM
YTD2025 
  LEFT JOIN tickets USING(location, date)
GROUP BY location, date
ORDER BY location, date

```


Example Output:

<img src="images/Daily_Sales_Example_Output.PNG" width="420" height="450" />

<br>


### Top 5 Selling Categories Per Store & Month


```SQL
--total sales per category, location and month
Totals AS(
  SELECT
    Location,
    DATE_TRUNC(date, MONTH) AS month,
    Category,
    SUM(Net_sales) AS CatSales
  FROM YTD2025 
  GROUP BY location, month, Category
),

--Assigns ranking to categories based on sales.
Ranking AS(
  SELECT
    Location,
    Month,
    Category,
    CatSales,
    ROW_NUMBER() OVER(PARTITION BY location, month ORDER BY CatSales DESC) AS CategoryRank
  FROM
    totals
)

--Displays only the top 5 categories.
SELECT
  Location,
  Month,
  Category,
  CatSales,
  CategoryRank AS CatRrank
FROM
  ranking
WHERE CategoryRank < 6
ORDER BY location, month, CategoryRank

```



Example Output

<img src="images/Top_Categories_Example_Output.PNG" width="510" height="530" />

<br>

### Top 10 Selling Products Per Store & Month


```SQL

--total sales per category, location and month
Totals AS(
  SELECT
    Location,
    DATE_TRUNC(date, MONTH) AS month,
    Product_name,
    SUM(Net_sales) AS ProductSales
  FROM YTD2025
  GROUP BY location, month, product_name
),



-- Assigns a ranking to products based on sales totals per store and month
ProductRankings as(
  SELECT
    Location,
    Month,
    Product_name,
    ProductSales,
    ROW_NUMBER() OVER(PARTITION BY location, month ORDER BY ProductSales DESC) AS ProductRank
  FROM
    totals
)



--Filters to show only the top 10 products
SELECT
  Location,
  Month,
  Product_name,
  ROUND(ProductSales,2) AS ProductSales,
  ProductRank
FROM ProductRankings
WHERE Product_Rank < 11
ORDER BY location, month, Product_Rank


```



Example Output

<img src="images/Top_Products_Example_Output.PNG" width="600" height="620" />

<br>

### Metrics Per Store & Month

```SQL

-- Gathers store and day totals for tickets excluding ecommerce transactions. 
WITH Tickets AS(
  SELECT
    date,
    location,
    COUNT(DISTINCT ticket_id) AS ticketcount
  FROM YTD2025
  WHERE
    LOWER(payment_methods) <> 'ecommerce payment' OR payment_methods IS NULL
  GROUP BY date, location
),


DailySales AS (
  SELECT
    date, Location,
    ROUND(SUM(Net_sales), 2) AS Sales
  FROM YTD2025
  GROUP BY date, Location
),



net_sales_per_store AS (
  SELECT
    Location,
    Date,
    ROUND(SUM(Net_sales), 2) AS Net_Sales
  FROM YTD2025
  WHERE product_name <> 'Bag_Fee'
  GROUP BY Date, Location
),



-- Calculates AOV per store and day.
avg_ticket_totals AS (
  SELECT
    Location,
    date,
    ROUND(
    fs.Sales / NULLIF(ticketcount,0),2 ) AS AOV
  FROM tickets
    LEFT JOIN DailySales fs USING(location, date)
),




upt_per_store AS (
  SELECT
    Location,
    Date,
    ROUND(SUM(quantity) / NULLIF(ticketcount,0),2) AS UPT
    FROM YTD2025
      LEFT JOIN 
      tickets t USING(location, date)
  WHERE
    (LOWER(payment_methods) <> 'ecommerce payment' OR payment_methods IS NULL)
    AND
    (product_name <> 'Bag_Fee')
  GROUP BY
     Date, Location, Ticketcount
),


--The following table calculates total quantities sold of key products and upsell metrics.
SalesByCategory AS(
  SELECT
    date,
    Location,
    -- Hat categories
      SUM(CASE WHEN category IN ('VISOR', 'SNAPBACK', 'ROPER', 'A_FRAME', 'BUCKET', 'HATS',
      'UNSTRUCTURED_ADJ', 'FITTED', 'FLEXONE_FITS', 'STRUCTURED_ADJ') 
      THEN Quantity ELSE 0 END) AS Hats,


    -- Hat Sprays
    SUM(CASE
          WHEN LOWER(product_name) IN('hat_spray', 'hat_spray_steam_bundle') THEN quantity
          WHEN Product_name = 'Jersey_Spray' THEN Quantity * 5
          WHEN Product_name = '12oz_Repelwell'
          OR product_name = 'DetraPel' THEN Quantity * 20
          ELSE 0
        END) AS Hat_Sprays,



    -- Hat steams, pins and accessories.
    SUM(CASE WHEN LOWER(product_name) IN('steamcurve', 'hat_spray_steam_bundle') THEN quantity ELSE 0 END) AS Hat_Steams,
    SUM(CASE WHEN Category = 'HAT_PINS' THEN Quantity ELSE 0 END) AS Hat_Pins,
    SUM(CASE WHEN Category = 'HAT_ACCESSORIESADD-ONS' THEN Quantity ELSE 0 END) AS Hat_Accessories,



    -- Calculating Total Accessories
    (
      SUM(CASE
          WHEN LOWER(product_name) IN('hat_spray', 'hat_spray_steam_bundle') THEN quantity
          WHEN Product_name = 'Jersey_Spray' THEN Quantity * 5
          WHEN Product_name = '12oz_Repelwell'
          OR product_name = 'DetraPel' THEN Quantity * 20
          ELSE 0 END)
      +
      SUM(CASE WHEN LOWER(product_name) IN('steamcurve', 'hat_spray_steam_bundle') THEN quantity ELSE 0 END)
      +
      SUM(CASE WHEN Category = 'HAT_PINS' THEN Quantity ELSE 0 END)
      +
      SUM(CASE WHEN Category = 'HAT_ACCESSORIESADD-ONS' THEN Quantity ELSE 0 END)
    ) AS Total_QT,



    -- Add_On % Rate
    ROUND(((
          SUM(CASE
          WHEN LOWER(product_name) IN('hat_spray', 'hat_spray_steam_bundle') THEN quantity
          WHEN Product_name = 'Jersey_Spray' THEN Quantity * 5
          WHEN Product_name = '12oz_Repelwell'
          OR product_name = 'DetraPel' THEN Quantity * 20
          ELSE 0 END)
          +
          SUM(CASE WHEN lower(product_name) IN('steamcurve', 'hat_spray_steam_bundle') THEN quantity ELSE 0 END)
          +
          SUM(CASE WHEN Category = 'HAT_PINS' THEN Quantity ELSE 0 END)
          +
          SUM(CASE WHEN Category = 'HAT_ACCESSORIESADD-ONS' THEN Quantity ELSE 0 END)
        ) *100.0
        /
        NULLIF(
        SUM(CASE WHEN Category IN ('SNAPBACK', 'ROPER', 'A_FRAME', 'BUCKET', 'HATS', 'VISOR',
                                  'UNSTRUCTURED_ADJ', 'FITTED', 'FLEXONE_FITS', 'STRUCTURED_ADJ') 
        THEN Quantity ELSE 0 END),
        0) ), 4
    ) AS Add_On_Percent,



--Percentage of hats sprayed (SUM of hat sprays / SUM of hats sold).

ROUND(SUM(CASE WHEN LOWER(product_name) IN('hat_spray', 'hat_spray_steam_bundle') THEN quantity else 0 END) *100.0
/
NULLIF(SUM(CASE WHEN category IN('SNAPBACK', 'ROPER', 'A_FRAME', 'BUCKET', 'HATS', 'VISOR',
                                 'UNSTRUCTURED_ADJ', 'FITTED', 'FLEXONE_FITS', 'STRUCTURED_ADJ')
                THEN Quantity ELSE 0 END),0),2) AS Hats_Sprayed_Percent,


--Percent of jerseys sprayed (SUM of jerseys sprayed / SUM of jerseys sold).
COALESCE(ROUND(SUM(CASE WHEN product_name = 'Jersey_Spray' THEN quantity ELSE 0 END) *100.0
/
NULLIF(SUM(CASE WHEN LOWER(Product_Name)
                LIKE '%jersey%' OR LOWER(product_name) LIKE '%jerseys%'
                THEN Quantity ELSE 0 END),0),2),0) AS Jerseys_Sprayed_Percent,

-- Total Returns
SUM(CASE WHEN quantity <0 THEN quantity END) AS returns

FROM YTD2025
GROUP BY date, location
),


combined AS(
  SELECT
  sbc.location,
  sbc.date,
  Net_Sales,
  AOV,
  UPT,
  ticketcount,

  sbc.Hats,
  sbc.Hat_Sprays,
  sbc.Hat_Steams,
  sbc.Hat_Pins,
  sbc.Hat_Accessories,
  sbc.Total_QT,
  sbc.Add_On_Percent,
  Hats_Sprayed_Percent,
  Jerseys_Sprayed_Percent,
  Returns


  FROM SalesByCategory sbc
    LEFT JOIN avg_ticket_totals avg ON avg.date = sbc.date AND avg.location = sbc.location
    LEFT JOIN upt_per_store uptt ON uptt.date = sbc.date AND uptt.location = sbc.location
    LEFT JOIN net_sales_per_store net ON net.date = sbc.date AND net.location = sbc.location
    LEFT JOIN tickets t ON t.date = sbc.date AND t.location = sbc.location
),



SELECT
  Location,
  date,
  Net_Sales,
  AOV,
  UPT,
  Ticketcount,
  Hats,
  Hat_Sprays,
  Hat_Steams,
  Hat_Pins,
  Hat_Accessories,
  Total_QT,
  Add_On_Percent,
  Hats_Sprayed_Percent,
  Jerseys_Sprayed_Percent,
  Returns,

FROM combined
ORDER BY location, date;

```




Example Output

<img src="images/Monthly_Metrics_Example_Output.PNG" width="690" height="710" />







