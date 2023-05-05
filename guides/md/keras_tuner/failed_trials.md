# Handling failed trials in KerasTuner

**Authors:** Haifeng Jin<br>
**Date created:** 2023/02/28<br>
**Last modified:** 2023/02/28<br>
**Description:** The basics of fault tolerance configurations in KerasTuner.


<img class="k-inline-icon" src="https://colab.research.google.com/img/colab_favicon.ico"/> [**View in Colab**](https://colab.research.google.com/github/keras-team/keras-io/blob/master/guides/ipynb/keras_tuner/failed_trials.ipynb)  <span class="k-dot">•</span><img class="k-inline-icon" src="https://github.com/favicon.ico"/> [**GitHub source**](https://github.com/keras-team/keras-io/blob/master/guides/keras_tuner/failed_trials.py)



---
## Introduction

A KerasTuner program may take a long time to run since each model may take a
long time to train. We do not want the program to fail just because some trials
failed randomly.

In this guide, we will show how to handle the failed trials in KerasTuner,
including:

* How to tolerate the failed trials during the search
* How to mark a trial as failed during building and evaluating the model
* How to terminate the search by raising a `FatalError`

---
## Setup


```python
!pip install keras-tuner -q
```


```python
from tensorflow import keras
from tensorflow.keras import layers
import keras_tuner
import numpy as np
```

---
## Tolerate failed trials

We will use the `max_retries_per_trial` and `max_consecutive_failed_trials`
arguments when initializing the tuners.

`max_retries_per_trial` controls the maximum number of retries to run if a trial
keeps failing. For example, if it is set to 3, the trial may run 4 times (1
failed run + 3 failed retries) before it is finally marked as failed. The
default value of `max_retries_per_trial` is 0.

`max_consecutive_failed_trials` controls how many consecutive failed trials
(failed trial here refers to a trial that failed all of its retries) occur
before terminating the search. For example, if it is set to 3 and Trial 2, Trial
3, and Trial 4 all failed, the search would be terminated. However, if it is set
to 3 and only Trial 2, Trial 3, Trial 5, and Trial 6 fail, the search would not
be terminated since the failed trials are not consecutive. The default value of
`max_consecutive_failed_trials` is 3.

The following code shows how these two arguments work in action.

* We define a search space with 2 hyperparameters for the number of units in the
  2 dense layers.
* When their product is larger than 800, we raise a `ValueError` for the model
  too large.


```python

def build_model(hp):
    # Define the 2 hyperparameters for the units in dense layers
    units_1 = hp.Int("units_1", 10, 40, step=10)
    units_2 = hp.Int("units_2", 10, 30, step=10)

    # Define the model
    model = keras.Sequential(
        [
            layers.Dense(units=units_1, input_shape=(20,)),
            layers.Dense(units=units_2),
            layers.Dense(units=1),
        ]
    )
    model.compile(loss="mse")

    # Raise an error when the model is too large
    num_params = model.count_params()
    if num_params > 1200:
        raise ValueError(f"Model too large! It contains {num_params} params.")
    return model

```

We set up the tuner as follows.

* We set `max_retries_per_trial=3`.
* We set `max_consecutive_failed_trials=8`.
* We use `GridSearch` to enumerate all hyperparameter value combinations.


```python
tuner = keras_tuner.GridSearch(
    hypermodel=build_model,
    objective="val_loss",
    overwrite=True,
    max_retries_per_trial=3,
    max_consecutive_failed_trials=8,
)

# Use random data to train the model.
tuner.search(
    x=np.random.rand(100, 20),
    y=np.random.rand(100, 1),
    validation_data=(
        np.random.rand(100, 20),
        np.random.rand(100, 1),
    ),
    epochs=10,
)

# Print the results.
tuner.results_summary()
```

<div class="k-default-codeblock">
```
Trial 12 Complete [00h 00m 00s]
```
</div>
    
<div class="k-default-codeblock">
```
Best val_loss So Far: 0.11403200030326843
Total elapsed time: 00h 00m 14s
INFO:tensorflow:Oracle triggered exit
Results summary
Results in ./untitled_project
Showing 10 best trials
<keras_tuner.engine.objective.Objective object at 0x7ff05a5c7280>
Trial summary
Hyperparameters:
units_1: 20
units_2: 30
Score: 0.11403200030326843
Trial summary
Hyperparameters:
units_1: 20
units_2: 20
Score: 0.12364783138036728
Trial summary
Hyperparameters:
units_1: 10
units_2: 30
Score: 0.1268368661403656
Trial summary
Hyperparameters:
units_1: 10
units_2: 20
Score: 0.15601204335689545
Trial summary
Hyperparameters:
units_1: 30
units_2: 10
Score: 0.16938944160938263
Trial summary
Hyperparameters:
units_1: 20
units_2: 10
Score: 0.20609712600708008
Trial summary
Hyperparameters:
units_1: 10
units_2: 10
Score: 0.36607497930526733

```
</div>
---
## Mark a trial as failed

When the model is too large, we do not need to retry it. No matter how many
times we try with the same hyperparameters, it is always too large.

We can set `max_retries_per_trial=0` to do it. However, it will not retry no
matter what errors are raised while we may still want to retry for other
unexpected errors. Is there a way to better handle this situation?

We can raise the `FailedTrialError` to skip the retries. Whenever, this error is
raised, the trial would not be retried. The retries will still run when other
errors occur. An example is shown as follows.


```python

def build_model(hp):
    # Define the 2 hyperparameters for the units in dense layers
    units_1 = hp.Int("units_1", 10, 40, step=10)
    units_2 = hp.Int("units_2", 10, 30, step=10)

    # Define the model
    model = keras.Sequential(
        [
            layers.Dense(units=units_1, input_shape=(20,)),
            layers.Dense(units=units_2),
            layers.Dense(units=1),
        ]
    )
    model.compile(loss="mse")

    # Raise an error when the model is too large
    num_params = model.count_params()
    if num_params > 1200:
        # When this error is raised, it skips the retries.
        raise keras_tuner.errors.FailedTrialError(
            f"Model too large! It contains {num_params} params."
        )
    return model


tuner = keras_tuner.GridSearch(
    hypermodel=build_model,
    objective="val_loss",
    overwrite=True,
    max_retries_per_trial=3,
    max_consecutive_failed_trials=8,
)

# Use random data to train the model.
tuner.search(
    x=np.random.rand(100, 20),
    y=np.random.rand(100, 1),
    validation_data=(
        np.random.rand(100, 20),
        np.random.rand(100, 1),
    ),
    epochs=10,
)

# Print the results.
tuner.results_summary()
```

<div class="k-default-codeblock">
```
Trial 12 Complete [00h 00m 00s]
```
</div>
    
<div class="k-default-codeblock">
```
Best val_loss So Far: 0.09288986027240753
Total elapsed time: 00h 00m 11s
INFO:tensorflow:Oracle triggered exit
Results summary
Results in ./untitled_project
Showing 10 best trials
<keras_tuner.engine.objective.Objective object at 0x7ff0583327a0>
Trial summary
Hyperparameters:
units_1: 10
units_2: 30
Score: 0.09288986027240753
Trial summary
Hyperparameters:
units_1: 20
units_2: 30
Score: 0.10828635841608047
Trial summary
Hyperparameters:
units_1: 20
units_2: 10
Score: 0.11322537064552307
Trial summary
Hyperparameters:
units_1: 20
units_2: 20
Score: 0.12243752181529999
Trial summary
Hyperparameters:
units_1: 10
units_2: 10
Score: 0.129277765750885
Trial summary
Hyperparameters:
units_1: 30
units_2: 10
Score: 0.16572774946689606
Trial summary
Hyperparameters:
units_1: 10
units_2: 20
Score: 0.2288801372051239

```
</div>
---
## Terminate the search programmatically

When there is a bug in the code we should terminate the search immediately and
fix the bug. You can terminate the search programmatically when your defined
conditions are met. Raising a `FatalError` (or its subclasses `FatalValueError`,
`FatalTypeError`, or `FatalRuntimeError`) will terminate the search regardless
of the `max_consecutive_failed_trials` argument.

Following is an example to terminate the search when the model is too large.


```python

def build_model(hp):
    # Define the 2 hyperparameters for the units in dense layers
    units_1 = hp.Int("units_1", 10, 40, step=10)
    units_2 = hp.Int("units_2", 10, 30, step=10)

    # Define the model
    model = keras.Sequential(
        [
            layers.Dense(units=units_1, input_shape=(20,)),
            layers.Dense(units=units_2),
            layers.Dense(units=1),
        ]
    )
    model.compile(loss="mse")

    # Raise an error when the model is too large
    num_params = model.count_params()
    if num_params > 1200:
        # When this error is raised, the search is terminated.
        raise keras_tuner.errors.FatalError(
            f"Model too large! It contains {num_params} params."
        )
    return model


tuner = keras_tuner.GridSearch(
    hypermodel=build_model,
    objective="val_loss",
    overwrite=True,
    max_retries_per_trial=3,
    max_consecutive_failed_trials=8,
)

try:
    # Use random data to train the model.
    tuner.search(
        x=np.random.rand(100, 20),
        y=np.random.rand(100, 1),
        validation_data=(
            np.random.rand(100, 20),
            np.random.rand(100, 1),
        ),
        epochs=10,
    )
except keras_tuner.errors.FatalError:
    print("The search is terminated.")
```

<div class="k-default-codeblock">
```
Trial 7 Complete [00h 00m 02s]
val_loss: 0.09336935728788376
```
</div>
    
<div class="k-default-codeblock">
```
Best val_loss So Far: 0.09336935728788376
Total elapsed time: 00h 00m 10s
```
</div>
    
<div class="k-default-codeblock">
```
Search: Running Trial #8
```
</div>
    
<div class="k-default-codeblock">
```
Value             |Best Value So Far |Hyperparameter
30                |30                |units_1
20                |10                |units_2
```
</div>
    
<div class="k-default-codeblock">
```
The search is terminated.

```
</div>
---
## Takeaways

In this guide, you learn how to handle failed trials in KerasTuner:

* Use `max_retries_per_trial` to specify the number of retries for a failed
  trial.
* Use `max_consecutive_failed_trials` to specify the maximum consecutive failed
  trials to tolerate.
* Raise `FailedTrialError` to directly mark a trial as failed and skip the
  retries.
* Raise `FatalError`, `FatalValueError`, `FatalTypeError`, `FatalRuntimeError`
  to terminate the search immediately.