---
layout: posts
comments: true
author_profile: true
title: "OOP design patterns applied in data science: Strategy"
excerpt: Use case of the Strategy pattern in a data science application
---

Recently I had the chance of reading parts of *[Design Patterns: Elements of Reusable Object-Oriented Software](https://en.wikipedia.org/wiki/Design_Patterns)* (AKA the book by the gang of four). I really enjoyed the book, the first 2 chapters in particular.

While working in the data science/ML space I picked up some software engineering practices. These made it way easier to improve output quality, reproducibility, and maintainability. I mostly use python, so eventually, OOP design patterns become the next tool I wanted to add to my toolbox. The goal of the series is twofold, the first is getting more familiar with the patterns, while the second is showing how data science/ML practitioners can benefit from broadly adopting software engineering practices, design patterns in this case.

I will try to go over as many design patterns as possible by finding one data science application that could benefit from applying them. I am not going to explain each pattern in detail, I will simply quote its *Intent,* as per book definition, so that it can be used as a refresher. The book goes in-depth for each one of them, so the interested reader should probably consider reading it.

## Strategy Pattern - data preprocessing

> Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

Let's consider data preprocessing as an potential area in which we can use this pattern. More specifically let's say I have a pandas series that is provided as an input (or an array) and I need to transform it. I may need to handle missing values, outliers and then apply some transformation, such as rescaling the feature.

As usual, I am really really excited about data preparation, so I quickly write down a class and have all the data processing logic living in the class itself. I can specify the processing behaviour by passing strings or booleans as arguments at instantiation time.

```python
import pandas as pd
import numpy as np


class Preprocessor:

    def __init__(
    	self, transformation=None, drop_nans=False, 
	nan_filling=None, min_val=None, max_value=None
	):
        self._drop_nans = drop_nans
        self._nan_fillings = nan_fillings
        self._min_val = min_val
        self._max_val = max_val
	 self._transformation = transformation

    def process_series(series):
        series = self._handle_nans(series)
        series = self._handle_outliers(series)
        if self._transformation == "standardize":
            series = self._standardize(series)
        elif self._transformation == "min_max":
            series = self._min_max_normalize(series)
        else:
            pass
        return series

    def _standardize(self, series):
        return (series - series.mean())/series.std()

    def _min_max_normalize(self, series):
        return (series - series.min())/(series.max() - series.min())

    def _handle_nans(self, series):
        if self.drop_nans:
            return series[series.notnull()]
        elif self.nan_filling:
            return series.fillna(nan_filling)
        else:
            logging.warning("you have nans in your series")
            return series

    def _handle_outliers(self, series):
        """clip will ignore Nones, amazing!"""
        return series.clip(self.min_val, self.max_val)
```

After having tested it I am pretty happy with that. It took little time to write it down, which makes me feel the best. I haven't realized yet that every time I will want to add a new transformation or change the way the data is being cleaned I will need to modify the class.

Who cares, you may say. 

Let's say that after few days I realize I need to work with a categorical series. No biggie, I just need to add another method for one-hot encoding.

```python
@staticmethod
def _one_hot_encoding(series):
    return pd.get_dummies(series)
```

Oh men, I also need to add a function to apply the box-cox transformation so that my continuous feature is normally distributed no what was its original distribution.



Every time I add a transformation I need to add another step to the if-else in process_series, which forces me to violate the Open-Closed principle. Which states that Objects should be open to extension but closed to modification.

You may not care about applying principles, you may think that they are just applied for the sake of it. Still, eventually, the class will become pretty big, the if-else in particular, and require some refactoring, and I guess that at that point you are likely to go back to SOLID principles and use them as a guide. So, you are just kicking the can down the road.

Also, whoever is not familiar with the codebase (this may include your future self) may end up having to read a good part of the class before being able to understand where he can find what he is looking for and do whatever change is necessary.

Finally, the when instantiating `Preprocessor` the user needs to remember the correct name of the transformation he wants to use.

What if we could extract the whole transformation logic and only pass to the preprocessor the transformation needed?!?! Ding ding ding you won the strategy pattern prize!

```python
import abc


class Strategy(abc.ABC):

    @abc.abstractclassmethod
    def transform(cls, series):
        raise NotImplementedError


class Preprocessor:

    def __init__(
    self, transformation: Optional[Strategy]=None, drop_nans: bool=False, 
    nan_filling: Any=None, min_val: Optional[Union[float, int]]=None, 
    max_value: Optional[Union[float, int]]=None
    ):
        self._drop_nans = drop_nans
        self._nan_fillings = nan_fillings
        self._min_val = min_val
        self._max_val = max_val
	 self._transformation = transformation

    def process_series(series):
        series = self._handle_nans(series)
        series = self._handle_outliers(series)
        if self._transformation
            return self._transformation.transform(series)
        else:
            return IdentityStrategy.transform(series)

    def _handle_nans(self, series):
        if self.drop_nans:
            return series[series.notnull()]
        elif self.nan_filling:
            return series.fillna(nan_filling)
        else:
            logging.warning("you have nans in your series")
            return series

    def _handle_outliers(self, series):
        """clip will ignore Nones, amazing!"""
        return series.clip(self.min_val, self.max_val)



class IdentityStrategy(Strategy):
    @classmethod
    def transform(cls, series):
        return series.to_frame()


class StandardizeStrategy(Strategy):
    @classmethod
    def transform(cls, series):
        return ((series - series.mean())/series.std()).to_frame()

class MinMaxStrategy(Strategy):
    @classmethod
    def transform(cls, series):
        return (series - series.min())/(series.max() - series.min()).to_frame()

class OneHotEncodingStrategy(Strategy):
    @classmethod
    def transform(cls, series):
        return pd.get_dummies(series)
```

This change leads to several advantages:

- No need to modify the Preprocessor class when you need to add a new transformation or remove an unused one
- Ability to re-use the transformations in other contexts
- Make the code much easier to unit-test
- Make the code much easier to read

If you have some experience with scikit-learn transformers you probably felt straight away that what I was doing in the first code block was somehow wrong. At the same time, you may have been perfectly fine with the logic to nan fill and remove outliers.

However, Soon enough you may end up having so many cleaning functions or slight variations that you will have similar problems to the ones we had with the transformers, although it may feel less natural to do a change like the one below (which is probably overkilling in this context).

```python
import abc
import functools
from typing import Optional, Any, List, Callable


class Strategy(abc.ABC):
    
    @abc.abstractclassmethod
    def transform(cls, series):
        raise NotImplementedError
        

class DataProcessor:
    
    def __init__(
    self, transformation: Optional[Strategy]=None, 
    cleaning_functions: Optional[List[Callable]]= None
    ):
        self._transformation = transformation
        self._cleaning_functions = cleaning_functions
        
    def process_series(self, series):
        for cleaner in self._cleaning_functions:
            series = cleaner(series)
        if self._transformation:
            return self._transformation.transform(series)
        else:
            return IdentityStrategy.transform(series)
        

def drop_nans(series):
    return series[series.notnull()]

def nanfill(series, value):
    return series.fillna(value)
        
def handle_outliers(series, min_val, max_val):
    """clip will ignore Nones, amazing!"""
    return series.clip(min_val, max_val)

class IdentityStrategy(Strategy):
    @classmethod    
    def transform(cls, series):
        return series.to_frame()


class StandardizeStrategy(Strategy):
    @classmethod    
    def transform(cls, series):
        return ((series - series.mean())/series.std()).to_frame()

class MinMaxStrategy(Strategy):
    @classmethod
    def transform(cls, series):
        return (series - series.min())/(series.max() - series.min()).to_frame()

class OneHotEncodingStrategy(Strategy):
    @classmethod
    def transform(cls, series):
        return pd.DataFrame([[int(cat is e) for cat in set(categories)] for e in categories], columns=set(categories))
    
    
if __name__ == "__main__":
    data = pd.Series([1., 2., 4., 7., 100., np.nan])
    data_processor = DataProcessor(
        preprocessor=StandardizeStrategy(), 
        cleaning_functions=[
                     functools.partial(nanfill, value=5),
                     functools.partial(handle_outliers, min_val=0, max_val=10)
                 ])
    data_processor.process_series(data)
```

Time taken to write post: 4 hours
