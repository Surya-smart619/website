# Deploy LightGBM model with InferenceService

This example walks you through how to deploy a `LightGBM` model leveraging
the `v1beta1` version of the `InferenceService` CRD.
Note that, by default the `v1beta1` version will expose your model through an
API compatible with the existing V1 Dataplane.
However, this example will show you how to serve a model through an API
compatible with the new [V2 Dataplane](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2)

## Training

The first step will be to train a sample `LightGBM` model.
Note that this model will be then saved as `model.bst`.

```python
import lightgbm as lgb
from sklearn.datasets import load_iris
import os

model_dir = "."
BST_FILE = "model.bst"

iris = load_iris()
y = iris['target']
X = iris['data']
dtrain = lgb.Dataset(X, label=y)

params = {
    'objective':'multiclass', 
    'metric':'softmax',
    'num_class': 3
}
lgb_model = lgb.train(params=params, train_set=dtrain)
model_file = os.path.join(model_dir, BST_FILE)
lgb_model.save_model(model_file)
```

## Testing locally

Once you've got your model serialised `model.bst`, we can then use
[MLServer](https://github.com/SeldonIO/MLServer) to spin up a local server.
For more details on MLServer, feel free to check the [SKLearn example doc](https://github.com/SeldonIO/MLServer/blob/master/docs/examples/lightgbm/README.md).

!!! Note
    this step is optional and just meant for testing, feel free to jump straight to [deploying with InferenceService](#deploy-with-inferenceservice).

### Pre-requisites

Firstly, to use MLServer locally, you will first need to install the `mlserver`
package in your local environment, as well as the LightGBM runtime.

```bash
pip install mlserver mlserver-lightgbm
```

### Model settings

The next step will be providing some model settings so that
MLServer knows:

- The inference runtime to serve your model (i.e. `mlserver_lightgbm.LightGBMModel`)
- The model's name and version

These can be specified through environment variables or by creating a local
`model-settings.json` file:

```json
{
  "name": "lightgbm-iris",
  "version": "v1.0.0",
  "implementation": "mlserver_lightgbm.LightGBMModel"
}
```

Note that, when you [deploy your model](#deployment), **KServe will already
inject some sensible defaults** so that it runs out-of-the-box without any
further configuration.
However, you can still override these defaults by providing a
`model-settings.json` file similar to your local one.
You can even provide a [set of `model-settings.json` files to load multiple
models](https://github.com/SeldonIO/MLServer/tree/master/docs/examples/mms).

### Serving model locally

With the `mlserver` package installed locally and a local `model-settings.json`
file, you should now be ready to start our server as:

```bash
mlserver start .
```

## Deploy with InferenceService

Lastly, you will use KServe to deploy the trained model.
For this, you will just need to use **version `v1beta1`** of the
`InferenceService` CRD and set the **`protocolVersion` field to `v2`**.

=== "Old Schema"

```yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "lightgbm-v2-iris"
spec:
  predictor:
    lightgbm:
      protocolVersion: v2
      storageUri: "gs://kfserving-examples/models/lightgbm/v2/iris"
```

=== "New Schema"

```yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "lightgbm-v2-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: lightgbm
      protocolVersion: v2
      storageUri: "gs://kfserving-examples/models/lightgbm/v2/iris"
```

Note that this makes the following assumptions:

- Your model weights (i.e. your `model.bst` file) have already been uploaded
  to a "model repository" (GCS in this example) and can be accessed as
  `gs://kfserving-examples/models/lightgbm/v2/iris`.
- There is a K8s cluster available, accessible through `kubectl`.
- KServe has already been [installed in your cluster](../../../../get_started/README.md).

=== "kubectl"

```bash
kubectl apply -f ./lightgbm.yaml
```

==**Expected Output**==

```bash
inferenceservice.serving.kserve.io/lightgbm-v2-iris created
```

## Testing deployed model

You can now test your deployed model by sending a sample request.

Note that this request **needs to follow the [V2 Dataplane
protocol](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2)**.
You can see an example payload below:

```json
{
  "inputs": [
    {
      "name": "input-0",
      "shape": [2, 4],
      "datatype": "FP32",
      "data": [
        [6.8, 2.8, 4.8, 1.4],
        [6.0, 3.4, 4.5, 1.6]
      ]
    }
  ]
}
```

Now, assuming that your ingress can be accessed at
`${INGRESS_HOST}:${INGRESS_PORT}` or you can follow [this instruction](../../../../get_started/first_isvc.md#3-determine-the-ingress-ip-and-ports)
to find out your ingress IP and port.

you can use `curl` to send the inference request as:

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice lightgbm-v2-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d @./iris-input.json \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/lightgbm-v2-iris/infer
```

==**Expected Output**==

```json
{
  "model_name":"lightgbm-v2-iris",
  "model_version":null,
  "id":"96253e27-83cf-4262-b279-1bd4b18d7922",
  "parameters":null,
  "outputs":[
    {
      "name":"predict",
      "shape":[2,3],
      "datatype":"FP64",
      "parameters":null,
      "data":
        [8.796664107010673e-06,0.9992300031041593,0.0007612002317336916,4.974786820804187e-06,0.9999919650711493,3.0601420299625077e-06]
    }
  ]
}
```
