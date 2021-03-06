---
published: true
---
Today I did some testing for seldon-core (1.1.0)

### Things covered
- Metrics endpoint 
- Metadata (Change of structure)
- Combined testing phase
- Tag testing
- CI/CD Jenkins with seldon-core
- Seldon-core testing / seldon-core testing framework


# Metadata 

Sample code:
```
import logging
import base64
class simple(object):
    """
    Model template. You can load your model parameters in __init__ from a location accessible at runtime
    """

    def __init__(self):
        """
        Add any initialization parameters. These will be passed at runtime from the graph definition parameters defined in your seldondeployment kubernetes resource manifest.
        """
        print("Initializing")

        return

    def health_status(self):
    #response = self.predict([1, 2], ["f1", "f2"])
    #assert len(response) == 2, "health check returning bad predictions" # or some other simple validation
        return "I am super healthy :)"

    def metadata(self):

        meta = {
            "name": "model-name",
            "versions": ["model-version"],
            "platform": "platform-name",
            "inputs": [{"name": "input", "datatype": "BYTES", "shape": [1]}],
            "outputs": [{"name": "output", "datatype": "BYTES", "shape": [1]}],
        }

        return meta

    def init_metadata(self):

        meta = {
            "name": "model-name",
            "versions": ["model-version"],
            "platform": "platform-name",
            "inputs": [{"name": "input", "datatype": "BYTES", "shape": [1]}],
            "outputs": [{"name": "output", "datatype": "BYTES", "shape": [1]}],
        }

        return meta

    def metrics(self):
        return [
            {"type": "COUNTER", "key": "mycounter", "value": 1}, # a counter which will increase by the given value
            {"type": "GAUGE", "key": "mygauge", "value": 100},   # a gauge which will be set to given value
            {"type": "TIMER", "key": "mytimer", "value": 20.2},  # a timer which will add sum and count metrics - assumed millisecs
        ]
        
    def predict(self, X, feature_names):
        """
        Return a prediction.

        Parameters
        ----------
        X : array-like
        """
        print("Predict called - will run plus2 function")
        logging.info(X)
        
        return X


```

Model_deployment.yaml:
```
apiVersion: machinelearning.seldon.io/v1alpha2
kind: SeldonDeployment
metadata:
  name: seldon-simple
  namespace: default
  labels:
    model_name: simple-model
    api_type: microservice
    microservice_type: ai
spec:
  name: times2-plus2
  predictors:
  - componentSpecs:
    - spec:
        containers:
        - name: model
          image: gcr.io/science-experiments-divya/simple:input-output
          imagePullPolicy: Always
    graph:
      name: model
      type: MODEL
      endpoint:
        type: REST
    name: meta-metrics-test
    replicas: 1

```

Executed commands:

`docker build -t gcr.io/science-experiments-divya/simple:input-output .`

`docker push gcr.io/science-experiments-divya/simple:input-output`

`kubectl apply -f model_deployment.yaml`

### Workability
Prefix used:
`/seldon/default/seldon-simple/`

Called prediction with `{"jsonData": {"X" : 4 }}`

Response:
```json
{
    "jsonData": {
        "X": 4
    },
    "meta": {
        "metrics": [
            {
                "key": "mycounter",
                "type": "COUNTER",
                "value": 1
            },
            {
                "key": "mygauge",
                "type": "GAUGE",
                "value": 100
            },
            {
                "key": "mytimer",
                "type": "TIMER",
                "value": 20.2
            }
        ]
    }
}
```

# Metrics endpoint

Sending with `{"jsonData": {"X" : 4 }}`

404 page not found:
- `/api/v0.1/prometheus/metrics`
- `/api/v0.1/metrics`


> Possible that metrics is only visible via prometheus


Auto generated:
![seldon_metrics_1.PNG]({{site.baseurl}}/img/seldon_metrics_1.PNG)
![seldon_metrics_2.PNG]({{site.baseurl}}/img/seldon_metrics_2.PNG)

Issue:
![seldon_metrics_3.PNG]({{site.baseurl}}/img/seldon_metrics_3.PNG)
[Link](https://github.com/SeldonIO/seldon-core/issues/1729#issuecomment-618362123)


## Using local docker

Setup:
`docker run --rm --name mymodel -p 5000:5000 mymodel:0.1`



[Reference link](https://docs.seldon.io/projects/seldon-core/en/latest/workflow/serving.html)

1) Testing:
`curl -X POST -H 'Content-Type: application/json' -d '{"jsonData": {"X" : 4 }}' http://localhost:5000/api/v1.0/predictions`

Response: 
`{"jsonData":{"X":4},"meta":{"metrics":[{"key":"mycounter","type":"COUNTER","value":1},{"key":"mygauge","type":"GAUGE","value":100},{"key":"mytimer","type":"TIMER","value":20.2}]}}`


2) Metric

