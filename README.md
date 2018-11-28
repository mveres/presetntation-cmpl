# Problem 1


## Part 1 - Choosing the technology

Decision: SQL vs. NoSQL database?

* Time to delivery - SHORT
* Team knowledge - SQL

Conclusin:
No time to ramp up on NoSQL databases.
MySQL chosen as technology to profit from team's expertise.

*Time Team = me, Time to delivery = about 2 days*


## Part 2 - Breaking down the problem into smaller, iterative problems

1. Schema structure
2. Populate with sample data
3. Query samples
4. Create bigger sample data
5. Analyse performance of queries and improve
6. Create same order of magnitude as requested sample data & Analyse again and improve


## 1. Schema structure

##### 1. Setup (*docker for speed and flexibility*)

MySQL server instance:
`docker run --name comply-mysql -e MYSQL_ROOT_PASSWORD=test -d mysql:latest`

MySQL command line client:
`docker run -it --link comply-mysql:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'`

##### 2. Schema

```
comply:
+------------------+
| Tables_in_comply |
+------------------+
| clients          |
| searchedTags     |
| searches         |
| tags             |
| users            |
+------------------+


clients:
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | varchar(36) | NO   | PRI | NULL    |       |
| name  | varchar(36) | NO   | UNI | NULL    |       |
+-------+-------------+------+-----+---------+-------+


users:
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| id       | varchar(36) | NO   | PRI | NULL    |       |
| clientId | varchar(36) | NO   |     | NULL    |       |
| name     | varchar(36) | NO   |     | NULL    |       |
+----------+-------------+------+-----+---------+-------+


tags:
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| id       | varchar(36) | NO   | PRI | NULL    |       |
| clientId | varchar(36) | NO   |     | NULL    |       |
| tag      | varchar(36) | NO   |     | NULL    |       |
+----------+-------------+------+-----+---------+-------+


searches:
+-------------+-------------+------+-----+---------+-------+
| Field       | Type        | Null | Key | Default | Extra |
+-------------+-------------+------+-----+---------+-------+
| id          | varchar(36) | NO   | PRI | NULL    |       |
| clientId    | varchar(36) | NO   |     | NULL    |       |
| userId      | varchar(36) | NO   |     | NULL    |       |
| searchTime  | timestamp   | NO   |     | NULL    |       |
| name        | varchar(36) | NO   |     | NULL    |       |
| yearOfBirth | smallint(6) | YES  |     | NULL    |       |
| matched     | tinyint(1)  | NO   |     | NULL    |       |
+-------------+-------------+------+-----+---------+-------+

searchedTags:
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| tagId    | varchar(36) | NO   |     | NULL    |       |
| searchId | varchar(36) | NO   |     | NULL    |       |
+----------+-------------+------+-----+---------+-------+
```

*Note: foreign keys are in place even though not shown above*
*Note: data types can be changed after questions*

SQL to create tables:

```
CREATE TABLE clients (
  id VARCHAR(36) NOT NULL UNIQUE,
  name VARCHAR(36) NOT NULL UNIQUE
);

CREATE TABLE users (
  id VARCHAR(36) NOT NULL UNIQUE,
  clientId VARCHAR(36) NOT NULL REFERENCES clients (id),
  name VARCHAR(36) NOT NULL
);

CREATE TABLE tags (
  id VARCHAR(36) NOT NULL UNIQUE,
  clientId VARCHAR(36) NOT NULL REFERENCES clients (id),
  tag VARCHAR(36) NOT NULL
);

CREATE TABLE searches (
  id VARCHAR(36) NOT NULL UNIQUE,
  clientId VARCHAR(36) NOT NULL REFERENCES clients (id),
  userId VARCHAR(36) NOT NULL REFERENCES users (id),
  searchTime TIMESTAMP NOT NULL,
  name VARCHAR(36) NOT NULL,
  yearOfBirth SMALLINT,
  matched BOOLEAN NOT NULL
);

CREATE TABLE searchedTags (
  tagId VARCHAR(36) NOT NULL REFERENCES tags (id),
  searchId VARCHAR(36) NOT NULL REFERENCES searches (id)
);
```


## 2. Populate with sample data

