---
order: 7
title: "Performance and Cost Optimization"
---

We've created our dashboard with a date filters in previous parts. In this
part we're going to work on performance and cost optimization of our queries.

Athena is great at handling large datasets, but will never give you a sub-second response, even on small datasets. As we saw previously, it leads to a wait time on dashboards and charts, especially dynamic, where users can select different date ranges or change filters.

To solve that issue we'll use Cube.js external pre-aggregations. We'll still leverage Athena's power to process large datasets, but will put all aggregated data into MySQL. Cube.js manages all the process of building and maintaining the pre-aggregations, including refreshes and partitioning.

The schema below shows the setup for Cube.js with Athena as main data source and MySQL for pre-aggregations.

SCHEMA

### Connecting to MySQL

To use the external pre-aggregations feature, we need to configure Cube.js to connect to both Athena and MySQL, as well as specify which pre-aggregation we want to build externally. We've already configured the connection to Athena, so all we need to setup now is MySQL connection.

First, we need to install Cube.js MySQL driver. Run the following command in the root folder of your project.

```bash
$ npm install --save @cubejs-backend/mysql-driver
```

Next, let's edit our `index.js` file in the root folder of the project.
Replace its content with the following.

```js
const CubejsServer = require("@cubejs-backend/server");
const MySQLDriver = require('@cubejs-backend/mysql-driver');

const server = new CubejsServer({
  externalDbType: 'mysql',
  externalDriverFactory: () => new MySQLDriver({
    host: process.env.CUBEJS_EXT_DB_HOST,
    database: process.env.CUBEJS_EXT_DB_NAME,
    user: process.env.CUBEJS_EXT_DB_USER,
    password: process.env.CUBEJS_EXT_DB_PASS.toString()
  })
});

server.listen().then(({ port }) => {
  console.log(`🚀 Cube.js server is listening on ${port}`);
});
```

Lastly, we're going to add credentials to connect to MySQL to the `.env` file. Please note, that in order to build pre-aggregations inside MySQL, Cube.js should have write access to the `stb_pre_aggregations` schema where pre-aggregation tables will be stored.

```bash
CUBEJS_EXT_DB_NAME=preags
CUBEJS_EXT_DB_HOST=localhost
CUBEJS_EXT_DB_USER=root
CUBEJS_EXT_DB_PASS=12345
```

That is all we need to let Cube.js connect to MySQL. Now, we can move forward and start defining pre-aggregations insides our data schema.

### Defining Pre-Aggregations in the Data Schema

The main idea of the pre-aggregation is to create a table with already aggregated data, which is going to be much smaller than the original table with the raw data. Querying such table is much faster that querying the raw data. Additionally, by inserting this table into external database, like MySQL, we'll be able to horizontally scale it, which is especially important in multi-tenant environments.

Cube.js can create and maintain such tables. To instruct it to do that we need to
define what measures and dimensions we want to pre-aggregate in the data schema.
The pre-aggregations are defined inside the `preAggregations` block. Let's
define the first simple pre-aggregation first and then take a closer look how it
works.

Inside the `Sessions` cube in the data schema add the following block.

```js
preAggregations: {
  additive: {
    type: `rollup`,
    measureReferences: [count],
    timeDimensionReference: timestamp,
    granularity: `day`,
    refreshKey: {
      every: `5 minutes`
    },
    external: true
  }
}
```

The code above will instruct Cube.js to create the pre-aggregation called
`additive` with two columns: `Sessions.count` and `Sessions.timestamp` with
daily granularity. The resulting table will look like the one below.

```
+-------------------------+-----------------+
| sessions__timestamp_day | sessions__count |
+-------------------------+-----------------+
| 2020-01-19 16:00:00     |               2 |
| 2020-01-20 16:00:00     |              71 |
| 2020-01-21 16:00:00     |             699 |
| 2020-01-22 16:00:00     |             608 |
| 2020-01-23 16:00:00     |             374 |
| 2020-01-24 16:00:00     |             139 |
| 2020-01-25 16:00:00     |              86 |
| 2020-01-26 16:00:00     |             128 |
| 2020-01-27 16:00:00     |             143 |
| 2020-01-28 16:00:00     |             123 |
+-------------------------+-----------------+
```

Also, note that we specify `external: true` property, which tells Cube.js to load that
table into MySQL, instead of keeping it inside Athena.

The `refreshKey` property defines how Cube.js should refresh that table. In our
case, the refresh strategy is quite simple, we just configure that
pre-aggregation to refresh every 5 minute. Refresh strategy can be much
complicated depending on the required use case, you can [learn more about it in
the docs](https://cube.dev/docs/caching#pre-aggregations-refresh-strategy).