404 Error:
- `curl -X GET http://localhost:5000/metric`
- `curl -X GET http://localhost:5000/metrics`
- `curl -X GET http://localhost:5000/api/v1.0/prometheus`
- `curl -X GET http://localhost:5000/api/v1.0/metrics`
- `curl -X GET http://localhost:5000/api/v1.0/metric`
- `curl -X GET http://localhost:5000/api/v1.0/model/metrics`
- `curl -X GET http://localhost:5000/api/v1.0/model/metric`



References:
[Seldon-core Metrics code](https://docs.seldon.io/projects/seldon-core/en/latest/_modules/seldon_core/metrics.html), [Seldon-core wrapper source code](https://docs.seldon.io/projects/seldon-core/en/latest/_modules/seldon_core/wrapper.html?highlight=%2Fhealth),[Exposing prometheus /metric wrapper issue](https://github.com/SeldonIO/seldon-core/issues/1476),[Seldon-core metrics documentation](https://docs.seldon.io/projects/seldon-core/en/v1.1.0/analytics/analytics.html?highlight=metrics%20endpoint#metrics)




# Custom meta-data

- Attempts to create custom metadata



## Local docker

404 error:
- ` curl -X GET http://localhost:5000/api/v1.0/model/metadata`
- `curl -X GET http://localhost:5000/model/metadata`

> Possibly the /model/metadata does not work due to having only one model

200:
- `curl -X GET http://localhost:5000/metadata` (Localy)
- ` curl -X GET http://35.240.217.69:80/seldon/default/seldon-simple/api/v1.0/metadata/model` (Ambassador)

Returns: 
```
{"inputs":[{"datatype":"BYTES","name":"input","shape":[1]}],"name":"model-name","outputs":[{"datatype":"BYTES","name":"output","shape":[1]}],"platform":"platform-name","versions":["model-version"]}
```


### Trial 1

Changing init_metadata:
```
    def init_metadata(self):

        meta = {
            "mymodel": "model-name",
            "versions": ["model-version"],
            "platform": "platform-name",
            "Something":"new",
            "inputs": [{"name": "input", "datatype": "BYTES", "shape": [1]}],
            "outputs": [{"name": "output", "datatype": "BYTES", "shape": [1]}],
        }

        return meta
```
> I remove metadata()

Return: `{}`

### Trial 2 - Putting back metadata

- Return metadata function 
- metadata return the same dictionary as init

Code:

```python3
def metadata(self):

        meta = {
            "mymodel": "model-name",
            "versions": ["model-version"],
            "platform": "platform-name",
            "Something":"new",
            "inputs": [{"name": "input", "datatype": "BYTES", "shape": [1]}],
            "outputs": [{"name": "output", "datatype": "BYTES", "shape": [1]}],
        }

        return meta

    def init_metadata(self):

        meta = {
            "mymodel": "model-name",
            "versions": ["model-version"],
            "platform": "platform-name",
            "Something":"new",
            "inputs": [{"name": "input", "datatype": "BYTES", "shape": [1]}],
            "outputs": [{"name": "output", "datatype": "BYTES", "shape": [1]}],
        }

        return meta
```
Returns: 
```json
{"Something":"new","inputs":[{"datatype":"BYTES","name":"input","shape":[1]}],"mymodel":"model-name","outputs":[{"datatype":"BYTES","name":"output","shape":[1]}],"platform":"platform-name","versions":["model-version"]}
```

### Trial 3 - remove init-metadata

- remove the function init-metadata

Returns:
```json
{"Something":"new","inputs":[{"datatype":"BYTES","name":"input","shape":[1]}],"mymodel":"model-name","outputs":[{"datatype":"BYTES","name":"output","shape":[1]}],"platform":"platform-name","versions":["model-version"]}
```


> Ability to self define our own metadata might be due to the version of seldon-core. 
Latest might not have custom meta data

## Workability

![seldon_meta_1.PNG]({{site.baseurl}}/img/seldon_meta_1.PNG)
[Link](https://docs.seldon.io/projects/seldon-core/en/v1.1.0/python/python_wrapping_docker.html?highlight=metrics%20endpoint#advanced-usage)

References:
[Metadata documentation](https://docs.seldon.io/projects/seldon-core/en/latest/examples/metadata.html)


# Combined testing phase - Metadata and metrics
## Changed code
- Added metrics in init function

Sample code:
(Image = input-output-v2)
```python
import logging
import base64
class simple(object):
    """
    Model template. You can load your model parameters in __init__ from a location accessible at runtime
    """

    def __init__(self, metrics_ok=True, ret_nparray=False, ret_meta=False):
        """
        Add any initialization parameters. These will be passed at runtime from the graph definition parameters defined in your seldondeployment kubernetes resource manifest.
        """
        print("Initializing")
        self.metrics_ok = metrics_ok
        self.ret_nparray = ret_nparray
        self.ret_meta = ret_meta

        return

    def health_status(self):
    #response = self.predict([1, 2], ["f1", "f2"])
    #assert len(response) == 2, "health check returning bad predictions" # or some other simple validation
        return "I am super healthy :)"

    def metadata(self):

        meta = {
            "Something":"new",
            "somedictionary": [{"name1": "input1", "datatype1": "BYTES1", "shape1": [1]}],
        }

        return meta


    def metrics(self):
        if self.metrics_ok:
            return [{"type": "COUNTER", "key": "mycounter", "value": 1}]
        else:
            return [{"type": "BAD", "key": "mycounter", "value": 1}]
        
    def predict(self, X, feature_names):
        """
        Return a prediction.

        Parameters
        ----------
        X : array-like
        """
        print("Predict called - will run plus2 function")
        logging.info(X)
        
        return X

```

Testing phase:
1) GET methods

404 not found:
- `http://35.240.217.69:80/seldon/default/seldon-simple/api/v0.1/metric/model`
- `http://35.240.217.69:80/seldon/default/seldon-simple/api/v0.1/metrics/model`
- `http://35.240.217.69:80/seldon/default/seldon-simple/metric/model`
- `http://35.240.217.69:80/seldon/default/seldon-simple/metric/model`
- `http://35.240.217.69:80/seldon/default/seldon-simple/model/metrics`
- `http://35.240.217.69:80/seldon/default/seldon-simple/model/metric`
- `http://35.240.217.69:80/seldon/default/seldon-simple/api/v0.1/model/metric`
- `http://35.240.217.69:80/seldon/default/seldon-simple/api/v0.1/model/metrics`


200 OK:
`http://35.240.217.69:80/seldon/default/seldon-simple/api/v0.1/metadata/model`

Returns:
```json
{
    "Something": "new",
    "somedictionary": [
        {
            "datatype1": "BYTES1",
            "name1": "input1",
            "shape1": [
                1
            ]
        }
    ]
}
```

2) Post methods

`http://35.240.217.69:80/seldon/default/seldon-simple/api/v0.1/predictions` (POST)

Input: `{"jsonData": {"X" : 4 }}`
Output:
```
{
    "jsonData": {
        "X": 4
    },
    "meta": {
        "metrics": [
            {
                "key": "mycounter",
                "type": "COUNTER",
                "value": 1
            }
        ]
    }
}
```

> Returned metrics together with data





#### Change of code 1 - ret_meta

(Image = input-output-v3)

Change code snippet:

```
   def __init__(self, metrics_ok=True, ret_nparray=False, ret_meta=True):
        """
        Add any initialization parameters. These will be passed at runtime from the graph definition parameters defined in your seldondeployment kubernetes resource manifest.
        """
        print("Initializing")
        self.metrics_ok = metrics_ok
        self.ret_nparray = ret_nparray
        self.ret_meta = ret_meta

        return
        
```

> No difference from output of image = input-output-v2 using the same test cases


#### Change code - metadata

- Attempts to make the predict call return the metadata instead of just the metrics

(Image = input-output-v3.1)

Changed code snippet:
```
    def metadata(self):

        if self.ret_meta:
            return {
            "Something TRUE":"new",
            "somedictionary": [{"name1": "input1", "datatype1": "BYTES1", "shape1": [1]}]
        }
        else:
            return {
            "NOT TRUE":"new",
            "somedictionary": [{"name1": "input1", "datatype1": "BYTES1", "shape1": [1]}]
        }
```

> Still returns only the metrics



#### Change code - More metrics

- Added more metrics

(Image = input-output-v5.1)

Code snippet

```python
   def metrics(self):
        if self.metrics_ok:
            return [
            {"type": "COUNTER", "key": "mycounter", "value": 1}, # a counter which will increase by the given value
            {"type": "GAUGE", "key": "mygauge", "value": 100},   # a gauge which will be set to given value
            {"type": "TIMER", "key": "mytimer", "value": 20.2},  # a timer which will add sum and count metrics - assumed millisecs
        ]
        else:
            return [{"type": "BAD", "key": "mycounter", "value": 1}]
        
```



Response:
```json
{
    "jsonData": {
        "X": 22222,
        "XXX": "s",
        "Y": 3
    },
    "meta": {
        "metrics": [
            {
                "key": "mycounter",
                "type": "COUNTER",
                "value": 1
            },
            {
                "key": "mygauge",
                "type": "GAUGE",
                "value": 100
            },
            {
                "key": "mytimer",
                "type": "TIMER",
                "value": 20.2
            }
        ],
        "tags": {
            "mytag": 1
        }
    }
}
```


> Conclusion: 
> - Metrics will return in the POST method no matter if metrics_ok is set to true or false. 
>
> - There seems to not exist an /metrics endpoint
> 
> - Metadata will not be returned using the POST method if metrics is defined
>
> - Metadata can be access by `/metadata/<MODELNAME>` endpoint
>
> - Declaring ret_meta = true in init function does not do anything
>
> - We can declare our own metadata in this version but suspected that future version will not allow our own custom metadata (we can make use of tags)
>
> - Init_metadata seems to be useless for now


References:
[Use of python init - superclass](https://stackoverflow.com/questions/3694371/how-do-i-initialize-the-base-super-class), [Init](https://micropyramid.com/blog/understand-self-and-__init__-method-in-python-class/)




## Changing of port 6000


Input tested:
1. `curl -X POST -H 'Content-Type: application/json' -d '{"jsonData": {"X" : 4 }}' http://localhost:5000/api/v1.0/metrics`

2. `curl -X GET http://localhost:6000/metrics`

3. `curl -X GET http://localhost:6000/metrics/model`


Command run at own shell:


- `docker run --rm --name mymodel -p 6000:6000 gcr.io/science-experiments-divya/simple:input-output-v4`

### Results:

1) Error 404

2) Nothing returned (Command line empty)

