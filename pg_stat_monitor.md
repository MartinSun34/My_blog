<p>Database monitoring is a very important for the database system, it can prevent a system crisis and improve performance. So a monitoring tool can provide useful insight of the internal of &nbsp;database is important. Postgresql has a extension called pg_stat_statements, which can capture lot of query execution details of query. Recently, percona release a new extension called pg_stat_monitor which can do what pg_stat_statements did and provide more useful query execution details.</p>

<!-- wp:tadv/classic-paragraph -->
<h4>How it was implementd</h4>
<p>Before I introduce pg_stat_monitor, I will briefly introduce the what pg_stat_statements dose first, because they have many common features. Just like following diagram demonstrated. Pg_stat_statements will capture data like buffer usage, shared memory usage, wal usage (From version 13) and response time when a query being processed. And at beginning and end of each phase, it will measure those metrics so it can get how many resources and how much time were costed from the difference of begin value and end value. when user try to get these information from view pg_stat_statements, function pg_stat_statements() will be called and use these data to build tuples then return these tuples as result set. Pg_stat_monitor was implemented almost in the same way.</p>
<!-- /wp:tadv/classic-paragraph -->

<!-- wp:image {"id":1979,"align":"center","width":498,"height":364} -->
<div class="wp-block-image"><figure class="aligncenter is-resized"><img src="https://www.highgo.ca/wp-content/uploads/2020/11/pg_stat_statements-diagram.png" alt="" class="wp-image-1979" width="498" height="364"/></figure></div>
<!-- /wp:image -->

<!-- wp:paragraph -->
<p>After successful installation and configuration pg_stat_statements, we can use CREATE EXTENSION pg_stat_statement command to create the extension. The sql scripts contained in the extension will create VIEW pg_stat_statements. the column of pg_stet_statements is as following.</p>
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
<td style="width: 109.728%; height: 94px;">Total time the statement spent reading blocks, in milliseconds (if <a class="xref" href="https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-IO-TIMING">track_io_timing</a> is enabled, otherwise zero)</td>
</tr>
<tr style="height: 92px;">
<td style="width: 16.6884%; height: 92px;" width="158">
<p>blk_write_time</p>
</td>
<td style="width: 25.8646%; height: 92px;" width="177">
<p>double precision</p>
</td>
<td style="width: 109.728%; height: 92px;">Total time the statement spent writing blocks, in milliseconds (if <a class="xref" href="https://www.postgresql.org/docs/12/runtime-config-statistics.html#GUC-TRACK-IO-TIMING">track_io_timing</a> is enabled, otherwise zero)</td>
</tr>
</tbody>
</table>
<p>As we can see, we can get a lot of query execution details from pg_stat_statement, and by combining those information we can get lots of useful information. For example we can get the top 5 query consume most I/O by running following query:</p>
<pre class="EnlighterJSRAW" data-enlighter-language="sql">select userid::regrole, dbid, query from pg_stat_statements order by (blk_read_time+blk_write_time)/calls desc limit 5  </pre>
<p>More useful examples and other details can be found in <a href="https://www.postgresql.org/docs/12/pgstatstatements.html"><u>pg_stat_statement PG official document</u></a>.</p>
<h4>New features</h4>
<p>And now we can see what information will be gathered by pg_stat_monitor. Just like pg_stat_statements, pg_stat_monitor create a similar VIEW pg_stat_monitor.</p>
<pre class="EnlighterJSRAW" data-enlighter-language="typescript">View "public.pg_stat_monitor"  
       Column        |           Type           | Collation | Nullable | Default   