```
-- sample data

INSERT INTO clients (id, name) VALUES (UUID(), 'first bank');
INSERT INTO clients (id, name) VALUES (UUID(), 'second bank');

INSERT INTO users (id, clientId, name)
SELECT UUID() AS id, id AS clientId, 'Joe Smith' AS name FROM clients;

INSERT INTO users (id, clientId, name)
SELECT UUID() AS id, id AS clientId, 'Tim Robinnson' AS name FROM clients;

INSERT INTO tags (id, clientId, tag)
SELECT UUID() AS id, id AS clientId, 'outgoing payment' AS tag FROM clients;
INSERT INTO tags (id, clientId, tag)
SELECT UUID() AS id, id AS clientId, 'payment to russia' AS tag FROM clients;
INSERT INTO tags (id, clientId, tag)
SELECT UUID() AS id, id AS clientId, 'high net worth individual' AS tag FROM clients;

INSERT INTO searches (id, clientId, userId, searchTime, name, yearOfBirth, matched)
SELECT
  UUID() AS id,
  clientId,
  id AS userId,
  TIMESTAMP('2018-07-20 09:11:11') AS searchTime,
  'Robert Baratheon' AS name,
  1980 AS yearOfBirth,
  1 AS matched
FROM users;

INSERT INTO searches (id, clientId, userId, searchTime, name, yearOfBirth, matched)
SELECT
  UUID() AS id,
  clientId,
  id AS userId,
  TIMESTAMP('2018-08-21 08:11:11') AS searchTime,
  'Stannis Baratheon' AS name,
  1970 AS yearOfBirth,
  0 AS matched
FROM users;

INSERT INTO searchedTags (searchId, tagId)
SELECT s.id AS searchId, t.id AS tagId
FROM searches s
INNER JOIN clients c ON c.id = s.clientId
INNER JOIN tags t ON c.id = t.clientId;
```


## Part 5: Query samples

#### All searches:
```
WITH allTagsBySearchId AS (
  SELECT st.searchId, GROUP_CONCAT(t.tag) AS tags
  FROM searchedTags st
  INNER JOIN tags t ON t.id = st.tagId
  GROUP BY st.searchId
)
SELECT c.name AS client, u.name as user, s.searchTime, s.name, s.yearOfBirth, s.matched, at.tags
FROM searches s
INNER JOIN allTagsBySearchId at ON s.id = at.searchId
INNER JOIN users u ON u.id = s.userId
INNER JOIN clients c ON c.id = s.clientId;
```

Results:
```
+-------------+---------------+---------------------+-------------------+-------------+---------+--------------------------------------------------------------+
| client      | user          | searchTime          | name              | yearOfBirth | matched | tags                                                         |
+-------------+---------------+---------------------+-------------------+-------------+---------+--------------------------------------------------------------+
| first bank  | Joe Smith     | 2018-07-20 09:11:11 | Robert Baratheon  |        1980 |       1 | payment to russia,high net worth individual,outgoing payment |
| second bank | Joe Smith     | 2018-07-20 09:11:11 | Robert Baratheon  |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
| first bank  | Tim Robinnson | 2018-07-20 09:11:11 | Robert Baratheon  |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
| second bank | Tim Robinnson | 2018-07-20 09:11:11 | Robert Baratheon  |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
| first bank  | Joe Smith     | 2018-08-21 08:11:11 | Stannis Baratheon |        1970 |       0 | outgoing payment,high net worth individual,payment to russia |
| second bank | Joe Smith     | 2018-08-21 08:11:11 | Stannis Baratheon |        1970 |       0 | outgoing payment,payment to russia,high net worth individual |
| first bank  | Tim Robinnson | 2018-08-21 08:11:11 | Stannis Baratheon |        1970 |       0 | outgoing payment,payment to russia,high net worth individual |
| second bank | Tim Robinnson | 2018-08-21 08:11:11 | Stannis Baratheon |        1970 |       0 | outgoing payment,payment to russia,high net worth individual |
+-------------+---------------+---------------------+-------------------+-------------+---------+--------------------------------------------------------------+
8 rows in set (0.00 sec)
```

#### Searches in July 2018 tagged “outgoing payment” and “payment to russia”
```
WITH searchesWithTags AS (
  SELECT st.searchId, COUNT(st.tagId) AS tagCount
  FROM searchedTags st
  INNER JOIN tags t ON t.id = st.tagId
  WHERE t.tag IN ('outgoing payment', 'payment to russia')
  GROUP BY st.searchId HAVING (tagCount = 2)
),
allTagsBySearchId AS (
  SELECT st.searchId, GROUP_CONCAT(t.tag) AS tags
  FROM searchedTags st
  INNER JOIN tags t ON t.id = st.tagId
  GROUP BY st.searchId
)
SELECT c.name AS client, u.name AS user, s.searchTime, s.name, s.yearOfBirth, s.matched, at.tags
FROM searches s
INNER JOIN searchesWithTags swt ON s.id = swt.searchId
INNER JOIN allTagsBySearchId at ON s.id = at.searchId
INNER JOIN users u ON u.id = s.userId
INNER JOIN clients c ON c.id = s.clientId
WHERE MONTHNAME(s.searchTime) = 'JULY' AND YEAR(s.searchTime) = '2018';
```

Results:
```
+-------------+---------------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
| client      | user          | searchTime          | name             | yearOfBirth | matched | tags                                                         |
+-------------+---------------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
| first bank  | Joe Smith     | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | payment to russia,high net worth individual,outgoing payment |
| second bank | Joe Smith     | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
| first bank  | Tim Robinnson | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
| second bank | Tim Robinnson | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
+-------------+---------------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
4 rows in set (0.00 sec)
```

