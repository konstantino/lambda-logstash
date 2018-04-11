## Lambda logstash

The main idea behind this "snippet" is to simplify the process of maintaining logstash pipelines in a k8s environment. Effectively this means, that every time you need a new data pipeline you get to re-use most of the k8s manifest. Additional configuration such as the specifics of the pipeline is handled by way of ConfigMaps. Essentially this turns logstash into a function hence why this project is labelled lambda logstash.

# Pipeline

For simplicity we assume the output of logstash is elasticsearch. Other outputs can be handled equally well by changing the pipeline file. Here's an example where we intentionally assume that input comes from an http request.

pipeline.conf

```
input {
	http {
        port => 8080
    }
}
filter {
	csv {
		columns => [
			'field1',
			'field2',
			'field3'
			]
		separator => ";"
	}
	date {
		match => [ "field2", "yyyy-MM-dd HH:mm:ss" ]
	}
	mutate {
		convert => {
			"field3" => "float"
		}
        remove_field => ["path", "host", "message"]
	}
}
output {
	elasticsearch {
		hosts => ["service-elasticsearch:9200"]
		action => "index"
		index => "my_index"
    }
    stdout { }
}
```

# Usage

1. First we need a config map to hold the pipeline we described earlier. Here's how:

```
kubectl create configmap pipeline-config --from-file=pipeline.conf
```

2. Now we can deploy lambda-logstash

```
kubectl create -f manifest.yaml
```

3. How to test?

```
curl --request POST \
  --url http://your_ip:your_port/ \
  --data '1;2018-10-12 09:30:10;20.3'
```

```
curl --request GET \
  --url http://your_ip:your_port/my_index/_search
```

4. To clean up

```
kubectl delete configmap pipeline-config
kubectl delete -f manifest.yaml
```
