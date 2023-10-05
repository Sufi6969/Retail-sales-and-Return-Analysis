# Retail-sales-and-Return-Analysis
/* Sanity Checks - Data Cleaning */
/* Tables Load */
proc import datafile='/home/u63391350/Poject 2/Customer.xlsx' out=Customer dbms=xlsx replace;
    sheet='Sheet1';
    getnames=yes;
run;
proc import datafile='/home/u63391350/Poject 2/Order.xlsx' out=Order dbms=xlsx replace;
    sheet='Sheet1';
    getnames=yes;
run;
/* Customer id is blank */
data Order;
  set ;
  if missing(Customer_id) then Customer_id = 99999;
run;
proc sql;
  create table Combined as
  select *
  from customer
  inner join order
  on customer.Customer_ID = order.Customer_ID;
quit;
/* Age greater than 18 */
data Combined;
  set Combined;
  if Age <= 18 then Age = 18;
run;
/* Apply automatic discount of 5% where Price is equal to Selling Price and has a Coupon Code */
data Combined;
  set Combined;
  if Price = Selling_Price and missing(Coupon_Code) then do;
    Discount_Amount = 0.05 * Price;
    Selling_Price = Selling_Price - Discount_Amount;
  end;
run;
/* Make sure Return Date is after Purchase Date */
data ReturnDate;
  set Combined;
  keep Customer_id Date Return_Date ;
  if Return_Date <= Date then Return_Date = Date + 1;
run;


/* If Coupon ID is NULL, ensure no discount is given and Selling Price is equal to Price */
data NoCoupon;
  set Combined;
  keep Customer_id Coupon_ID Selling_price Price;
  if missing(Coupon_ID) then Selling_Price = Price;
run;


/****************************************************111****************************************************/
/* Create customer segments based on age, gender, and total spending */
proc sort data=combined;
  by Customer_id;
run;

data CustSegSpending;
  set combined;
  by Customer_id;
  retain Total_Spending 0;
  if first.Customer_id then Total_Spending = 0;
  Total_Spending + Selling_price;
  if Gender = 'F' then do;
    if Age < 35 then Segment = 'Young Females';
    else if Age < 55 then Segment = 'Mid-age Females';
    else Segment = 'Old Females';
  end;
  else if Gender = 'M' then do;
    if Age < 35 then Segment = 'Young Males';
    else if Age < 55 then Segment = 'Mid-age Males';
    else Segment = 'Old Males';
  end;
  if Total_Spending < 1000 then Spending_Segment = 'Low';
  else if Total_Spending < 5000 then Spending_Segment = 'Moderate';
  else Spending_Segment = 'High';
keep Customer_id Name City Age Gender Segment Spending_Segment Product_ID P_CATEGORY;
run;
proc sql;
  select Customer_id, Gender, Age, City, Segment, Spending_Segment
  from CustSegSpending
  where monotonic() <= 10 ;
quit;


/************************************************111********************************************************/
/* Create customer segments based on age, gender, and number of transactions */
proc sort data=combined;
  by Customer_id;
run;

data CustSegtranction;
  set combined;
  by Customer_id;
  retain Total_Transactions 0;
  if first.Customer_id then Total_Transactions = 0;
  Total_Transactions + 1;
if Gender = 'F' then do;
    if Age < 35 then Segment = 'Young Females';
    else if Age < 55 then Segment = 'Mid-age Females';
    else Segment = 'Old Females';
  end;
  else if Gender = 'M' then do;
    if Age < 35 then Segment = 'Young Males';
    else if Age < 55 then Segment = 'Mid-age Males';
    else Segment = 'Old Males';
  end;
if Total_Transactions < 10 then Transaction_Segment = 'Low';
  else if Total_Transactions < 50 then Transaction_Segment = 'Moderate';
  else Transaction_Segment = 'High';
keep Customer_id Name City Age Gender Segment Total_Transactions Transaction_Segment;
run;
proc sql;
  select Customer_id, Gender, Age, City, Segment, Transaction_Segment
  from CustSegtranction
  where monotonic() <= 10 ;
quit;


/***************************************************222*****************************************************/

/* Calculate overall spend based on products */
proc sql;
    create table product_spend as
    select Product_ID, P_CATEGORY, sum(Selling_price) as Total_Spend
    from combined
    group by Product_ID, P_CATEGORY
    order by Total_Spend desc;