#### Searches beginning with “Robert” conducted by “Joe Smith”.
```
WITH allTagsBySearchId AS (
  SELECT st.searchId, GROUP_CONCAT(t.tag) AS tags
  FROM searchedTags st
  INNER JOIN tags t ON t.id = st.tagId
  GROUP BY st.searchId
)
SELECT c.name AS client, u.name as user, s.searchTime, s.name, s.yearOfBirth, s.matched, at.tags
FROM searches s
INNER JOIN allTagsBySearchId at ON s.id = at.searchId
INNER JOIN users u ON u.id = s.userId
INNER JOIN clients c ON c.id = s.clientId
WHERE s.name LIKE 'Robert%' AND u.name = 'Joe Smith';
```

Results:
```
+-------------+-----------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
| client      | user      | searchTime          | name             | yearOfBirth | matched | tags                                                         |
+-------------+-----------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
| first bank  | Joe Smith | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | payment to russia,high net worth individual,outgoing payment |
| second bank | Joe Smith | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
+-------------+-----------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
2 rows in set (0.00 sec)
```

#### Searches that had a result (matched = true) and tagged with “outgoing payment” and “high net worth individual”
```
WITH allTagsBySearchId AS (
  SELECT st.searchId, GROUP_CONCAT(t.tag) AS tags
  FROM searchedTags st
  INNER JOIN tags t ON t.id = st.tagId
  GROUP BY st.searchId
)
SELECT c.name AS client, u.name as user, s.searchTime, s.name, s.yearOfBirth, s.matched, at.tags
FROM searches s
INNER JOIN allTagsBySearchId at ON s.id = at.searchId
INNER JOIN users u ON u.id = s.userId
INNER JOIN clients c ON c.id = s.clientId
WHERE s.matched = 1;
```

Results:
```
+-------------+---------------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
| client      | user          | searchTime          | name             | yearOfBirth | matched | tags                                                         |
+-------------+---------------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
| first bank  | Joe Smith     | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | payment to russia,high net worth individual,outgoing payment |
| second bank | Joe Smith     | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
| first bank  | Tim Robinnson | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
| second bank | Tim Robinnson | 2018-07-20 09:11:11 | Robert Baratheon |        1980 |       1 | outgoing payment,payment to russia,high net worth individual |
+-------------+---------------+---------------------+------------------+-------------+---------+--------------------------------------------------------------+
4 rows in set (0.00 sec)
```

NOTE: Tenant-wise all queries will be filtered by `clientId`


## 3. Create bigger sample data

#### procedure to create sample data

`big_data(totalClients, totalUsersPerClient, totalTagsPerClient, totalSearchesPerUser)`

```
CREATE PROCEDURE big_data
(
    totalClients INT,
    totalUsersPerClient INT,
    totalTagsPerClient INT,
    totalSearchesPerUser INT
)
BEGIN

    DECLARE clientsCount, usersCount, tagsCount, searchesCount INT;
    DECLARE clientId, userId, tagId, searchId VARCHAR(36);
    DECLARE searchTime TIMESTAMP;
    DECLARE yearOfBirth, tagStart, tagEnd INT;
    DECLARE matched TINYINT;

    SELECT totalClients, totalUsersPerClient, totalTagsPerClient, totalSearchesPerUser;

    SET clientsCount = 0;
    WHILE clientsCount < totalClients DO
        SET clientId = UUID();

        INSERT INTO clients (id, name)
        VALUES (clientId, CONCAT('bank ', LEFT(clientId, 8)));

        SET clientsCount = clientsCount + 1;

        -- tags
        SET tagsCount = 0;
        WHILE tagsCount < totalTagsPerClient DO
          SET tagId = UUID();

          INSERT INTO tags (id, clientId, tag)
          VALUES (tagId, clientId, CONCAT('tag ', LEFT(tagId, 8)));

          SET tagsCount = tagsCount + 1;
        END WHILE;

        -- users
        SET usersCount = 0;
        WHILE usersCount < totalUsersPerClient DO
          SET userId = UUID();

          INSERT INTO users (id, clientId, name)
          VALUES (userId, clientId, CONCAT('user ', LEFT(clientId, 8)));

          -- searches
          SET searchesCount = 0;
          WHILE searchesCount < totalSearchesPerUser DO
            SET searchId = UUID();
            SET searchTime = FROM_UNIXTIME(UNIX_TIMESTAMP('2016-11-01 12:00:00') + FLOOR((RAND() * 63072000)));
            SET yearOfBirth = 1940 + FLOOR((RAND() * 80));
            SET matched = FLOOR((RAND() * 2));

            SET tagEnd = FLOOR((RAND() * totalTagsPerClient));
            SET tagStart = FLOOR((RAND() * tagEnd));

            -- logging
            SELECT clientsCount, usersCount, searchesCount, searchTime, yearOfBirth, matched, tagStart, tagEnd;

            INSERT INTO searches (id, clientId, userId, searchTime, name, yearOfBirth, matched)
            SELECT searchId, clientId, userId, searchTime, CONCAT('name ', LEFT(searchId, 8)), yearOfBirth, matched;

            INSERT INTO searchedTags (searchId, tagId)
            SELECT searchId, t.id AS tagId
            FROM tags t WHERE t.clientId = clientId LIMIT tagStart, tagEnd;

            SET searchesCount = searchesCount + 1;
          END WHILE;

          SET usersCount = usersCount + 1;
        END WHILE;


    END WHILE;
END$$
```

