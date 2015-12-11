# Auditing (Enterprise only)

Studio 2.2 includes a new functionality called [Auditing](Auditing.md). To understand how Auditing works, please read the [Auditing](https://github.com/orientechnologies/orientdb-docs/blob/master/Auditing.md) page on the [OrientDB Manual](http://orientdb.com/docs/last/index.html).

Studio Auditing panel helps on Auditing configuration, avoiding to edit the `auditing-config.json` file.

![](images/studio-auditing-configuration.png)

By default all the auditing logs are saved as documents of class `AuditingLog`. If your account has enough priviledges, can directly query the auditing log. Example on retrieving last 20 logs: `select from AuditingLog order by @rid desc limit 20`. 

However, Studio provides a panel to filter the Auditing Log messages without using SQL.

![](images/studio-auditing-log.png)

