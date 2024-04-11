# Distributed Training and Inference

---

**Orca `Estimator` provides sklearn-style APIs for transparently distributed model training and inference** 

### 1. Estimator

To perform distributed training and inference, the user can first create an Orca `Estimator` from any standard (single-node) TensorFlow, Kera or PyTorch model, and then call `Estimator.fit` or `Estimator.predict`  methods (using the [data-parallel processing pipeline](./data-parallel-processing.md) as input).

Under the hood, the Orca `Estimator` will replicate the model on each node in the cluster, feed the data partition (generated by the data-parallel processing pipeline) on each node to the local model replica, and synchronize model parameters using various *backend* technologies (such as *Horovod*, `tf.distribute.MirroredStrategy`, `torch.distributed`, or the parameter sync layer in [*BigDL*](https://github.com/intel-analytics/BigDL)).

### 2. TensorFlow/Keras Estimator

#### 2.1 TensorFlow 1.15 and Keras 2.3

There are two ways to create an Estimator for TensorFlow 1.15, either from a low level computation graph or a Keras model. Examples are as follow:

TensorFlow Computation Graph:
```python
# define inputs to the graph
images = tf.placeholder(dtype=tf.float32, shape=(None, 28, 28, 1))
labels = tf.placeholder(dtype=tf.int32, shape=(None,))

# define the network and loss
logits = lenet(images)
loss = tf.reduce_mean(tf.losses.sparse_softmax_cross_entropy(logits=logits, labels=labels))

# define a metric
acc = accuracy(logits, labels)

# create an estimator using endpoints of the graph
est = Estimator.from_graph(inputs=images,
                           outputs=logits,
                           labels=labels,
                           loss=loss,
                           optimizer=tf.train.AdamOptimizer(),
                           metrics={"acc": acc})
```

Keras Model:
```python
model = create_keras_lenet_model()
model.compile(optimizer=keras.optimizers.RMSprop(),
              loss='sparse_categorical_crossentropy',
              metrics=['accuracy'])
est = Estimator.from_keras(keras_model=model)
```

Then users can perform distributed model training and inference as follows:

```python
dataset = tfds.load(name="mnist", split="train")
dataset = dataset.map(preprocess)
est.fit(data=mnist_train,
        batch_size=320,
        epochs=max_epoch)
predictions = est.predict(data=df,
                          feature_cols=['image'])
```
The `data` argument in `fit` method can be a Spark DataFrame, an *XShards* or a `tf.data.Dataset`. The `data` argument in `predict` method can be a spark DataFrame or an *XShards*. See the *data-parallel processing pipeline* [page](./data-parallel-processing.md) for more details.

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#module-bigdl.orca.learn.tf.estimator) for more details.

#### 2.2 TensorFlow 2.x and Keras 2.4+

**Using `ray` or *Horovod* backend**

Users can create an `Estimator` for TensorFlow 2.x from a Keras model (using a _Model Creator Function_) when the backend is
`ray` (currently default for TF2) or *Horovod*. For example:

```python
def model_creator(config):
    model = create_keras_lenet_model()
    model.compile(optimizer=keras.optimizers.RMSprop(),
                  loss='sparse_categorical_crossentropy',
                  metrics=['accuracy'])
    return model
est = Estimator.from_keras(model_creator=model_creator) # or backend="horovod"
```

The `model_creator` argument should be a function that takes a `config` dictionary and returns a compiled Keras model.

Then users can perform distributed model training and inference as follows:

```python
def train_data_creator(config, batch_size):
    dataset = tfds.load(name="mnist", split="train")
    dataset = dataset.map(preprocess)
    dataset = dataset.batch(batch_size)
    return dataset
stats = est.fit(data=train_data_creator,
                epochs=max_epoch,
                steps_per_epoch=total_size // batch_size)
predictions = est.predict(data=df,
                          feature_cols=['image'])
```

The `data` argument in `fit` method can be a spark DataFrame, an *XShards* or a *Data Creator Function* (that returns a `tf.data.Dataset`). The `data` argument in `predict` method can be a spark DataFrame or an *XShards*. See the *data-parallel processing pipeline* [page](./data-parallel-processing.md) for more details.

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#orca-learn-tf2-tf2-ray-estimator) for more details.

**Using *spark* backend**

Users can create an `Estimator` for TensorFlow 2.x using the *spark* backend as follows:

```python
def model_creator(config):
    model = create_keras_lenet_model()
    model.compile(**compile_args(config))
    return model

def compile_args(config):
    if "lr" in config:
        lr = config["lr"]
    else:
        lr = 1e-2
    args = {
        "optimizer": keras.optimizers.SGD(lr),
        "loss": "mean_squared_error",
        "metrics": ["mean_squared_error"]
    }
    return args

est = Estimator.from_keras(model_creator=model_creator,
                           config={"lr": 1e-2},
                           workers_per_node=2,
                           backend="spark",
                           model_dir=model_dir)
```

The `model_creator` argument should be a function that takes a `config` dictionary and returns a compiled Keras model.
The `model_dir` argument is required for *spark* backend, it should be a share filesystem path which can be accessed by executors for culster mode.  

Then users can perform distributed model training and inference as follows:

```python
def train_data_creator(config, batch_size):
    dataset = tfds.load(name="mnist", split="train")
    dataset = dataset.map(preprocess)
    dataset = dataset.batch(batch_size)
    return dataset
stats = est.fit(data=train_data_creator,
                epochs=max_epoch,
                steps_per_epoch=total_size // batch_size)
predictions = est.predict(data=df,
                          feature_cols=['image']).collect()
```

The `data` argument in `fit` method can be a spark DataFrame, an *XShards* or a *Data Creator Function* (that returns a `tf.data.Dataset`). The `data` argument in `predict` method can be a spark DataFrame or an *XShards*. See the *data-parallel processing pipeline* [page](./data-parallel-processing.md) for more details.

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#orca-learn-tf2-tf2-spark-estimator) for more details.

### 3. PyTorch Estimator

**Using *BigDL* backend**

Users may create a PyTorch `Estimator` using the *Spark* backend (currently default for PyTorch) as follows:

```python
def model_creator(config):
    model = LeNet() # a torch.nn.Module
    model.train()
    return model

def optimizer_creator(model, config):
    return torch.optim.Adam(model.parameters(), config["lr"])

est = Estimator.from_torch(model=model_creator,
                           optimizer=optimizer_creator,
                           loss=nn.NLLLoss(),
                           config={"lr": 1e-2})
```

Then users can perform distributed model training and inference as follows:

```python
est.fit(data=train_loader, epochs=args.epochs)
predictions = est.predict(xshards)
```

The input to `fit` methods can be a `torch.utils.data.DataLoader`, a Spark Dataframe, an *XShards*, or a *Data Creator Function* (that returns a `torch.utils.data.DataLoader`). The input to `predict` methods should be a Spark Dataframe, or an *XShards*. See the *data-parallel processing pipeline* [page](./data-parallel-processing.md) for more details.

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#orca-learn-pytorch-pytorch-spark-estimator) for more details.

**Using `torch.distributed` or *Horovod* backend**

Alternatively, users can create a PyTorch `Estimator` using `torch.distributed` or *Horovod* backend by specifying the `backend` argument to be "ray" or "horovod". In this case, the `model` and `optimizer` should be wrapped in _Creater Functions_. For example:

```python
def model_creator(config):
    model = LeNet() # a torch.nn.Module
    model.train()
    return model

def optimizer_creator(model, config):
    return torch.optim.Adam(model.parameters(), config["lr"])

est = Estimator.from_torch(model=model_creator,
                           optimizer=optimizer_creator,
                           loss=nn.NLLLoss(),
                           config={"lr": 1e-2},
                           backend="ray") # or backend="horovod"
```

Then users can perform distributed model training and inference as follows:

```python
est.fit(data=train_loader_func, epochs=args.epochs)
predictions = est.predict(data=df,
                          feature_cols=['image'])
```

The input to `fit` methods can be a Spark DataFrame, an *XShards*, or a *Data Creator Function* (that returns a `torch.utils.data.DataLoader`). The `data` argument in `predict` method can be a Spark DataFrame or an *XShards*. See the *data-parallel processing pipeline* [page](./data-parallel-processing.md) for more details.

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#orca-learn-pytorch-pytorch-ray-estimator) for more details.

### 4. MXNet Estimator

The user may create a MXNet `Estimator` as follows:
```python
from bigdl.orca.learn.mxnet import Estimator, create_config

def get_model(config):
    net = LeNet() # a mxnet.gluon.Block
    return net

def get_loss(config):
    return gluon.loss.SoftmaxCrossEntropyLoss()

config = create_config(log_interval=2, optimizer="adam",
                       optimizer_params={'learning_rate': 0.02})
est = Estimator.from_mxnet(config=config,
                           model_creator=get_model,
                           loss_creator=get_loss,
                           num_workers=2)
```

Then the user can perform distributed model training as follows:
```python
import numpy as np

def get_train_data_iter(config, kv):
    train = mx.io.NDArrayIter(data_ndarray, label_ndarray,
                              batch_size=config["batch_size"], shuffle=True)
    return train

est.fit(get_train_data_iter, epochs=2)
```

The input to `fit` methods can be an *XShards*, or a *Data Creator Function* (that returns an `MXNet DataIter/DataLoader`). See the *data-parallel processing pipeline* [page](./data-parallel-processing.html) for more details.

### 5. BigDL Estimator

The user may create a BigDL `Estimator` as follows:
```python
from bigdl.dllib.nn.criterion import *
from bigdl.dllib.nn.layer import *
from bigdl.dllib.optim.optimizer import *
from bigdl.orca.learn.bigdl import Estimator

linear_model = Sequential().add(Linear(2, 2))
mse_criterion = MSECriterion()
est = Estimator.from_bigdl(model=linear_model, loss=mse_criterion, optimizer=Adam())
```

Then the user can perform distributed model training and inference as follows:
```python
# read spark Dataframe
df = spark.read.parquet("data.parquet")

# distributed model training
est.fit(df, 1, batch_size=4)

#distributed model inference
result_df = est.predict(df)
```

The input to `fit` and `predict` methods can be a *Spark Dataframe*, or an *XShards*. See the *data-parallel processing pipeline* [page](./data-parallel-processing.html) for more details.

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#module-bigdl.orca.learn.bigdl.estimator) for more details.

### 6. OpenVINO Estimator

The user may create a OpenVINO `Estimator` as follows:
```python
from bigdl.orca.learn.openvino import Estimator

model_path = "The/file_path/to/the/OpenVINO_IR_xml_file"
est = Estimator.from_openvino(model_path=model_path)
```

Then the user can perform distributed model inference as follows:
```python
# ndarray
input_data = np.random.random([20, 4, 3, 224, 224])
result = est.predict(input_data)

# xshards
shards = XShards.partition({"x": input_data})
result_shards = est.predict(shards)
```

The input to `predict` methods can be an *XShards*, or a *numpy array*. See the *data-parallel processing pipeline* [page](./data-parallel-processing.html) for more details.

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#orca-learn-openvino-estimator) for more details.

### 7. MPI Estimator
The Orca MPI Estimator is to run distributed training job based on MPI.

#### Preparation:
* Configure password-less ssh from the master node (the one you'll launch training from) to all other nodes.

* All hosts have the same working directory.
* All hosts have the same Python environment in the same location.

#### Train
Then the user may create a MPI Estimator as follows:
```python
from bigdl.orca.learn.mpi import MPIEstimator

est = MPIEstimator(model_creator=model_creator,
                   optimizer_creator=optimizer_creator,
                   loss_creator=None,
                   metrics=None,
                   scheduler_creator=None,
                   config=config,
                   init_func=init,  # Init the distributed environment for MPI if any
                   hosts=hosts,
                   workers_per_node=workers_per_node,
                   env=None)                   
```
Then the user can perform distributed model training as follows:
```python
# read spark Dataframe
df = spark.read.parquet("data.parquet")

# distributed model training
est.fit(data=df, epochs=1, batch_size=4, feature_col="feature", label_cols="label")

```
The input to `fit` methods can be an Spark Dataframe, or a callable  function to return a `torch.utils.data.DataLoader`. 

View the related [Python API doc](https://bigdl.readthedocs.io/en/latest/doc/PythonAPI/Orca/orca.html#orca-learn-mpi-mpi-estimator) for more details.