```
generated data size:
520 clients
1700 users
15400 tags defined
161000 searches
1924905 searched tags (between 0 and 30 / search)
```

## 4. Analyse performance of queries and improve

#### Searches in July 2018 tagged “outgoing payment” and “payment to russia”
*i.e. 'tag e6ed82b3' and 'tag e6ee2ff4' for sample data for client "bank e6e3966e"*

```
WITH searchesWithTags AS (
  SELECT st.searchId, COUNT(st.tagId) AS tagCount
  FROM searchedTags st
  INNER JOIN tags t ON t.id = st.tagId
  WHERE t.tag IN ('tag e6ed82b3', 'tag e6ee2ff4')
  AND t.clientId = 'e6e3966e-f0d8-11e8-b606-0242ac110002'
  GROUP BY st.searchId HAVING (tagCount = 2)
),
allTagsBySearchId AS (
  SELECT st.searchId, GROUP_CONCAT(t.tag) AS tags
  FROM searchedTags st
  INNER JOIN tags t ON t.id = st.tagId
  AND t.clientId = 'e6e3966e-f0d8-11e8-b606-0242ac110002'
  GROUP BY st.searchId
)
SELECT c.name AS client, u.name AS user, s.searchTime, s.name, s.yearOfBirth, s.matched, at.tags
FROM searches s
INNER JOIN searchesWithTags swt ON s.id = swt.searchId
INNER JOIN allTagsBySearchId at ON s.id = at.searchId
INNER JOIN users u ON u.id = s.userId
INNER JOIN clients c ON (c.id = s.clientId AND c.id = 'e6e3966e-f0d8-11e8-b606-0242ac110002')
WHERE MONTHNAME(s.searchTime) = 'JULY' AND YEAR(s.searchTime) = '2018';
```


Result:
**5 rows in set (41.12 sec)** - time to improve


