<p>Database monitoring is a very essential for most of the database system, it can prevent system crises, improve performance, database failure and auditing the traffic for identify any unwanted access. PostgreSQL has an extension called pg_stat_statements, which can capture a lot of query execution details. Recently, percona release a new extension called pg_stat_monitor which can do what pg_stat_statements did and additionally provide more useful query execution details.</p>

<!-- wp:tadv/classic-paragraph -->
<h3><strong data-renderer-mark="true">Implementation Details</strong></h3>
<p>Before I introduce pg_stat_monitor, I will briefly introduce pg_stat_statements and what it does? Both extensions have many common features, just like following diagram demonstrated, pg_stat_statements will capture data like buffer usage, shared memory usage, wal usage (From version 13) and response time when a query being processed. At the end of each phase, it will measure and store those metrics that can show how many resources were costed in that phase. For example, at start of execution pg_stat_statements will set current system time as start time, then at the end of excution it also fetch end time. So the total time is the difference between start time and end time. Finally, at the end of query execution all these collected data will be stored in shared memory. When user try to get these information from view pg_stat_statements, function pg_stat_statements() will be called. This function use those data which stored in shared memory build tuples, then return tuples as result set. Pg_stat_monitor was implemented almost in the same way.”</p>
<!-- /wp:tadv/classic-paragraph -->

<!-- wp:image {"id":1979,"width":498,"height":364} -->
<figure class="wp-block-image is-resized"><img src="https://www.highgo.ca/wp-content/uploads/2020/11/pg_stat_statements-diagram.png" alt="" class="wp-image-1979" width="498" height="364"/></figure>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>After successful installation and configuration pg_stat_statements, we can use CREATE EXTENSION pg_stat_statements command to create the extension. The sql scripts contained in the extension will create VIEW pg_stat_statements. The column of pg_stet_statements is as following.</p>
<!-- /wp:paragraph -->

<!-- wp:tadv/classic-paragraph -->
<table class="table" style="height: 1669px; width: 100.694%;" border="1" summary="pg_stat_statements Columns">
<thead>
<tr style="height: 24px;">
<th style="width: 16.6884%; height: 24px;">Name</th>
<th style="width: 25.8646%; height: 24px;">Type</th>
<th style="width: 109.728%; height: 24px;">Description</th>
</tr>
</thead>
<tbody>
<tr style="height: 12px;">
<td style="width: 16.6884%; height: 12px;" width="158">
<p>userid</p>
</td>
<td style="width: 25.8646%; height: 12px;" width="177">
<p>oid</p>
</td>
<td style="width: 109.728%; height: 12px;">OID of user who executed the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>dbid</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>oid</p>
</td>
<td style="width: 109.728%; height: 64px;">OID of database in which the statement was executed</td>
</tr>
<tr style="height: 73px;">
<td style="width: 16.6884%; height: 73px;" width="158">
<p>queryid</p>
</td>
<td style="width: 25.8646%; height: 73px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 73px;">Internal hash code, computed from the statement's parse tree</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>query</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>text</p>
</td>
<td style="width: 109.728%; height: 64px;">Text of a representative statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>calls</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Number of times executed</td>
</tr>
<tr style="height: 92px;">
<td style="width: 16.6884%; height: 92px;" width="158">
<p>total_time</p>
</td>
<td style="width: 25.8646%; height: 92px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 92px;">Total time spent in the statement, in milliseconds</td>
</tr>
<tr style="height: 92px;">
<td style="width: 16.6884%; height: 92px;" width="158">
<p>min_time</p>
</td>
<td style="width: 25.8646%; height: 92px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 92px;">Minimum time spent in the statement, in milliseconds</td>
</tr>
<tr style="height: 92px;">
<td style="width: 16.6884%; height: 92px;" width="158">
<p>max_time</p>
</td>
<td style="width: 25.8646%; height: 92px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 92px;">Maximum time spent in the statement, in milliseconds</td>
</tr>
<tr style="height: 92px;">
<td style="width: 16.6884%; height: 92px;" width="158">
<p>mean_time</p>
</td>
<td style="width: 25.8646%; height: 92px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 92px;">Mean time spent in the statement, in milliseconds</td>
</tr>
<tr style="height: 92px;">
<td style="width: 16.6884%; height: 92px;" width="158">
<p>stddev_time</p>
</td>
<td style="width: 25.8646%; height: 92px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 92px;">Population standard deviation of time spent in the statement, in milliseconds</td>
</tr>
<tr style="height: 73px;">
<td style="width: 16.6884%; height: 73px;" width="158">
<p>rows</p>
</td>
<td style="width: 25.8646%; height: 73px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 73px;">Total number of rows retrieved or affected by the statement</td>
</tr>
<tr style="height: 73px;">
<td style="width: 16.6884%; height: 73px;" width="158">
<p>shared_blks_hit</p>
</td>
<td style="width: 25.8646%; height: 73px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 73px;">Total number of shared block cache hits by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>shared_blks_read</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of shared blocks read by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>shared_blks_dirtied</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of shared blocks dirtied by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>shared_blks_written</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of shared blocks written by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>local_blks_hit</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of local block cache hits by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>local_blks_read</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of local blocks read by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>local_blks_dirtied</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of local blocks dirtied by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>local_blks_written</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of local blocks written by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>temp_blks_read</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of temp blocks read by the statement</td>
</tr>
<tr style="height: 64px;">
<td style="width: 16.6884%; height: 64px;" width="158">
<p>temp_blks_written</p>
</td>
<td style="width: 25.8646%; height: 64px;" width="177">
<p>bigint</p>
</td>
<td style="width: 109.728%; height: 64px;">Total number of temp blocks written by the statement</td>
</tr>
<tr style="height: 94px;">
<td style="width: 16.6884%; height: 94px;" width="158">
<p>blk_read_time</p>
</td>
<td style="width: 25.8646%; height: 94px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 94px;">Total time the statement spent reading blocks, in milliseconds (if <a class="xref" href="https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-IO-TIMING">track_io_timing</a> is enabled, otherwise zero)</td>
</tr>
<tr style="height: 92px;">
<td style="width: 16.6884%; height: 92px;" width="158">
<p>blk_write_time</p>
</td>
<td style="width: 25.8646%; height: 92px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 92px;">Total time the statement spent writing blocks, in milliseconds (if <a class="xref" href="https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-IO-TIMING">track_io_timing</a> is enabled, otherwise zero)</td>
</tr>
</tbody>
</table>
<p>As we can see, we can get a lot of query execution details from pg_stat_statement, and by combining those information we can get lots of useful information. For example we can get the top 5 query consume most execution time by running following query:</p>
<pre class="EnlighterJSRAW" data-enlighter-language="typescript">bench=# SELECT pg_stat_statements_reset();

