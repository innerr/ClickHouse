HTTP interface
==============

The HTTP interface lets you use ClickHouse on any platform from any programming language. We use it for working from Java and Perl, as well as shell scripts. In other departments, the HTTP interface is used from Perl, Python, and Go. The HTTP interface is more limited than the native interface, but it has better compatibility.

By default, clickhouse-server listens for HTTP on port 8123 (this can be changed in the config).
If you make a GET / request without parameters, it returns the string "Ok" (with a line break at the end). You can use this in health-check scripts.

.. code-block:: bash

    $ curl 'http://localhost:8123/'
    Ok.

Send the request as a URL 'query' parameter, or as a POST. Or send the beginning of the request in the 'query' parameter, and the rest in the POST (we'll explain later why this is necessary). URL length is limited by 16KB, this limit should be taken into account when sending long queries in the 'query' parameter.

If successful, you receive the 200 response code and the result in the response body.
If an error occurs, you receive the 500 response code and an error description text in the response body.

When using the GET method, 'readonly' is set. In other words, for queries that modify data, you can only use the POST method. You can send the query itself either in the POST body, or in the URL parameter.

Examples:

.. code-block:: bash

    $ curl 'http://localhost:8123/?query=SELECT%201'
    1

    $ wget -O- -q 'http://localhost:8123/?query=SELECT 1'
    1

    $ GET 'http://localhost:8123/?query=SELECT 1'
    1

    $ echo -ne 'GET /?query=SELECT%201 HTTP/1.0\r\n\r\n' | nc localhost 8123
    HTTP/1.0 200 OK
    Connection: Close
    Date: Fri, 16 Nov 2012 19:21:50 GMT

    1

As you can see, curl is not very convenient because spaces have to be URL-escaped. Although wget escapes everything on its own, we don't recommend it because it doesn't work well over HTTP 1.1 when using keep-alive and Transfer-Encoding: chunked.

.. code-block:: bash

    $ echo 'SELECT 1' | curl 'http://localhost:8123/' --data-binary @-
    1

    $ echo 'SELECT 1' | curl 'http://localhost:8123/?query=' --data-binary @-
    1

    $ echo '1' | curl 'http://localhost:8123/?query=SELECT' --data-binary @-
    1

If part of the query is sent in the parameter, and part in the POST, a line break is inserted between these two data parts.
Example (this won't work):

.. code-block:: bash

    $ echo 'ECT 1' | curl 'http://localhost:8123/?query=SEL' --data-binary @-
    Code: 59, e.displayText() = DB::Exception: Syntax error: failed at position 0: SEL
    ECT 1
    , expected One of: SHOW TABLES, SHOW DATABASES, SELECT, INSERT, CREATE, ATTACH, RENAME, DROP, DETACH, USE, SET, OPTIMIZE., e.what() = DB::Exception

By default, data is returned in TabSeparated format (for more information, see the "Formats" section).
You use the FORMAT clause of the query to request any other format.

.. code-block:: bash

    $ echo 'SELECT 1 FORMAT Pretty' | curl 'http://localhost:8123/?' --data-binary @-
    ┏━━━┓
    ┃ 1 ┃
    ┡━━━┩
    │ 1 │
    └───┘

The POST method of transmitting data is necessary for INSERT queries. In this case, you can write the beginning of the query in the URL parameter, and use POST to pass the data to insert. The data to insert could be, for example, a tab-separated dump from MySQL. In this way, the INSERT query replaces LOAD DATA LOCAL INFILE from MySQL.

Examples:

Creating a table:

.. code-block:: bash

    echo 'CREATE TABLE t (a UInt8) ENGINE = Memory' | POST 'http://localhost:8123/'

Using the familiar INSERT query for data insertion:

.. code-block:: bash

    echo 'INSERT INTO t VALUES (1),(2),(3)' | POST 'http://localhost:8123/'

Data can be sent separately from the query:

.. code-block:: bash

    echo '(4),(5),(6)' | POST 'http://localhost:8123/?query=INSERT INTO t VALUES'

You can specify any data format. The 'Values' format is the same as what is used when writing INSERT INTO t VALUES:

.. code-block:: bash

    echo '(7),(8),(9)' | POST 'http://localhost:8123/?query=INSERT INTO t FORMAT Values'

To insert data from a tab-separated dump, specify the corresponding format:

.. code-block:: bash

    echo -ne '10\n11\n12\n' | POST 'http://localhost:8123/?query=INSERT INTO t FORMAT TabSeparated'

Reading the table contents. Data is output in random order due to parallel query processing:

.. code-block:: bash

    $ GET 'http://localhost:8123/?query=SELECT a FROM t'
    7
    8
    9
    10
    11
    12
    1
    2
    3
    4
    5
    6

Deleting the table.

.. code-block:: bash

    POST 'http://localhost:8123/?query=DROP TABLE t'

For successful requests that don't return a data table, an empty response body is returned.

You can use compression when transmitting data. The compressed data has a non-standard format, and you will need to use a special compressor program to work with it  (`sudo apt-get install compressor-metrika-yandex`).

If you specified 'compress=1' in the URL, the server will compress the data it sends you.
If you specified 'decompress=1' in the URL, the server will decompress the same data that you pass in the POST method.

You can use this to reduce network traffic when transmitting a large amount of data, or for creating dumps that are immediately compressed.

You can use the 'database' URL parameter to specify the default database.

.. code-block:: bash

    $ echo 'SELECT number FROM numbers LIMIT 10' | curl 'http://localhost:8123/?database=system' --data-binary @-
    0
    1
    2
    3
    4
    5
    6
    7
    8
    9

By default, the database that is registered in the server settings is used as the default database. By default, this is the database called 'default'. Alternatively, you can always specify the database using a dot before the table name.

The username and password can be indicated in one of two ways:

1. Using HTTP Basic Authentication. Example:
.. code-block:: bash

    echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-

2. In the 'user' and 'password' URL parameters. Example:
.. code-block:: bash

    echo 'SELECT 1' | curl 'http://localhost:8123/?user=user&password=password' -d @-

3. Using 'X-ClickHouse-User' and 'X-ClickHouse-Key' headers. Example:
.. code-block:: bash

	echo 'SELECT 1' | curl -H "X-ClickHouse-User: user" -H "X-ClickHouse-Key: password"  'http://localhost:8123/' -d @-
	

If the user name is not indicated, the username 'default' is used. If the password is not indicated, an empty password is used.
You can also use the URL parameters to specify any settings for processing a single query, or entire profiles of settings. Example:
`http://localhost:8123/?profile=web&max_rows_to_read=1000000000&query=SELECT+1`

For more information, see the section "Settings".

.. code-block:: bash

    $ echo 'SELECT number FROM system.numbers LIMIT 10' | curl 'http://localhost:8123/?' --data-binary @-
    0
    1
    2
    3
    4
    5
    6
    7
    8
    9

For information about other parameters, see the section "SET".

In contrast to the native interface, the HTTP interface does not support the concept of sessions or session settings, does not allow aborting a query (to be exact, it allows this in only a few cases),  and does not show the progress of query processing. Parsing and data formatting are performed on the server side, and using the network might be ineffective.

The optional 'query_id' parameter can be passed as the query ID (any string). For more information, see the section "Settings, replace_running_query".

The optional 'quota_key' parameter can be passed as the quota key (any string). It can also be passed as 'X-ClickHouse-Quota' header. For more information, see the section "Quotas".

The HTTP interface allows passing external data (external temporary tables) for querying. For more information, see the section "External data for query processing".
