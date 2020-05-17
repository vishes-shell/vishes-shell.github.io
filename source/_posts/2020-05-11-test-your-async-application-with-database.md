---
layout: post
title: Test your async application with database
description: With rise of python async support with asyncio library new async frameworks/libraries/clients started to pop out and grow it's popularities. More and more people went to try async features and enter this magical world where everying does somethinf while waiting for IO guys.
tags:
- async
- tests
- databases
- python
toc: true
---

With rise of python *async* support with [`asyncio`](https://docs.python.org/3/library/asyncio.html)
library new async frameworks/libraries/clients started to pop out and grow their
popularities. More and more people want to try _async_ features and enter this
magical world where everything does something useful while waiting for _IO guys_.

And you are one of those who want to do more than making
[`asyncio.sleep(n)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep)
and be proud of how cool it is to be on the edge of technology. Putting your
program to sleep is easy, but you are not stopping there, you want HTTP requests,
databases and other real world scenarios to solve, also you want to test this.
That's what this article is for, to _find a way to test your async application
that has some connections to database_.


## Purpose

Here is a list of testing rules (one of many): [Testing Your Code](https://docs.python-guide.org/writing/tests/)

And the part that we want to cover:

> Each test unit must be fully independent. Each test must be able to run alone,
and also within the test suite, regardless of the order that they are called.
The implication of this rule is that each test must be loaded with a fresh
dataset and may have to do some cleanup afterwards.

Overall: we want to work with database in tests, but want it to be isolated and
clean everything after each test ran.


## Ways to keep db fresh

 1. Run everything in transaction and then roll them back (e.g.
    [`django` test ecosystem](https://docs.djangoproject.com/en/dev/topics/testing/overview/#testcase)
    does so)

    *Thoughts*: Would be perfect

 1. Drop database after every test (this sure keep database clean)

    *Thoughts*: Not good (heavy)

 1. Delete data in all tables

    *Thoughts*: Fast but what would you do if you have lot of tables?

 1. Truncate data in all tables

    *Thoughts*: Faster than deleting, however you still need to go through all
    tables, and `sqlalchemy` does not have method for that, but who cares.

 1. Unapply and then reapply migrations (if you have them, maybe
    [`alembic`](https://alembic.sqlalchemy.org/en/latest/))

    *Thoughts*: Same as droping database, not good


The best way is to put tests in transactions and roll them after test finishes.


## Tools

For *testing* purposes we would use awesome
[`pytest`](https://docs.pytest.org/en/latest/). I won't dive in to how good
`pytest` is, there is already documentation, lots of articles, conference talks
and stackoverflow questions. But if you are still loyal to `unittest` i'm not
quite sure that this article would be for you.

Async *database client* would be
[`databases`](https://github.com/encode/databases), because it's ready to use
and [Tom Christie](http://www.tomchristie.com/)
([creator](https://github.com/encode/databases/commit/388f163096c9f4e274b57523b18b6c59ff33b306))
knows the way to build a beautiful software.

For `databases` we would use `sqlite` since it's so easy to use, you can just
copy and paste lines (_isn't that the one that we love the most?_).


## Project

For testing purpose Note model will be used:

```python
#  app/models.py
import sqlalchemy as sa

metadata = sa.MetaData()

notes = sa.Table(
    "notes",
    metadata,
    sa.Column("id", sa.Integer, primary_key=True),
    sa.Column("text", sa.String),
    sa.Column("completed", sa.Boolean),
)
```


## Tests setup

We create _session_ scope fixture that creates our database with defined tables
and deletes after test run is completed:

```python
#  tests/conftest.py
import pytest
import sqlalchemy as sa
from app.models import metadata
from databases import Database

SQLITE_DATABASE_URL = "sqlite:///./test.db"


@pytest.fixture(scope="session")
def sqlite_db() -> Database:
    engine = sa.create_engine(SQLITE_DATABASE_URL)
    metadata.create_all(engine)

    yield Database(SQLITE_DATABASE_URL)

    metadata.drop_all(engine)
```

## Implementation

Note: _For running async tests (they should be async since you are dealing with
async code) you would require
[`pytest-asyncio`](https://github.com/pytest-dev/pytest-asyncio) and decorate
your test with `@pytest.mark.asyncio` or apply asyncio mark to all tests with:_

```python
pytestmark = pytest.mark.asyncio
```

Another way to test async is to create your own decorator as in `databases`:
[`async_adapter`](https://github.com/encode/databases/blob/master/tests/test_databases.py#L100-L111):

```python
def async_adapter(wrapped_func):
    """
    Decorator used to run async test cases.
    """

    @functools.wraps(wrapped_func)
    def run_sync(*args, **kwargs):
        loop = asyncio.new_event_loop()
        task = wrapped_func(*args, **kwargs)
        return loop.run_until_complete(task)

    return run_sync
```


### Test function

Define generic function that inserts data and check if it's there.
We use `pytest.fail` since we want to use this function to be used by every
test.

```python
async def main(sqlite_db):
    """Insert data and check that it exist in database
    """
    query = notes.insert().values(text="text", completed=False)
    await sqlite_db.execute(query)

    query = notes.select()
    if not await sqlite_db.fetch_all(query):
        pytest.fail("no data in database")
```


### Check that database is clean after test

We can achieve that with fixture that run after test completes:

```python
@pytest.fixture
async def check_empty_database(sqlite_db):
    """All tests use transaction and database clean after tests
    """
    yield

    query = notes.select()
    data = await sqlite_db.fetch_all(query)
    if data:
        pytest.fail("db data should be rolled back")
```


### Rollback transaction fixture

The most obvious way is to implement transactional fixture with force rollback
transaction context manager as this:

```python
@pytest.fixture
async def rollback_transaction(sqlite_db):
    async with sqlite_db.transaction(force_rollback=True):
        yield
```

And to test this:

```python
@pytest.mark.asyncio
async def test_rollback_transaction_fixture_and_asyncio_mark(
    sqlite_db, rollback_transaction, check_empty_database
):
    await main(sqlite_db)
```

```console
$ pytest -k test_rollback_transaction_fixture_and_asyncio_mark
tests/test_databases_sqlite.py .E                               [100%]

=========================== ERRORS ===================================
__ ERROR at teardown of test_rollback_transaction_fixture_and_asyncio_mark __

sqlite_db = <databases.core.Database object at 0x10736d640>

    @pytest.fixture
    async def check_empty_database(sqlite_db):
        """All tests use transaction and database clean after tests
        """
        yield

        query = notes.select()
        data = await sqlite_db.fetch_all(query)
        if data:
>           pytest.fail("db data should be rolled back")
E           Failed: db data should be rolled back

tests/test_databases_sqlite.py:16: Failed
======================= short test summary info ======================
ERROR tests/test_databases_sqlite.py::test_rollback_transaction_fixture_and_asyncio_mark - Failed: db data should be rolled back
```

We can see that the test is passed, but due to `pytest` fixtures implementation
we cannot make them to fail test, only create an error. But in this case it's
enough to say that _data is not cleaned_.

But why is that? Maybe `force_rollback` transaction don't even work? We can try
and make sure that it's not our fault:

```python
@pytest.mark.asyncio
async def test_base_context_manager(sqlite_db, check_empty_database):
    """Transaction context manager with force_rollback
    rolls transaction after all
    """
    async with sqlite_db.transaction(force_rollback=True):
        await main(sqlite_db)
```

```console
$ pytest -k test_base_context_manager
======================= test session starts ==========================
platform darwin -- Python 3.8.2, pytest-5.4.2, py-1.8.1, pluggy-0.13.1
rootdir: /Users/vishes/Personal/github/pytest-asyncio-db-transactions
plugins: asyncio-0.12.0
collected 5 items / 4 deselected / 1 selected
tests/test_databases_sqlite.py .                                [100%]
```

Force rollback transaction works as expected, but why transaction is not rolled
back in fixture? The reason is that async fixtures are executed in it's own
context, so maybe transaction is rolled back, however not the one that we
wanted to.


### Any other ways?

Writing context manager in each test is not
[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) at all, however it
will get us where we want, but there is another way: *decorator*.

```python
import functools

def transaction_wrapper(func, rollback=True):
    @functools.wraps(func)
    async def wrapper(sqlite_db, *args, **kwargs):
        async with sqlite_db.transaction(force_rollback=rollback):
            return await func(sqlite_db, *args, **kwargs)

    return wrapper
```

And all we need to do is to decorate our test method:

```python
@transaction_wrapper
@pytest.mark.asyncio
async def test_transaction_wrapper(sqlite_db, check_empty_database):
    """Transaction wrapper with default rolling back
    rolls transaction after tests completes with pytest asyncio mark
    """
    await main(sqlite_db)
```

```console
$ pytest -k test_transaction_wrapper_and_async_adapter
======================== test session starts =========================
platform darwin -- Python 3.8.2, pytest-5.4.2, py-1.8.1, pluggy-0.13.1
rootdir: /Users/vishes/Personal/github/pytest-asyncio-db-transactions
plugins: asyncio-0.12.0
collected 5 items / 4 deselected / 1 selected

tests/test_databases_sqlite.py .                                [100%]
```


## Conclusion

We couldn't make database test setup through _pytest fixture_ (maybe even
with `autouse=True`), however **we've achieved tests with rolled back
transactions trough decorator**, which is quite good enough.

I hope someday we can all use async fixtures and by now _decoration_ is the one
of the cleanest methods at the moment.


Source code can be found here:
[vishes-shell/pytest-asyncio-db-transactions](https://github.com/vishes-shell/pytest-asyncio-db-transactions)
