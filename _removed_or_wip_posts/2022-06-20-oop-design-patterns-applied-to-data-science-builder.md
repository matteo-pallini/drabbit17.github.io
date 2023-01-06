---
layout: posts
comments: true
author_profile: true
title: "OOP design patterns applied in data science: Builder"
excerpt: Using the Builder pattern to make more maintainable and flexible creating complex objects
---

During this weekend I had the chance to attend [PyData London 2022 as a volunteer](https://pydata.org/london2022/). It was a fun experience, which gave me the chance to learn more about the PyData internals, and also to attend a bunch of talks. There were a few talks about applying software engineering and OOP principles when writing data science related code. The presence of these talks combined with how trendy MLOps is nowadays shows how much this market is now focusing more and more on maintainability, which is great.

## Clean Architecture
Regarding OOP principles I attended the ["Clean Architecture" talk](https://laszlo.substack.com/p/slides-for-my-talk-at-pydata-london) from Laszlo Sragner. I particularly enjoyed how much emphasis he put in how applying OOP design patterns can improve code maintainability also in data science pipelines.

In one of his examples he replaced a pandas dataframe (`df = self.source.get_data()`) returned by a data loading function with a domain specific abstraction (a list of `Customer`). I don't want to put words in his mouth regarding the reason behind this change, so I suggest anyone interested to watch his talk (whenever it is available). Below you can find the original script and the modified version.

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

Modified example:
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

Generally speaking, I also prefer using domain specific abstractions (like `Customer`) rather than constructs unrelated to the domain (like a pandas df) whenever possible. I also don't love using pandas in production code. However, I feel that it's not always easy to renounce to the performance that this brings to the table. In this regard Laszlo suggested Numba as a performing alternative to pandas. Even though I don't have extensive experience with it, I feel that given how well known pandas is at this point it's worth being able to use it. Particularly in not very mature companies/start-ups where data scientist are supposed to contribute to production code. 

So, now the question is, how to keep pandas in the game while keeping your code maintainable and decoupled?

## The Builder pattern
A single blog post wouldn't be enough to answer that question. Also, probably there are multiple acceptable answers, and none is really perfect. Still I will try to use the Builder pattern to keep domain specific abstractions in our code example, but also support using pandas to do computations. Specifically, to build some features. The use of the Builder pattern to do so is the result of a suggestion coming from the public during the talk. Also, as usual I am not going to go in detail on how the Builder pattern works, the gang of 4 book and many other online sources would do a much better job than me in this regard.

So, as per the gang of 4 book the Builder pattern intent is:
> Separate the construction of a complex object from its representation so that the same construction process can create different representations.

So, to apply the general statement above to this example. We want to be able to create an arbitrary number of configurations of complex objects, ie different combinations of features, while keeping how we create them, ie possibly through a pandas dataframe, decoupled from how we represent them, ie possibly not through a dataframe but instead a domain related abstraction.

So, let's say that starting from our `customers` abstraction we want to build two features, the number of days passed since the sign-up date and the number of products purchased. We could simply write a couple of pure python function or pandas operations to do so, and simply wrap them in a `add_features` function to do, an then add new features when necessary. There are a few problem with this, first by adding the new functions in the wrapper we would violate the `open for extension, closed to modification` principle. Secondly, we may want to be able to add features both using pure python and pandas, so we would need to have an if-else in the function, which again violates the above mention principle and makes the code harder to maintain/read.

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
Then we can create the two concrete classes for the features described above.
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
Now that we have some logic to create a few features we can now write two Directors. One will create a features set using only pure python, and another one will do the same thing only using pandas. Through the second director we can also decide whether to return a pandas dataframe as result, or a list of dictionaries, which can then be used to instantiate whatever domain abstraction is considered more appropriate.
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
    
    def compute_features(self, data: List[Customer]) -> Union[List[Dict[str, Any]], pd.DataFrame]:
        df = pd.DataFrame([dataclasses.asdict(val) for val in data])
        for feature in self._features:
            df = feature.compute_as_df(df)
        if self._return_df:
            return df
        else:
            return df.to_dict("records")


class FeaturePythonComputer(FeaturesComputer):
    def compute_features(self, data: List[Customer]) -> Union[List[Dict[str, Any]], pd.DataFrame]:
        data = [val.asdict() for val in data]
        for feature in self._features:
            data = feature.compute_as_list(data)
        return data
```

## How it fits all together
So, finally we can combine our newly created builders and directors with the original pipeline to create a feature set with an arbitrary number of features, whose complexity is isolated at individual feature level. Also, the way these are computed is decoupled from the final object representation. We can easily create a new director that returns a domain related abstraction rather than a list of dictionaries, or a dataframe as output.
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




