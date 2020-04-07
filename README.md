# rsyslog-nginx-clickhouse

As we know, nginx writes logs about itself. We can read those logs, search something in it, parse and analyze it, make graphical interpretations. Usually, we use ELK stack: the filebeat to send logs, logstash to parse and convert and elasticsearch to store. It's normal practice, but to this, we have to install logstash. Also, we will be faced with a different problem: elasticsearch needs resources for work, and sometimes works slow. Also, kibana as GUI for elasticsearch works slow as well and don't support regular SQL syntax.
But we have a nice solution.

Main idea is:

Logfiles, produced by nginx, should be parsed with the rsyslog, and put into the clickhouse, then they are can be used to make graphics or other reports.
We use the grafana as GUI for the clickhouse instead of a tabix, which integrated into clickhouse. 
access.log files -> rsyslog -> clickhouse -> grafana.

We choose rsyslog, because it's a  widely distributed software for the Linux, it's simple, flexible, has a specific module for the clickhouse and already installed in centos and ubuntu distributives. 
Rsyslog can send log's data and receive it from multiple systems, also it can parse data and use different rulesets for different kinds of logs. 
We choose clickhouse because it works fast for the "select" queries type, those we have to make to build graphics. Also, clickhouse can compress saved data, and clickhouse takes fewer resources then elasticsearch.


### 1. nginx setup:

We have to do nothing for the basic setup. 
Default nginx access logs format is:
```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```
if the remote_user variable is empty, the '-' sets as value. 
#### It's perfect already and we nothing to do here. 

But of course, we can add to the access log format some variables,  like $upstream_addr $upstream_connect_time $upstream_header_time $upstream_response_time. If we will not use those variables as parameters for the "select" queries - we could put is after the $http_x_forwarded_for, and then we don't need to change everything else. On the other hand, if we need to use some of those variables as a parameter for the "select", then we should make a special column for this variable and a special field in a normalizer rule. 
The text below about a default log format. 

### 2. Rsyslog setup:

Usually, rsyslog uses as log collector, which grabs logs from a diffent places and store them according a some rules. In our case, we will also parse nginx access logs and it in clickhouse. 

Rsyslog - powerful and flexible software, its functionality can be extended with different plugins. We will use three of them:

1. imfile
2. mmnormalize
3. omclickhouse

Imfile - uses to read from a file. It's an embedded module for the rsyslog. Usually, we just need to load it in the config file. 
Mmnormalize - log normalizer plugin, uses to parse log's string as an array of values with special rules. Rules can be set in the config. Mmnormalize can be installed from the repo. Also, for the test reasons, we recommend installing lognormalizer software (for example liblognorm1, liblognorm1-utils ). 
We will show below how we using lognormalizer for testing parse rules.
Omclickhouse - the specific module for rsyslog, which can send logs into clickhouse. Unfortunately, it may be not in the repo yet, so it can be easily installed according to the manual: 
https://www.rsyslog.com/doc/master/configuration/modules/omclickhouse.html 

Use lognormalize to slice access logs as variables.
Basically, nginx log looks like:

127.0.0.1 - - [06/Apr/2020:09:54:48 -0400] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"

We interpret it this kind:

| log variable       | example   | parsed variable|  comment                         |
|--------------------|:---------:|----------------|----------------------------------|
| remote_addr        | 127.0.0.1 | clientip       |                                  |
|        -           |     -     | ident          |reserved variable                 | 
| remote_user        |     -     | auth           |                                  |
| time local         |    06     | day            |separated to multiple values      | 
|                    |   Apr     | month          |                                  |
|                    |   2020    | year           |                                  |
|                    | 09:54:48  | rtime          |                                  |
|                    | -0400     | tz             |                                  | 
| request            |   GET     | verb           | separated as well                |
|                    | /         | request        |                                  | 
|                    | HTTP/1.1  | httpversion    |                                  |
| status             | 200       | response       |                                  |
| body_bytes_sent    | 612       | bytes          |                                  |
| http_referer       | -         | referrer       |                                  | 
| http_user_agent    |curl/7.29.0| referrer       |                                  | 
|http_x_forwarded_for| -         | blob           | all after this value will be here| 

