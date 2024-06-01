CREATE TABLE goldusers_signup(userid integer,gold_signup_date date); 

INSERT INTO goldusers_signup(userid,gold_signup_date) 
 VALUES (1,'09-22-2017'),
(3,'04-21-2017');

CREATE TABLE users(userid integer,signup_date date); 

INSERT INTO users(userid,signup_date) 
 VALUES (1,'09-02-2014'),
(2,'01-15-2015'),
(3,'04-11-2014');

CREATE TABLE sales(userid integer,created_date date,product_id integer); 

INSERT INTO sales(userid,created_date,product_id) 
 VALUES (1,'04-19-2017',2),
(3,'12-18-2019',1),
(2,'07-20-2020',3),
(1,'10-23-2019',2),
(1,'03-19-2018',3),
(3,'12-20-2016',2),
(1,'11-09-2016',1),
(1,'05-20-2016',3),
(2,'09-24-2017',1),
(1,'03-11-2017',2),
(1,'03-11-2016',1),
(3,'11-10-2016',1),
(3,'12-07-2017',2),
(3,'12-15-2016',2),
(2,'11-08-2017',2),
(2,'09-10-2018',3);

CREATE TABLE product(product_id integer,product_name text,price integer); 

INSERT INTO product(product_id,product_name,price) 
 VALUES
(1,'p1',980),
(2,'p2',870),
(3,'p3',330);


select * from sales;
select * from product;
select * from goldusers_signup;
select * from users;

--1. What is the total amount each customer spent on Zomato?

Select 
s.userid,
sum(price) as total_amount
from sales s join product p 
on s.product_id=p.product_id
group by s.userid

--2. How many days each customer visited Zomato?

Select
userid,
count(distinct created_date) visited_days
from sales
group by userid

--3. What was the first product purchased by each of the customer?

Select * 
from(Select
*,
rank()over(partition by userid order by created_date) as rnk
from sales) a
where rnk =1

--4. What is the most purchased item on the menu and how many times was it purchased by all customers?

Select top 1
product_id
from sales
group by product_id
order by count(product_id) desc

Select userid, count(product_id) count_products
from sales 
where product_id = (Select top 1
product_id
from sales
group by product_id
order by count(product_id) desc)
group by userid

--5. Which item was the most popular for each customer?

with t1 as (
select
userid, product_id,
count(product_id) cnt
from sales
group by userid, product_id),
t2 as (
Select
userid,
product_id,
cnt,
rank()over (partition by userid order by cnt desc) rn
from t1)
Select
userid,
product_id,
cnt
from t2
where rn=1

--6. Which item was purchased by the customer after becoming the member?

With t1 as (
Select
s.userid,
s.product_id,
s.created_date,
g.gold_signup_date,
rank()over(partition by s.userid order by s.created_date) rnk
from sales s join goldusers_signup g
on s.userid=g.userid
where s.created_date>=g.gold_signup_date)
select
userid,
product_id,
created_date,
gold_signup_date
from t1 
where rnk=1 

--7. Which item has purchased just before the customer become a member?

With t1 as (
Select
s.userid,
s.product_id,
s.created_date,
g.gold_signup_date,
rank()over(partition by s.userid order by s.created_date desc) rnk
from sales s join goldusers_signup g
on s.userid=g.userid
where s.created_date<g.gold_signup_date)
select
userid,
product_id,
created_date,
gold_signup_date
from t1 
where rnk=1

--8. What is the total orders and amount before they become member?
With t1 as (
Select
s.userid,
s.product_id,
s.created_date,
g.gold_signup_date
from sales s join goldusers_signup g
on s.userid=g.userid
where s.created_date<g.gold_signup_date)
select
a.userid,
count(a.created_date) number_orders,
sum(p.price) price
from t1 a join product p on a.product_id = p.product_id
group by a.userid

--9. If buying each product for eg 5 rs -2 zomato points and each product has different purchase points eg for p1 10 rs - 5 zomato points , for p2 5 rs - 1 zomato points, for p3 5 rs - 1 zomato points.
Calculate points collected by each customes and for which product most points have been given till now.

--Part 1
with t1 as(Select
s.userid,
s.product_id,
sum(p.price) amt,
case when s.product_id =1 then 5
when s.product_id =2 then 2
when s.product_id =3 then 5 else 0 end as points
from sales s join product p
on s.product_id=p.product_id
group by s.userid,s.product_id),
t2 as (Select
userid,
product_id,
amt,
points,
amt/points as total_points
from t1)
Select
userid,
sum(total_points)*2.5 total_earned_points
from t2
group by userid

--Part 2
with t1 as(Select
s.userid,
s.product_id,
sum(p.price) amt,
case when s.product_id =1 then 5
when s.product_id =2 then 2
when s.product_id =3 then 5 else 0 end as points
from sales s join product p
on s.product_id=p.product_id
group by s.userid,s.product_id),
t2 as (Select
userid,
product_id,
amt,
points,
amt/points as total_points
from t1)
Select top 1
product_id,
sum(total_points) total_earned_points
from t2
group by product_id
order by sum(total_points) desc

--10. In the first one year after a customer joins the gold program(including their join date rate) irrespective of what the customer has purchased they earn 5 zomato points for every 10 rs spent who earned more 1 or 3 and what was the their points earnings in their first year?

With t1 as (
Select
s.userid,
s.product_id,
s.created_date,
g.gold_signup_date,
p.price
from sales s join product p
on s.product_id=p.product_id
join goldusers_signup g
on s.userid=g.userid
WHERE s.created_date BETWEEN g.gold_signup_date AND DATEADD(year, 1, g.gold_signup_date))
select
userid,
sum(price)*0.5 price
from t1
group by userid

--11. Rank all the transactions of the customers
Select
*, rank() over(partition by userid order by created_date ) rnk
from sales

--12. Rank all the transactions for each member whenever they are a --gold member for every non gold member transaction mark as na

With t1 as(Select
s.userid,
s.product_id,
s.created_date,
g.gold_signup_date,
cast((case when g.gold_signup_date is null then 0 else rank()over(partition by s.userid order by s.created_date desc) end)as varchar) as rnk
from sales s left join goldusers_signup g
on s.userid=g.userid and s.created_date>g.gold_signup_date)
Select
userid,
product_id,
created_date,
gold_signup_date,
case when rnk=0 then 'na' else rnk end as rnkk
from t1





