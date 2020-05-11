During the last three months my team wrote a lot of integration and repository tests, due a couple of to new features.
Over time those tests became extremely slow.

It was such a pain, that we came up to think about how many of those tests we can cope, 
because a full test run took at least half an hour now.
## The environment
* loopback 3 with mysql-connector-module and mysql.js under the hood
* mocha, chai as test modules
* mariadb

So nothing special in a NodeJS world.

## We ran into a couple of problems
* The deployment pipelines get slower test by test in big steps. 
* The developers tend accept that slowness.
* Thinking about kicking some tests out.
* Usually servers providing CI-CD-pipelines have a certain amount of build minutes per month.
 So maybe the tests run into a quota.
* Running all tests before committing sucks a developers lifetime

# Steps to speed up the integration- and repository tests
Here is a collection of ideas, o got during my research
* [executing tests in parallel](https://medium.com/expedia-group-tech/do-you-want-to-speed-up-your-integration-tests-by-10x-eb047c72a252)
: not an option, because the database needs to be cleaned up after every single test
* [keep mysql data files in a RAM-disk](https://vladmihalcea.com/how-to-run-database-integration-tests-20-times-faster/)
: too tricky, because of different OS between developers

I focused on
1. tweaking database cleanups
2. the idea, running integration tests in chunks

## Tweaking Database Cleanups
### truncate vs delete
As already mentioned on other sites, ``truncate`` may be faster than ``delete``. 
In our situation ``delete`` was the better choice. 

If you want to read more about the possible reasons, please read the performance section on [truncate documentation at mariadb.com](https://mariadb.com/kb/en/truncate-table/)

### Reducing simultaneous db connections
Using Promises for every single table generates a full CPU-load by the mysql daemon, when running in parallel-tasks, caught by ``Promise.all()``.
```javascript
const deletionTasks = [];

tableModels.forEach((tableModel) => {
    promiseTasks.push(mySingleTableDeletion(tableModel))
});

return Promise.all(deletionTasks);
```
The daemon needs to deal 
* with multiple connections.
* foreign keys

``multipleStatements`` was set to enabled in the configuration just for the testing environment. A good solution is to create an extra database config.

In case of integration tests setting this flag allowed me to write one single empty-database-query block resetting all tables.
```sql
DELETE QUICK FROM table1; ALTER TABLE table1 AUTO_INCREMENT = 1;
DELETE QUICK FROM table2; ALTER TABLE table2 AUTO_INCREMENT = 1;
...
```
__Attention:__ Please *don't* enable multipleStatements in the default db config: 
It opens the doors for mysql injections! 
It has a good reason, why it's disabled by default in several frameworks.

With ``"multipleStatements": "on"`` the CPU-load decreased to the pure script execution.
* Sending a single query for each table opens a lot of connections, the mysql daemon have to deal with.
* Sending one query per table uses just one connection at all.

### Disable foreign_key_checks while emptying
Since the auto increment is reset to 1 for every table, no data will be left over for sure. 
Mysql and MariaDB don't need to care about constraints
Finally, the sql query block got another little tweak:
```sql
SET foreign_key_checks = 0; ${deleteBlock} SET foreign_key_checks = 1;
```
### The benefit
Allowing multipleStatements reduced the cpu load of the mysql daemon from over 100% to less than 20%.

## Run integration tests in chunks
I would like to show you, what I recognized, when I came up with the idea if chunking integration tests.

### Speed slows down from test to test
As mentioned above, after writing a lot of tests over tine, they now take more than a half hour. 

It was always the same experience, regardless using junit in java in old days or mocha in a NodeJS project now.
The speed of the first integration tests from the first 2-3 test-files are lighting fast (for an integration test),
but after the first 20 - 30 tests the runner begins to slow down remarkably. 

#### Assumptions
* __It's the database!__ No! It's not! the next 30 Tests should have a similar speed than the first 30.
And the database-bottleneck is eliminated.
* __The tests are asynchronous!__ Well that would be fatal, because one test would manipulate data from the other test concurrently. Luckily mocha executes tests one-by-one.

It needs to be something different.

#### Chunking integration tests
In the beginning of the tests the server is started once with a test db.

Since the speed is decreasing while testing, the server has to be restarted 
* from time to time 
* or every time.

Restarting the Server for every singe test means a lot of boot- and shut down time. 

A good choice was to run integration tests per folder. In a project I'm currently working on, there are 5 folders representing a domain context.

Executing all tests in one run took 30+ min.

Starting integration tests per context just took 6 min in summary!

#### Test coverage
``nyc`` is used for test coverage. It collects data about coverage as long as it is running. 

```json
{
    "scripts": {
        "test": "npm run test:unit && npm run test:integration && npm run test:acceptance",
        "test:coverage": "cross-env ./node_modules/.bin/nyc --reporter=text-summary --reporter=lcov npm run test",
        "test:integration": "cross-env npm run test:integration:context:context1 && npm run test:integration:context:context2 && npm run test:integration:context:context3 && ...",
        "test:integration:context:context1": "cross-env DB_CONNECTOR=mysql DB_SCHEMA=test-db NODE_ENV=test mocha --opts 'test-integration/mocha.opts' --recursive './test-integration/contexts/context1/**/*.spec.js'",
        "test:integration:context:context2": "cross-env ..."
    }
}
```
In this case ``nyc`` executes ``npm run test`` running a couple of test runners of test runners.

#### The benefit
Splitting the tests into chunks, in order to restart the server in beween, reduced the execution time from above 30 minutes to 6 minutes.

# Conclusions
* Using multiple statements in form of a single db-cleanup-query reduced CPU consumption.
* Chunking integration- and repository tests reduced local and deployment execution time. 
 
More research needs to be done. Integration tests should not slow down each other! 
