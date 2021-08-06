# CLAIMS-SQL

sql analysis for claims and providers

## Prep

#### Data Given:

````
Claim_id member_id service_dat billed_amt	paid_amt cptode orig_claim adj_num provider_id
1001	526752834 2018-05-01	199.99	50.00	97018	1001	00	127
1002	070535274 2018-05-01	300.00	150.00	97036	1002	00	231
1003	171265761 2018-05-02	15600.00	12500.00	27445	1003	00	142
1001	526752834 2018-05-01	199.99	100.00	97018	1001	01	127
1004	171265761 2018-05-02	16500.00	13000.00	27486	1003	00	142
1005	184368945 2018-05-02	29999.99	2000.00	93303	1005	00	129


Provider_id Provider_name  address	State Zipcode
127	EZ Rehab	1 Long Rd.	FL	33101
129	Dr. Howser	796 Amalfi Dr	CA	90272
142	ACME Ortho	1 Bridge St.	CA	94402


````

#### Table Creation

````
CREATE TABLE CLAIMS
(claim_id integer not null primary key,
Member_id integer, Service_dat date, Billed_amount decimal(8,2), Allowed_amount decimal(8,2), CPTCode varchar(6), Orig_claim_id integer, Adj_num integer,
Provider_id integer);

CREATE TABLE PROVIDER
(provider_id integer not null, Provider_name varchar(20), Address varchar(40),
State char(2), Zipcode varchar(10));

CREATE TABLE name_data
(patient_name varchar(100) NOT NULL,
 dob varchar(10));

````

#### Data Insertion

````
insert into CLAIMS
	(claim_id, Member_id, Service_dat, Billed_amount,Allowed_amount,CPTCode,Orig_claim_id,Adj_num,Provider_id)
values
	(1001,526752834,'2018-05-01',199.99,50.00,'97018',1001,00,127),
    (1002,070535274,'2018-05-01',300.00,150.00,'97036',1002,00,231),
    (1003,171265761,'2018-05-02',15600.00,12500.00,'27445',1003,00,142),
    (1001,526752834,'2018-05-01',199.99,100.00,'97018',1001,01,127),
    (1004,171265761,'2018-05-02',16500.00,13000.00,'27486',1003,00,142),
    (1005,184368945,'2018-05-02',29999.99,2000.00,'93303',1005,00,129);
    
    
insert into PROVIDER
	(provider_id, Provider_name, Address, State, Zipcode)
values
	(127,'EZ Rehab','1 Long Rd.','FL','33101'),
    (129,'Dr. Howser','796 Amalfi Dr','CA','90272'),
    (142,'ACME Ortho','1 Bridge St.','CA','94402');
    
 insert into name_data
  (patient_name, dob)
values
  ('John Doe','2000-02-13'),
  ('John Doe','2000-06-07'),
  ('Dexter Lab','1997-08-19'),
  ('Dexter Lab','1997-08-19'),
  ('Dee Dee','2005-03-18'),
  ('Dee Dee','1995-11-25'),
  ('Roger Bunny','1993-09-10')
    ;

````

## Questions

##### 1.  Construct a query that ranks US states by paid amount.

````
# no partition by in RANK..so that it can rank all rows as one category

SELECT 
 us_state,
 sum(paid_amount) as total_paid_amount,
 RANK() OVER (ORDER BY sum(paid_amount) DESC) as sales_rank 
 from 
  (
select A.Allowed_amount as paid_amount,B.State as us_state from CLAIMS A INNER JOIN PROVIDER B ON A.Provider_id = B.provider_id
  )TAB1 group by us_state
  
Output:
+----------+-------------------+------------+
| us_state | total_paid_amount | sales_rank |
+----------+-------------------+------------+
| CA       |          27500.00 |          1 |
| FL       |             50.00 |          2 |
+----------+-------------------+------------+

  
  
````

##### 2.  In the results for (1) you notice state CA near the top, and also states Ca and 94. Why might this happen, and what do you do next? After fixing this, you also notice a pretty high ranking next to an “empty string” state. What does this mean, and what might you do next to address this issue?

````
#Assuming the question(as question has Ca and 94 states but data don't hold it) 

select A.Allowed_amount as paid_amount,B.State as us_state from CLAIMS A INNER JOIN PROVIDER B ON A.Provider_id = B.provider_id order by paid_amount desc

Above SQL would result in multiple rows with same cities where aggregation not took place. This will not allow user to rank it correctly. Therefore Aggregation needs to be implemented and also RANK function would help end user to tag the ranks. 