```
+---------------+---------------+---------------------+---------------+-------------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| client        | user          | searchTime          | name          | yearOfBirth | matched | tags                                                                                                                                                                                                                                      |
+---------------+---------------+---------------------+---------------+-------------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| bank e6e3966e | user e6e3966e | 2018-07-24 11:41:45 | name e7affeb2 |        1946 |       1 | tag e6f46a65,tag e6f40348,tag e6f35ebb,tag e6f2eeda,tag e6f26fac,tag e6f1d602,tag e6f15981,tag e6f0dfce,tag e6f03f16,tag e6ef4e98,tag e6eeb42a,tag e6ee2ff4,tag e6ed82b3,tag e6ed1072,tag e6ec45d1,tag e6eb87ea,tag e6eb1143,tag e6efc28c |
| bank e6e3966e | user e6e3966e | 2018-07-24 12:06:57 | name e7f23ea3 |        1941 |       1 | tag e6eb1143,tag e6ee2ff4,tag e6ed82b3,tag e6ed1072,tag e6ec45d1,tag e6eb87ea,tag e6ea80cf,tag e6ea063c,tag e6e8face,tag e6e881e7,tag e6e80a1e,tag e6e76a91,tag e6e70165,tag e6e6745f,tag e6e58459,tag e6e99d3b                           |
| bank e6e3966e | user e6e3966e | 2018-07-18 00:59:04 | name e81cc38e |        1952 |       0 | tag e6ed1072,tag e6f1d602,tag e6f15981,tag e6f0dfce,tag e6f03f16,tag e6efc28c,tag e6ef4e98,tag e6eeb42a,tag e6ee2ff4,tag e6ec45d1,tag e6eb87ea,tag e6eb1143,tag e6ea80cf,tag e6ea063c,tag e6e99d3b,tag e6e8face,tag e6e881e7,tag e6ed82b3 |
| bank e6e3966e | user e6e3966e | 2018-07-16 02:30:27 | name e8b61898 |        1995 |       1 | tag e6f1d602,tag e6f46a65,tag e6f40348,tag e6f35ebb,tag e6f2eeda,tag e6f26fac,tag e6f15981,tag e6f03f16,tag e6ed1072,tag e6efc28c,tag e6ef4e98,tag e6eeb42a,tag e6ee2ff4,tag e6ed82b3,tag e6f0dfce                                        |
| bank e6e3966e | user e6e3966e | 2018-07-29 06:12:49 | name e8e3f37a |        1941 |       0 | tag e6eb1143,tag e6eeb42a,tag e6ee2ff4,tag e6ed82b3,tag e6ed1072,tag e6ec45d1,tag e6eb87ea,tag e6ea80cf,tag e6ea063c,tag e6e99d3b,tag e6e8face,tag e6e881e7                                                                               |
+---------------+---------------+---------------------+---------------+-------------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

#### analyse subparts

`searchesWithTags `
```
SELECT st.searchId, COUNT(st.tagId) AS tagCount
FROM tags t
INNER JOIN searchedTags st ON t.id = st.tagId
WHERE t.tag IN ('tag e6ed82b3', 'tag e6ee2ff4')
AND t.clientId = 'e6e3966e-f0d8-11e8-b606-0242ac110002'
GROUP BY st.searchId HAVING (tagCount = 2);
```
`148 rows in set (20.28 sec)`

```
EXPLAIN SELECT st.searchId ...
+----+-------------+-------+------------+--------+---------------+------+---------+-----------------+---------+----------+-----------------+
| id | select_type | table | partitions | type   | possible_keys | key  | key_len | ref             | rows    | filtered | Extra           |
+----+-------------+-------+------------+--------+---------------+------+---------+-----------------+---------+----------+-----------------+
|  1 | SIMPLE      | st    | NULL       | ALL    | NULL          | NULL | NULL    | NULL            | 1913362 |   100.00 | Using temporary |
|  1 | SIMPLE      | t     | NULL       | eq_ref | id            | id   | 146     | comply.st.tagId |       1 |     5.00 | Using where     |
+----+-------------+-------+------------+--------+---------------+------+---------+-----------------+---------+----------+-----------------+
```

Observations on query plan:

* a full scan will be done on `searchedTags` table, over 2M rows
* usually columns to index are those used for searching, grouping, or sorting records
* in this case: searchId for grouping, tagId for joining (searching), tag for searching


```
ALTER TABLE searchedTags ADD INDEX(tagId);
ALTER TABLE searchedTags ADD INDEX(searchId);
```

#### Retry: Searches in July 2018 tagged “outgoing payment” and “payment to russia”
`5 rows in set (0.06 sec)`


Increased sample data size by one order of magnitude:
```
searches 1M
clients 700
users 4K
tags 20K
searchedTags 8M
```

Analyse again:
```
EXPLAIN SELECT st.searchId, ...
+----+-------------+-------+------------+------+----------------+-------+---------+-------------+-------+----------+------------------------------+
| id | select_type | table | partitions | type | possible_keys  | key   | key_len | ref         | rows  | filtered | Extra                        |
+----+-------------+-------+------------+------+----------------+-------+---------+-------------+-------+----------+------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | id             | NULL  | NULL    | NULL        | 20401 |     2.00 | Using where; Using temporary |
|  1 | SIMPLE      | st    | NULL       | ref  | tagId,searchId | tagId | 146     | comply.t.id |   209 |   100.00 | NULL                         |
+----+-------------+-------+------------+------+----------------+-------+---------+-------------+-------+----------+------------------------------+
```

Observations:
 * scan over 20K rows then 2% filtered by conditions, so 400 rows.
 * can be improved with a good index
 * id is indexed because it's a primary key
 * we're searching by tags -> add index on `tag`

`ALTER TABLE tags ADD INDEX(tag);`

```
EXPLAIN SELECT st.searchId, ...
+----+-------------+-------+------------+-------+----------------+-------+---------+-------------+------+----------+-----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys  | key   | key_len | ref         | rows | filtered | Extra                                               |
+----+-------------+-------+------------+-------+----------------+-------+---------+-------------+------+----------+-----------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | range | id,tag         | tag   | 146     | NULL        |    2 |    10.00 | Using index condition; Using where; Using temporary |
|  1 | SIMPLE      | st    | NULL       | ref   | tagId,searchId | tagId | 146     | comply.t.id |  209 |   100.00 | NULL                                                |
+----+-------------+-------+------------+-------+----------------+-------+---------+-------------+------+----------+-----------------------------------------------------+
```

#### Next subpart to improve:

* Searches beginning with “Robert” conducted by “Joe Smith”
`155 rows in set (0.93 sec)`

subpart:
```
SELECT c.name AS client, u.name as user, s.searchTime, s.name, s.yearOfBirth, s.matched
FROM searches s
INNER JOIN users u ON u.id = s.userId
INNER JOIN clients c ON (c.id = s.clientId AND c.id = 'e6e3966e-f0d8-11e8-b606-0242ac110002')
WHERE s.name LIKE 'name e8%' AND u.name = 'user e6e3966e';
```
`159 rows in set (0.87 sec)`

```
EXPLAIN SELECT c.name AS client, u.name as user, s.searchTime, s.name, s.yearOfBirth, s.matched FROM searches s INNER JOIN users u ON u.id = s.userId INNER JOIN clients c ON (c.id = s.clientId AND c.id = 'e6e3966e-f0d8-11e8-b606-0242ac110002') WHERE s.name LIKE 'name e8%' AND u.name = 'user e6e3966e';
+----+-------------+-------+------------+--------+---------------+------+---------+-----------------+---------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key  | key_len | ref             | rows    | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+------+---------+-----------------+---------+----------+-------------+
|  1 | SIMPLE      | c     | NULL       | const  | id            | id   | 146     | const           |       1 |   100.00 | NULL        |
|  1 | SIMPLE      | s     | NULL       | ALL    | NULL          | NULL | NULL    | NULL            | 1081245 |     1.11 | Using where |
|  1 | SIMPLE      | u     | NULL       | eq_ref | id            | id   | 146     | comply.s.userId |       1 |    10.00 | Using where |
+----+-------------+-------+------------+--------+---------------+------+---------+-----------------+---------+----------+-------------+
```

Observations:
 * scan over 1M rows then 1% filtered by conditions
 * can be improved with a good index

`CREATE INDEX search_name ON searches (name);`


* Searches beginning with “Robert” conducted by “Joe Smith”
`155 rows in set (0.03 sec)`
*(improved by one order of magnitude)*


```
 EXPLAIN SELECT c.name AS client, u.name as user, s.searchTime, s.name, s.yearOfBirth, s.matched FROM searches s INNER JOIN users u ON u.id = s.userId INNER JOIN clients c ON (c.id = s.clientId AND c.id = 'e6e3966e-f0d8-11e8-b606-0242ac110002') WHERE s.name LIKE 'name e8%' AND u.name = 'user e6e3966e';
 +----+-------------+-------+------------+--------+---------------+-------------+---------+-----------------+------+----------+------------------------------------+
 | id | select_type | table | partitions | type   | possible_keys | key         | key_len | ref             | rows | filtered | Extra                              |
 +----+-------------+-------+------------+--------+---------------+-------------+---------+-----------------+------+----------+------------------------------------+
 |  1 | SIMPLE      | c     | NULL       | const  | id            | id          | 146     | const           |    1 |   100.00 | NULL                               |
 |  1 | SIMPLE      | s     | NULL       | range  | search_name   | search_name | 146     | NULL            | 6566 |    10.00 | Using index condition; Using where |
 |  1 | SIMPLE      | u     | NULL       | eq_ref | id            | id          | 146     | comply.s.userId |    1 |    10.00 | Using where                        |
 +----+-------------+-------+------------+--------+---------------+-------------+---------+-----------------+------+----------+------------------------------------+
 3 rows in set, 1 warning (0.01 sec)
 ```

**Observations:**
  * index works because `LIKE 'text%'` (startsWith) query, it would not work for `LIKE '%text%'` type of query
  * full text searches needed for `LIKE '%text%'` type of queries (`FULLTEXT INDEX` !?)

Insights:
  * "aha-moment": foreign keys on `searchedTags` where not created initially (wrong setup query!?) - might have avoided indexes (false start...)
  * InnoDB and MYISAM, are storage engines for MySQL (different locking implementation)


##### Searches that had a result (matched = true) and tagged with “outgoing payment” and “high net worth individual”
`148 rows in set (1.84 sec)`
*...same as before*
`CREATE INDEX search_client_id ON searches (clientId);`
`CREATE INDEX tag_client_id ON tags (clientId);`


#### Summary
```
searches 2M
clients 700
users 4K
tags 20K
searchedTags 9M
```
All sample searches time: < 0.01 s


## Part 8: Create same order of magnitude as requested sample data & Analyse again and improve

Attempt:
```
searches 13M
clients 850
users 4K
tags 24K
searchedTags 9M
```
All sample searches time: < 0.02 s

Questions:
How many writes do we expect in the database?




# Problem 2

## Script solution

  * script to extract url for each senator from master page
  * script to extract relevant data from child pages given their urls
  * possible technologies: `curl + grep`, `javascript`

Given the constraints (site updates), as stated, scripts solution is not good

## Machine Learning solution

  #### Note: Team (me) is not knowledgeable in ML solutions, training needed


#### Breaking the problem down into iterative subparts

  1. Quick training on ML solutions
  2. Define problem from ML point of view
  3. Tools
  4. Prepare data
  5. Evaluate algorithms
  6. Train chosen ML algo
  7. Check results, improve and present


### 1. Quick training on ML solutions

  *Note: deeper training would be needed but there are time constraints*

  #### The iris flowers dataset problem
    * “hello world” dataset in machine learning and statistics
    * 150 records of measurements
    * algo predicts iris flower class based on measurements with high accuracy
    * study solution and try to apply a similar one


### 2. Define problem from ML point of view  

  #### Set of raw data extracted from senator's pages to be transformed into set of senators relevant info


### 3. Tools

   * `javascript` for extracting raw data
   * python and libraries for ML algo: `scipy numpy matplotlib pandas sklearn`


### 4. Prepare data

  40 records dataset - more needed but there are time constraints

  #### js script to extract raw data from URLs:
    * GET content from each URL
    * remove irrelevant tags from content: `a`, `script`, `meta`, `nav`, `footer`
    * remove all tags, keep just text

  #### Sample output for a page:

  ```
  "	Senator Brian Burston
