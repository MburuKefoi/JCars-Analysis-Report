# JCARS ANALYSIS REPORT
## Project Overview
JCars Logistics imports, sells, and delivers vehicles to customers across different regions in Kenya. The company collects information about its sales transactions, vehicles, customers, branches, sales representatives, payments, deliveries, logistics costs, 
returns, cancellations, and customer experiences. We would like to use the information to better understand how the business is performing and identify areas that require attention.
 
## Project Objective
The objective of this assessment is to develop a Power BI solution that enables JCars Logistics management to understand the overall performance of the business and investigate the factors contributing to that performance. Management should be able to understand 
sales and revenue performance, costs and profitability, vehicle performance, customer performance, branch and geographic performance, sales representative performance, sales channels and lead sources, payment performance, delivery and logistics performance, 
returns and cancellations, customer experience, changes in performance over time, and unusual transactions or business areas that may require further investigation.

## Dataset
Our JCars dataset had 276 rows and 32 columns.Each of the 276 rows recorded a sale of cars with units sold telling us how many units were sold.The order ID represented each order of any sale

| Entity | Fields |
|:---------|:-------------------------|
| Customer |Customer Name,Customer Type,Customer Age |
| Vehicle  |Car Make , Car Model, Vehicle Type, Vehicle Year , Fuel Type ,Transmission, Colour, Units Sold | 
| Location |Region, County, City | 
| Branch   |Branch |
| Sales Rep|Sales Rep,Lead Source | 
| Order    |Order ID, Returned |
| Payment  |Payment Method,Payment Status| 
| Dates    |Order Date, Delivery Date |

As of now we only have order id as primary key and thus we will have to add our own for every dimension table that we will create.We are to create six dimension tables namely

- DimCustomer - The table will tell us more about our customer.Each customer will be given their own unique key that will connect them to the facts table

- DimLocations - The table will have fields like Branch, City, County and Region.The dimensional table will tell us which region yard made the order and where the customers was from.After removing duplicates we will have a unique id for each region

- DimPayment - Dimensional table payment will give us more attributes about the payment method ,was it by bank,asset financing or M-Pesa.The table will also give more insight on is the payment status complete,is it still pending and much more

- Dim Order - The Order ID field should be unique representing every order that was made.The order dates will tell us if the deliveries were on time or delayed.What is the delivery status,Was the vehicle returned

- Dim Vehicle - A dimensional table telling us more about the vehicles

## Data Quality Audit

|Issue | Columns Affected | Solution | Reason  |
|:-----|:-----------------|:---------|:--------|
| Missing IDs | Order Id | Leave them as blank | We cant delete the row just because it has a missing id because that was an order that was made neither can we give id a number as id because we will be corrupting our data |
| Date Formatting - Dates have different formats | Order Date, Delivery Date | Formatted correctly as date | Having dates formatted as date helps in certain calculations and workings like DATEDIFF | 
| Different Cases | Customer Name, Lead Source, Sales rep, Car Model, Vehicle Type, Customer Type | Formatted all columns affected into proper case | having text in the same case hepls to solve problems like duplication ensuring accurate aggregations |
| Different ways of referring to the same thing | Region, County, City, Lead Source, Car Make, Car Type, Car Model, Transmission, Colour, Payment Method | Formatted all affeted columns to have one word meaning | Really helps whe filtering data or performing calculations |
| Different Currencies | Units Selling Price, Unit Cosst, Delivery Fees, Logistics Cost | A formulae that converted all other currencies to Kenya | We cant calculate anything dealing with money if at all we do have different currencies|
|"Out of 5 "| Customer Rating | Usually Ratings are out of 5 | We would want its data type as number so that it becomes possible to even get the average ratings | 
| Unknowns, Text, Outliers | Age, Units Sold, Discount, Vehicle Year, Customer Rating, Review Count, Revenue Recorded |Outliers like 121 years in Age wwere deleted,missing values left as blanks and columns formatted appropriately i.e text converted to numbers | This is to enable performing calculations and manipulations accurately |

## Data Cleaning Approach 
After first going through the data and understanding the whole data,using power query this was my cleaning approach

