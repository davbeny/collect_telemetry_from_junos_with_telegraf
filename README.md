# about this repo

collect openconfig telemetry from junos devices using telegraf.  
store collected data in influxdb  
query influxdb database with cli and python to extract data   

# about telegraf

telegraf is an open source collector written in GO.  
Telegraf collects data and writes them into a database.  
It is plugin-driven (it has input plugins, output plugins, ...)  

# requirements 

## docker

you need to install docker.  
This is not covered in this repository

## Junos 

here's some details from the junos devices   
```
jcluser@vMX-1> show version | match "na telemetry"
JUNOS na telemetry [18.2R1.9-C1]
```
```
jcluser@vMX-1> show version | match openconfig
JUNOS Openconfig [0.0.0.10-1]
```
```
jcluser@vMX-1> show configuration system services netconf | display set
set system services netconf ssh
```
```
jcluser@vMX-1> show configuration system services extension-service | display set
set system services extension-service request-response grpc clear-text port 32768
set system services extension-service request-response grpc skip-authentication
set system services extension-service notification allow-clients address 0.0.0.0/0
```

# influxdb

pull docker images 
```
$ docker pull influxdb
```
Verify
```
$ docker images influxdb
```
Instanciate an influxdb container
```
$ docker run -d --name influxdb -p 8083:8083 -p 8086:8086 influxdb
```
Verify
```
$ docker ps | grep influxdb
```
for troubleshooting purposes you can run this command
```
$ docker logs influxdb
```
start a shell session in the influxdb container
```
$ docker exec -it influxdb bash
```
run this command to read the influxdb configuration file
```
# more /etc/influxdb/influxdb.conf
```
create a user and a database
```
# influx
Connected to http://localhost:8086 version 1.7.0
InfluxDB shell version: 1.7.0
Enter an InfluxQL query
> CREATE DATABASE juniper
> show databases
name: databases
name
----
_internal
juniper
>  CREATE USER "juniper" WITH PASSWORD 'juniper'
> show users
user   admin
----   -----
influx false
> exit
# 
```
exit the influxdb container
```
# exit
```

# telegraf

get ip address used by containers
```
$ ifconfig docker0
```

pull docker images 
```
$ docker pull telegraf
```
Verify
```
$ docker images telegraf
```
create a telegraf configuration file ([use this file](telegraf.conf))  
it will use jti_openconfig_telemetry input plugin (grpc client to collect telemetry on junos devices) and influxb output plugin (database to store the data collected)  

```
$ vi telegraf.conf
```
instanciate a telegraf container
```
$ docker run --name telegraf -d -v $PWD/telegraf.conf:/etc/telegraf/telegraf.conf:ro telegraf
```
verify
```
$ docker ps | grep telegraf
```
for troubleshooting purposes you can run this command
```
$ docker logs telegraf
```
start a shell session in the telegraf container
```
$ docker exec -it telegraf bash
```
verify the telegraf configuration file
```
# more /etc/telegraf/telegraf.conf
```
exit the telegraf container
```
# exit
```
# query the influxdb database with cli to verify
start a shell session in the influxdb container
```
$ docker exec -it influxdb bash
```
query the database
```
# influx
Connected to http://localhost:8086 version 1.7.0
InfluxDB shell version: 1.7.0
Enter an InfluxQL query
> show databases
name: databases
name
----
_internal
juniper
> use juniper
Using database juniper
> show measurements
name: measurements
name
----
```
All devices
```
> SHOW TAG VALUES FROM  "/network-instances/network-instance/protocols/protocol/bgp/" with KEY = "device"
name: /network-instances/network-instance/protocols/protocol/bgp/
key    value
---    -----
device 100.123.1.0
device 100.123.1.1
device 100.123.1.2
device 100.123.1.4
device 100.123.1.5
device 100.123.1.6
>
```
BGP neighbors address
```
> SHOW TAG VALUES FROM  "/network-instances/network-instance/protocols/protocol/bgp/" with KEY = "/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address"
name: /network-instances/network-instance/protocols/protocol/bgp/
key                                                                                             value
---                                                                                             -----
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.1
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.2
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.3
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.4
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.5
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.6
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.7
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.2.1
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.2.2
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.2.3
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.2.4
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.2.5
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.2.6
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.2.7
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.3.1
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.3.2
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.3.3
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.3.4
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.3.5
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.3.6
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.3.7
```

