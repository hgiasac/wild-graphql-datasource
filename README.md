# Wild GraphQL Datasource

**This plugin is in early development. Breaking changes may happen at any time.**

This is a Grafana datasource that aims to make requesting time series data via a GraphQL endpoint easy.
This datasource is similar to https://github.com/fifemon/graphql-datasource, but is not compatible.
This datasource tries to reimagine how GraphQL queries should be made from Grafana.

Requests are made in the backend. Results are consistent between queries and alerting.


## Variables

### Provided Variables

Certain variables are provided to every query. These include:

| Variable      | Type   | Description                                                | Grafana counterpart                                                                                                |
|---------------|--------|------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `from`        | Number | Epoch milliseconds of the "from" time. Passed as a number. | [$__from](https://grafana.com/docs/grafana/latest/dashboards/variables/add-template-variables/#__from-and-__to)    |
| `to`          | Number | Epoch milliseconds of the "to" time. Passed as a number.   | [$__to](https://grafana.com/docs/grafana/latest/dashboards/variables/add-template-variables/#__from-and-__to)      |
| `interval_ms` | Number | Milliseconds of the interval. Equivalent to `to - from`.   | [$__interval_ms](https://grafana.com/docs/grafana/latest/dashboards/variables/add-template-variables/#__interval)  |

An example usage is shown in the most basic query:

```graphql
query ($from: Long!, $to: Long!) {
  queryStatus(from: $from, to: $to) {
    # ...
  }
}
```

In the above example, the query asks for two Longs, `$from` and `$to`.
The value is provided by the provided variables as seen in the above table.
Notice that while `interval_ms` is provided, we do not use it or define it anywhere in our query.
One thing to keep in mind for your own queries is the type accepted by the GraphQL server for a given variable.
In the case of that specific schema, the type of a `Long` is allowed to be a number or a string.
If your specific schema explicitly asks for a `String`, these variables may not work.
Please [raise and issue](https://github.com/wildmountainfarms/wild-graphql-datasource/issues) if this limitation becomes a roadblock.


### Query specific variables and Grafana variable interpolation

For each query you define, the query editor has a place for you to add variables.
This section is optional, but if you wish to provide constant variables for your query or [simplify the specification of input types](https://graphql.org/graphql-js/mutations-and-input-types/).
You may do so.

The variables section is the most useful for variable interpolation.
Any value inside a string, whether that string is nested inside an object, or a top-most value of the variables object, can be interpolated.
Please note that interpolation does not work for alerting queries.
You may use any configuration of [variables](https://grafana.com/docs/grafana/latest/dashboards/variables/add-template-variables/) you see fit.
An example is this:

```graphql
query ($sourceId: String!, $from: Long!, $to: Long!) {
  queryStatus(sourceId: $sourceId, from: $from, to: $to) {
    # ...
  }
}
```

```json
{
  "sourceId": "$sourceId"
}
```

Here, `$sourceId` inside of the variables section will be interpolated with a value defined in your Grafana dashboard.
`$sourceId` inside of the GraphQL query pane is a regular [variable](https://graphql.org/learn/queries/#variables) that is passed to the query.

NOTE: Interpolating the entirety of the JSON text is not supported at this time.
This means that interpolated variables cannot be passed as numbers and interpolated variables cannot define complex JSON objects.
One of the pros of this is simplicity, with the advantage of not having to worry about escaping your strings.

REMEMBER: Variable interpolation does not work for alerting queries, or any query that is executed without the frontend component.


## Uses for Timeseries data

### Using a field as the display name

If you have data that needs to be "grouped by" or "partitioned by", you first need to add "Partition by values"
as a transform and select `packet.identifier.representation` and `packet.identityInfo.displayName` as fields.
Once you do that, you can go into the "Standard options" and find "Display name" and set it to
`${__field.labels["packet.identityInfo.displayName"]}`.

References:

* [documentation for Display name under standard options](https://grafana.com/docs/grafana/latest/panels-visualizations/configure-standard-options/#display-name).
* [partition by values](https://grafana.com/docs/grafana/latest/panels-visualizations/query-transform-data/transform-data/)
  * Note: Partition by values was [added in 9.3](https://grafana.com/docs/grafana/latest/whatsnew/whats-new-in-v9-3/#new-transformation-partition-by-values) ([blog](https://grafana.com/blog/2022/11/29/grafana-9.3-release/))

## FAQ

* Can I use variable interpolation within the GraphQL query itself?
  * No, but you may use variable interpolation inside the values of the variables passed to the query
* Is this a drop-in replacement for [fifemon-graphql-datasource](https://grafana.com/grafana/plugins/fifemon-graphql-datasource/)?
  * No, but both data sources have similar goals and can be ported between with little effort.

## Common errors

### Alerting Specific Errors

* `Failed to evaluate queries and expressions: input data must be a wide series but got type long (input refid)`
  * This error indicates that the query returns more fields than just the time and the datapoint.
  * For alerts, the response from the GraphQL query cannot contain more than the time and datapoint. At this time, you cannot use other attributes from the result to filter the data.

## To-Do

* Add ability to have multiple parsing options
  * Add "advanced options" section that has "Partition by" and "Alias by"
    * Including these might be necessary as you may want to partition by and alias by different things for different parts of a GraphQL query
    * Advances options can also include the ability to add custom labels to the response - this allows different parsing options to be distinguished by Grafana
* Add support for secure variable data defined in the data source configuration
  * The variables defined here cannot be overridden for any request - this is for security
  * Also add support for secure HTTP headers
* See what minimum Grafana version we can support
* Add support for variables: https://grafana.com/developers/plugin-tools/create-a-plugin/extend-a-plugin/add-support-for-variables
* Add metrics to backend component: https://grafana.com/developers/plugin-tools/create-a-plugin/extend-a-plugin/add-logs-metrics-traces-for-backend-plugins#implement-metrics-in-your-plugin
* Support returning logs data: https://grafana.com/developers/plugin-tools/tutorials/build-a-logs-data-source-plugin
  * We could just add `"logs": true` to `plugin.json`, however we need to support the renaming of fields because sometimes the `body` or `timestamp` fields will be nested
* Publish as a plugin
  * https://grafana.com/developers/plugin-tools/publish-a-plugin/publish-a-plugin
  * https://grafana.com/developers/plugin-tools/publish-a-plugin/sign-a-plugin#generate-an-access-policy-token
  * https://grafana.com/legal/plugins/
  * https://grafana.com/developers/plugin-tools/publish-a-plugin/provide-test-environment