#Note#: With give data, no EMPTY Strings occured, but if at all empty string exists, I would filter with "where" condition first place to ensure I have clean data(avoid mising info data)

Output:
+-------------+----------+
| paid_amount | us_state |
+-------------+----------+
|    13000.00 | CA       |
|    12500.00 | CA       |
|     2000.00 | CA       |
|       50.00 | FL       |
+-------------+----------+


````

##### 3.  During the above analysis, you notice that sum(paid_amount) for all of the states together is a lot lower than expected (e.g. Florida total is lower than Kansas). What are some possible explanations, and what approach would you take to address this issue?

````
#Assuming the question: 
With assumption of both PREPAY and POSTPAY exists in CLAIMS table, Florida has two different records with column 'Adj_num'(00,01). After post adjudication, FL state might went to negative paid amount(overpayment). We can choose the ranking on paid_amount either by PREPAY or POSTPAY by filtering Adj_num'
````

##### 4.  Your colleagues decide that the above analysis would be easier to understand if there weren’t so many weird states (e.g. PR, GU, VI). How would you lump all of the commonwealth territories into “ISLANDS”.

````
I would introduce an New column with CASE condition in SQL. This column would act like another State column except takes value 'island' if state value fall in  commonwealth territories
#CASE 
#      WHEN state IN(PR, GU, VI) THEN 'island'
#      ELSE state
#  END as 'state2'
````

##### 5.  For the same tables write a query to find the names of the providers with the two highest charges for each procedure.

````
select P.Provider_name,clm.CPTCode,clm.total_billed,bill_rank from 
(select sum(Billed_amount) as total_billed,
       CPTCode,
       Provider_id,
       RANK() OVER (PARTITION BY Provider_id ORDER BY sum(Billed_amount) DESC) as bill_rank       
       from CLAIMS group by CPTCode,Provider_id
       )clm INNER JOIN PROVIDER P ON clm.Provider_id = P.Provider_id WHERE bill_rank >= 1 and bill_rank < 3 order by clm.total_billed desc

Output:
+---------------+---------+--------------+-----------+
| Provider_name | CPTCode | total_billed | bill_rank |
+---------------+---------+--------------+-----------+
| Dr. Howser    | 93303   |     29999.99 |         1 |
| ACME Ortho    | 27486   |     16500.00 |         1 |
| ACME Ortho    | 27445   |     15600.00 |         2 |
| EZ Rehab      | 97018   |       199.99 |         1 |
+---------------+---------+--------------+-----------+


````

##### 6.  Your colleague points out that PMPM on this data set looks 30% higher than expected, based on financial data supplied by the customer. What might be some possible explanations for this, and how would you attempt to explain or reconcile this with your data?

````
I would check the service_dates whether appropriate to the "financial year" that we are calculating(needs to be same financial year supplied by the customer). If we have more than that specific year, filter the data on service_date to that financial year and get the distinct count of "Member_id" and multiply by 12(12 months in an year). total_cost/above calculation, gives you PMPM approx which can be verified.
````

##### 7.  Are all three doing the same thing? Explain.


````
Query1: Is trying to eliminate nulls, but this would not eliminate the EMPTY STRINGS

Query2: Eliminates both NULLS and EMPTY STRINGS

Query3: Also does same as query2 by eliminating both NULLS and EMPTY STRINGS but the approach is different. Instead of using two “AND” conditions, it is making use of “NVL” which converts NULLS to empty string. (More matured way of querying)

Conclusion: Query2 and Query3 are doing same. But Query1 is just doing sub/half of the job.

````

##### 8.  How do you find all patients that have multiple different distinct dob values (e.g., John Doe has two different values for dob).

````
select patient_name,count(distinct dob) from name_data group by patient_name having count(distinct dob)>1

Output:
+--------------+---------------------+
| patient_name | count(distinct dob) |
+--------------+---------------------+
| Dee Dee      |                   2 |
| John Doe     |                   2 |
+--------------+---------------------+

````
##### 9.  Will these return same results? Does one return more rows than the other? Explain.


````
Query1 returns More records(row count EQUAL to LEFT TABLE mostly) as the 'authorization_number' is checked for empty string before/while the left join occurs. 

If empty 'authorization_number' data not exists(empty string) all data comes as NULL. If not, data populates for empty 'authorization_number' column and rest will be NULL

Where as Query2, “WHERE” condition is applied after LEFT JOIN executes and as a result only specified data(authorization_number = '') is filtered if exists. If not, zero records occur.

````