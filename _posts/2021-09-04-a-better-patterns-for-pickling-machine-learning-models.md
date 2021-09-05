---
layout: posts
comments: true
author_profile: true
title: "A better pattern for pickling machine learning models"
excerpt: Can we do better than a straightforward pickle.dump?
---

## TL;DR

You may do this

```python
with open("model.pkl", "wb") as handle:
	pickle.dump(model, handle)
```

But this is probably better

```python
pickled_model = pickle.dumps(model)
model_and_details = {
    "model": pickled_model,
    "versions": {"model_library": "1.1.1"},
    "commit_id": "commit_id_value",
}
with open("model.pkl", "wb") as handle:
	pickle.dump(model_and_details, handle)
```

## The common way

If you google something along the lines of "how to store machine learning models with pickle" the vast majority of results will suggest doing something like in the snippet below:

```python
from sklearn import svm
import pickle

X = [[0, 0], [1, 1]]
y = [0, 1]
model = svm.SVC()
model.fit(X, y)

model.predict([[1, 2]])
# array([1])

# save the model
with open("model.pkl", "bw") as handle:
	pickle.dump(model, handle)

# load the model
with open("model.pkl", "br") as handle:
	model = pickle.load(handle)

model.predict([[1, 2]])
# array([1])
```

Which arguably gets the job done. What happens though if you try to re-use the model in an environment that is using different versions of the libraries the model needs (`sklearn` in this case)?

## Some details on how pickle stores the object (your model)

