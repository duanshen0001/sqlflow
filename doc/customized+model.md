# Design Doc: Define Models for SQLFlow

SQLFlow enables SQL programs to call deep learning models defined in Python. This document is about how to define models for SQLFlow.

## Keras v.s. Estimator

Many deep leareners define models using Keras API or as an Estimator derived class.
We prefer [Keras](https://keras.io/) over [Estimator](https://www.tensorflow.org/guide/estimators) for some reasons:

1. [TensorFlow Submit 2019](https://www.youtube.com/watch?v=k5c-vg4rjBw) announced that TensorFlow 2.x will closely integrate with Keras.

2. We found more documents about Keras than Estimator.

3. We found more models defined using Keras than Estimator.

## Keras APIs

Keras provides three approaches to define models.

### 1. Subclassing `tf.keras.Model`

  ```python
  class DNNClassifier(tf.keras.Model):
      def __init__(self, feature_columns, hidden_units, n_classes):
          super(DNNClassifier, self).__init__()
          self.feature_layer = tf.keras.layers.DenseFeatures(feature_columns)
          self.hidden_layers = []
          for hidden_unit in hidden_units:
              self.hidden_layers.append(tf.keras.layers.Dense(hidden_unit))
          self.prediction_layer = tf.keras.layers.Dense(n_classes, activation='softmax')
  
      def call(self, inputs):
          x = self.feature_layer(inputs)
          for hidden_layer in self.hidden_layers:
              x = hidden_layer(x)
          return self.prediction_layer(x)
  
  model = DNNClassifier(feature_columns, hidden_units, n_classes)
  ```

  Please be aware that `tf.keras.Model` has methods `save_weights` and `load_weights`, which save/load model parameters but no the topology, as explained in [this guidance](https://stackoverflow.com/questions/51806852/cant-save-custom-subclassed-model) and [this example list](https://stackoverflow.com/questions/52826134/keras-model-subclassing-examples).

### 2. Functional API

  ```python
  x = tf.feature_column.input_layer(shape=(5,))
  for n in hidden_units:
      x = tf.keras.layers.Dense(n, activation='relu')(x)
  pred = tf.keras.layers.Dense(n_classes, activation='softmax')(x)
  model = tf.keras.models.Model(inputs=feature_columns, outputs=pred)
  ```

  Please be aware that functional API doesn't work with feature column API, as reported [here](https://github.com/tensorflow/tensorflow/issues/27416). However, the approach of deriving classes from `keras.Model` works with the feature column API.

### 3. `keras.Sequential`

  ```python
  model = tf.keras.Sequential()
  model.add(tf.keras.layers.DenseFeatures(feature_columns))
  for n in hidden_units:
    model.add(tf.keras.layers.Dense(n, activation='relu'))
  model.add(tf.keras.layers.Dense(n_classes, activation='softmax'))
  ```

  Please be aware that  `tf.keras.Sequential()` only covers a small variety of models.  It doesn't cover many well-known models including ResNet, Transforms, and WideAndDeep.

### The Choice

We chose the approach of subclassing `tf.keras.Model` according to the following table.

| Keras APIs         | Work with feature column API | Save/load models           | Model coverage |
| ------------------ | ---------------------------- | -------------------------- | -------------- |
| `tf.keras.Model`   | ☑️                            | weights-only, no topology  | High           |
| Functional API     | ❌                           | ☑️                          | High           |
| Sequential Model   | ☑️                            | ☑️                          | Low            |


## An Example

Here is an example `DNNClassifier` of multiple hidden layers as a Python class derived from `tf.keras.Model`. To run it, please use TensorFlow 2.0 alpha or newer versions.

```python
class DNNClassifier(tf.keras.Model):
    def __init__(self, feature_columns, hidden_units, n_classes):
        """DNNClassifier
        :param feature_columns: feature columns.
        :type feature_columns: list[tf.feature_column].
        :param hidden_units: number of hidden units.
        :type hidden_units: list[int].
        :param n_classes: List of hidden units per layer.
        :type n_classes: int.
        """
        super(DNNClassifier, self).__init__()

        # combines all the data as a dense tensor
        self.feature_layer = tf.keras.layers.DenseFeatures(feature_columns)
        self.hidden_layers = []
        for hidden_unit in hidden_units:
            self.hidden_layers.append(tf.keras.layers.Dense(hidden_unit))
        self.prediction_layer = tf.keras.layers.Dense(n_classes, activation='softmax')

    def call(self, inputs):
        x = self.feature_layer(inputs)
        for hidden_layer in self.hidden_layers:
            x = hidden_layer(x)
        return self.prediction_layer(x)

    def default_optimizer(self):
        """Default optimizer name. Used in model.compile."""
        return 'adam'

    def default_loss(self):
        """Default loss function. Used in model.compile."""
        return 'categorical_crossentropy'

    def default_training_epochs(self):
        """Default training epochs. Used in model.fit."""
        return 5

    def prepare_prediction_column(self, prediction):
        """Return the class label of highest probability."""
        return prediction.argmax(axis=-1)
```

## Further Reading

We read the following Keras source code files: [models.py](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/keras/models.py), [network.py](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/keras/engine/network.py), and [training.py](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/keras/engine/training.py).