– Parliament of Australia
Senator Brian Burston
Senator for
New South Wales
Party
United Australia Party
Chamber
Senate
Seating Plan
Electorate Office
(Principal Office)
Suite 103, Level 1
43-43a The Boulevarde
Toronto, NSW, 2283
Postal address
Suite 103, Level 1, 43-43a The Boulevarde
Toronto, NSW, 2283
Telephone:
(02) 4959 1044
Fax:
(02) 4950 4833
Toll Free:
1300 498 639
Parliament Office
PO Box 6100
Senate
Parliament House
Canberra ACT 2600
Telephone:
(02) 6277 3197
Fax:
(02) 6277 5812
Connect with Senator Brian Burston
Email
Websites
Alternative URL
State/Territory
Speeches
Biography
Committee service
Senate Select: Red Tape from 09.11.2016 to 12.09.2018; Funding for Research into Cancers with Low Survival Rates from 01.12.2016; Charity Fundraising in the 21st Century from 27.06.2018.
Parliamentary party positions
Pauline Hanson's One Nation Party Whip from 1.9.16
Party positions
PHON National Director, 2001-03
Personal
Born 25.02.1948, Cessnock, Australia.Married
Qualifications and occupation before entering Federal Parliament
Dip. T.Assoc. Dip. Structural EngineeringApprentice Boilermaker, 1963-68Boilermaker, 1968-70Draftsman (trainee), 1971-73Design Draftsman, 1973-78Teacher of Engineering Drawing, 1978-82Lecturer, Teacher Education, 1982-87Architectural building design consultant, 1987-2016
Local government service
Councillor, Cessnock City Council from 1987 to 1999Deputy Mayor, Cessnock City Council, 1997
Senate
House of Representatives
Get informed
Bills
Committees
Get involved
Visit Parliament
Website features
Parliamentary Departments
```

Script:

```
const https = require('https');
const fs = require('fs');
const { JSDOM } = require("jsdom");

