# Project Baikal

## Main goal
Provide a Keras/TensorFlow like API to develop complex machine learning pipelines. The API is mainly for pipelines that use algorithms other than neural networks (that's what TensorFlow, pytorch, Caffe, etc is for. That said, a neural network could be a component in the pipeline).

## Design guidelines
- API should be something like:

```python
# x1, x2, x1 ... y are Data objects with pointers to the actual results,
# the actual results will be realized once the graph is run (feed_forward)
# (conceptually, kinda similar to a Tensor in Tensorflow)
x1 = Input(name='input1')
x2 = Input(name='input2')

z1 = SomePreprocessor(...)(x1)
z2 = SomePreprocessor(...)(x2)
c = Concatenate(...)([z1, z2])
y = SVM(..., name='y')(c)

# Define a model
model = Model([x1, x2], y)  # Could also be a sub-graph

# Train model a la Keras:
model.fit([x1_data, x2_data], y_data)

# or a la TensorFlow:
model.fit({'input1': x1_data,
           'input2': x2_data,
           'y': y_data})
             
# predict and feed_forward return concrete data (numpy arrays, pandas dataframes, etc)
y_pred = model.predict([x1_data, x2_data])

# Can also query specific outputs (and only give the necessary inputs)
outs = model.predict({'input1': x1_data}, outputs=['z1'])
```

## Processor abstract class
```python
```

## Some desired features
- Add model (graph) serialization and de-serialization
    - Something like `model.save(filename)` and `model.load(filename)`
    - Most likely pickle
        - Define `__getstate__` and `__setstate__` API
    - Consider using joblib, yaml?
- Should be able to use models as components of a bigger graph
- Choose the type of output of classifiers (predicted label, predicted score)
    - Useful for model ensembling
- It should be possible to freeze models to avoid fitting (e.g. when we loaded a pretrained model)
- Should have a `set_params` and `get_params` for compatiblity with sklearn's GridSearch API.
- Provide a factory function to extend sklearn classes:
    - e.g.: `LogisticRegression = make_sklearn_processor(sklearn.linear_model.logistic.LogisticRegression)`
- A cache to avoid repeating the computations if the input data hasn't changed

## Implementation ideas
- DiGraph should be built dynamically (like TensorFlow)
    - There should be a default graph
    - Nodes should be added to the graph on instantiation
    - Edges should be added on calls to node `__call__` (e.g. `SomeNode(...)([n1, n2])`)
- Represent graphs as dicts-of-sets (adjacency list)
    - This is similar to the data structure used in networkx (dicts-of-dicts)
    - Maybe only dicts (nodes) of sets (successors) are necessary
        - Do we also need a similar structure with the predecessors?
- The following abstractions should be made public to code:
    - Processor
        - A node of the graph
        - Analogous to TensorFlow's Operation
        - Internally derived from a Node class
    - Data
        - An semi-edge of the graph
        - Analogous to TensorFlow's Tensor
        - Internally derived from a Edge class
    - Input
        - An special node of the graph that allows inputting data (arrays, dataframes, etc) from client code
            - At instantiation, internally it creates a special InputNode, and returns a Data object
        - Analogous to TensorFlow's Placeholder and Input in Keras
        - Must specify an input shape
    - Model
        - A graph (pf Processor's that pass Data along each other) with defined inputs and outputs.
        - A graph with fit/predict API
        - Models should be callable on Data inputs
        - Internally derived from a DiGraph class
- Need to implement check/inference of input/output shapes
    - Shape information is delegated to Data class
    - Processors like Concatenate, Split and Merge need to know about the input shapes
    - sklearn Processors inputs and outputs are of shape (n_samples, n_features) and (n_samples,), respectively
    - Also necessary to infer the number of outputs of a Processor
        - There is no way in Python to know a priori the type and number of outputs of a function
    - Provide a `compute_output_shapes` API
- Use `networkx` for graph stuff. After having a working API, replace with a in-house module.
    - We don't need much from `networkx`. Just the primitive classes and the topological sort algorithm.

### Compilation (i.e. call to `Model(...)`)
- Do topological sort to get the order of execution of the processors.

### Feedforward
- Find the required processors with a recursive predecessor search
    - Backtrace the predecesor nodes to find the inputs required to compute the specified outputs
        - Stop a backtrace path if the node's output is found in the provided inputs
- Execute the required processors according to the topological sort
    - Use a results cache (just a dict)

## Tests TODO list

### API test cases:

- [x] Can create an Input with a name
    - [x] Takes a shape argument (mandatory). The shape should not include n_samples
    - [x] If name is not specified, a unique name should be generated
        - [x] Input (Node) naming format: `graph_name/node_name`
        - [x] Data (Node output, semi-edge) naming format: `graph_name/node_name/output_name` ?
    - [x] Creates another instance with an unique name if an Input is created with a name already used by another Input
    - [x] At instantiation:
        - [x] An InputNode is added to the default graph
        - [ ] A Data object with the specified shape and name is returned
    
```python
x1 = Input(name='x1')
x2 = Input(name='x2')
```

- [x] Can extend a user-defined class by mixing with ProcessorMixin.
    - The user-defined class must implement the sklearn API:
        - fit/predict ( + predict_proba/decision_function)
        - fit/transform
        - transform only (this is for nodes that do not require fit and just transform the data, e.g. Concatenate, Merge, Split, etc.)
        - get_params
        - set_params
    - Though not implemented explicitly, any class that is extended with ProcessorMixin will be referred as Processor hereafter.

- [x] A Processor can be instantiated with a name.
    - [x] If name is not specified, a unique name should be generated
    - [x] Creates another instance with a unique name if a Processor is created with a name already used by another Processor
- [x] A Processor must check its inputs shapes and provide its output shapes
    
```python
class Processor(ProcessorMixin, SomeSklearnClass):
    pass
    
p0 = Processor(name='p0')
p1 = Processor(name='p1')
```

- [ ] Can call a Processor component instance with (possibly several Data objects).
    - A component must be defined by extending the original component with Processor mixin class
    - A component can only be called with Data objects as inputs
    - A call to Processor must return Data objects
```python
pred = SVC(...)(inputs=[x1, x2])
```

- [ ] Can instantiate a Model with specified inputs and outputs (inputs and outputs are Data objects)
```python
model = Model(inputs=[x1, x2], outputs=pred)
```

- [ ] Can fit the model a la Keras with lists of actual data (numpy arrays, pandas dataframes, etc)
```python
model.fit([x1_data, x2_data], pred_data)
```

- [ ] Can fit the model a la TensorFlow, with a dictionary of actual data (numpy arrays, pandas dataframes, etc)
```python
model.fit({'x1': ..., 'pred': ...})  # dictionary keys could also be the Data objects themselves?
```

- [ ] Can predict with the model a la Keras with lists of actual data (numpy arrays, pandas dataframes, etc)
    - If multiple outputs were specified, a list of actual result data is returned
```python
out = model.predict([x1_data, x2_data])
# out is actual result data (e.g. a numpy array) 
```

- [ ] Can predict with the model a la TensorFlow, with a dictionary of actual data (numpy arrays, pandas dataframes, etc)
```python
model.predict({'x1': ...})  # dictionary keys could also be the Data objects themselves?
# out = {'pred': ...}
```

- [ ] model.fit fails if any of the required inputs was not passed in the dictionary
    - This includes target data for outputs in the case of supervised learning

- [ ] model.predict fails if any of the required inputs is not in the input dictionary