3) Error 404


> Conclusion: Ignored 

# Tag testing

(Image = input-output-v4)

Added snippet:
```
    def tags(self):
        return {"mytag": 1}
```


Post methods:
- `http://35.240.217.69:80/seldon/default/seldon-simple/api/v1.0/predictions`

Input:`{"jsonData": {"X" : 4 }}`

Output:
```json
{
    "jsonData": {
        "X": 4
    },
    "meta": {
        "metrics": [
            {
                "key": "mycounter",
                "type": "COUNTER",
                "value": 1
            }
        ],
        "tags": {
            "mytag": 1
        }
    }
}
```

Get methods:
`http://35.240.217.69:80/seldon/default/seldon-simple/api/v1.0/metadata/model`

```json
{
    "Something TRUE": "new",
    "somedictionary": [
        {
            "datatype1": "BYTES1",
            "name1": "input1",
            "shape1": [
                1
            ]
        }
    ]
}
```

> No sign of tags


> Conclusion: Tags will appear with the return value when using POST method, it will not return anything if using a get method with the endpoint "metadata/model" (This case model is the only deployment used)

# Jenkins CI/CD with seldon

How the work flow is suppose to be like:

1. A Data Scientist or ML Engineer trains a new model.
2. The Data Scientist or ML Engineer pushes the updated configuration to the model implementation repository.
3. The CI tool automatically builds and tests the model implementation.
4. The CI tool automatically pushes the change into the GitOps staging repository.
5. The CI tool automatically opens a PR into the GitOps production repository.


