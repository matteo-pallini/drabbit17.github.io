---
layout: posts
comments: true
author_profile: true
title: "OOP design patterns applied in data science: Builder"
excerpt: Using the Builder pattern to make more maintainable and flexible creating complex objects
---

During the last weekend I attended [PyData London 2022 as a volunteer](https://pydata.org/london2022/). It was a fun experience, which gave me the chance to learn more about the PyData org, and also to attend a bunch of talks. There were several talks related to applying software engineering and OOP principles when writing data science related code. The presence of these talks combined with how trendy MLOps is nowadays shows how much this market is now focusing more and more on maintainability rather than fancy new algorithms, which is great.

## Clean Architectureh
The ["Clean Architecture" talk](https://laszlo.substack.com/p/slides-for-my-talk-at-pydata-london) from Laszlo Sragner touched some OOP principles. I particularly enjoyed how much emphasis he put in how applying OOP design patterns can improve code maintainability also in data science.

In one of his examples, meant to show the factory pattern, he replaced a pandas dataframe (`df = self.source.get_data()`) returned by a data loading function with a domain specific abstraction (a list of `Customer`). I don't want to put words in his mouth regarding the reasons behind this change, so I suggest anyone interested to watch his talk (whenever it is available). This blog post is an attempt to further build on top of that example and try to keep domain relevant abstractions but also pandas constructs and leverage their performance. 

In order to set the ground it may be worth quickly reading Laszlo's original scripts, below. You can also find them on slide 19 of his [slide deck](https://laszlo.substack.com/p/slides-for-my-talk-at-pydata-london).

Original example:
```python
class SqlSource:
    def __init__(self):
        self.query = "SELECT * FROM tbl_customers"
        self.engine = create_engine(SQL_CONNECTION_ENGINE)

    def get_data(self):
        return pd.read_sql(self.query, self.engine)


class Main:
    def __init__(self, source):
        self.source = source

    def run(self):
        df = self.source.get_data()


if __name__ == "__main__":
    app = Main(source=SqlSource())
    app.run()
```

Improved example:
```python
class Customer:
    ...


class CustomerSource:

    def __init__(self, query=None):
        ...

    def get_customers(self):
        return [Customer.from_tuple(value) 
			for value in pd.read_sql(...).itertuples()]


class Main:
    ...
    def run(self):
        customers = self.source.get_customers()


if __name__ == "__main__":
    app = Main(source=CustomerSource())
    app.run()
```

Generally speaking, I also prefer using domain specific abstractions (like `Customer`) rather than constructs unrelated to the domain (like a pandas df, or a list, etc) whenever possible. I also don't love using pandas in production code. However, I feel that it's not always easy to renounce to its performance. In this regard Laszlo suggested Numba as a performing alternative to pandas. Still, I believe that given how well known pandas APIs are it may still be worth trying to find some decent way of using it in production code. Probably this is even more true within not very mature companies/start-ups where data scientist are also contributing extensively to production code. 

So, now the question is, how to keep pandas in the game while keeping your code maintainable and decoupled?

## The Builder pattern
I am sure that there are multiple acceptable answers to this question, and probably none is perfect. Ideally this blog post is going to be one of them. I will try to use the Builder pattern to keep domain specific abstractions in our code example, but also support pandas use to do computations. Specifically, to compute some features. The use of the Builder pattern to do so is the result of a suggestion coming from the public during the talk. Also, I am not going to go in detail on how the Builder pattern works, the gang of 4 book and many other online sources can explain it much better than me.

As per the gang of 4 book the Builder pattern intent is:
> Separate the construction of a complex object from its representation so that the same construction process can create different representations.

So, to let's try to translate that general statement to this example. We want to be able to create an arbitrary number of representations, ie different combinations of features in this case. But, we want to keep how we create them, ie possibly through pandas vectorized operations, decoupled from how we represent them, ie not through a pandas dataframe, but maybe using a domain related abstraction or pure python constructs.

So, let's say that starting from our `customers` abstraction we want to build two features, the number of days passed since the sign-up date and the number of products purchased. We could simply write a couple of pure python function or pandas operations to do so, and simply wrap them in an `add_features` function to do so, and then add new feature functions to this when necessary. There are a few problem with this, first by adding the new functions in the wrapper we would violate the `open for extension, closed to modification` principle. Second, we may want to be able to add features both using pure python and pandas, so we would need to have (at least) an if-else in the function, which again violates the above mention principle and makes the code harder to maintain/read.

## The Builders
Let's instead start by writing our Builders, which will take care of encapsulating the logic needed to create a specific feature and support both pure python and pandas. I am going to start by writing an abstract class defining their interfaces:
```python
class Feature(abc.ABC):
    """Abstract class defining our interfaces for our builder pattern implementation"""

    FEATURE_NAME = "abstract_feature"

    def __init__(self, input_data_field: str, feature_name: Optional[str] = None):
        self._feature_name = feature_name
        self._input_data_field = input_data_field

    @abc.abstractmethod
    def compute_as_list(self, data: List[Any]) -> List[Any]:
        pass

    @abc.abstractmethod
    def compute_as_df(self, data: pd.DataFrame) -> pd.DataFrame:
        pass

    @property
    def feature_name(self):
        return self._feature_name
```
Then we can create the two concrete builder classes for the features described above.
```
class NumberOfPurchasesFeature(Feature):
    FEATURE_NAME = "number_of_purchases"

    def compute_as_list(self, data: List[Dict]) -> List[Any]:
        for record in data:
            record[self.feature_name] = len(record[self._input_data_field])
        return data

    def compute_as_df(self, data: pd.DataFrame) -> pd.DataFrame:
        data[self.feature_name] = data[self._input_data_field].len()
        return data


class DaysFromSignupDatetimeFeature(Feature):
    FEATURE_NAME = "days_from_signup"

    def __init__(self, input_data_field, feature_name: Optional[str] = None):
        super().__init__(input_data_field=input_data_field, feature_name=feature_name)
        self._current_date = datetime.datetime.today()

    def compute_as_list(self, data: List[Any]) -> List[Any]:
        for record in data:
            record[self.feature_name] = (
                self._current_date - record[self._input_data_field]
            ).days
        return data

    def compute_as_df(self, data: pd.DataFrame):
        data[self.feature_name] = (
            self._current_date - data[self._input_data_field]
        ).dt.days
        return data
```
## The Directors
Now that we have some logic to create a few features we can also write two Directors. One will create a features set using only pure python, and another one will do the same thing but only using pandas. Through the second director we can also decide whether to return a pandas dataframe as result, or a list of dictionaries, which can then be used to instantiate whatever domain abstraction is considered more appropriate. I didn't define any domain specific abstraction for the final output here, but it should be easy to do so if needed.
```python
class FeaturesComputer(abc.ABC):
    """Abstract class defining the interfaces of our Director pattern implementation"""

    def __init__(self, features: List[Feature]) -> None:
        self._features = features

    @abc.abstractclassmethod
    def compute_features(
        self, data: List[Customer]
    ) -> Union[List[Dict[str, Any]], pd.DataFrame]:
        pass


class FeaturesPandasComputer(FeaturesComputer):

    def __init__(self, features: List[Feature], return_df=False) -> None:
        super().__init__(features)
        self._return_df = return_df
    
    def compute_features(self, data: List[Customer]) -> pd.DataFrame:
        df = pd.DataFrame([dataclasses.asdict(val) for val in data])
        for feature in self._features:
            df = feature.compute_as_df(df)
        if self._return_df:
            return df
        else:
            return df.to_dict("records")


class FeaturePythonComputer(FeaturesComputer):
    def compute_features(self, data: List[Customer]) -> List[Dict[str, Any]]:
        data = [val.asdict() for val in data]
        for feature in self._features:
            data = feature.compute_as_df(data)
        return data
```

## How it fits all together
So, finally we can combine our newly created builders and directors with the original pipeline to create a feature set with an arbitrary number of features, whose complexity is isolated at individual feature level. Also, the way these are computed is decoupled from the final object representation. We can easily modify the directors so that return a domain related abstraction rather than a list of dictionaries, or a dataframe as output.
```python
class Main:
    def __init__(self, source, features_computer: FeaturesComputer):
        self._source = source
        self._features_computer = features_computer

    def run(self):
        customers = self._source.get_customers()
        data_with_features = self._features_computer.compute_features(data=customers)
        ...


if __name__ == "__main__":
    features_computer_df = FeaturesPandasComputer(
        features=[
            NumberOfPurchasesFeature(input_data_field="list_of_purchases"),
            DaysFromSignupDatetimeFeature(input_data_field="signup_datetime"),
        ]
    )
    app = Main(source=CustomerSource(), features_computer=features_computer_df)
    app.run()
```

Time taken to write post: 6 hours