As the [pickle documentation](https://docs.python.org/3/library/pickle.html#what-can-be-pickled-and-unpickled) explains:

`[...] only the function name is pickled, along with the name of the module the function is defined in. Neither the function’s code, nor any of its function attributes are pickled. Thus the defining module must be importable in the unpickling environment [...] Similarly, classes are pickled by named reference, so the same restrictions in the unpickling environment apply. Note that none of the class’s code or data is pickled`

Needless to say, the library the model was originally imported from has to be installed in the environment in which the unpickling happens.

Another interesting bit from the docs:

`when class instances are pickled, their class’s code and data are not pickled along with them. Only the instance data are pickled.`

So, when pickling an object instance we store on disk the name of the class, the name of the module in which this is defined, and the state of the instance (data attached to it).

## Should I care?

So, when pickling `model` in the first example I am effectively only pickling a class name, a reference to a `sklearn` module, and the attributes attached to the model instance. Which seems reasonable, but may lead to issues if a library upgrade/downgrade changes some of the attributes used by the model, either by renaming them, either by modifying how they are used in the model logic.

### Unpickling potentially misbehaving models

As an example, in the new library version, the implementation of `predict` has changed and the same private attribute is now used (and computed) differently. So, even when using the same attributes, the same model will now output a different prediction. Not that this ever happened to me, but it should be possible in principle.

You can get a better grasp of this behavior by running the following snippet:

```python
import pickle

class Model:
    MODEL_INTERNAL_PARAMETER = 2

    def __init__(self, parameter):
        self._parameter = parameter

    def predict(self, x):
        return self._parameter ** self.MODEL_INTERNAL_PARAMETER * x

model = Model(parameter=2)
prediction_pre_dump = model.predict(3)
# prediction_pre_dump will be 12

with open("model.pkl", "wb") as handle:
    pickle.dump(model, handle)

# The model internal implementation has changed, now MODEL_INTERNAL_PARAMETER is 3
class Model:
    MODEL_INTERNAL_PARAMETER = 3

    def __init__(self, parameter):
        self._parameter = parameter

    def predict(self, x):
        return self._parameter ** self.MODEL_INTERNAL_PARAMETER * x

with open("model.pkl", "rb") as handle:
    model = pickle.load(handle)

prediction_post_dump = model.predict(3)
# prediction_post_dump will now be 24
```

In the example, I preferred to use customer classes to showcase the pickling behavior, rather than looking for a similar change in two versions of the same library. A case of the latter would be harder to find (I hope similar changes don't happen often).

Given that, I think it would be much safer to also store with the model the name and the version of the libraries used.

Would storing something like this then be enough?

```python
model_with_meta = {
    "model": model,
    "versions": {"model_library": "1.1.1"}
}

with open("model.pkl", "wb") as handle:
    pickle.dump(model_with_meta, handle)
```

Not quite, but we are getting on the right track.

### Simply not being able to unpickle

By storing the model with the relevant versions like in the last code block we would still have issues if with the upgrade the `Model` class definition is moved from one module to another. In that case, we won't be able to unpickle the file at all.

Again you can check this by running the snippet below

```python
# model the Model class defined above in a models.py module and import it
from models import Model

model = Model(parameter=2)
prediction_pre_dump = model.predict(3)
model_with_meta = {
    "model": model,
    "versions": {"model_library": "1.1.1"}
}

with open("model.pkl", "wb") as handle:
    pickle.dump(model_with_meta, handle)

# this works as expected
with open("model.pkl", "rb") as handle:
    model = pickle.load(handle)
model.predict(3)

# now let's rename the models.py file to classifiers.py
import os; os.rename("model.pkl", "classifiers.py")
# renaming the module is a bit like simulating the model being defined in a different module with the library upgrade

with open("model.pkl", "rb") as handle:
    model = pickle.load(handle)
# trying to unpickle model.pkl will raise `ModuleNotFoundError: No module named 'models'`
```

Alternatively, it's also possible that the old parameters are used in the same way and the model class is defined in the same location but the `predict` method now requires some other new attribute to run. In that case, the model will be loaded but running `predict` will raise an exception.

## So, how to pickle the model?

What we may want to do is decoupling the pickling of the versions (and any other model metadata that should be saved) from the one of the model itself. We can easily achieve this by only pickling the model first, and then pickling the versions (+ metadata) together with the bytes of the model just pickled.

Doing so allows us to safely unpickle the versions details without having to worry about what should be accessible in the environment so that the model can be loaded and ran correctly. Once we are happy with the versions and our environment we can unpickle the model itself.

```python
from models import Model

model = Model(parameter=2)
pickled_model = pickle.dumps(model)
model_with_meta = {
    "model": pickled_model,
    "versions": {"model_library": "1.1.1"}
}

with open("model.pkl", "wb") as handle:
    pickle.dump(model_with_meta, handle)

# now let's rename the models.py file to classifiers.py
import os; os.rename("model.pkl", "classifiers.py")
# in this case we can load the model.pkl content
with open("model.pkl", "rb") as handle:
    model = pickle.load(handle)

# the content of model will be something like:
# {'model': b'\x80\x04\x95*\x00\x00\x00\x00\x00\x00\x00\x8c\x06models\x94\x8c\x05Model\x94\x93\x94)\x81\x94}\x94\x8c\n_parameter\x94K\x02sb.',
# 'versions': {'model_library': '1.1.1'}}

# if we are happy with the versions in our environment we can unpickle the mode itself and use it
unpickled_model = pickle.loads(model['model'])
unpickled_model.predict(3)

```

Given the examples I used, it should be straightforward that the pickling/unpickling will be an issue also with custom models (or models wrappers), and not simply with models coming from third-party libraries.

While tracking changes to code internally may not be as simple as checking library versions, a decent enough approach would be storing the git commit id of the codebase used to train and pickle the model.

So, we would end up pickling an object similar to the one below:

```python
pickled_model = pickle.dumps(model)
model_and_details = {
    "model": pickled_model,
    "versions": {"model_library": "1.1.1"},
    "commit_id": "commit_id_value",
}
with open("model.pkl", "wb") as handle:
    pickle.dump(model_and_details, handle)
```

By going through these simple extra steps it should become less painful to manage models stored through the pickling.

Time taken to write post: approximately 2.5 hours
