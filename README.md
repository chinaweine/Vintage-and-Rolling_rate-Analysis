# Vintage-and-Rolling_rate-Analysis
**In this repository, I will explaine how to conduct vintage, Rolling_rate and Observe_Window analysis with sql
Vintage analysis is a widely used method for managing credit risk. It illustrate the behaviour after an account was opened. Based on same origination period, it calculates charge off ratio of a loan portfolio. In recent years, as the python population, most of research and people prefer to use python to do data analysis. And we can find a lot of articles and videos on the forum regarding how to use python to realize vintage analysis. So, in this post, I want to explain how to use sql to realize vintage analysis step by step. The data I use is “credit card approval prediction”, it is a free data for learner to practice their data analysis skill. So if you are interesting going through the whole process of vintage analysis, you can go to download the data. 
**There are two tables in the data sources, which is application_record.csv and credit_record.csv, for vintage analysis, we just need to use the credit_record.csv file. The credit_record.csv file contains loan accounts’ credit records, the detailed data explanation is here:
|Feature name	|Explanation	|Remarks|
|---|---|---|
|ID	|Client number|	
|MONTHS_BALANCE	|Record month	|The month of the extracted data is the starting point, backwards, 0 is the current month, -1 is the previous month, and so on|
|STATUS	|status	|0: 1-29 days past due 1: 30-59 days past due 2: 60-89 days overdue 3: 90-119 days overdue 4: 120-149 days overdue 5: Overdue or bad debts, write-offs for more than 150 days C: paid off that month X: No loan for the month|


