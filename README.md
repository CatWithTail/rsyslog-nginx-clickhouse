# rsyslog-nginx-clickhouse


It's common knowledge that NGINX is quite capable of writing its own logs for us to read, search, parse, analyze, and visualize. A popular solution here is the ELK (ELK - elasticsearch, logstash, kibana, so why filebeat?) stack that uses Filebeat (link needed) to transport the logs, Logstash (link needed) to parse and convert them, Elasticsearch (link needed) to store the resulting data, and Kibana to visualize them nicely.


However, this approach comes with costs at every phase. You may have to install Logstash; Elasticsearch is often resource-intensive and sometimes slow; Kibana, used as a GUI on top of Elasticsearch, also runs slowly and doesn't support regular SQL syntax. Our goal is to offer a nice solution to all of these issues.

TL;DR
(CONSOLE COMMANDS GO INLINE, NO GIST NEEDED)

```
git clone https://github.com/CatWithTail/rsyslog-nginx-clickhouse.git
cd rsyslog-nginx-clickhouse
clickhouse-client  --query="$(cat nginx.click);"
cp nginx.{conf,rule,table} /etc/rsyslog.d/
service rsyslog restart
```

Enjoy!


Background


The key idea here is to parse NGINX-generated log files with Rsyslog and feed the results to ClickHouse for visualization and reporting. On top of that, we can add Grafana as the GUI instead of ClickHouse's native Tabix client.


(IMAGE NEEDED): access.log files -> rsyslog -> clickhouse -> grafana.


These choices are fueled by the following considerations. First, Rsyslog is widely used on *nix systems; it's simple and flexible, has a ClickHouse module and comes pre-installed with CentOS and Ubuntu. Rsyslog can interface with multiple systems, applying various parsing rulesets for different kinds of logs. Second, ClickHouse is inherently well-suited to SELECT queries that are used in visualization. Also, it compresses stored data, requiring fewer resources than Elasticsearch.


### 1. NGINX


A basic setup requires no action from us; NGINX's default access log format looks like this:
(GIST NEEDED)
```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```

if the remote_user variable is empty, '-' is used in its place. There's not much to add, so nothing to do here, move along, citizen!


However, you can add some sugar with extra variables such as $upstream_addr, $upstream_connect_time, $upstream_header_time, or $upstream_response_time. If you aren't going to use them as SELECT parameters, you can safely put them after $http_x_forwarded_for to avoid breaking changes. On the other hand, if you need some of these in your SELECT queries, you'll need to create a special column and a special field in the normalizer rule for each variable.


For the purposes of this article, let's assume the default log format is used.


### 2. Rsyslog


Usually, Rsyslog serves as the log collector, picking up logs from different locations and storing them according to a set of pre-configured rules. In this case, we are parsing NGINX access logs to store them in ClickHouse. Also, mind that much of Rsyslog's functionality stems from its many plugins, or modules. Here, we suggest using three of them: imfile, mmnormalize, and omclickhouse.


The imfile module is used to read text files; it comes embedded in Rsyslog. Usually, loading it requires only an appropriate config file setting. The ``mmnormalize`` module is used by Rsyslog to parse log strings as value arrays according to a set of rules that are set in the configuration; you can install it from this repo (LINK NEEDED). Otherwise, if only for the sake of testing, you can try other log normalizers such as liblognorm1 or liblognorm1-utils.


Finally, omclickhouse is Rsyslog's support module that enables feeding logs to ClickHouse. Unfortunately, it may be unavailable in the repo yet, so please refer to these instructions (LINK https://www.rsyslog.com/doc/master/configuration/modules/omclickhouse.html).
To split the access logs into variables, we use lognormalize. Basically, an NGINX log looks similar to this:
(GIST NEEDED)
```
127.0.0.1 - - [06/Apr/2020:09:54:48 -0400] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
```

It can be interpreted this way:

