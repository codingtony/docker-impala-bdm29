#Presentation material for Big Data Montreal #29
## Docker on Impala
Github repo for codingtony/impala : https://github.com/codingtony/docker-impala
Please read the README.md of that repo to understand how the image has been built.

### Pull the docker image from Docker Hub (~1.5GB)
```
docker pull codingtony/impala
```

###Create a container named "impala-bdm"
The container will run daemonized and NAT port 25000 (impala server web interface) to the Host.
```
docker run -d --name "impala-bdm" -p 21000:21000 -p 25000:25000 codingtony/impala


docker logs -f impala-bdm
```

We see that upon startup, it's formating HDFS (only at the first start) , then starts the namenode, the datanode and all other impala processes.


###Create a temporary container to run impala-shell connecting on impala-bdm
```
docker run --rm -ti --link impala-bdm:impala-server codingtony/impala impala-shell -i impala-server
```

###Create database and table  (one that we used in our presentation BDM 27)

```
DROP DATABASE IF EXISTS bdm29;
CREATE DATABASE bdm29;


DROP TABLE IF EXISTS bdm29.hits2013;
CREATE EXTERNAL TABLE bdm29.hits2013 (
        day INT,
        ts INT,
        synthetic INT,
        localtradetimestamp INT,
        ticker STRING,
        lastPrice DOUBLE,
        vol INT,
        locnum INT,
        eventtype STRING,
        priceQual STRING,
        tradeQual STRING,
        eventQual STRING,
        contributor STRING,
        bidprice DOUBLE,
        bidsize INT,
        bidqual STRING,
        bidtime INT,
        askprice DOUBLE,
        asksize INT,
        askqual STRING,
        asktime INT,
        session INT


) 
ROW FORMAT DELIMITED FIELDS TERMINATED BY ';'
LOCATION '/user/impala/bdm29/2013/';
```


###Load one file (138M) in HDFS
From my host to the HDFS running in the impala-bdm container.
This will be done using a temporary container. This is more complicated than I expected. I will try to make it simpler with helper scripts in the docker image.

```
docker run --rm   -ti --link impala-bdm:impala-server -v /home/tbussier/work/HITS-321-20130107.csv:/tmp/HITS-321-20130107.csv:ro -v /home/tbussier/work/impala_hdp:/usr/bin/impala_hdp:ro codingtony/impala su impala -c "impala_hdp impala-server  hadoop --config /tmp fs -put -f  /tmp/HITS-321-20130107.csv /user/impala/bdm29/2013/"
```
See script **impala_hdp** in this repo.

###Check that the file has been loaded correctly
```
docker run --rm   -ti --link impala-bdm:impala-server codingtony/impala hdp impala-server  hadoop fs -ls -h /user/impala/bdm29/2013/
```

###Run queries (using a temporary container)
Refresh metadata, compute stats and query the file (using a temporary container)
```
docker run --rm -ti --link impala-bdm:impala-server codingtony/impala impala-shell -i impala-server -q 'REFRESH bdm29.hits2013; COMPUTE STATS bdm29.hits2013; select ticker,count(*) from bdm29.hits2013 group by ticker order by ticker;'
```

Run another query 
```
docker run --rm -ti --link impala-bdm:impala-server codingtony/impala impala-shell -i impala-server -q 'select day,ticker,max(lastPrice) as High ,min(lastPrice) as Low  from   bdm29.hits2013 WHERE ticker="IBM" AND lastPrice IS NOT NULL   GROUP by day,ticker ORDER by day;'
```


###Visit the Impala web interface (of the impala-bdm container)

[Impala web interface on localhost](http://localhost:25000/queries)
