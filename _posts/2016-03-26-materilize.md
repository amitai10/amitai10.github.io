---
title: Materialize your views!
teaser: How, why and when to use materialized views in postgres
category: Database
tags: [postgres]
---

I value performance a great deal. I believe a web application today should be as responsive as possible. Our customer's patience for latency is close to zero. It is easier when the application renders static pages, but what should we do when the application has to present the results of complex calculations or analysis? In order to display the data fast we cannot run the calculation on demand. It just wouldn't be fast enough.

In this blog post i will demonstrate one method that can help us achieve this goal. This notion is called _materialized views_.

## What are materialized views?
Materialized views are database objects that contain the result of a query ([wiki definition](https://en.wikipedia.org/wiki/Materialized_view)). The query can be a calculation of an aggregate function or a join of tables or any other query. The incentive for it is to replace a long timed query. Materialized views are generally used for demonstrating reports, graphs and any other data that is a result of calculation. Another common use case is to do an abstraction of a complex data structure (join of tables).

Several databases support materialized views. The main ones are PostgreSQL, Oracle and DB2 of IBM.

The idea of materialized views is to cache the result of the query (In the disk - like regular table) in order to serve it to the customer fast, without the need for calculations or joins. The application can refresh the results whenever it needed according to the use case (manually, schedule or event driven). Extra performance boost can be achieved by adding indexes to the materialized view.

The cons of materialized view are that the result is not 'real time'. If the materialized view is not refreshed, it will serve a result that is different from the actual data in the database (Many times, it is fine because of the manner of the data). A part from that, it is taking  disk space and  CPU time for refreshing. This can hurt the performance, especially if there are many materialized views and the refresh rate is high.

## Dosn't it sounds like OLAP?
OLAP (Online analytic processing) is a database that is used for business intelligent (BI) and reports. OLAP has operations like consolidation, drill-down, and 'slicing and dicing'. Although materialized views can be used for reports and analytic results, OLAP has much more strength an should be used in case the application has significant BI module that has a lot of reports and calculations on those reports.
The down side of OLAP is the complexity and cost of maintaining another machine and software, and the transformation of the data from the OLTP (_Online transaction processing_ - the database that stores the data of the application) to the OLAP.

## Materialized views in practice
In order to demonstrate the power of materialized views I will perform and measure the following:
- A simple query
- A simple view
- Materialized view

I will use PostgerSQL 9.4. In previous versions, you had to  lock the materialized view during refresh, and no queries could be made. In version 9.4 it has been [solved](https://wiki.postgresql.org/wiki/What's_new_in_PostgreSQL_9.4#REFRESH_MATERIALIZED_VIEW_CONCURRENTLY).

The performance test took place on my laptop. I inserted 1M records into my database using [Faker](https://github.com/stympy/faker). The records represent _'Invoices_', and the use case is a monthly report of the total amount of money per customer.

##### Query:
The query I will perform is:
```SQL
SELECT sum(total), customer, extract(month from invoice_date) AS month
FROM invoices
GROUP BY customer, month;
```
Running this query with _EXPLAIN ANALYZE_ (in order to test the performance) will get the following results:
```
"GroupAggregate  (cost=146051.21..157399.90 rows=96585 width=28) (actual time=5724.266..9894.844 rows=647075 loops=1)"
"  Group Key: customer, (date_part('month'::text, (invoice_date)::timestamp without time zone))"
"  ->  Sort  (cost=146051.21..148465.82 rows=965846 width=28) (actual time=5724.254..9563.611 rows=965788 loops=1)"
"        Sort Key: customer, (date_part('month'::text, (invoice_date)::timestamp without time zone))"
"        Sort Method: external merge  Disk: 44400kB"
"        ->  Seq Scan on invoices  (cost=0.00..26928.69 rows=965846 width=28) (actual time=0.019..252.742 rows=965788 loops=1)"
"Planning time: 0.140 ms"
"Execution time: 9919.586 ms"
```
Almost 10 seconds!!
##### View:
```SQL
CREATE monthly_view
SELECT sum(total), customer, extract(month from invoice_date) AS month
FROM invoices
GROUP BY customer, month;

EXPLAIN ANALYZE select * from monthly_view;
```
The results are almost the same:
```
"GroupAggregate  (cost=146051.21..157399.90 rows=96585 width=28) (actual time=5711.102..9863.692 rows=647075 loops=1)"
"  Group Key: invoices.customer, (date_part('month'::text, (invoices.invoice_date)::timestamp without time zone))"
"  ->  Sort  (cost=146051.21..148465.82 rows=965846 width=28) (actual time=5711.088..9534.372 rows=965788 loops=1)"
"        Sort Key: invoices.customer, (date_part('month'::text, (invoices.invoice_date)::timestamp without time zone))"
"        Sort Method: external merge  Disk: 44400kB"
"        ->  Seq Scan on invoices  (cost=0.00..26928.69 rows=965846 width=28) (actual time=0.021..256.500 rows=965788 loops=1)"
"Planning time: 0.259 ms"
"Execution time: 9888.132 ms"
```
It makes a lot of sense because the regular view runs its query in order to serve the result. The advantage of regular view is to abstract the query, so the application can use it easily.

##### Materialized view:
```SQL
CREATE materialized view materialized_monthly_view as
SELECT sum(total), customer, extract(month from invoice_date) AS month
FROM invoices
GROUP BY customer, month;


EXPLAIN ANALYZE select * from materialized_monthly_view;
```

And it took...
```
"Seq Scan on materialized_monthly_view  (cost=0.00..11852.75 rows=647075 width=35) (actual time=0.021..51.068 rows=647075 loops=1)"
"Planning time: 0.022 ms"
"Execution time: 69.179 ms"
```

WOW!, less then 100ms, more than 1000 times faster.
This is of course because returning the result do not run the complex query, but rather returning the results as it was a table.

## Refreshing the materialized view
The command for refreshing the view is:
```SQL
REFRESH MATERIALIZED VIEW materialized_monthly_view;
```
If we will add a unique index to the materialized view, we can use:
```SQL
REFRESH MATERIALIZED VIEW CONCURRENTLY materialized_monthly_view;
```
This is the new feature I mentioned before, this way, the
materialized view will not be locked for queries.


## Using materialized views in Rails
If we needed to perform a complex query from rails we probeblly had used the ActiveRecord API such as:
```ruby
ActiveRecord::Base.connection.execute(...)
```
Materialized view let us use the regular ActiveRecord API.
All we have to do is create a model that represent the materialized view. Then, we can query the same as other records.
```ruby
class CreateCustomerMonthlyMaterializedView < ActiveRecord::Migration
  def change
    execute %{
      CREATE MATERIALIZED VIEW customer_monthlies AS
      SELECT sum(total), customer, extract(month from invoice_date) AS month
      FROM invoices
      GROUP BY customer, month;
    }

    execute %{
      CREATE UNIQUE INDEX month_index on customer_monthlies(month)
    }
  end
end
```
Notice that I added the unique index for concurrently refresh.

In order to query he materialized view I can run:
```ruby
CustomerMonthly.all
```

## Conclusion
Materialized views are great performance booster for responsiveness. It is very easy to use and maintain. It is not a substitution for OLAP or other analytic software, but it can be helpful in case you need to serve data to a simple report or join tables together to construct a record that its data stores in few tables. This is only an introduction, there is a lot more to it and you can explore it yourself. This is a good place to [start](http://tech.jonathangardner.net/wiki/PostgreSQL/Materialized_Views).

#### Other references I used:
- https://hashrocket.com/blog/posts/materialized-view-strategies-using-postgresql
- Rails, Angular, Postgres, and Bootstrap Powerful, Effective, and Efficient Full-Stack Web Development by David Bryant Copeland, ISBN 978-1-68050-126-1