quit;
proc sql;
  select Product_ID, P_CATEGORY, Total_Spend
  from product_spend
  where monotonic() <= 5 ;
quit;
/* Calculate total spend per state */
proc sql;
    create table state_spend as
    select State, sum(Selling_price) as Total_Spend
    from combined
    group by State
    order by Total_Spend desc;
quit;
proc sql;
  select State, Total_Spend
  from state_spend
  where monotonic() <= 5 ;
quit;

/* Calculate total spend per payment method */
proc sql;
    create table payment_spend as
    select PaymentMethod, sum(Selling_price) as Total_Spend
    from combined
    group by PaymentMethod
    order by Total_Spend desc;
quit;
proc sql;
  select PaymentMethod, Total_Spend
  from payment_spend
  where monotonic() <= 5 ;
quit;


/*****************************************************444******************************************************/
/* Calculate by age */
proc sql;
     create table return_count as
     select Age, sum(Return_ind) as Return_Count
     from combined
     group by Age;
quit;
proc sql;
     select Age
     from return_count
     where Return_Count = (select max(Return_Count) from return_count);
quit;

/* Calculate product category */
proc sql;
     create table return_count as
     select P_CATEGORY, sum(Return_ind) as Return_Count
     from combined
     group by P_CATEGORY;
quit;
proc sql;
     select P_CATEGORY
     from return_count
     where Return_Count = (select max(Return_Count) from return_count);
quit;
/* Calculate by state */
proc sql;
     create table return_count_state as
     select State, sum(Return_ind) as Return_Count
     from combined
     group by State;
quit;
proc sql;
     select State
     from return_count_state
     where Return_Count = (select max(Return_Count) from return_count_state);
quit;

/* Calculate by discount */
proc sql;
     create table return_count_discount as
     select Selling_price - Price as Discount, sum(Return_ind) as Return_Count
     from combined
     group by Discount;
quit;
proc sql;
     select Discount
     from return_count_discount
     where Return_Count = (select max(Return_Count) from return_count_discount);
quit;

/***************************************************444 ALL**************************************************/

/* Calculate by age, state, discount, and product category */
proc sql;
     create table return_count_combined as
     select Age, State, Selling_price - Price as Discount, P_CATEGORY, count(*) as Return_Count
     from combined
     where Return_ind = 1 
     group by Age, State, Discount, P_CATEGORY;
quit;
/* Find Highest return count */
proc sql;
     select Age, State, Discount, P_CATEGORY
     from return_count_combined
     where Return_Count = (select max(Return_Count) from return_count_combined);
quit;



/******************************************************5555***************************************************/
/* Profile Based on Timing */
data combined;
    set combined;
    OrderHour = hour(time);
run;
data customer_profiles;
    set combined;
    format OrderHour Time_Hour.;

    if OrderHour >= 5 and OrderHour < 12 then do;
        OrderTiming = "EarlyBird";
    end;
    else if OrderHour >= 12 and OrderHour < 17 then do;
        OrderTiming = "DayShift";
    end;
    else if OrderHour >= 17 and OrderHour < 21 then do;
        OrderTiming = "WorkClass";
    end;
    else do;
        OrderTiming = "NightOwl";
    end;
    keep Customer_id Age Gender Transaction_ID OrderHour OrderTiming;
run;



/*********************************************************666**************************************************/
/* Calculate the total discount by payment method */
proc sql;
     create table total_discount as
     select PaymentMethod, sum(Selling_price - Price) as Total_Discount
     from combined
     group by PaymentMethod;
quit;
proc sql;
     select PaymentMethod
     from total_discount
     where Total_Discount = (select max(Total_Discount) from total_discount);
quit;


/************************************************************777***********************************************/
/* Calculate the count of orders with selling price less than 500 */
proc sql;
   select count(*) as LowValProCount
    from combined
    where Selling_price < 1000;
quit;

/* Calculate the count of high-value products by category */
proc sql;
    select count(*) as HighValProCount
    from combined
    where Selling_price > 1000;
quit;

/***********************************************************888************************************************/
/* Calculate the average number of orders for each discount level */
proc sql;
     create table Discount_N_OrderC as
     select (Selling_price - Price) as Discount, count(*) as Order_Count
     from combined
     group by Discount
     order by Order_Count desc;
quit;
proc sql;
 select Discount, Order_Count
 from Discount_N_OrderC
 where monotonic() <= 5 ;
quit;  

