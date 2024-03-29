# Deprecated

We switched over using this helm chart here: https://github.com/iits-consulting/charts/tree/main/charts/db-init

## Kubernetes Cloud Container Engine Terraform module

A module for running an initialization SQL script on an existing database server.

### Supported DB engines
postgres, mysql, mariadb

#### Limitations:
Setting pod annotations/labels through this module only works when creating the pod for the first time.
Pod annotations and labels are listed in **lifecycle.ignorechanges** list inside the **kubernetes_pod** ressource.
An update on pod annotations/labels after creation is only possible via recreation (taint, destroy, etc...).

Usage example
```hcl
locals {
  # List all databases you want to create
  databases = toset([
    "mordor",
    "customers",
  ])
}

module "db-init" {
  source          = "registry.terraform.io/iits-consulting/db-init/kubernetes"
  database_engine = "postgres"
  database_root_credentials = {
    username = "root"
    password = "root_password"
    address  = "mydatabase.mydomain.de"
  }
  initdb_script = templatefile("./initdb.sql", { #Path to the SQL Script
    databases = [for database in local.databases : {# This populates variables set inside the initdb.sql script
      name     = database
      username = database
      password = random_password.database_passwords[database].result
    }]
  })
  pod_config = {
    namespace = "database"
    namespace_create = true
    name             = "db-init"
    image            = "alpine:3.15.5"
    annotations      = {}
    labels = {
      "app.kubernetes.io/instance" = "alpine"
      "app.kubernetes.io/name"     = "db-init"
    }
  }
}
```
Example **initdb.sql** script. Uses variables which are populated by terraform's templatefile() Function (see above)
```sql
%{ for database in databases }
CREATE DATABASE IF NOT EXISTS ${database.name};
CREATE USER IF NOT EXISTS '${database.username}'@'localhost' IDENTIFIED BY '${database.password}';
GRANT ALL ON ${database.name}.* TO '${database.username}'@'localhost';
ALTER USER IF EXISTS '${database.username}'@'localhost' IDENTIFIED BY '${database.password}';
FLUSH PRIVILEGES;
%{ endfor ~}
```
