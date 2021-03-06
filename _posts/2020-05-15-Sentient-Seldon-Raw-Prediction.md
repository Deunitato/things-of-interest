---
published: true
---
Here I am going to do the testing of the raw prediction method that is avaliable in Seldon

# Base Code 

plus2.py (model) - v1.0
```
import logging
import base64
class plus2(object):
   
    def predict(self, X, feature_names):
        """
        Return a prediction.

        Parameters
        ----------
        X : array-like
        """
        print("Predict called - will run plus2 function")
        logging.info("Char's log (model): " + str(X))
        #myStrin = X.decode() + " appended byte string in times2"
        value = X.get("X") + 2
        
        return "The return value is " + str(value)

```

times2.py (transformer) - v1.0:
```
import logging
class times2(object):

    def __init__(self):
        """
        Add any initialization parameters. These will be passed at runtime from the graph definition parameters defined in your seldondeployment kubernetes resource manifest.
        """
        print("Initializing")

        return


    def transform_input(self, X, feature_names):
        logging.info("Char's log (transform): " + str(X))
        value = X.get("X") * 2
        mydict = {} 
        mydict["X"] = value
        return mydict


```

yaml file:
```
apiVersion: machinelearning.seldon.io/v1alpha2
kind: SeldonDeployment
metadata:
  name: seldon-example
  labels:
    model_name: times2-plus2-raw
    api_type: microservice
    microservice_type: ai
spec:
  name: times2-plus2-raw
  predictors:
  - componentSpecs:
    - spec:
        containers:
        - name: main
          image: gcr.io/science-experiments-divya/plus2:raw-0.1
          imagePullPolicy: Always
        - name: preprocess
          image: gcr.io/science-experiments-divya/times2:raw-preprocess-0.1
        imagePullSecrets: 
          - name: gcr-json-key
    graph:
      name: preprocess
      type: TRANSFORMER
      endpoint:
        type: REST
      children: 
      - name: main
        type: MODEL
        endpoint:
          type: REST
    name: raw-example
    replicas: 1

```

# Attempt to use predict-raw

## V0.1.1 (both)
- Successful sending of data

> Predefined json
![kube_raw_1.PNG]({{site.baseurl}}/img/kube_raw_1.PNG)


> Own Json Type
![kube_raw_2.PNG]({{site.baseurl}}/img/kube_raw_2.PNG)

> More than one values
![kube_raw_3.PNG]({{site.baseurl}}/img/kube_raw_3.PNG)

Log file:

Input: `{"data": {"X": 10}}`

Log output:
`INFO:  Char's log (model): {'data': {'X': 10}}`

## Model - V0.2

Updated predict_raw snippet:

Input: `{"data":{"X":10}}`

(0.2.1)
```
def predict_raw(self, X):
        """
        Return a prediction.

        Parameters
        ----------
        X : array-like
        """
        print("Predict called - will run plus2 function")
        logging.info("Char's log (model): " + str(X))
        
        data = X.get("data", {}).get("X")
        value = int(data) + 2
        return value
```

> There was an error with this

```
Traceback (most recent call last): File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 2447, in wsgi_app response = self.full_dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1952, in full_dispatch_request rv = self.handle_user_exception(e) File "/usr/local/lib/python3.7/site-packages/flask_cors/extension.py", 

line 161, in wrapped_function return cors_after_request(app.make_response(f(*args, **kwargs))) File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1821, in handle_user_exception reraise(exc_type, exc_value, tb) File "/usr/local/lib/python3.7/site-packages/flask/_compat.py",

line 39, in reraise raise value File "/usr/local/lib/python3.7/site-packages/flask/app.py", 

line 1950, in full_dispatch_request rv = self.dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1936, in dispatch_request return self.view_functions[rule.endpoint](**req.view_args) File "/usr/local/lib/python3.7/site-packages/seldon_core/wrapper.py", line 50, in Predict user_model, requestJson, seldon_metrics File "/usr/local/lib/python3.7/site-packages/seldon_core/seldon_methods.py", 

line 84, in predict handle_raw_custom_metrics(response, seldon_metrics, is_proto) File "/usr/local/lib/python3.7/site-packages/seldon_core/seldon_methods.py", line 48, in handle_raw_custom_metrics metrics = msg.get("meta", {}).get("metrics", []) AttributeError: 'int' object has no attribute 'get'
```

