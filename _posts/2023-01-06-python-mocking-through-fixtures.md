---
layout: posts
comments: true
author_profile: true
title: Patching python objects through pytest fixtures
excerpt: A simple piece on how to patch objects and use pytest fixtures in tests at the same time
---

**TL;DR**

*It is possible to patch objects and pass pytest fixtures to a test at the same time. It's enough to apply the patching through a fixture*

## Bugfixes while maintaining legacy code
Recently I had to do a bugfix on some legacy code written a while ago. As part of the bugfix I also wanted to add a test to make sure the issue was actually solved. The abstraction I had to test was enstablishing a connection to a service at instantiation time. 

Ideally the connection would have been enstablished lazily when it was needed, and also it would have been better to inject an instance of the connection client rather than have its creation baked into the constructor. Before considering any of these refactorings I had to write some tests though (and merge the bugfix!).

```python
# main.py
from typing import Any
import dataclasses


# for the sake of showing how this works I am going to replace the third party library client with a mock one that raises when instantiated
class Client:
    def __init__(self, connection_params: dict[str, Any]):
        raise Exception("You shall not instantiate me!")


@dataclasses.dataclass
class Params:
    a: int
    b: int
    c: int


class MainAbstraction:

    def __init__(self, connection_params: dict[str, Any]):
        self._client = Client(connection_params)

    def method_to_test(self, params: Params):
        return params.a ** 2
```

The simplest solution seemed to be to patch the `Client` and pass it to the test as an argument. However, I was also passing at least one resource needed for testing as a pytest fixture to the test, which made it seem as this approach was unfeasible. 

```python
# test_main.py
import pytest
from main import Params, MainAbstraction

@pytest.fixture
def params():
    return Params(a=3, b=10, c=1)

@pytest.fixture
def connection_params():
    return {"host": "foo", "port": 123}

def test_main(connection_params, params):
    abstraction = MainAbstraction(connection_params)
    actual = abstraction.method_to_test(params)
    assert actual == 9
```

## Patching through fixtures
Luckily first impressions are not always right. It is possible to create a patch for a specific test through a pytest fixture, then applying the patch is just a matter of passing the fixture as an argument to the test.

```python
# test_main.py
import pytest
from main import Params, MainAbstraction

@pytest.fixture
def params():
    return Params(a=3, b=10, c=1)

@pytest.fixture
def connection_params():
    return {"host": "foo", "port": 123}

@pytest.fixture
def mocked_client():
    with patch('main.Client', return_value=None) as m:
        yield m

def test_main(mocked_client, connection_params, params):
    abstraction = MainAbstraction(connection_params)
    actual = abstraction.method_to_test(params)
    assert actual == 9
```


Time taken to write post: 1 hour