BGP neighbors address for device 100.123.1.0
```
> SHOW TAG VALUES FROM  "/network-instances/network-instance/protocols/protocol/bgp/" with KEY = "/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address" WHERE device='100.123.1.0'
name: /network-instances/network-instance/protocols/protocol/bgp/
key                                                                                             value
---                                                                                             -----
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.1
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.3
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.5
/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address 192.168.1.7
>
```
BGP groups for device 100.123.1.0
```
> SHOW TAG VALUES FROM  "/network-instances/network-instance/protocols/protocol/bgp/" with KEY ="/network-instances/network-instance/protocols/protocol/bgp/peer-groups/peer-group/@peer-group-name" WHERE device='100.123.1.0'
name: /network-instances/network-instance/protocols/protocol/bgp/
key                                                                                                value
---                                                                                                -----
/network-instances/network-instance/protocols/protocol/bgp/peer-groups/peer-group/@peer-group-name underlay
>
```
BGP groups 
```
> SHOW TAG VALUES FROM  "/network-instances/network-instance/protocols/protocol/bgp/" with KEY ="/network-instances/network-instance/protocols/protocol/bgp/peer-groups/peer-group/@peer-group-name"
name: /network-instances/network-instance/protocols/protocol/bgp/
key                                                                                                value
---                                                                                                -----
/network-instances/network-instance/protocols/protocol/bgp/peer-groups/peer-group/@peer-group-name underlay
>
```
Sessions state on device 100.123.1.0
```
> SELECT "/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state" from "/network-instances/network-instance/protocols/protocol/bgp/" WHERE device='100.123.1.0' ORDER BY DESC LIMIT 4
name: /network-instances/network-instance/protocols/protocol/bgp/
time                /network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state
----                -------------------------------------------------------------------------------------------------
1545167962424407403 ESTABLISHED
1545167962424407403 ESTABLISHED
1545167962424407403 ESTABLISHED
1545167962424407403 ESTABLISHED
>

```
Sessions state
```
> SELECT "/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state" FROM "/network-instances/network-instance/protocols/protocol/bgp/" ORDER BY DESC LIMIT 10
name: /network-instances/network-instance/protocols/protocol/bgp/
time                /network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state
----                -------------------------------------------------------------------------------------------------
1545168049575284624 ESTABLISHED
1545168049575284624 ESTABLISHED
1545168049575284624 ESTABLISHED
1545168049575284624 ESTABLISHED
1545168049092114919 ESTABLISHED
1545168049092114919 ESTABLISHED
1545168049092114919 ESTABLISHED
1545168049092114919 ESTABLISHED
1545168048906507987 ESTABLISHED
1545168048906507987 ESTABLISHED
>
```
Total prefixes on device 100.123.1.4
```
> SELECT MEAN("/network-instances/network-instance/protocols/protocol/bgp/global/state/total-prefixes") FROM "/network-instances/network-instance/protocols/protocol/bgp/" WHERE "device" = '100.123.1.4'
name: /network-instances/network-instance/protocols/protocol/bgp/
time mean
---- ----
0    36
>
```
Total prefixes on device 100.123.1.0 since one minute
```
> SELECT mean("/network-instances/network-instance/protocols/protocol/bgp/global/state/total-prefixes") FROM "/network-instances/network-instance/protocols/protocol/bgp/" WHERE ("device" = '100.123.1.0') AND time >= now() - 1m
name: /network-instances/network-instance/protocols/protocol/bgp/
time                mean
----                ----
1545168168469112825 44
>
```
Sessions state and total prefixes
```
> SELECT "/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state",  "/network-instances/network-instance/protocols/protocol/bgp/global/state/total-prefixes" FROM "/network-instances/network-instance/protocols/protocol/bgp/" ORDER BY DESC LIMIT 4
name: /network-instances/network-instance/protocols/protocol/bgp/
time                /network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state /network-instances/network-instance/protocols/protocol/bgp/global/state/total-prefixes
----                ------------------------------------------------------------------------------------------------- --------------------------------------------------------------------------------------
1545168279586592682 ESTABLISHED
1545168279586592682 ESTABLISHED
1545168279586592682                                                                                                   44
1545168279586592682 ESTABLISHED
>
```
Sessions state and total prefixes since 10 secondes for device 100.123.1.0
```
> SELECT "/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state", "/network-instances/network-instance/protocols/protocol/bgp/global/state/total-prefixes" FROM "/network-instances/network-instance/protocols/protocol/bgp/" WHERE "device" = '100.123.1.0' AND time >= now() - 10s
name: /network-instances/network-instance/protocols/protocol/bgp/
time                /network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/state/session-state /network-instances/network-instance/protocols/protocol/bgp/global/state/total-prefixes
----                ------------------------------------------------------------------------------------------------- --------------------------------------------------------------------------------------
1545168296433372717 ESTABLISHED
1545168296433372717                                                                                                   44
1545168296433372717 ESTABLISHED
1545168296433372717 ESTABLISHED
1545168296433372717 ESTABLISHED
1545168298425250176 ESTABLISHED
1545168298425250176 ESTABLISHED
1545168298425250176                                                                                                   44
1545168298425250176 ESTABLISHED
1545168298425250176 ESTABLISHED
1545168300432730351 ESTABLISHED
1545168300432730351 ESTABLISHED
1545168300432730351 ESTABLISHED
1545168300432730351 ESTABLISHED
1545168300432730351                                                                                                   44
1545168302438985906 ESTABLISHED
1545168302438985906                                                                                                   44
1545168302438985906 ESTABLISHED
1545168302438985906 ESTABLISHED
1545168302438985906 ESTABLISHED
>
```
Others influxdb queries examples
```
> SELECT * FROM "/interfaces/" ORDER BY DESC LIMIT 1
...
> SELECT * FROM "/network-instances/network-instance/protocols/protocol/bgp/" ORDER BY DESC LIMIT 1
...
> SELECT * FROM "/network-instances/network-instance/protocols/protocol/bgp/" WHERE device='100.123.1.0' AND time >= now() - 10s
...
> SELECT * FROM "/network-instances/network-instance/protocols/protocol/bgp/" WHERE "/network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/@neighbor-address" ='192.168.1.7' ORDER BY DESC LIMIT 1
...
> SELECT COUNT("/interfaces/interface/state/oper-status") FROM /interfaces/ WHERE "device" = '100.123.1.0' AND "/interfaces/interface/@name" = 'ge-0/0/0' AND "/interfaces/interface/state/oper-status" = 'UP' AND time >= now() - 1m GROUP BY time(10s)
name: /interfaces/
time                count
----                -----
1545168180000000000 1
1545168190000000000 5
1545168200000000000 5
1545168210000000000 5
1545168220000000000 5
1545168230000000000 5
1545168240000000000 4
>
```
exit 
```
> exit
#
```
exit the influxdb container
```
# exit 
```
# query the influxdb database with python to verify
install the influxdb python library.
This python library is a client for interacting with InfluxDB.
```
$ pip install influxdb
```
Verify
```
$ pip list | grep influx
```
you can now interact with InfluxDB using Python

run this command to start a python interactive session

```
$ python
```
connect to InfluxDB using Python
```
>>> from influxdb import InfluxDBClient
>>> influx_client = InfluxDBClient('localhost',8086)
```
list the databases
```
>>> influx_client.query('show databases')
ResultSet({'(u'databases', None)': [{u'name': u'_internal'}, {u'name': u'juniper'}]})
```
list measurements for a database
```
>>> influx_client.query('show measurements', database='juniper')
ResultSet({'(u'measurements', None)': [{u'name': u'/interfaces/'}, {u'name': u'/network-instances/network-instance/protocols/protocol/bgp/'}]})
```
query data from a particular measurement and database
```
>>> gp = influx_client.query('select * from "/interfaces/"  order by desc limit 2 ', database='juniper').get_points()
>>> for item in gp:
...     print item['/interfaces/interface/@name']
...
ge-0/0/7
ge-0/0/6
>>> gp = influx_client.query('select * from "/interfaces/"  order by desc limit 2 ', database='juniper').get_points()
>>> for item in gp:
...     print item['/interfaces/interface/subinterfaces/subinterface/state/counters/in-octets']
...
36
26277618
>>>