| log variable       | example   | parsed variable|  comment                       |
|--------------------|:---------:|----------------|--------------------------------|
| remote_addr        | 127.0.0.1 | clientip       |                                |
|        -           |     -     | ident          |Reserved variable               | 
| remote_user        |     -     | auth           |                                |
| time local         |    06     | day            |Split into several values       | 
|                    |   Apr     | month          |                                |
|                    |   2020    | year           |                                |
|                    | 09:54:48  | rtime          |                                |
|                    | -0400     | tz             |                                | 
| request            |   GET     | verb           | Split as well                  |
|                    | /         | request        |                                | 
|                    | HTTP/1.1  | httpversion    |                                |
| status             | 200       | response       |                                |
| body_bytes_sent    | 612       | bytes          |                                |
| http_referer       | -         | referrer       |                                | 
| http_user_agent    |curl/7.29.0| referrer       |                                | 
|http_x_forwarded_for| -         | blob           | Contains the remaining log part| 


Dates should be parsed because NGINX's default time format is not suitable for ClickHouse, and changing this format may not be a good idea: other software may rely on the same logs. All in all, we need to convert month values from strings to numbers, storing them in a separate variable.


To achieve all that, let's create a ``lognormalizer`` rule in /etc/rsyslog.d/nginx.rule:
(GIST NEEDED)

```
version=2


rule=:%clientip:word% %ident:word% %auth:word% [%day:char-to:/%/%month:char-to:/%/%year:number%:%rtime:word% %tz:char-to:]%] "%verb:word% %request:word% HTTP/%httpversion:float%" %response:number% %bytes:number% "%referrer:char-to:"%" "%agent:char-to:"%"%blob:rest%
```

Now test the rule:
(CONSOLE COMMANDS GO INLINE, NO GIST NEEDED)
```
curl 127.0.0.1; tail -n 1 /var/log/nginx/access.log | lognormalizer  -r /etc/rsyslog.d/nginx.rule -e json
```
The command's output should resemble the following JSON:


```
{ "blob": " \"-\"", "agent": "curl\/7.29.0", "referrer": "-", "bytes": "612", "response": "200", "httpversion": "1.1", "request": "\/", "verb": "GET", "tz": "-0400", "rtime": "09:54:48", "year": "2020", "month": "Apr", "day": "06", "auth": "-", "ident": "-", "clientip": "127.0.0.1" }
```

You can see that the month value in the data is still a string, so we should transform t it into a number using a conversion table; store the following in /etc/rsyslog.d/nginx.table:
(GIST NEEDED)
```
{ "version":1, "nomatch":"unk", "type":"string",
 "table":[ {"index":"Jan", "value":"01" },
      {"index":"Feb", "value":"02" },
      {"index":"Mar", "value":"03" },
      {"index":"Apr", "value":"04" },
      {"index":"May", "value":"05" },
      {"index":"Jun", "value":"06" },
      {"index":"Jul", "value":"07" },
      {"index":"Aug", "value":"08" },
      {"index":"Sep", "value":"09" },
      {"index":"Oct", "value":"10" },
      {"index":"Nov", "value":"11" },
      {"index":"Dec", "value":"12" }
     ]
}
```

This means we end up with the following algorithm:


	•	Read a log row
	•	Split it into separate variables
	•	Convert month value to the new format
	•	Store the data in the database using a template


NOTE ADMONITION:
Mind that Rsyslog has different zones for variables. To feed our month value into the !usr!nxm variable that is used in the template, we need to use a separate "out" ruleset that is invoked by the first ruleset. Instructions on writing templates can be found in omclickhouse documentation (LINK NEEDED); basically, a ClickHouse template looks like a common INSERT query.


For testin purposes, we can put some parsed data into a file and check it using the output rule in config:
(GIST NEEDED)
```
action(type="omfile" file="/var/log/nginx.parsed" template="ng")
```

