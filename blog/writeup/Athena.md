# AWS Athena is based on Presto SQL, which may take some time to get comfortable with. 
#### note: 
- To make a comment or note, use " **--** note_text" or "**/*** note_text ***/**"
- SQL best practice: Correctness, readability, then optimization; 
- Athena query is case sensitive (Upper case and lower case keywords lead to different functions); 
- In Athena, if you select certain section with cursor, Athena will only run the selected section (this feature comes in handy for debugging long queries);
- To date, creating tables can only be done in Athena not PyAthena; 
- Quotation " " is different from ' '.
- Be patient. Often the error is caused by typo, especially the extra *comma* before **FROM**; 
- Once you ran a complicated query successfully, make a copy just in case. 
- Reminder: Direct copying from text editors such as MS Word can sometimes cause error due to autocorrection such as replacing ' with ‘.

## 1. [Setup](https://aws.amazon.com/blogs/machine-learning/run-sql-queries-from-your-sagemaker-notebooks-using-amazon-athena/) (assume you have installed the perm to access AWS S3)   
   import sys  
   !{sys.executable} -m pip install PyAthena  

   from pyathena import connect  
   import pandas as pd  
   conn = connect(s3_staging_dir='your_query_location_can_be_found_in_Athena_settings',  region_name='YOUR REGION, for example, us-east-1')  

   test = pd.read_sql("""  
   SELECT *  
   FROM "db"."table"  
   LIMIT 3  
   ;""", conn)  

#### (Note: pay attention to the number of quotation marks, different from the AWS tutorial which is strangly not working).

## 2. How to convert a numerical string to date (e.g. 20201010 to 2020-10-10)  
   DATE(DATE_PARSE('20201010', '%y%m%d'))  
   
## 3. Day of the week, Week of the year
  - EXTRACT(DAY_OF_WEEK FROM DATE('2020-10-10'))     (e.g. 1,2,3,4,5,6,7).   
  - DATE_FORMAT(DATE('2020-10-10'), '%W')   (e.g. Monday, Tuesday, ..., Sunday). 
  - EXTRACT(WEEK FROM DATE('2020-10-10'))     (e.g. 1,2,3,..., 53).  
   
## 4. How to calculate weekdays (my way, many popular methods are not applicable to Athena) 
To calculate weekdays is essentially to calcualte weekends.  
e.g. 2020-12-03 (Thu) to 2020-12-16 (Wed) across three weeks but only one whole week.  
- - 2020-12-03 (Thu) to 2020-12-04 (Fri): 1. 
- - 2020-12-07 (Mon) to 2020-12-11 (Fri): 5. 
- - 2020-12-14 (Mon) to 2020-12-16 (Wed): 3. 
- - - Two weekends (four days in total), 9 business days. (9 business days = 13 days - 4 days). 
#### Note:
- In Python Pandas, len(pd.bdate_range('2020-12-03', '2020-12-14')) **-1** = 9. 
- In Office Excel, NETWORKDAYS(date1, date2) **- 1** = 9. 

#### In Athena,
**DATE_DIFF, DATE_TRUNC, DATE_FORMAT, and CASE**  

#### <<<<<<<< week 1 >>>>>>> start_date >>
#### <<<<<<<< week x >>>>>>> end_date >>
#### Note: Drawing the timeline (bulk) on a sketch paper is helpful to understand their relationship.

### 4.1 How many days between start_date and end_date:   
    DATE_DIFF('DAY', DATE(start_date), DATE(end_date))  

### 4.2 How many weeks between start_date and end_date:   
    DATE_DIFF('WEEK', DATE_TRUC('WEEK', start_date), DATE_TRUNC('WEEK', end_date))    

### 4.2.1 Edge cases I, what if start date is in the weekend:   
    (CASE WHEN DATE_FORMAT(start_date, '%W') = 'Saturday' THEN -1   
    WHEN DATE_FORMAT(start_date, '%W') = 'Sunday' THEN -2   
    ELSE 0 END).  

### 4.2.2 Edge case II, what if end date is in the weekend:   
    (CASE WHEN DATE_FORMAT(end_date, '%W') = 'Saturday' THEN +1   
    WHEN DATE_FORMAT(end_date, '%W') = 'Sunday' THEN +2   
    ELSE 0 END)     

### To sum up, how many days of weekends between start_date and end_date:    
    (DATE_DIFF('WEEK', DATE_TRUC('WEEK', start_date), DATE_TRUNC('WEEK', end_date))) x 2 + 
    (CASE WHEN DATE_FORMAT(start_date, '%W') = 'Saturday' THEN -1 WHEN DATE_FORMAT(start_date, '%W') = 'Sunday' THEN -2 ELSE 0 END) +  
    (CASE WHEN DATE_FORMAT(end_date, '%W') = 'Saturday' THEN +1 WHEN DATE_FORMAT(end_date, '%W') = 'Sunday' THEN +2 ELSE 0 END)  
    AS weekend_day.   
### Therefore, how many weekdays: DATE_DIFF('DAY', DATE(start_date), DATE(end_date)) - **weekend_day**

## 5. Add / minus 7 days of a given date: 
    DATE_ADD('day', 7, DATE(given_date)) 
    DATE_ADD('day', -7, DATE(given_date))

## 6. Improve readability with CTE (Common Table Expressions)
    WITH Table_x AS (SELECT ... FROM Original_table)  
    SELECT ...  
    FROM Table_x  
## 7. Window functions (with OVER)
    ROW_NUMBER():  for rows that have duplicate values,numbers are arbitarily assigned., e.g. 1, 2, 3, 4, ...    
    RANK(): 1, 1, 1, 4, ...  
    DENSE_RANK(): 1, 1, 1, 2, ...  
    LAG() AND LEAD(): LAG pulls from previous rows and LEAD pulls from following rows.  
### To date, Athena does not support window alias.

## 8. Additional reading 
    https://athena.guide/ (better than official documentation)  
    https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/  (JOIN small table on the right)  
    https://skeptric.com/presto-integer-division/ (integer to percent)
    
## 9. Tricks
    ARRAY_JOIN(ARRAY_DISTINCT(ARRAY_SORT(ARRAY_AGG(value))), ', ') AS values
    
``` <b>One way to automate Athena</b>
    def sampler(value='123xyz'):
    import pandas as pd
    from pyathena import connect

    s3_dir = 's3://xxx'
    region = 'xxx'
    conn = connect(s3_staging_dir= s3_dir, region_name= region)
    
    param = """
    SELECT  *
    FROM "db_xxx"."table_xxx"  
    WHERE ##Var## IN ('123xyz')
    ORDER BY claim_tcn_id, CAST(service_line_number AS INTEGER)
    ;  """
    
    param = param.replace('123xyz',value)
    sample = pd.read_sql(param, conn)
    
    return sample
```