The first step, we use select * from credit_record, we can see the following information.
 ![image](https://user-images.githubusercontent.com/50256538/198986508-f1d0e090-8f44-48e6-b734-99daea11dd0c.png)

 
 
 
Because the MONTHS_BALANCE column is a number, this is not what we are familiar with information, We can use MIN() function and MAX() function to get Open_month and End_month  which each borrowers starts and closes their credit relationship with the lending institution. Then, we can get the observe window by End_month minus Open_month and get the month_on_book(MOB) by MONTHS_BALANCE  minus Open_month. 
When we set up the conditions to filter the data to meet our analysis demand, There are two points we need to be concerned. Firstly, we know the observe window should be long enough to observe the borrows’ behaviour, so we will delete the windows less than 20. You can also try setting the number as 12, that means the loan term should be at least one year. You can see what the result will be going on. Second, according to the official explanation of the STATUS, ‘X’ means there is no loan in that month. So for my personal opinion, I think when we calculate the total loan number in each month, we should set STATUS not equal to ‘X’.
```
select ID, MONTHS_BALANCE, STATUS, MIN(MONTHS_BALANCE) as Open_month
from credit_record
group by ID
```
then we will name the above query as a subquery. And using this subquery, we can calculate the total loan number and the number past due in 30-60 days
```
select c.ID, c.MONTHS_BALANCE, c.STATUS, c.MONTHS_BALANCE - t1.Open_month as MOB
from credit_record c
join(select ID, MIN(MONTHS_BALANCE) AS Open_month
from credit_record
group by ID) t1
on c.ID = t1.ID) t2
join(select ID, MIN(MONTHS_BALANCE) AS Open_month, MAX(MONTHS_BALANCE) - MIN(MONTHS_BALANCE) AS Window 
from credit_record
group by ID) t3
on t2.ID = t3.ID
where t2.STATUS ='2' or t2.STATUS ='3' or t2.STATUS ='4' or t2.STATUS ='5') t4
where t4.Window >= 20
group by t4.Open_month, t4.MOB
```
Because when one borrower can’t pay his(or her) principle and interest in 30-60 days, we record his loan status as ‘1’. If he can’t repay the loan properly next month, we will record his loan status as ‘2’ according to the credit policy. So, there is little opportunity that we will get the duplication record result. Finally, we put the two query result together, and calculate the running total number of past due more than 60 days. In real working situation, you can set up different conditions to meet your different analysis demands, such as if we want to figure out the results regarding past due 90 days, we can set the status number just as 2. So we will go through the whole sql code in the following contents.
```
select t6.Open_month, count(t6.ID) AS Total_num
from(select ID, MIN(MONTHS_BALANCE) AS Open_month
from credit_record
where status <> ‘x’
group by ID) t6
group by t6.Open_month
```
Finally, we can put the two subqueries together with join… on, and calculate the running total default number. 
![image](https://user-images.githubusercontent.com/50256538/198987090-5591f052-4e2d-49db-8e74-fb7161a2ec29.png)



From the following screenshot, we can see the final result. However you should pay more attention, the running total number is not correct. The reason is we have 60 Open_months loan data, the loan was granted in each month has at least maximum 60 months loan term and minimum 20 months loan term. so the number in the Running_total column is already aggregate the same MOB in different Open_month. For calculating cumulative number, we need using circulating query. The following screenshot is what we want to get correct result.
![image](https://user-images.githubusercontent.com/50256538/198987267-357bf282-f00a-46d9-82a2-b65b281965b6.png)


 
Finally, I prefer to use power BI to visualize the query result.
![image](https://user-images.githubusercontent.com/50256538/198987429-eb83b0a0-6cb2-4ee8-ad21-49bc2375b50a.png)

 
According to the above screenshot, we can see the credit management in the first three month in this lending institution is very poor. The worst loan granted  happened in -60, -59 and -55. the cumulative rate past due more than 60 days are both greater than 1.5%. However, I think as they learn some lesson from the past experience, and they should change some loan’s entrance requirements. So their loan quality was improved significantly. If we just choose parts of Open_month grant loan for analysis, it is more easy for us to watch the trend of the past due clearly. For example, from the following screenshot, we can see the grant loan in -49 month, after 10 months, it reaches its peak of default, and begins going down, keeping stable at the rate of 0.5% at most of time.
![image](https://user-images.githubusercontent.com/50256538/198987494-b5a01870-75b2-4c95-bcfa-6820dea9128f.png)

 
*****Observe Window Analysis
Because of two reasons, account cancellation and observe over, our observe on accounts will be truncated. Observe window is a significant parameter to be considered. If observe window is too short, users’ behaviour will not fully show off, which will bring unnecessary noise to our data. In order to observe how many accounts increase as observe window extend, we plot this.
![image](https://user-images.githubusercontent.com/50256538/198987550-a5f9d667-6939-47d9-9c0c-4632528df96e.png)

 
From the above screenshot, if we setting the past-due conditions is not so strict, as the time pass, the past-due number will increase dramatically. If we just regard the situation which past-due more than 60 days as default, we can see the default number are much more stable.

*****Rolling Rate Analysis
**Roll rate analysis is used for solving various type of problems. Most common usage is loss forecasting and it is also used to determine the definition of 'bad' customers (defaulters). Most common definition of 'bad' customer is customer delinquent for 90 days or more. In simple words, if payment has been due 90 days or more, it is considered as 'bad'. It includes if it is partially or fully charged-off.

**A roll rate is the percentage of a lender’s portfolio that transitions from one 30-day delinquent period to another. Roll rate methodologies are sometimes called transition rates, flow models, or migration analysis. A roll rate is used by analysts to predict losses based on linquency. For carrying out the Rolling_rate analysis, we still use the credit card approve data. According to the data dictionary explanation, the “-60” in the MONTHS_BALANCE column means 60 months ago, so I will use the convert() function to transform the MONTHS_BALANCE to the exactly date. When I did this analysis, the date is on 10th August 2022. In dateadd() function, I set ‘20220810’ as the current date. So we can know the first date which the lending institution grant loan is 20170810. As the inflation hit our economy in the past twelve months, for controlling the inflation, most of the central bank will hike the rate. That will unavoidably lead to increase mortgage rate. So using the Rolling_rate analysis, we can figure out how many borrowers will enter into past due 90 days. After that, we can set the observe point at 20210910, the time period between 20200910 and 20210910 is the observe period, and the time period between 20210910 and 20220810 is the behaviour period
```
select t5.previous_status, t5.New_status, count(t5.ID) AS count_num
from(select t2.ID, t2.status as previous_status, t4.STATUS as New_status
from (select t1.ID, t1.STATUS
from(select *, CONVERT(varchar(8), dateadd(mm, MONTHS_BALANCE, '20220810'), 112) as Date
from credit_record
group by ID, MONTHS_BALANCE, STATUS) t1
where t1.Date = 20210910) t2
join (select t3.ID, t3.STATUS
from(select *, CONVERT(varchar(8), dateadd(mm, MONTHS_BALANCE, '20220810'), 112) as Date
from credit_record
group by ID, MONTHS_BALANCE, STATUS) t3
where t3.Date = 20220810) t4
on t2.ID = t4.ID) t5
group by t5.previous_status, t5.New_status
order by t5.previous_status
```
Finally, we put two time period data together and using powet pivot or power BI to visualization data. We can get the following two tables, the first table is displaying as count number, the second table is displaying as percent number. From the second table, we can figure out if the previous status is past due 0-30 days, in the next 12 months, there is 40.40% borrowers still keeping past due 0-30 days, and there are 43.35% borrowers will pay off their debt on time, and only less than 10% borrowers’ status will escalate. For those who old status is past due 60 days, 31.28% will roll back, and 2.64% will continue escalating.
![image](https://user-images.githubusercontent.com/50256538/198988561-9bcc0b81-acd8-4df6-9d00-5593a6e57807.png)


When we use the Flow_Rate Analysis method, we can learn more information in detail each month. Using the following SQL code, we can get a data list that shows us the number of each status in each month. Then we use power pivot to transform the data into the following table.

'''
select t1.STATUS, t1.date, count(t1.ID) AS count_num
from (select *, convert(varchar(8), dateadd(mm, MONTHS_BALANCE, '20220810'),112) as date
from credit_record) t1
where T1.STATUS <> 'X' and T1.date between 20210910 and 20220810
group by t1.STATUS, t1.date
order by t1.STATUS, t1.date
'''
![image](https://user-images.githubusercontent.com/50256538/198988973-69e77ddd-b3e7-4b38-a206-c1288e899be4.png)


We can see from the above table, that in October 2021, there a is 3.07% probability borrowers’ status go into M1 from M0 and 5.30% from M1 to M2. And there are 3.17% average probability for borrowers to go into M1 from M0 in the past 12 months.
![image](https://user-images.githubusercontent.com/50256538/198989071-770c82c3-d57c-4ed9-999f-a1829faa8fe6.png)

 
bad debt loss rate = (MO-M1)*(M1-M2)*(M2-M3)*(M3-M4)*(M4-M5)
                             = 3.18%*12.02%*58.69%*53.34%*311.82%=0.37%
So, if we just want to calculate the Bad debt loss rate DPD90, we can choose(M2-M3)*(M3-M4)*(M4-M5), in this case, the bad debt loss rate is 97.62%.
Furthermore, we can analyze the borrowers’ repayment behavior to see how many months the borrowers can afford this kind of high mortgage rate. From the following table and column chart, we can learn about three facts. The first is that there are a total of 67 clients who spent an average of six MOB to enter default in the past 12 months. Secondly, almost 30% of clients already enter into PDD more than 90 days after three MOB. We can consider checking their account balance monthly change and expense to income ratio carefully. Thirdly, almost 80% of clients who enter PDD more than 90 days just need only eight MOB. Combining the Flow_rate analysis result, we can have an idea that there are 6.54% of M1 clients after six MOB will enter M2. And in this 6.54% of M1 clients, there are almost 80% of clients who enter into M2 just need six MOB. 
![image](https://user-images.githubusercontent.com/50256538/198989231-07745d66-a799-48f9-a79e-6bbd73514887.png)

![image](https://user-images.githubusercontent.com/50256538/198989270-b7938dd2-14e5-45e5-b805-063cec3c0c5d.png)


 