Returned server error: 500

Failed test input:
1. `{"data":{"X":10}, "meta":{"metrics":[]}}`

2. 
```
			{"data":{"X":10}, "meta":{"metrics":[
            {"type": "COUNTER", "key": "mycounter", "value": 1}, 
            {"type": "GAUGE", "key": "mygauge", "value": 100},   
            {"type": "TIMER", "key": "mytimer", "value": 20.2}]
            }}
```
3. `{"data":{"X":10}, "meta":{"metrics": {"type": 1}}}`
4. `{"data":{"X":10}, "meta":{"metrics": []}}`
5. `{"data":{"X":10}, "meta":{"metrics": "some metric"}}`
6. `{"data":{"X":10},"meta":{}}`


(0.2.2)
plus2 code snippet:

```
        x = json.loads(X)
        data = x.get("data").get("X")
        value = int(data) + 2
        return value
```
> Attempt to load it as a Json file instead

- Fails

```
line 2447, in wsgi_app response = self.full_dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1952, in full_dispatch_request rv = self.handle_user_exception(e) File "/usr/local/lib/python3.7/site-packages/flask_cors/extension.py", 

line 161, in wrapped_function return cors_after_request(app.make_response(f(*args, **kwargs))) File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1821, in handle_user_exception reraise(exc_type, exc_value, tb) File "/usr/local/lib/python3.7/site-packages/flask/_compat.py", line 39, in reraise raise value File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1950, in full_dispatch_request rv = self.dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", 

line 1936, in dispatch_request return self.view_functions[rule.endpoint](**req.view_args) File "/usr/local/lib/python3.7/site-packages/seldon_core/wrapper.py", line 50, in Predict user_model, requestJson, seldon_metrics File "/usr/local/lib/python3.7/site-packages/seldon_core/seldon_methods.py",

line 83, in predict response = user_model.predict_raw(request) File "/app/plus2.py", line 66, in predict_raw x = json.loads(X) File "/usr/local/lib/python3.7/json/__init__.py", line 341, in loads raise TypeError(f'the JSON object must be str, bytes or bytearray, ' TypeError: the JSON object must be str, bytes or bytearray, not dict
```

> Suspicion that X is a dict

(0.2.3)

- Added metric function

> No difference


Input:
`{"data":{"X":10},"meta":{"metrics":{"type": 1, "counter": 3}}}`

Error
```
Traceback (most recent call last): File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 2447, in wsgi_app response = self.full_dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1952, in full_dispatch_request rv = self.handle_user_exception(e) File "/usr/local/lib/python3.7/site-packages/flask_cors/extension.py", line 161, in wrapped_function return cors_after_request(app.make_response(f(*args, **kwargs))) File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1821, in handle_user_exception reraise(exc_type, exc_value, tb) File "/usr/local/lib/python3.7/site-packages/flask/_compat.py", line 39, in reraise raise value File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1950, in full_dispatch_request rv = self.dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1936, in dispatch_request return self.view_functions[rule.endpoint](**req.view_args) File "/usr/local/lib/python3.7/site-packages/seldon_core/wrapper.py", line 78, in TransformInput user_model, requestJson, seldon_metrics File "/usr/local/lib/python3.7/site-packages/seldon_core/seldon_methods.py", line 198, in transform_input handle_raw_custom_metrics(response, seldon_metrics, is_proto) File "/usr/local/lib/python3.7/site-packages/seldon_core/seldon_methods.py", line 51, in handle_raw_custom_metrics seldon_metrics.update(metrics) File "/usr/local/lib/python3.7/site-packages/seldon_core/metrics.py", line 79, in update metrics_type = metrics.get("type", "COUNTER") AttributeError: 'str' object has no attribute 'get'
```

