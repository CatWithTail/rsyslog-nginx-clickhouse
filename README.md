# rsyslog-nginx-clickhouse
rsyslog-nginx-clickhouse

Main idea is:

Logfiles, produced by nginx, should be parsed with the rsyslog, and put into the clickhouse, then they are can be used to make graphics or other reports.
We use the grafana as GUI for the clickhouse instead of a tabix, which integrated into clickhouse. 
access.log files -> rsyslog -> clickhouse -> grafana.

### 1. nginx setup:

We have to do nothing for the basic setup. 
Default nginx access logs format is:
```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```
if the remote_user variable is empty, the '-' sets as value. 

### 2. Rsyslog setup:

First, we need to use pre-builded module omclickhouse. How to make this module (https://www.rsyslog.com/doc/master/configuration/modules/omclickhouse.html).
Also, the system should contain mmnormalize module and the lognormalizer utility.


Use lognormalize to slice access logs as variables.
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