If everything goes OK, the file will contain something like this:
(GIST NEEDED)
```
INSERT INTO nginx.nginx (logdate, logdatetime, hostname, syslogtag, message, clientip, ident, auth, verb, request, httpv, response, bytes, referrer, agent, blob ) values ('2020-04-07', '2020-04-07 19:21:13', 'unit', 'nginx', '127.0.0.1 - - [07/Apr/2020:19:21:13 -0400] \"GET / HTTP/1.1\" 200 612 \"-\" \"curl/7.29.0\" \"-\"', '127.0.0.1', '-', '-', 'GET', '/', '1.1', '200', '612', '-', 'curl/7.29.0', ' \"-\"');
```

The rules and the template are stored in the same nginx.conf file:
(GIST NEEDED)

```
lookup_table(name="months" file="/etc/rsyslog.d/nginx.table" reloadOnHUP="off")
template(name="ng" type="list" option.json="on") {
  constant(value="INSERT INTO nginx (logdate, logdatetime, hostname, syslogtag, message, clientip, ident, auth, verb, request, httpv, response, bytes, referrer, agent, blob ) values ('")
  property(name="$!year")
  constant(value="-")
  property(name="$!usr!nxm")
  constant(value="-")
  property(name="$!day")
  constant(value="', '")
  property(name="$!year")
  constant(value="-")
  property(name="$!usr!nxm")
  constant(value="-")
  property(name="$!day")
  constant(value=" ")
  property(name="$!rtime")
  constant(value="', '")
  property(name="hostname")
  constant(value="', '")
  property(name="syslogtag")
  constant(value="', '")
  property(name="msg")
  constant(value="', '")
  property(name="$!clientip")
  constant(value="', '")
  property(name="$!ident")
  constant(value="', '")
  property(name="$!auth")
  constant(value="', '")
  property(name="$!verb")
  constant(value="', '")
  property(name="$!request")
  constant(value="', '")
  property(name="$!httpversion")
  constant(value="', '")
  property(name="$!response")
  constant(value="', '")
  property(name="$!bytes")
  constant(value="', '")
  property(name="$!referrer")
  constant(value="', '")
  property(name="$!agent")
  constant(value="', '")
  property(name="$!blob")
  constant(value="');\n")
}
# if clickhouse version <20, remove ' near the UNIT values

module(load="imfile")
module(load="omclickhouse")
module(load="mmnormalize")
input(type="imfile" file="/var/log/nginx/access.log" tag="nginx" ruleset="norm")
ruleset(name="norm") {
  action(type="mmnormalize" rulebase="/etc/rsyslog.d/nginx.rule" useRawMsg="on")
  set $!usr!nxm = lookup("months", $!month);
  call out
}
ruleset(name="out") {
# action(type="omfile" file="/var/log/nginx.parsed" template="ng")
  action(type="omclickhouse" server="127.0.0.1" port="8123"
  usehttps="off"
  template="ng")
}
```


### 3. ClickHouse


Next, let's create a table to store log data in the ClickHouse database. For this, we employ the developer-recommended method, adding a few designated fields:
	•	logdate - to partition the table
	•	logdatetime - to sort data
	•	hostname - to store the log creator host name
	•	syslogtag - to store the tag from syslog/Rsyslog
	•	message - to store the raw log data in case a dump is needed
	•	blob - to store all values that follow the agent variable


The resulting query:
(GIST NEEDED)
```
CREATE TABLE nginx
(
    `logdate` Date,
    `logdatetime` DateTime,
    `hostname` String,
    `syslogtag` String,
    `message` String,
    `clientip` String,
    `ident` String,
    `auth` String,
    `verb` String,
    `request` String,
    `httpv` String,
    `response` UInt16,
    `bytes` UInt64,
    `referrer` String,
    `agent` String,
    `blob` String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(logdate)
ORDER BY (logdate, logdatetime)
SETTINGS index_granularity = 8192

```

Grafana
Finally, to make Grafana consume data from ClickHouse for visualization, we need to install the ClickHouse plugin (LINK https://grafana.com/grafana/plugins/vertamedia-clickhouse-datasource) and use it with the simplest query:


```
SELECT
    $timeSeries as t,
    count(*) as Count
FROM $table
WHERE $timeFilter
GROUP BY t
ORDER BY t
```

That's basically it. Parse your logs, stay home, stay NGINX!
