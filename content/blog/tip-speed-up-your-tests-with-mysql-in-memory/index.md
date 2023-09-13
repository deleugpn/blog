---
title: "Tip: Speed up your tests with MySQL in-memory"
date: "2022-03-06T15:09:30.284Z"
description: Using Docker tmpfs for MySQL performance
---

> Byte size tip 

It's very common in the Laravel community to use SQLite in-memory for phpunit tests.
However, sometimes you may need functionality that differ between MySQL and SQLite.
Not writing test for that or struggling to find a compatible SQL operation for
the two databases are options that I consider less than ideal.

We can instruct Docker to mount a file system that will only be persisted on memory

```yaml 
  database:
    image: mysql:5.7
    ports:
    - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD={your_password_here}
      - MYSQL_DATABASE={your_default_database}
    command: mysqld --sql_mode="NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"
    tmpfs:
      - /var/lib/mysql:rw
```

Setting up `/var/lib/mysql` as a read-write temporary file system will instruct docker to skip the local
storage and only mount the volume in memory, speeding up it's storage manipulation.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.
