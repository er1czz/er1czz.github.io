AWS Athena is based on Presto SQL, which may take some time to get comfortable with. 
note: 
- Athena query is case sensitive; 
- Creating tables can only be done in Athena not PyAthena (to date); 
- Quotation "" is different from ''.

1. [Setup](https://aws.amazon.com/blogs/machine-learning/run-sql-queries-from-your-sagemaker-notebooks-using-amazon-athena/) (assume you have installed the perm to access AWS S3)   
import sys  
!{sys.executable} -m pip install PyAthena  

from pyathena import connect  
import pandas as pd  
conn = connect(s3_staging_dir='<ATHENA QUERY RESULTS LOCATION>',  region_name='<YOUR REGION, for example, us-east-1>')  

test = pd.read_sql("""  
SELECT *  
FROM "athenaquery"."<YOUR TABLE NAME>"  
LIMIT 3  
;""", conn)  
  
2. How to convert a numerical string to date (e.g. 20201010 to 2020-10-10)  
   DATE(DATE_PARSE('20201010', '%Y%M%D'))  
   
3. How to calculate weekdays (many popular methods are not applicable)  
**DATE_DIFF, DATE_TRUNC, DATE_FORMAT, and CASE**  
3.1 How many days between start_date and end_date: DATE_DIFF('DAY', DATE(start_date), DATE(end_date))  
3.2 How many weeks between start_date and end_date: DATE_DIFF('WEEK', DATE_TRUC('WEEK', start_date), DATE_TRUNC('WEEK', end_date))  
3.3 Edge cases I what if start date is in the weekend: CASE WHEN DATE_FORMAT('W', start_date) = 'Saturday' THEN -2 WHEN DATE_FORMAT('W', start_date) = 'Sunday' THEN -1 ELSE 0 END  
3.4 Edge case II what if end date is in the weekend: CASE WHEN DATE_FORMAT('W', end_date) = 'Saturday' THEN +1 WHEN DATE_FORMAT('W', end_date) = 'Sunday' THEN +2 ELSE 0 END  
To sum up, DATE_DIFF  