| Phase | Action  |
|:------|:----------|
| Structure | Fix data types, Clean Casing & Typos |
| Missing Data | Left missing data as nulls   |
| Outliers | Dropped outliers like 121 in age and left them as blank |
| Deduplication | Removed  exact duplicates |

## Currency Standardization Challenge
When auditing our raw data, we noticed various fields like 'Unit Selling Price','Unit Cost' were in various currencies like some records were in Kenyan currency,others in dollars, others in Euros and many other different currencies.
We therefore came up with an M code that we used to standardize the currencies

```powerquery
(RawValue as any) =>
let
    // 1. Force value to text and handle blanks or null entries safely
    TextValue = if RawValue = null then "" else Text.Upper(Text.Trim(Text.From(RawValue))),
    
    // 2. Scan for specific international currency flags and shorthand millions tags
    IsUSD = Text.Contains(TextValue, "USD") or Text.Contains(TextValue, "$"),
    IsEUR = Text.Contains(TextValue, "EUR"),
    IsZAR = Text.Contains(TextValue, "ZAR") or Text.Contains(TextValue, "R ") or (Text.StartsWith(TextValue, "R") and Text.Length(TextValue) > 1 and Text.PositionOfAny(TextValue, {"0".."9"}) = 1),
    IsMillion = Text.EndsWith(TextValue, "M"),
    
    // 3. Strip all alphabetical letters, symbols, commas, and spaces—leaving only numbers, decimals, and signs
    CleanDigits = Text.Select(TextValue, {"0".."9", ".", "-"}),
    NumericValue = try Number.FromText(CleanDigits) otherwise null,
    
    // 4. Handle conversion scales for values using the million ("M") shortcut
    BaseValue = if NumericValue = null then null 
                 else if IsMillion then NumericValue * 1000000 
                 else NumericValue,
                 
    // 5. Convert value to KES based on CBK exchange benchmarks (129.50 USD, 148.43 EUR, 8.01 ZAR)
    ConvertedKES = if BaseValue = null then null
                    else if IsUSD then BaseValue * 129.50
                    else if IsEUR then BaseValue * 148.43
                    else if IsZAR then BaseValue * 8.01
                    else BaseValue
in
    ConvertedKES
 ```
We went ahead to also use a M code to format our dates because we had dates formatted in different formats 

```powerquery
let
    // 1. Standardize spacing and handle known text typos
    RawValue = Text.Replace(Text.Trim([Order Date]), "-Sept-", "-Sep-"),
    
    // 2. Fix the impossible 13th month typo (e.g., 2026-13-04 -> 2026-04-13)
    FixedThirteen = if Text.Contains(RawValue, "-13-") then Text.Replace(RawValue, "-13-", "-04-") else RawValue,

    // 3. Fix impossible text dates like April 31st (changes to April 30th)
    FixedApril = if Text.Contains(FixedThirteen, "April 31") then Text.Replace(FixedThirteen, "April 31", "April 30") else FixedThirteen,

    // 4. Force catch impossible February entry typos (e.g., 31/02/2026 -> 28/02/2026)
    FixedFebruary = if Text.Contains(FixedApril, "31/02/") then Text.Replace(FixedApril, "31/02/", "28/02/")
                     else if Text.Contains(FixedApril, "30/02/") then Text.Replace(FixedApril, "30/02/", "28/02/")
                     else FixedApril,

    // 5. Try parsing 5-digit Excel serial numbers
    NumToDate = try Date.From(Number.FromText(FixedFebruary)) otherwise null,

    // 6. Try parsing text variations sequentially (UK format first, then US, then generic)
    IntlTextToDate = try Date.FromText(FixedFebruary, [Culture = "en-GB"]) otherwise null,
    UsTextToDate = try Date.FromText(FixedFebruary, [Culture = "en-US"]) otherwise null,
    GenericTextToDate = try Date.FromText(FixedFebruary) otherwise null,

    // 7. Output final combined fallback logic
    FinalResult = if NumToDate <> null then NumToDate 
                   else if IntlTextToDate <> null then IntlTextToDate 
                   else if UsTextToDate <> null then UsTextToDate 
                   else GenericTextToDate
in
    FinalResult
```
 
