# logspout-redis-logstash
[Logspout](https://github.com/gliderlabs/logspout) adapter for writing Docker container stdout/stderr logs to Redis in Logstash jsonevent layout.

See the example below for more information.


## Docker image available

Logspout including this adapter is available on [Docker Hub](https://registry.hub.docker.com/u/rtoma/logspout-redis-logstash/). Pull it with:

```
$ docker pull rtoma/logspout-redis-logstash
```

## How to use the Docker image

```
$ docker run -d --name logspout /var/run/docker.sock:/var/run/docker.sock:ro rtoma/logspout-redis-logstash redis://<your-redis-server>
```

## Configuration

Some configuration can be passed via container environment keys. Some can be appended as query parameters to the endpoint.


### Environment keys

Behaviour of the adapter can be changed by passing configuration via docker environment vars (e.g. -e key=val).

- DEBUG=1: will enable logspout debug mode (default is disabled);
- REDIS\_PASSWORD=\<password\>: will force the adapter to do an AUTH to Redis (default is none);
- REDIS\_KEY=\<key\>: will configure the Redis list key to add events to (default is 'logspout');
- REDIS\_DOCKER\_HOST=\<host\>: will add a docker.host=<host> field, allows you to add the hostname of your docker host, identifying where your container was running (think mesos cluster);
- REDIS\_USE\_V0\_LAYOUT=1: logstash jsonevent layout v0 will be used (default is v1 layout).

### Endpoint query parameters

Two keys can also be set as endpoint query parameters.

The REDIS\_KEY can be set as redis://host?key=\<key\>.

The REDIS\_USE\_V0\_LAYOUT switch can be set with ?use\_v0\_layout=1


## Ignoring containers
To make logspout ignore a containers logs, set the environmental variable and value of 'LOGSPOUT=ignore' on the container being ignored for it to be ignored.

Logspout with the redis module can also be configured to pass options in to the ELK stack by way of environmental variables at the logspout container level, which can be overrode by the individual containers logging messages, for example:
```
docker run -it -v /var/run/docker.sock:/tmp/docker.sock -e 'OPTIONS={"test":"logspout"}' brendangibat/docker-logspout-redis:latest redis://redis.url:6379
```

Sample container logging to stdout
```
docker run -d -e TEST=LEON -e TEST2=LEON2 -e 'LOGSPOUT_OPTIONS={"test":"leon"}' ubuntu echo 'hello world'
```

In the case above, when the sample container logs any messages, the logspout container forwarding to redis will set an object at the document root of options.test=leon
In this way, individual containers or the logspout container running on a server can set the logging policies and extra information to tag forwarded messages. Such values in the options document set include overrides about what index to use, and if the message should be decoded as JSON data.

Logspout-redis can be configured to at its own container level for what type of logs it expects to process for all of the containers it monitors.

Logspout-logstash sending json logs to custom_index index:

```
docker run -it -v /var/run/docker.sock:/tmp/docker.sock -e 'OPTIONS={"_logging_index":"custom_index","codec":"json"}' brendangibat/docker-logspout-redis:latest redis://redis.url:6379
```

Additionally, Logspout-redis looks at the environmental variables of each container for a hash object in the env var 'LOGSPOUT_OPTIONS' and overrides any setting at the logspout-logstash container level of log processing.

Logspout-logstash will process these as json logs to test index

```
docker run -d -e 'LOGSPOUT_OPTIONS={"test":"hello world", "_logging_index":"test", "codec":"json", "type":"test_type"}' ubuntu echo '{"testKeyName": "hello world test"}'
```

The above docker command will send its logs to the test index and be processed as a json object with the root type "test_type" which will merge its document on to the root document ingested in to ELK.


## Logstash Filters

This works well in tandem with logstash filters. By passing in custom options on individual containers or for all messages logged by a running logspout container, you can partition your environments logs in to different indexes or ElasticSearch hosts.

```

input {
    redis {
        host => 'some_host.url'
        key => 'some_key'
        data_type => 'list'
        type => 'redis'
    }
}


filter {
  if [options][type] {
    mutate {
      replace => {
        "type" => "%{[options][type]}"
      }
    }
  }
  if [data][message][type] {
    mutate {
      replace => {
        "type" => "%{[data][message][type]}"
      }
    }
  }
  if [options][codec] == "json" {
    json {
      source => "message"
    }
  }
}

output {
  if [options][_logging_index] {
      amazon_es {
          hosts => ['es_host.url']
          region => 'aws_region'
          index => '%{[options][_logging_index]}-%{+YYYY.MM.dd}'
      }
  } else {
      amazon_es {
          hosts => ['es_host.url']
          region => 'aws_region'
          index => 'logging-%{+YYYY.MM.dd}'
      }
  }
}
```

## ELK integration

Try out logspout with redis-logstash adapter in a full ELK stack. A docker-compose.yml can be found in the example/ directory.

When logspout with adapter is running. Executing something like:

```
docker run --rm centos:7 echo hello from a container
```

Will result in a corresponding event in Elasticsearch. Below is a screenshot from Kibana4:

![](event-in-k4.png)


## Credits

Thanks to [Gliderlabs](https://github.com/gliderlabs) for creating Logspout!

For other credits see the header of the redis.go source file.