function sanitizeContent(content) {
  const dom = new JSDOM(content);
  const document = dom.window.document;

  const div = document.createElement('div');
  div.innerHTML = content;

  function removeElementsByTagName(tagName) {
    const elements = div.getElementsByTagName(tagName);

    while (elements[0])
       elements[0].parentNode.removeChild(elements[0])
  };

  removeElementsByTagName('a');
  removeElementsByTagName('script');
  removeElementsByTagName('meta');
  removeElementsByTagName('nav');
  removeElementsByTagName('footer');

  const res =
    (div.textContent || div.innerText || "")
    .replace(/^\s*[\r\n]/gm, '')
    .replace(/^ +/gm, '');

  return res;
};

function getContentFromUrls(urls) {

  urls.forEach(url => {

    https.get(url, (res) => {
      let rawData = '';
      res.on('data', chunk => { rawData += chunk; });
      res.on('end', () => {
        try {
          const res = sanitizeContent(rawData);
          fs.appendFileSync('dataset.csv', `"${res}", "placeholder ${res}"\n`);
        } catch (e) {
          console.error('end error', e.message);
        }
      });
    }).on('error', (e) => {
      console.error(`Got error: ${e.message}`);
    });

  });

};

getContentFromUrls([
  'https://www.aph.gov.au/Senators_and_Members/Parliamentarian?MPID=N26',
  'https://www.aph.gov.au/Senators_and_Members/Parliamentarian?MPID=273829',
  'https://www.aph.gov.au/Senators_and_Members/Parliamentarian?MPID=G0D',
  'https://www.aph.gov.au/Senators_and_Members/Parliamentarian?MPID=HZB',
  ... (40 urls)
]);
```
#### Expected extracted info from a page:

```
Senator the Hon Doug Cameron| Parliamentary service| Elected to the Senate for New South Wales 2007 (term began 1.7.2008), 2013 and 2016.| Born 27.1.1951"
```

Extraction method for the 40 records dataset: manual (scripting needed)

#### Load the data with `pandas` and take a look at it:

```
names = ['content', 'person-details']
dataset = pandas.read_csv('./dataset.csv', names=names)

