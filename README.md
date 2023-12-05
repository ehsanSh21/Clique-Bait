# Clique-Bait
SQL Case Study #6

### Introduction:

In this case study - you are required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.

More information about this case study: https://8weeksqlchallenge.com/case-study-6/


### Enterprise Relationship Diagram: 


<div class="language-sql highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">CREATE</span> <span class="k">TABLE</span> <span class="n">clique_bait</span><span class="p">.</span><span class="n">event_identifier</span> <span class="p">(</span>
  <span class="nv">"event_type"</span> <span class="nb">INTEGER</span><span class="p">,</span>
  <span class="nv">"event_name"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">13</span><span class="p">)</span>
<span class="p">);</span>

<span class="k">CREATE</span> <span class="k">TABLE</span> <span class="n">clique_bait</span><span class="p">.</span><span class="n">campaign_identifier</span> <span class="p">(</span>
  <span class="nv">"campaign_id"</span> <span class="nb">INTEGER</span><span class="p">,</span>
  <span class="nv">"products"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">3</span><span class="p">),</span>
  <span class="nv">"campaign_name"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">33</span><span class="p">),</span>
  <span class="nv">"start_date"</span> <span class="nb">TIMESTAMP</span><span class="p">,</span>
  <span class="nv">"end_date"</span> <span class="nb">TIMESTAMP</span>
<span class="p">);</span>

<span class="k">CREATE</span> <span class="k">TABLE</span> <span class="n">clique_bait</span><span class="p">.</span><span class="n">page_hierarchy</span> <span class="p">(</span>
  <span class="nv">"page_id"</span> <span class="nb">INTEGER</span><span class="p">,</span>
  <span class="nv">"page_name"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">14</span><span class="p">),</span>
  <span class="nv">"product_category"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">9</span><span class="p">),</span>
  <span class="nv">"product_id"</span> <span class="nb">INTEGER</span>
<span class="p">);</span>

<span class="k">CREATE</span> <span class="k">TABLE</span> <span class="n">clique_bait</span><span class="p">.</span><span class="n">users</span> <span class="p">(</span>
  <span class="nv">"user_id"</span> <span class="nb">INTEGER</span><span class="p">,</span>
  <span class="nv">"cookie_id"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">6</span><span class="p">),</span>
  <span class="nv">"start_date"</span> <span class="nb">TIMESTAMP</span>
<span class="p">);</span>

<span class="k">CREATE</span> <span class="k">TABLE</span> <span class="n">clique_bait</span><span class="p">.</span><span class="n">events</span> <span class="p">(</span>
  <span class="nv">"visit_id"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">6</span><span class="p">),</span>
  <span class="nv">"cookie_id"</span> <span class="nb">VARCHAR</span><span class="p">(</span><span class="mi">6</span><span class="p">),</span>
  <span class="nv">"page_id"</span> <span class="nb">INTEGER</span><span class="p">,</span>
  <span class="nv">"event_type"</span> <span class="nb">INTEGER</span><span class="p">,</span>
  <span class="nv">"sequence_number"</span> <span class="nb">INTEGER</span><span class="p">,</span>
  <span class="nv">"event_time"</span> <span class="nb">TIMESTAMP</span>
<span class="p">);</span>
</code></pre></div></div>


### Case Study Questions

You can use the link of DB Fiddle below to easily access this datasets and use my queries!

<code class="language-plaintext highlighter-rouge">DB Fiddle link: </code> https://www.db-fiddle.com/f/jmnwogTsUE8hGqkZv9H7E8/17


#### Product Funnel Analysis:

Using a single SQL query - create a new output table which has the following details:

<ul>
  <li>How many times was each product viewed?</li>
  <li>How many times was each product added to cart?</li>
  <li>How many times was each product added to a cart but not purchased (abandoned)?</li>
  <li>How many times was each product purchased?</li>
</ul>

```sql
WITH products_cte AS (
SELECT *
FROM
clique_bait.page_hierarchy
WHERE product_id IS NOT NULL
),

detailed_cte AS(
SELECT
all_product.product_id as prod_id,
all_product.page_name as prod_name,
events.*,
events.page_id as event_page,
products_cte.product_id as viewed_product_id,
cart.product_id as cart_product_id,
   CASE
     WHEN MAX (CASE WHEN events.page_id = 13 THEN 1 ELSE 0 END) OVER
(PARTITION BY visit_id) = 0
     THEN 1
     ELSE 0
   END AS but_not_purchased,

          CASE
     WHEN MAX (CASE WHEN events.page_id = 13 THEN 1 ELSE 0 END) OVER
(PARTITION BY visit_id) = 1
     THEN 1
     ELSE 0
   END AS purchased

FROM
clique_bait.events
left JOIN

products_cte as all_product ON events.page_id = all_product.page_id

left JOIN
products_cte ON events.page_id = products_cte.page_id AND
events.event_type = 1

LEFT JOIN products_cte as cart ON events.page_id = cart.page_id AND
events.event_type = 2

ORDER BY visit_id,sequence_number ASC)

SELECT
prod_id,
prod_name,
sum(case WHEN viewed_product_id is not null then 1 else 0 END) as
views_count,
sum(case WHEN cart_product_id is not null then 1 else 0 END) as
added_to_cart_count,
sum(case WHEN but_not_purchased = 1 AND cart_product_id is not null then
1 else 0 END) as but_not_purchased_count,
sum(case WHEN purchased = 1 AND cart_product_id is not null then 1 else
0 END) as purchased_count
from detailed_cte
WHERE prod_id is not NULL
GROUP BY prod_id,prod_name
;
```

###### output:

| prod_id | prod_name      | views_count | added_to_cart_count | but_not_purchased_count | purchased_count |
| ------- | -------------- | ----------- | ------------------- | ----------------------- | --------------- |
| 9       | Oyster         | 1568        | 943                 | 217                     | 726             |
| 6       | Abalone        | 1525        | 932                 | 233                     | 699             |
| 3       | Tuna           | 1515        | 931                 | 234                     | 697             |
| 5       | Black Truffle  | 1469        | 924                 | 217                     | 707             |
| 8       | Crab           | 1564        | 949                 | 230                     | 719             |
| 1       | Salmon         | 1559        | 938                 | 227                     | 711             |
| 2       | Kingfish       | 1559        | 920                 | 213                     | 707             |
| 4       | Russian Caviar | 1563        | 946                 | 249                     | 697             |
| 7       | Lobster        | 1547        | 968                 | 214                     | 754             |