$ pgbench -i bench
$ pgbench -c10 -t300 bench

bench=# \x
bench=# SELECT query, calls, total_time, rows, 100.0 * shared_blks_hit /
               nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
          FROM pg_stat_statements ORDER BY total_time DESC LIMIT 5;
-[ RECORD 1 ]--------------------------------------------------------------
query       | UPDATE pgbench_branches SET bbalance = bbalance + $1 WHERE bid = $2
calls       | 3000
total_time  | 25565.855387
rows        | 3000
hit_percent | 100.0000000000000000
-[ RECORD 2 ]--------------------------------------------------------------
query       | UPDATE pgbench_tellers SET tbalance = tbalance + $1 WHERE tid = $2
calls       | 3000
total_time  | 20756.669379
rows        | 3000
hit_percent | 100.0000000000000000
-[ RECORD 3 ]--------------------------------------------------------------
query       | copy pgbench_accounts from stdin
calls       | 1
total_time  | 291.865911
rows        | 100000
hit_percent | 100.0000000000000000
-[ RECORD 4 ]--------------------------------------------------------------
query       | UPDATE pgbench_accounts SET abalance = abalance + $1 WHERE aid = $2
calls       | 3000
total_time  | 271.232977
rows        | 3000
hit_percent | 98.5723926698852723
-[ RECORD 5 ]--------------------------------------------------------------
query       | alter table pgbench_accounts add primary key (aid)
calls       | 1
total_time  | 160.588563
rows        | 0
hit_percent | 100.0000000000000000
</pre>
<p>More useful examples and other details can be found in <a href="https://www.postgresql.org/docs/12/pgstatstatements.html"><u>pg_stat_statement PG official document</u></a>.</p>
<h3>New features</h3>
<p>Now we can see what information will be gathered by pg_stat_monitor. Just like pg_stat_statements, pg_stat_monitor create a similar VIEW pg_stat_monitor.</p>
<pre class="EnlighterJSRAW" data-enlighter-language="typescript">View "public.pg_stat_monitor"  
       Column        |           Type           | Collation | Nullable | Default   