(0.2.4)

Changed the return type to dictionary
![kube_raw_4.PNG]({{site.baseurl}}/img/kube_raw_4.PNG)

> Successful data sent

```
def predict_raw(self, X):
        """
        Return a prediction.

        Parameters
        ----------
        X : array-like
        """
        print("Predict called - will run plus2 function")
        logging.info("Char's log (model): " + str(X))
        #myStrin = X.decode() + " appended byte string in times2"

        data = X.get("data").get("X")
        value = int(data) + 2

        mydict = {}
        mydict["tree"] = value
        return mydict
```

# Testing of Multipart-form

## Use of V0.1.1

Input: 
- Key: "data"
- File:"Readme.md"


Output:
```
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>
```

Note: Ended with 500 internal server error


```
Traceback (most recent call last): File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 2447, in wsgi_app response = self.full_dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1952, in full_dispatch_request rv = self.handle_user_exception(e) File "/usr/local/lib/python3.7/site-packages/flask_cors/extension.py", line 161, in wrapped_function return cors_after_request(app.make_response(f(*args, **kwargs))) File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1821, in handle_user_exception reraise(exc_type, exc_value, tb) File "/usr/local/lib/python3.7/site-packages/flask/_compat.py",

line 39, in reraise raise value File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1950, in full_dispatch_request rv = self.dispatch_request() File "/usr/local/lib/python3.7/site-packages/flask/app.py", line 1936, in dispatch_request return self.view_functions[rule.endpoint](**req.view_args) File "/usr/local/lib/python3.7/site-packages/seldon_core/wrapper.py", line 75, in TransformInput requestJson = get_request() File "/usr/local/lib/python3.7/site-packages/seldon_core/flask_utils.py",

line 54, in get_request return get_multi_form_data_request() File "/usr/local/lib/python3.7/site-packages/seldon_core/flask_utils.py", line 25, in get_multi_form_data_request req_dict[key] = json.loads(request.form.get(key)) File "/usr/local/lib/python3.7/json/__init__.py", line 348, in loads return _default_decoder.decode(s) File "/usr/local/lib/python3.7/json/decoder.py",

line 337, in decode obj, end = self.raw_decode(s, idx=_w(s, 0).end()) File "/usr/local/lib/python3.7/json/decoder.py", line 355, in raw_decode raise JSONDecodeError("Expecting value", s, err.value) from None json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```


# Model to Model 



(0.4)

### Json
- Input output

Input: `{"tree":{"X":5, "Y": 6}}`

![kube_raw_5.PNG]({{site.baseurl}}/img/kube_raw_5.PNG)

Output: `{"tree":{"X":5, "Y": 6}}`

> Works fine


### Multipart

> Error

Input: 
- key: readme
- Data: README.md

Output: `{}`

#### Log:
1) Preprocess log
```
2020-05-18 08:00:53,963 - root:predict_raw:32 - INFO: Char's log (preprocess): {'readme': "# experiments-seldon-plus2 (raw:0.4)\r\nSimple experiment using Seldon-Core to serve models\r\n\r\nThis specific example make used of predict_raw() to test out.\r\n\r\n### Version Info\r\n\r\nV0.4 Specialized in the testing of model to model raw\r\n\r\n## Description\r\n- Learn how to use Seldon-Core to serve out models\r\n- Use a simple model --> x+2\r\n- This means that the feature is just a number and the prediction is the feature plus 2\r\n- Most important is to understand how to dockerise and chain preprocessing step with prediction step using Selon's inference graph\r\n- Also use this to begin to decide on ML project file hierarchy\r\n\r\n## Folder Structure\r\n- Here we only have the `preprocess` and `deployment_model` folder\r\n- This is because here we are not concerned with training yet. **only deployment**\r\n- The `preprocess` folder contains the preprocessing step\r\n + we need not build this into a container if this is not needed at serving time\r\n- The `deployment_model` folder contains the model loading and prediction functions\r\n + Note that for common ML frameworks like tensorflow, sklearn and pytorch, Seldon core already has dedicated modules to serve out predictions\r\n\r\n# Experiment Log\r\n\r\n## V0.4.0"}
```
2) Model Log
```
2020-05-18 08:00:53,968 - root:predict_raw:61 - INFO: Char's log (model): {}
```

