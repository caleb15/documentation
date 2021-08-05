---
title: Setting Up Database Monitoring for Google Cloud SQL managed Postgres
kind: documentation
description: Install and configure Database Monitoring for Postgres managed on Google Cloud SQL.
further_reading:
- link: "/integrations/postgres/"
  tag: "Documentation"
  text: "Basic Postgres Integration"
---

{{< site-region region="us3,gov" >}}
<div class="alert alert-warning">Database Monitoring is not supported for this site.</div>
{{< /site-region >}}

Database Monitoring provides deep visibility into your Postgres databases by exposing query metrics, query samples, explain plans, database states, failovers, and events.

The Agent collects telemetry directly from the database by logging in as a read-only user. Do the following setup to enable Database Monitoring with your Postgres database:

1. [Configure database parameters](#configure-postgres-settings)
1. [Grant the Agent access to the database](#grant-the-agent-access)
1. [Install the Agent](#install-the-agent)
1. [Configure the Agent](#configure-the-agent)
1. [Install the Cloud SQL Integration](#install-the-cloud-sql-integration)

## Before you begin

Supported PostgreSQL versions
: 9.6, 10, 11, 12, 13

Supported Agent versions
: 7.30.0+

Performance impact
: The default Agent configuration for Database Monitoring is conservative, but you can adjust settings such as the collection interval and query sampling rate to better suit your needs. For most workloads, the Agent represents less than one percent of query execution time on the database and less than one percent of CPU. <br/><br/>
Database Monitoring runs as an integration on top of the base Agent ([see benchmarks][1]).

Proxies, load balancers, and connection poolers
: The Agent must connect directly to the host being monitored. For self-hosted databases, `127.0.0.1` or the socket is preferred. The Agent should not connect to the database through a proxy, load balancer, or connection pooler such as `pgbouncer`. While this can be an anti-pattern for client applications, each Agent must have knowledge of the underlying hostname and should stick to a single host for its lifetime, even in cases of failover. If the Datadog Agent connects to different hosts while it is running, the values of metrics will be incorrect.

Data security considerations
: See [Sensitive information][2] for information about what data the Agent collects from your databases and how to ensure it is secure.

## Configure Postgres settings

Configure the following [parameters][3] in [Database flags][4] and then **restart the server** for the settings to take effect. For more information about these parameters, see the [Postgres documentation][5].

| Parameter | Value | Description |
| --- | --- | --- |
| `shared_preload_libraries` | `pg_stat_statements` | Required for `postgresql.queries.*` metrics. Enables collection of query metrics via the the [pg_stat_statements][5] extension. |
| `track_activity_query_size` | `4096` | Required for collection of larger queries. Increases the size of SQL text in `pg_stat_activity` and `pg_stat_statements`. If left at the default value then queries longer than `1024` characters will not be collected. |
| `pg_stat_statements.track` | `ALL` | Optional. Enables tracking of statements within stored procedures and functions. |
| `pg_stat_statements.max` | `10000` | Optional. Increases the number of normalized queries tracked in `pg_stat_statements`. This setting is recommended for high-volume databases that see many different types of queries from many different clients. |


## Grant the Agent access

The Datadog Agent requires read-only access to the database server in order to collect statistics and queries.

Choose a PostgreSQL database on the database server to which the Agent will connect. The Agent can collect telemetry from all databases on the database server regardless of which one it connects to, so a good option is to use the default `postgres` database. Choose a different database only if you need the Agent to run [custom queries against data unique to that database][6].

Connect to the chosen database as a superuser (or another user with sufficient permissions). For example, if your chosen database is `postgres`, connect as the `postgres` user using [psql][7] by running:

 ```bash
 psql -h mydb.example.com -d postgres -U postgres
 ```

Run the following SQL commands:

{{< tabs >}}
{{% tab "Postgres ≥ 10" %}}

```SQL
CREATE USER datadog WITH password '<PASSWORD>';
CREATE SCHEMA datadog;
GRANT USAGE ON SCHEMA datadog TO datadog;
GRANT USAGE ON SCHEMA public TO datadog;
GRANT pg_monitor TO datadog;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

{{% /tab %}}
{{% tab "Postgres 9.6" %}}

```SQL
CREATE USER datadog WITH password '<PASSWORD>';
CREATE SCHEMA datadog;
GRANT USAGE ON SCHEMA datadog TO datadog;
GRANT USAGE ON SCHEMA public TO datadog;
GRANT SELECT ON pg_stat_database TO datadog;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Create functions to enable the Agent to read the full contents of `pg_stat_activity` and `pg_stat_statements`:

```SQL
CREATE OR REPLACE FUNCTION datadog.pg_stat_activity() RETURNS SETOF pg_stat_activity AS
  $$ SELECT * FROM pg_catalog.pg_stat_activity; $$
LANGUAGE sql
SECURITY DEFINER;
CREATE OR REPLACE FUNCTION datadog.pg_stat_statements() RETURNS SETOF pg_stat_statements AS
    $$ SELECT * FROM pg_stat_statements; $$
LANGUAGE sql
SECURITY DEFINER;
```

{{% /tab %}}
{{< /tabs >}}

**Note**: When generating custom metrics that require querying additional tables, you may need to grant the `SELECT` permission on those tables to the `datadog` user. Example: `grant SELECT on <TABLE_NAME> to datadog;`. See [PostgreSQL custom metric collection explained][6] for more information.


Create the function to enable the Agent to collect explain plans.

```SQL
CREATE OR REPLACE FUNCTION datadog.explain_statement (
   l_query text,
   out explain JSON
)
RETURNS SETOF JSON AS
$$
BEGIN
   RETURN QUERY EXECUTE 'EXPLAIN (FORMAT JSON) ' || l_query;
END;
$$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT
SECURITY DEFINER;
```

### Verify

To verify the permissions are correct, run the following commands to confirm the Agent user is able to connect to the database and read the core tables:

{{< tabs >}}
{{% tab "Postgres ≥ 10" %}}

```shell
psql -h localhost -U datadog postgres -A \
  -c "select * from pg_stat_database limit 1;" \
  && echo -e "\e[0;32mPostgres connection - OK\e[0m" \
  || echo -e "\e[0;31mCannot connect to Postgres\e[0m"
psql -h localhost -U datadog postgres -A \
  -c "select * from pg_stat_activity limit 1;" \
  && echo -e "\e[0;32mPostgres pg_stat_activity read OK\e[0m" \
  || echo -e "\e[0;31mCannot read from pg_stat_activity\e[0m"
psql -h localhost -U datadog postgres -A \
  -c "select * from pg_stat_statements limit 1;" \
  && echo -e "\e[0;32mPostgres pg_stat_statements read OK\e[0m" \
  || echo -e "\e[0;31mCannot read from pg_stat_statements\e[0m"
```
{{% /tab %}}
{{% tab "Postgres 9.6" %}}

```shell
psql -h localhost -U datadog postgres -A \
  -c "select * from pg_stat_database() limit 1;" \
  && echo -e "\e[0;32mPostgres connection - OK\e[0m" \
  || echo -e "\e[0;31mCannot connect to Postgres\e[0m"
psql -h localhost -U datadog postgres -A \
  -c "select * from pg_stat_activity() limit 1;" \
  && echo -e "\e[0;32mPostgres pg_stat_activity read OK\e[0m" \
  || echo -e "\e[0;31mCannot read from pg_stat_activity\e[0m"
psql -h localhost -U datadog postgres -A \
  -c "select * from pg_stat_statements() limit 1;" \
  && echo -e "\e[0;32mPostgres pg_stat_statements read OK\e[0m" \
  || echo -e "\e[0;31mCannot read from pg_stat_statements\e[0m"
```

{{% /tab %}}
{{< /tabs >}}

When it prompts for a password, use the password you entered when you created the `datadog` user.

## Install the Agent

To monitor Cloud SQL hosts, the Agent should be installed somewhere in your infrastructure and configured to connect to each instance remotely.

Installing the Datadog Agent also installs the Postgres check which is required for Database Monitoring on Postgres. If you haven't already installed the Agent for your Postgres database host, see the [Agent installation instructions][8].

## Configure the Agent

{{< tabs >}}
{{% tab "Host" %}}

To configure Database Monitoring metrics collection for an Agent running on a host, for example when you provision a small GCE instance for the Agent to collect from a Google Cloud SQL database:

1. Edit the `postgres.d/conf.yaml` file to point to your `host` / `port` and set the masters to monitor. See the [sample postgres.d/conf.yaml][1] for all available configuration options.
   ```yaml
   init_config:
   instances:
     - dbm: true
       host: '<INSTANCE_ADDRESS>'
       port: 5432
       username: datadog
       password: '<PASSWORD>'
       ## Optional: Connect to a different database if needed for `custom_queries`
       # dbname: '<DB_NAME>'
   ```
2. [Restart the Agent][2].


[1]: https://github.com/DataDog/integrations-core/blob/master/postgres/datadog_checks/postgres/data/conf.yaml.example
[2]: /agent/guide/agent-commands/#start-stop-and-restart-the-agent
{{% /tab %}}
{{% tab "Docker" %}}

To configure this check for an Agent running on a container:

Set [Autodiscovery Integrations Templates][1] as Docker labels on your application container:
```yaml
LABEL "com.datadoghq.ad.check_names"='["postgres"]'
LABEL "com.datadoghq.ad.init_configs"='[{}]'
LABEL "com.datadoghq.ad.instances"='[{"dbm": true, "host": "<INSTANCE_ADDRESS>", "port":5432,"username":"datadog","password":"<PASSWORD>"}]'
```
See the [Autodiscovery template variables documentation][2] to learn how to pass `<PASSWORD>` as an environment variable instead of a label.


[1]: /agent/docker/integrations/?tab=docker
[2]: /agent/faq/template_variables/
{{% /tab %}}
{{% tab "Kubernetes" %}}

If you have a Kubernetes cluster, use the [Datadog Cluster Agent][1] for Database Monitoring.

Follow the instructions to [enable the cluster checks][2] if not already enabled in your Kubernetes cluster. You can declare the Postgres configuration either with static files mounted in the Cluster Agent container or using service annotations:

### Configure with mounted files

To configure a cluster check with a mounted configuration file, mount the configuration file in the Cluster Agent container on the path: `/conf.d/postgres.yaml`:

```yaml
cluster_check: true  # Make sure to include this flag
init_config:
instances:
  - dbm: true
    host: '<AWS_INSTANCE_ENDPOINT>'
    port: 5432
    username: datadog
    password: '<PASSWORD>'
```

### Configure with Kubernetes service annotations

Rather than mounting a file, you can declare the instance configuration as a Kubernetes Service. To configure this check for an Agent running on Kubernetes, create a Service in the same namespace as the Datadog Cluster Agent:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    tags.datadoghq.com/env: '<ENV>'
    tags.datadoghq.com/service: '<SERVICE>'
  annotations:
    ad.datadoghq.com/postgres.check_names: '["postgres"]'
    ad.datadoghq.com/postgres.init_configs: '[{}]'
    ad.datadoghq.com/postgres.instances: |
      [
        {
          "dbm": true,
          "host": "<INSTANCE_ADDRESS>",
          "port": 5432,
          "username": "datadog",
          "password": "<UNIQUEPASSWORD>"
        }
      ]
spec:
  ports:
  - port: 5432
    protocol: TCP
    targetPort: 5432
    name: postgres
```

The Cluster Agent automatically registers this configuration and begin running the Postgres check.

To avoid exposing the `datadog` user's password in plain text, use the Agent's [secret management package][3] and declare the password using the `ENC[]` syntax.

[1]: /agent/cluster_agent
[2]: /agent/cluster_agent/clusterchecks/
[3]: /agent/guide/secrets-management
{{% /tab %}}
{{< /tabs >}}

### Validate

[Run the Agent's status subcommand][9] and look for `postgres` under the Checks section. Or visit the [Databases][10] page to get started!

## Install the Cloud SQL integration

To collect more comprehensive database metrics from GCP, install the [Cloud SQL integration][11] (optional).

## Troubleshooting

If you have installed and configured the integrations and Agent as described and it is not working as expected, see [Troubleshooting][12]

## Further reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /agent/basic_agent_usage#agent-overhead
[2]: /database_monitoring/data_collected/#sensitive-information
[3]: https://www.postgresql.org/docs/current/config-setting.html
[4]: https://cloud.google.com/sql/docs/postgres/flags
[5]: https://www.postgresql.org/docs/current/pgstatstatements.html
[6]: /integrations/faq/postgres-custom-metric-collection-explained/
[7]: https://www.postgresql.org/docs/current/app-psql.html
[8]: https://app.datadoghq.com/account/settings#agent
[9]: /agent/guide/agent-commands/#agent-status-and-information
[10]: https://app.datadoghq.com/databases
[11]: /integrations/google_cloudsql
[12]: /database_monitoring/troubleshooting/?tab=postgres