Our decision to change the currencies into one type of currency was to ensure we have having one standard currency because a million dollars does not equate to a million kenya shillings.A million dollar is approximately 130 million kenya shillings.Having currencies as one ensured we did accurate calculations later in power bi
For dates we needed them as one in order to able to even calculate on time delivery time, enabling us to know if delivery was late or early

## Data Modelling

The supplied dataset is a raw flat table.We designed an appropriate data model to enable time intelligence functions and even manipulation of data later in power BI .From the dataset we made six different dimensional tables as discussed above that we connected to one fact table using primary and foreign keys
Here is a screenshot of how i did it

<img width="1826" height="1011" alt="Image" src="https://github.com/user-attachments/assets/1bf39844-43c1-45b3-90f4-ead61ec1251f" />


## Dax Functions
The dataset wanted us to answer different questions to the management.Below are some of the questions the dataset answered
- What is the total number of cars sold ?
- What is the total sales revenue generated by the company ?
- What is the total gross profit ?
- Which car makes generate strongest business performance ?
- Who are the company's highest-value customers ?
- How are individual sales representatives performing ?

Not to mention all but a few of the questions our analysis answered.We didnt create DAX functions for all the qquestions because one DAX formulae can be used to answer very different questions
Some of the DAX formulaes we created were 
 
Average Customer Rating

```dax
AVERAGE(Facts_Data[Clean Customer Rating])
```

Average Selling Price 

```dax
CALCULATE(AVERAGE(Facts_Data[Clean Selling Price]),DimOrder[Returned]="No")
```

Total Cars Sold 

```dax
CALCULATE(SUM(Facts_Data[Clean Units Sold]),DimOrder[Returned]="No")
```

Total Discount

```dax
CALCULATE(SUM(Facts_Data[Clean Discount]),DimOrder[Returned]="No")
```

Total Revenue 

```dax
CALCULATE(SUM(Facts_Data[Clean Revenue Recorded]),DimOrder[Returned]="No")
```

Total Gross Profit

```dax 
CALCULATE(SUMX(Facts_Data,Facts_Data[Clean Revenue Recorded] - Facts_Data[Clean Logistics Cost]-(Facts_Data[Unit Cost]*Facts_Data[Clean Unit Cost])),DimOrder[Returned]="No")
```

## Assumptions 
- Revenue was collected for only cars that were not returned and thats why almost evey DAX formulae we created had a filter ([Returned]="No").Those records that had Returned field as blank were not included because we cant tell if the customer returned the car or didnt
- We assumed gross profit = Revenue - Logistics - (Units Sold * Unit Cost)
- For records in 'Returned' field that were negative we assumed there were just errors from data entry and went ahead to have them as absolutes

## Executive Dashboard

<img width="1899" height="1019" alt="Image" src="https://github.com/user-attachments/assets/eda0c5c8-b1fc-4924-8830-baa4ffe2becc" />


BUSINESS INSIGHTS AND RECOMMENDATIONS

J Cars Limited Analysis proved the following
Nakuru Branch is the most profitable branch with a total revenue of 195.74M and total gross profit of 40M . It had 31 cars delivered and only 9 being cancelled
Mombasa with a total of 26 cars sold made a profit of 18.88M with only 13 cars being delivered.E250 the luxurious Mercedes was the most car model sold in the branch
Kakamega Branch selling the most cars made only 9.52M in profits.15 cars from  the branch had its delivery status cancelled.
Athi River with 27 delivered cars made a loss of 16.07M with over 118M in revenue

What do Analysts recommend 
74.9% of cars sold use petrol. We can have more petrol fueled cars in our branches and import more cars using petrol
Asset Financing was the most preferred mode of payment .As a company if we can introduce financing we may be able to increase our profits although the idea needs more investigation into knowing what kind of financing our customers preferred
For Nakuru Branch we can have more of our products being advertised through Whatsapp because we had Whatsapp as a source giving us 23M in profits 
In general we need to improve "modern social media platforms" because instagram and website are amongst our lead sources doing poorly

What do we investigate
Why are having cars being returned yet they averagely have a rating of 3.65,a rating higher than all the cars averaged
Why is Athi River, Nairobi and Kisumu making such losses 
Why are cars deliveries being cancelled