print(dataset.shape)
print(dataset.head(10))
print(dataset.describe())
```

Output:
```
content                                     person-details
0  \tSenator Brian Burston\n– Parliament of Austr...           "Senator Brian Burston| Born 25.02.1948"
1  \tSenator the Hon Doug Cameron\n– Parliament o...   "Senator the Hon Doug Cameron| Parliamentary ...
2  \tSenator the Hon Ian Macdonald\n– Parliament ...   "Senator the Hon Ian Macdonald| Positions| Ch...
3  \tSenator Gavin Marshall\n– Parliament of Aust...   "Senator Gavin Marshall| Positions| Chair of ...
4  \tSenator the Hon Simon Birmingham\n– Parliame...   "Senator the Hon Simon Birmingham| Positions|...
5  \tSenator David Leyonhjelm\n– Parliament of Au...   "Senator David Leyonhjelm| Positions| Chair o...
6  \tSenator David Bushby\n– Parliament of Austra...   "Senator David Bushby| Positions| Chief Gover...
7  \tSenator the Hon Jacinta Collins\n– Parliamen...   "Senator the Hon Jacinta Collins| Positions| ...
8  \tSenator Slade Brockman\n– Parliament of Aust...   "Senator Slade Brockman| Positions| Chair of ...
9  \tSenator Peter Georgiou\n– Parliament of Aust...           "Senator Peter Georgiou| Born 13.1.1974"
     content                             person-details
count                                                  40                                         40
unique                                                 40                                         40
top     \tSenator the Hon Concetta Fierravanti-Wells\n...   "Senator Peter Georgiou| Born 13.1.1974"
freq                                                    1                                          1
```

Split the data into training set and validation set (80/20), for Machine Learning algorithms:
`X_train, X_validation, Y_train, Y_validation = model_selection.train_test_split(X, Y, test_size=validation_size, random_state=seed)`



### 5. Evaluate algorithms

  * Try to use same methods as for the iris problem.... FALSE START
  * We have different a kind of data with different representation (text vs numbers) - cannot use the same method

  * Taking a step back - **further reading needed**

  * Identified the problem as similar to **Name Entity Recognition***, e.g.:

```
  Input: Google bought IBM for 10 dollars. Mike was happy about this deal.

Output:

Google              	ORGANIZATION
IBM                 	ORGANIZATION
10 dollars          	MONEY
Mike                	PERSON
```

  * multiple `easy-to-plug-in` technologies that can be used for this problem

##### Peak at `https://github.com/philipperemy/Stanford-NER-Python`

  * try it out with a raw data record -> output:

```
python main.py -f input.txt
Brian Burston       	PERSON
Parliament of Australia	ORGANIZATION
Brian Burston       	PERSON
New South Wales Party United Australia Party Chamber Senate Seating Plan Electorate Office	ORGANIZATION
Principal Office    	ORGANIZATION
103                 	NUMBER
1                   	NUMBER
Boulevarde Toronto  	ORGANIZATION
NSW                 	ORGANIZATION
2283                	DATE
103                 	NUMBER
1                   	NUMBER
Boulevarde Toronto  	ORGANIZATION
NSW                 	ORGANIZATION
2283                	DATE
Parliament Office PO Box 6100 Senate Parliament House	ORGANIZATION
Canberra            	LOCATION
2600                	DATE
Brian Burston       	PERSON
Senate              	ORGANIZATION
09.11.2016          	DATE
12.09.2018          	DATE
01.12.2016          	DATE
Charity Fundraising 	ORGANIZATION
the 21st Century    	DATE
27.06.2018          	DATE
Pauline Hanson      	PERSON
One                 	NUMBER
1.9.16              	DATE
PHON National       	ORGANIZATION
2001-03             	DATE
25.02.1948          	DATE
Federal Parliament  	ORGANIZATION
1987-2016           	DURATION
Cessnock City Council	ORGANIZATION
1987                	DATE
Cessnock City Council	ORGANIZATION
1997                	DATE
Senate House of Representatives	ORGANIZATION
```

#### Conclusion: This looks better, maybe it's a better start.

### 6. Train chosen ML algo  (TO DO)
### 7. Check results, improve and present (TO DO)


# Problem 3

## Design and implementation of call queue

### SPECS:

#### Previously implemented parts - existing calling system that can:
* take incoming calls
* associate incoming calls with existing entities, or create new ones
* makes outgoing calls
* break down by teams

 #### Needed implementation following rules:
* incoming call is played a message + music
* call goes to queue
* when agent becomes available he is called for the queue:
    * he picks up -> call is dequeued and transferred to him
    * he declines -> call is not dequeued but will not be transferred to this agent again

* when time in queue expires -> call goes to voicemail and is marked as missed
    * at end of business hours -> all queued calls go to voicemail (different message) and are marked as missed
    * when caller hangs up -> call is marked as missed
    * when all agents decline -> send to voicemail

* Interactive Voice Response (IVR) - caller can press a key to:
    * request transfer to voicemail
    * request being called back

* Configurable dequeuing strategies when multiple agents are available:
  * Round Robin: agents are called in turn for taking queued calls
  * Everybody: all agents are called for the oldest queued call

* Queue priority system:
  * when agent becomes available and a known caller associated to him is queued - he takes the call with priority, otherwise he takes the oldest call
  ...

  #### Other system notes:
    * high availability: multiple servers can receive calls and related events - centralized queue is needed


### Implementation:
* message queue
* database queries: one statement, lock calls for dequeue for one/many agents
* telephony provider API
