---
title: Data Migration Task Configuration File
summary: This document introduces the task configuration file of Data Migration. 
category: tools
---

# Data Migration Task Configuration File

This document introduces the task configuration file of Data Migration --
[`task.yaml`](https://github.com/pingcap/tidb-tools/blob/docs/docs/dm/zh_CN/configuration/task.yaml), including [Global configuration](#global-configuration) and [Instance configuration](#instance-configuration).

For description of configuration items, see [Data Migration Task Configuration Options](../tools/dm-task-config-argument-description.md).

## Important concepts

For description of important concepts including `instance-id` and the DM-worker ID, see [Important concepts](../tools/dm-configuration-file-overview.md#important-concepts). 

## Global configuration

### Basic information configuration

```
name: test                      # The name of the task. Should be globally unique.
task-mode: all                  # The task mode. Can be set to `full`/`incremental`/`all`.
is-sharding: true               # Whether it is a sharding task
meta-schema: "dm_meta"          # The downstream database that stores the `meta` information
remove-meta: false              # Whether to remove the `meta` information (`checkpoint` and `onlineddl`) before starting the 
                                # synchronization task 

target-database:                # Configuration of the downstream database instance
    host: "192.168.0.1"
    port: 4000
    user: "root"
    password: ""
```

For more details of `task-mode`, see [Task configuration argument description](../tools/dm-task-config-argument-description.md).

### Feature configuration set

Global configuration includes the following feature configuration set.

```
routes:                                             # The routing mapping rule set between the upstream and downstream tables
    user-route-rules-schema:                        # `schema-pattern`/`table-pattern` uses the wildcard matching rule.
        schema-pattern: "test_*"                
        table-pattern: "t_*"
        target-schema: "test"
        target-table: "t"

filters:                                            # The binlog event filter rule set of the matched table of the upstream
                                                    # database instance
    user-filter-1:
        schema-pattern: "test_*"
        table-pattern: "t_*"
        events: ["truncate table", "drop table"]
        action: Ignore

black-white-list:                                   # The filter rule set of the black white list of the matched table of the 
                                                    # upstream database instance
    instance:                                  
        do-dbs: ["~^test.*", "do"]
        ignore-dbs: ["mysql", "ignored"]
        do-tables:
        - db-name: "~^test.*"
          tbl-name: "~^t.*"

column-mappings:                                    # The column mapping rule set of the matched table of the upstream database 
                                                    # instance
    instance-1:                                     
        schema-pattern: "test_*"
        table-pattern: "t_*"
        expression: "partition id"
        source-column: "id"
        target-column: "id"
        arguments: ["1", "test_", "t_"]

mydumpers:                                          # Configuration arguments of running mydumper
    global:
        mydumper-path: "./mydumper"                 # The mydumper binary file path. It is generated by the Ansible deployment                                                     # application automatically and needs no configuration.
        threads: 16                                 # The number of the threads mydumper dumps from the upstream database instance
        chunk-filesize: 64                          # The size of the file mydumper generates
        skip-tz-utc: true						
        extra-args: "-B test -T t1,t2 --no-locks"

loaders:                                            # Configuration arguments of running Loader
    global:
        pool-size: 16                               # The number of threads that execute mydumper SQL files concurrently in Loader
        dir: "./dumped_data"                        # The directory output by mydumper that Loader reads. Directories for
                                                    # different tasks of the same instance must be different. (mydumper outputs the 
                                                    # SQL file based on the directory)

syncers:                                            # Configuration arguments of running Syncer
    global:
        worker-count: 16                            # The number of threads that synchronize binlog events concurrently in Syncer
        batch: 1000                                 # The number of SQL statements in a transaction batch that Syncer 
                                                    # synchronizes to the downstream database
        max-retry: 100                              # The retry times of the transactions with an error that Syncer synchronizes
                                                    # to the downstream database (only for DML operations)
```

References:

- For details of `user-filter-1`, see [Filtering rules of binlog events](../tools/dm-task-config-argument-description.md#filtering-rules-of-binlog-events).
- For details of `instance`, see [Black and white lists filtering rule](../tools/dm-task-config-argument-description.md#black-and-white-lists-filtering-rule).
- For details of `instance-1` of `column-mappings`, see [Column mapping rule](../tools/dm-task-config-argument-description.md#column-mapping-rule).

## Instance configuration

This part defines the subtask of data synchronization. DM supports synchronizing data from one or multiple MySQL instances to the same instance.

```
mysql-instances:
    -
        config:                                    # The upstream database configuration corresponding to `instance-id`
            host: "192.168.199.118"
            port: 4306
            user: "root"
            password: "1234"                       # Requires the password encrypted by dmctl
        instance-id: "instance118-4306"            # The MySQL instance ID. It corresponds to the upstream MySQL instance. It is 
                                                   # not allowed to set it to an ID of a MySQL instance that is not within the 
                                                   # DM-master cluster topology.

        meta:                                      # The position where the binlog synchronization starts when the checkpoint of 
                                                   # the downstream database does not exist. If the checkpoint exits, this 
                                                   # configuration does not work. 
            binlog-name: binlog-00001
            binlog-pos: 4

        route-rules: ["user-route-rules-schema", "user-route-rules"]       # Routing rules selected from `routes` above
        filter-rules: ["user-filter-1", "user-filter-2"]                   # Filter rules selected from `filters` above
        column-mapping-rules: ["instance-1"]                               # Column mapping rules selected from `column-mappings` above 
        black-white-list:  "instance"                                      # The black white list item selected from `black-white-list` above 

        mydumper-config-name: "global"                                     # The mydumper configuration name. You cannot set it 
                                                                           # and `mydumper` at the same time. 
        loader-config-name: "global"                                       # The Loader configuration name. You cannot set it and
                                                                           # `loader` at the same time.
        syncer-config-name: "global"                                       # The Syncer configuration name. You cannot set it and 
                                                                           # `syncer` at the same time.

    -
        config:
            host: "192.168.199.118"
            port: 5306
            user: "root"
            password: "1234"
        instance-id: "instance118-5306"

        mydumper:                                                          # The mydumper configuration. You cannot set it and 
                                                                           # `mydumper-config-name` at the same time.
            mydumper-path: "./mydumper"                                    # The mydumper binary file path. It is generated by 
                                                                           # Ansible deployment application and needs no 
                                                                           # configuration.
            threads: 4
            chunk-filesize: 8
            skip-tz-utc: true
            extra-args: "-B test -T t1,t2"
    
        loader:                                                            # The Loader configuration. You cannot set it and 
                                                                           # `loader-config-name` at the same time.
            pool-size: 32                                                  # The number of threads that execute mydumper SQL 
                                                                           # files concurrently in Loader
            dir: "./dumped_data"
    
        syncer:                                                            # The Syncer configuration. You cannot set it and 
                                                                           # `syncer-config-name` at the same time.
            worker-count: 32                                               # The number of threads that synchronize binlog events 
                                                                           # concurrently in Syncer
            batch: 2000
            max-retry: 200
```

For the configuration details of the above options, see the corresponding part in [Feature configuration set](#feature-configuration-set), as shown in the following table.

| Option | Corresponding part |
| ------ | ------------------ |
| `route-rules` | `routes` |
| `filter-rules` | `filters` |
| `column-mapping-rules` | `column-mappings` |
| `black-white-list` | `black-white-list` |
| `mydumper-config-name` | `mydumpers` |
| `loader-config-name` | `loaders` |
| `syncer-config-name` | `syncers`  |