---------------------+--------------------------+-----------+----------+---------  
 bucket              | integer                  |           |          |   
 bucket_start_time   | timestamp with time zone |           |          |   
 userid              | oid                      |           |          |   
 dbid                | oid                      |           |          |   
 client_ip           | inet                     |           |          |   
 queryid             | text                     |           |          |   
 query               | text                     |           |          |   
 plans               | bigint                   |           |          |   
 plan_total_time     | double precision         |           |          |   
 plan_min_timei      | double precision         |           |          |   
 plan_max_time       | double precision         |           |          |   
 plan_mean_time      | double precision         |           |          |   
 plan_stddev_time    | double precision         |           |          |   
 calls               | bigint                   |           |          |   
 total_time          | double precision         |           |          |   
 min_time            | double precision         |           |          |   
 max_time            | double precision         |           |          |   
 mean_time           | double precision         |           |          |   
 stddev_time         | double precision         |           |          |   
 rows                | bigint                   |           |          |   
 shared_blks_hit     | bigint                   |           |          |   
 shared_blks_read    | bigint                   |           |          |   
 shared_blks_dirtied | bigint                   |           |          |   
 shared_blks_written | bigint                   |           |          |   
 local_blks_hit      | bigint                   |           |          |   
 local_blks_read     | bigint                   |           |          |   
 local_blks_dirtied  | bigint                   |           |          |   
 local_blks_written  | bigint                   |           |          |   
 temp_blks_read      | bigint                   |           |          |   
 temp_blks_written   | bigint                   |           |          |   
 blk_read_time       | double precision         |           |          |   
 blk_write_time      | double precision         |           |          |   
 resp_calls          | text[]                   |           |          |   
 cpu_user_time       | double precision         |           |          |   
 cpu_sys_time        | double precision         |           |          |   
 tables_names        | text[]                   |           |          |   </pre>
<p>As we can see, I highlighted these new attributes which related with new features. And lets see what are these new features and how can we use them.</p>
<!-- /wp:tadv/classic-paragraph -->

<!-- wp:list -->
<ul><li> <strong>Time bucketing:</strong>&nbsp;The biggest difference is time bucketing. If we look at the code of pg_stat_statements, we can found that all the tracked queries are stored in a hash table in shared memory. And userid, dbid and queryid will be used as key to create hash entry. If a query is executed in the same database by same user all the time, pg_stat_statements will keep gathering data from it. With time bucketing, all the queries executed in the same period of time will be divided into same time bucket, and use the bucket id as part of the hash key. So we can get focus on the statistic generated in a short period of time. This could make data like avg/max/min of query more accurate, especially in the case like high resolution or unreliable network.  </li><li> And the way they store query text is also changed. Unlike pg_stat_statement puts queries in a temporary file, pg_stat_monitor stores the query text in shared memory, this could save the trouble to write/read query to file but it also could cause shared memory expansion if there are too many large queries. But it’s totally depends on configuration. </li><li> <strong>Response time histogram:</strong>&nbsp;this new feature will create a array with 10 integer elements to represent response time distribution. Every time after query execution, pg_stat_monitor use the execution time minus the lower bound and calculate how many steps away from the lower bound then add 1 to the corresponding element in the array,&nbsp;so we can get the response time histogram of same query. User can adjust the value of lower bound and step to get the histogram they want, but these value cannot be set while the datbase is running. For same query, same database, same user and same time bucket, if there’s a surge in&nbsp;response time&nbsp;histogram, it can warn DBA or&nbsp;system admin to do some check. </li><li> <strong>Non-normalized query:</strong>&nbsp;pg_stat_statements&nbsp;will replace the const value with placeholder(the location token) in the query text. In pg_stat_monitor&nbsp;user&nbsp;can choose to store the normalized or actual query,&nbsp;so user can use EXPLAIN to analyze the query. And it’s easier to discuss with others with an actual query. </li><li> <strong>Add table name:</strong>&nbsp;pg_stat_monitor also captures what tables will be accessed by a query. I thought it just provide another perspective to analyze the information. This allow users to&nbsp;analyze the those queries which access same table.&nbsp;For example, we can find the slowest query executing on the same table and try to improve its performance. </li><li><strong>Add ip address as a new dimension:</strong>&nbsp;add user's ip address as part of hash key to create a hash entry.&nbsp;If same query are executed at same time bucket in same database as same user but have very different performance, it&nbsp;can notify DBA to check&nbsp;client side or connection configuration.&nbsp;</li></ul>
<!-- /wp:list -->

<!-- wp:paragraph -->
<p>In short, pg_stat_monitor provides a more detailed version of pg_stat_statement without adding too much overhead . Current release version is just as a Technical Preview. There are several planned features still on working. Once it completed, I believe it will be a powerful tool in Postgresql tool kit.</p>
<!-- /wp:paragraph -->