## Private repo (WebHook)

- Use ssh keys to enable the webhook

```
%%bash
export GITHUB_USER=john.doe
export GITHUB_TOKEN=12341234

argocd repo add https://github.com/$GITHUB_ORG/$REPONAME --username $GITHUB_USER --password $GITHUB_TOKEN
```




Resource:

[Official document - jenkins seldon](https://docs.seldon.io/projects/seldon-core/en/v1.1.0/examples/jenkins_classic.html),



> Conclusion: 
>
> We can just make use of the Jenkins sample we did and it is dependent on the application we are making. I have already tested an automated deployment of Jenkins

# Seldon-core Load testing

The definition of load testing/testing framework is very broad.

## Load testing

- Found [iago](https://github.com/twitter-archive/iago) load testing.





References:
[Helm installation](https://docs.seldon.io/projects/seldon-core/en/v1.1.0/examples/autoscaling_example.html?highlight=load%20testing#Create-Load)




# Other references

[Issue regarding metadata and raw](https://github.com/SeldonIO/seldon-core/issues/1899), [How does custom metrics works issue](https://github.com/SeldonIO/seldon-core/issues/1898), [Accessing custom metrics issue](https://github.com/SeldonIO/seldon-core/issues/245),[Test combiner seldon code](https://github.com/SeldonIO/seldon-core/blob/master/python/tests/test_combiner_microservice.py),[Add model metadata support issue](https://github.com/SeldonIO/seldon-core/issues/1638)