it's necessary for parsing date because of nginx regular time format not acceptable for the clickhouse. Change nginx time format - could be not a good idea if some different software uses nginx's logs. 
Also,  we have to change month format from char type into digit type then we have to put a month into a separated variable. 

Create a rule for lognormalizer in file /etc/rsyslog.d/nginx.rule: 
```

version=2

rule=:%clientip:word% %ident:word% %auth:word% [%day:char-to:/%/%month:char-to:/%/%year:number%:%rtime:word% %tz:char-to:]%] "%verb:word% %request:word% HTTP/%httpversion:float%" %response:number% %bytes:number% "%referrer:char-to:"%" "%agent:char-to:"%"%blob:rest%
```

To test the rule, run: 
```

curl 127.0.0.1; tail -n 1 /var/log/nginx/access.log | lognormalizer  -r /etc/rsyslog.d/nginx.rule -e json
```

the output should be like:
```

{ "blob": " \"-\"", "agent": "curl\/7.29.0", "referrer": "-", "bytes": "612", "response": "200", "httpversion": "1.1", "request": "\/", "verb": "GET", "tz": "-0400", "rtime": "09:54:48", "year": "2020", "month": "Apr", "day": "06", "auth": "-", "ident": "-", "clientip": "127.0.0.1" }
```


In the data part of the string, we get also a month as word, we transform it into dight in the table part.

Create the table file /etc/rsyslog.d/nginx.table:
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

The algorithm is: 
   1. read string from log
   2. slice it to variables
   3. change month format
   4. put string to database according the template
    
    
     
###### The important thing about rsyslog: 
rsyslog has different zones for variables. To transmit variable month into the variable !usr!nxm which uses in the template for output, we use separated ruleset "out" which called from the first ruleset. 
How to write template we can find in documentation for omclickhouse above.
Basically, the template should be looks like a regular clickhouse's "insert"  query. 

For the test, we can put parsed data into some file, to check it with output rule in config:
action(type="omfile" file="/var/log/nginx.parsed" template="ng")

If everything is ok, the file should be filled with strings like this:
```
INSERT INTO nginx.nginx (logdate, logdatetime, hostname, syslogtag, message, clientip, ident, auth, verb, request, httpv, response, bytes, referrer, agent, blob ) values ('2020-04-07', '2020-04-07 19:21:13', 'unit', 'nginx', '127.0.0.1 - - [07/Apr/2020:19:21:13 -0400] \"GET / HTTP/1.1\" 200 612 \"-\" \"curl/7.29.0\" \"-\"', '127.0.0.1', '-', '-', 'GET', '/', '1.1', '200', '612', '-', 'curl/7.29.0', ' \"-\"');
```


All this rules and the template are in the same file nginx.conf:

```
lookup_table(name="monthes" file="/etc/rsyslog.d/nginx.table" reloadOnHUP="off")
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

module(load="imfile")
module(load="omclickhouse")
module(load="mmnormalize")
input(type="imfile" file="/var/log/nginx/access.log" tag="nginx" ruleset="norm")
ruleset(name="norm") {
  action(type="mmnormalize" rulebase="/etc/rsyslog.d/nginx.rule" useRawMsg="on")
  set $!usr!nxm = lookup("monthes", $!month);
  call out
}
ruleset(name="out") {
# action(type="omfile" file="/var/log/nginx.parsed" template="ng")
  action(type="omclickhouse" server="127.0.0.1" port="8123" 
  usehttps="off"
  template="ng")
}
```

### 3. Clickhouse setup:

Create table for the logs in the clickhosue database.
We using a recommended by clickhouse's developers method and engine of creation. 
We will use specific fields: 
logdate - for partitioning this table,
logdatetime - for sorting data,
hostname - name of host - log creator,
syslogtag - tag from syslog/rsyslog,
message - log string as is, for dump cases,
blob - all values after agent variable. 

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


## It isn't necessary to read all the above! 
####### You can copy-paste strings below, and everything should works. (If rsyslog modules were installed successfully)

```
clickhouse-client  --query="$(cat nginx.click);"
```