---------------------+--------------------------+-----------+----------+---------  
 bucket              | integer                  |           |          |   
 bucket_start_time   | timestamp with time zone |           |          |   
 userid              | oid                      |           |          |   
 dbid                | oid                      |           |          |   
 client_ip           | inet                     |           |          |   
 queryid             | text                     |           |          |   
 query               | text                     |           |          |   
 plans               | bigint                   |           |          |   
 plan_total_time     | double precision         |           |          |   
 plan_min_timei      | double precision         |           |          |   
 plan_max_time       | double precision         |           |          |   
 plan_mean_time      | double precision         |           |          |   
 plan_stddev_time    | double precision         |           |          |   
 calls               | bigint                   |           |          |   
 total_time          | double precision         |           |          |   
 min_time            | double precision         |           |          |   
 max_time            | double precision         |           |          |   
 mean_time           | double precision         |           |          |   
 stddev_time         | double precision         |           |          |   
 rows                | bigint                   |           |          |   
 shared_blks_hit     | bigint                   |           |          |   
 shared_blks_read    | bigint                   |           |          |   
 shared_blks_dirtied | bigint                   |           |          |   
 shared_blks_written | bigint                   |           |          |   
 local_blks_hit      | bigint                   |           |          |   
 local_blks_read     | bigint                   |           |          |   
 local_blks_dirtied  | bigint                   |           |          |   
 local_blks_written  | bigint                   |           |          |   
 temp_blks_read      | bigint                   |           |          |   
 temp_blks_written   | bigint                   |           |          |   
 blk_read_time       | double precision         |           |          |   
 blk_write_time      | double precision         |           |          |   
 resp_calls          | text[]                   |           |          |   
 cpu_user_time       | double precision         |           |          |   
 cpu_sys_time        | double precision         |           |          |   
 tables_names        | text[]                   |           |          |   </pre>
<p>As we can see, there are some new attributes like bucket, client_ip, resp_calls and tables_names which are related with new features. Let's see what are these new features and how can we use them.</p>
<!-- /wp:tadv/classic-paragraph -->

<!-- wp:list -->
<ul><li> <strong>Time bucketing:</strong> The biggest difference is time bucketing. If we look at the code of pg_stat_statements, we can found that all the tracked queries are stored in a hash table in shared memory. Userid, dbid and queryid will be used as key to create hash entry. If a query is executed in the same database by same user all the time, pg_stat_statements will keep gathering data from it. With time bucketing, all the queries executed in the same period of time will be divided into same time bucket, and use the bucket id as part of the hash key. So we can focus on the statistic generated in a short period of time. As a result, this could make data like avg/max/min of query more accurate, especially in the case like high resolution or unreliable network.  </li><li><strong>The way they store query text is also changed.</strong> Unlike pg_stat_statements, which stores the query text in a temporary file, pg_stat_monitor stores the query text in shared memory. This method can save the trouble of writing/reading queries to files, but if there are too many large queries, it may also cause Shared memory expansion. But it's totally depends on configuration. </li><li> <strong>Response time histogram:</strong> this new feature will create a array with 10 integer elements to represent response time distribution. After  query is executed, pg_stat_monitor will use the execution time to subtract the lower limit and calculate the number of steps away from the lower limit, and then add 1 to the corresponding element in the array. So we can get the response time histogram of same query. User can adjust the value of lower bound and step to get the histogram they want. Please note that these value cannot be set while the database is running. In a relatively long period of time, the response time histogram can provide a better insight than just using avg/max/min execution time.</li><li> <strong>Non-normalized query:</strong> pg_stat_statements will replace the const value with placeholder(the token location) in query text. In pg_stat_monitor, user can choose to store the normalized or actual query, so user can use EXPLAIN to analyze the query. It's easier to discuss with others with an actual query. </li><li> <strong>Add table name:</strong> pg_stat_monitor also captures names of those tables accessed by the query. This is methor is more reliable than getting table name from the query text. I think it provides another perspective for analyzing information. It allows users to analyze queries that access the same table.</li><li><strong>Add ip address as a new dimension:</strong> Add the user's IP address as part of the hash key to create a hash entry. This allows you to get the performance of queries from specific client addresses. It is valuable in some situations.</li></ul>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>In short, pg_stat_monitor provide a more detailed version of pg_stat_statements without adding too much overhead . Current release version is just as a Technical Preview. There are several planned features still on working. Once it completed, I believe it will be a powerful tool in Postgresql tool kit.</p>
<!-- /wp:paragraph -->
