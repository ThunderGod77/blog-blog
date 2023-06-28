---
author: Kshitij Gang
pubDatetime: 2023-06-28T13:10:15Z
title: "Materialized Views in Postgres SQL"
postSlug: materialized-view-postgres
featured: false
draft: false
tags:
  - postgres
  - materialized views
ogImage: ""
description: How to optimize your queries using Materialized Views?
---

### What is a materialized view?

Materialized views allow us to store the result of a very complicated long-running query on the disk. We can now use the saved query as any other table in the database and query results from it. Materialize view is a cached result of a query, so it will not update its values even if the underlying tables it's derived from are updated/changed. To get the new values we will have to refresh our materialized view.

## When will a materialized view be beneficial to us?

For example, we have an e-commerce application where we are selling different kinds of jams in certain regions of India, and our business team might be interested in how many jars of jam are we selling in each city. If we have more than a million orders per day, it might become a complicated query and will take a significant time every time we execute it. So the better approach will be to store our results in a materialized view.

Example Tables in our database:

```sql
--creating our jams table
CREATE TABLE jams (
  id serial PRIMARY KEY,
  name varchar(50),
  price int,
  description text,
  metadata text
)
-- Inserting 10 sample jams into our jams table

INSERT INTO jams (name, price, description, metadata)
  VALUES ('Strawberry Jam', 5, 'Delicious strawberry jam made from fresh berries', 'Organic, no preservatives'),
  ('Blueberry Jam', 6, 'Sweet and tangy blueberry jam with whole berries', 'Locally sourced ingredients'),
  ('Raspberry Jam', 4, 'Smooth raspberry jam with a hint of tartness', 'Made from hand-picked raspberries'),
  ('Apricot Jam', 5, 'Rich and flavorful apricot jam made from ripe fruits', 'Naturally sweetened'),
  ('Peach Jam', 5, 'Juicy peach jam with chunks of fruit', 'Homemade recipe'),
  ('Blackberry Jam', 6, 'Bold and robust blackberry jam with natural sweetness', 'Handcrafted in small batches'),
  ('Mixed Berry Jam', 7, 'A delightful blend of strawberries, blueberries, and raspberries', 'No artificial colors or flavors'),
  ('Mango Jam', 6, 'Exotic mango jam with a tropical twist', 'Made from sun-ripened mangoes'),
  ('Pineapple Jam', 5, 'Tropical pineapple jam bursting with flavor', 'Perfect for spreading on toast'),
  ('Cherry Jam', 4, 'Sweet cherry jam made from plump cherries', 'Locally sourced and freshly made');

```

```sql
--creating our orders table

CREATE TABLE orders (
  id serial PRIMARY KEY,
  jam_id int,
  quantity int,
  region varchar(20),
  user_id int,
  shipper text,
  CONSTRAINT fk_jam
      FOREIGN KEY(jam_id)
    REFERENCES jams(id)
)

```

We will now insert approximately 5 million rows into our orders table.[(You can find the golang code to do so here)](https://github.com/ThunderGod77/blog-code/tree/main/mt-view)

We will now write a SQL query to join the jams and orders table and find the aggregate sales for every jam in each region

```sql
SELECT jams.name,SUM(orders.quantity),orders.region,SUM(orders.quantity*jams.price)
FROM orders
LEFT JOIN jams
ON orders.jam_id = jams.id
GROUP by jams.name, orders.region;
```

![Result of the query](/assets/materialized-views/mv-blog-agg-query.png)
![Time take by the query](/assets/materialized-views/time-take-agg-q.png)

This query took almost 90 seconds to run, so every time we will have to generate reports using this query it will take a huge amount of time, so it makes much more sense to use a materialized view. This will cache the result on the disk and we can further query within the resulting table generated from the query. It will save us a lot of time and if we want to update the result, we can easily do so by refreshing the materialized views.

So we will create a materialized view now

```sql
CREATE MATERIALIZED VIEW jam_order_agg
AS
SELECT jams.name,SUM(orders.quantity),orders.region,SUM(orders.quantity*jams.price) as total
FROM orders
LEFT JOIN jams
ON orders.jam_id = jams.id
GROUP by jams.name, orders.region;
```

If we query the materialized view instead of executing the original query now, it will take less than an ms.

![Time take when querying the materialized view](/assets/materialized-views/mv-query-time.png)

We can also filter data when querying. from a materialized view

![materialized view filter](/assets/materialized-views/querying-the-mv-filter.png)

If for some reason we will delete all the orders from the 'east' region(due to some logistical issues in our jam company).

```sql
delete from orders where region='east'
```

If we query the materialized view we will realize the data is not updated, and we can still see the data of 'east' region. We will now have to refresh the materialized view to see the latest data.

```sql
REFRESH MATERIALIZED VIEW jam_order_agg;
select * from jam_order_agg where region='east';
```

![materialized view result after refresh](/assets/materialized-views/refresh-materialized-view-result.png)

Please note that REFRESH MATERIALIZED VIEW command does block the view in AccessExclusive mode, so while it is working, you can't even do SELECT on the table. Although, if you are in version 9.4 or newer, you can give it the CONCURRENTLY option - this will acquire an ExclusiveLock, and will not block SELECT queries, but may have a bigger overhead.

## Differenced between view and materialized view in Postgres?

View is a virtual table from one or more base tables, they actually just represent a select query (does not store the result of the select query). It acts as a placeholder for a particular select query, they are processed each time when they are used and always contain the updated data.

```sql
CREATE VIEW av
AS
SELECT jams.name,SUM(orders.quantity),orders.region,SUM(orders.quantity*jams.price) as total
FROM orders
LEFT JOIN jams
ON orders.jam_id = jams.id
GROUP by jams.name, orders.region;

```

![view query twice](/assets/materialized-views/view-query-2.png)

As we can see this query took 90 seconds double the time to run the select query, which tells us that views do not cache the result.

While materialized views on the other hand store the select query and cache it's result in the disk. So they are computed only at the time of creation or while refreshing, They are much faster to access but might not contain the latest data.

## Conclusion

So we learned how materialized views can be used in our applications and can save us significant time when querying aggregated/transformed data. Please note you can use 'select' statements and create indexes on a materialized view but cannot use the 'insert' command on it. Sometimes you need to move, replace, or add particular elements within a materialized view, to complete this task, the ALTER MATERIALIZED VIEW command can be used you can delete the materialized view using DROP MATERIALIZED VIEW command.