> Assume same error as the transformer - model



# Combiner_raw

## Code base

Combiner snippet:

```
def aggregate_raw(self, Xs):
        """average out the probabilities from multiple classifier and return that as a result"""
        # In this case we return the larger
        print("Combiner called..\n")
        logging.info("(In Combiner):"+str(Xs))
        model1 = Xs[0]
        model2 = Xs[1]
        mydict = {}
        assert (len(model1) == len(model2), "Dictionary size must be the same" )

        for (k,v), (k2,v2) in zip(model1.items(), model2.items()):
            if(v > v2):
                mydict[k] = v
            elif (v < v2):
                mydict[k2] = v2
            else: #they are both equal
                mydict[k] = 0
        
        return mydict

```

Plus2 snippet (Submodel):

```
def predict_raw(self, X):
        """
        Return a prediction.

        Parameters
        ----------
        X : array-like
        """
        print("Predict called - will run plus2 function\n")
        logging.info("(In plus2):"+str(X))
        value = X.get("info").get("X") + 2 #Change this code to make it accept multiple input
        mydict = {}
        mydict["X"] = value

        return mydict

```

times2 snippet (Submodel):

```
def predict_raw(self, X):
        print("Predict call, will run times2 function\n")
        logging.info("(In times2):"+str(X))
        value = X.get("info").get("X") *2 #Change this code to make it accept multiple input
        mydict = {}
        mydict["X"] = value

        return mydict

```

Yaml file graph:

```
apiVersion: machinelearning.seldon.io/v1alpha2
kind: SeldonDeployment
metadata:
  name: seldon-raw-v5
  labels:
    model_name: t2-p2-combiner
    api_type: microservice
    microservice_type: ai
spec:
  name: times2-plus2-raw
  predictors:
  - componentSpecs:
    - spec:
        containers:
        - name: combiner-raw
          image: gcr.io/science-experiments-divya/combiner-raw:0.1
          imagePullPolicy: IfNotPresent
        - name: sub-t2
          image: gcr.io/science-experiments-divya/times2:raw-t2-0.1
          imagePullPolicy: Always
        - name: sub-p2
          image: gcr.io/science-experiments-divya/plus2:raw-p2-0.1
          imagePullPolicy: Always
    graph:
        name: combiner-raw
        type: COMBINER
        endpoint:
          type: REST
        children: 
        - name: sub-t2
          type: MODEL
          endpoint:
            type: REST
        - name: sub-p2
          type: MODEL
          endpoint:
            type: REST
    name: raw
    replicas: 1

```

## Experimentation Log

### combiner-raw (0.1)
- Input: `{"info":{"X":40}}`
- Output:`{"X":80}`

> Works fine

#### Log file:

`
2020-05-18 16:53:56.064 HKT
2020-05-18 08:53:56,064 - root:predict_raw:14 - INFO: (In times2):{'info': {'X': 40}}`

`2020-05-18 16:53:56.065 HKT
2020-05-18 08:53:56,064 - root:predict_raw:54 - INFO: (In plus2):{'info': {'X': 40}}`

`2020-05-18 08:53:56,069 - root:aggregate_raw:9 - INFO: (In Combiner):[{'X': 80}, {'X': 42}]`



# Version History
- 0.1: Base code
- 0.1.1: Input-output, changes all methods to raw
- 0.2s: Modification of data in raw methods
- 0.3s: Attempt to run multiform 
- 0.4: Change model to model
