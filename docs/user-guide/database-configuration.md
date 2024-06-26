### Database Configuration

#### PostgreSQL Version

The default PostgreSQL version for the version of `AnsibleAIConnect` bundled with the latest version of the `ansible-ai-connect-operator` is PostgreSQL 15. You can find this default for a given version by at the default value for [supported_pg_version](./roles/postgres/vars/main.yml).

We only have coverage for the default version of PostgreSQL. Newer versions of PostgreSQL will likely work, but should only be configured as an external database. If your database is managed by the operator (default if you don't specify a `postgres_configuration_secret`), then you should not override the default version as this may cause issues when operator tries to upgrade your postgresql pod.

#### External PostgreSQL Service

To configure `AnsibleAIConnect` to use an external database, the Custom Resource needs to know about the connection details. To do this, create a k8s secret with those connection details and specify the name of the secret as `postgres_configuration_secret` at the CR spec level.


The secret should be formatted as follows:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: <resourcename>-postgres-configuration
  namespace: <target namespace>
stringData:
  host: <external ip or url resolvable by the cluster>
  port: <external port, this usually defaults to 5432>
  database: <desired database name>
  username: <username to connect as>
  password: <password to connect with>
  sslmode: prefer
  type: unmanaged
type: Opaque
```

> Please ensure that the value for the variable `password` should _not_ contain single or double quotes (`'`, `"`) or backslashes (`\`) to avoid any issues during deployment.

> It is possible to set a specific username, password, port, or database, but still have the database managed by the operator. In this case, when creating the postgres-configuration secret, the `type: managed` field should be added.

**Note**: The variable `sslmode` is valid for `external` databases only. The allowed values are: `prefer`, `disable`, `allow`, `require`, `verify-ca`, `verify-full`.

Once the secret is created, you can specify it on your spec:

```yaml
---
spec:
  ...
  postgres_configuration_secret: <name-of-your-secret>
```

#### Managed PostgreSQL Service

If you don't have access to an external PostgreSQL service, the `AnsibleAIConnect` operator can deploy one for you alongside the `AnsibleAIConnect` instance itself.

The following variables are customizable for the managed PostgreSQL service

| Name                                          | Description                                   | Default                                |
| --------------------------------------------- | --------------------------------------------- | -------------------------------------- |
| postgres_image                                | Path of the image to pull                     | quay.io/sclorg/postgresql-15-c9s       |
| postgres_image_version                        | Image version to pull                         | c9s                                    |
| database.resource_requirements                | PostgreSQL container resource requirements    | requests: {cpu: 50m, memory: 100Mi}    |
| database.storage_requirements                 | PostgreSQL container storage requirements     | requests: {storage: 8Gi}               |
| database.postgres_storage_class               | PostgreSQL PV storage class                   | Empty string                           |
| database.priority_class                       | Priority class used for PostgreSQL pod        | Empty string                           |

Example of customization could be:

```yaml
---
spec:
  ...
  database:
    resource_requirements:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: '1'
        memory: 4Gi
    storage_requirements:
      requests:
        storage: 8Gi
    postgres_storage_class: fast-ssd
    postgres_extra_args:
      - '-c'
      - 'max_connections=1000'
```

**Note**: If `database.postgres_storage_class` is not defined, PostgreSQL will store it's data on a volume using the default storage class for your cluster.

#### Note about overriding the postgres image

We recommend you use the default image sclorg image. If you override the postgres image to use a custom postgres image like `postgres:15` for example, the default data directory path may be different. These images cannot be used interchangeably.

You can no longer configure a custom `postgres_data_path` because it is hardcoded in the `quay.io/sclorg/postgresql-15-c9s` image.