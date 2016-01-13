There are many tutorials on PDO already, but unfortunately, they all written by people who have no clue. Including tutorials from tutsplus, culttt, lynda, phpro and many others (in case you're curious what's `wrong with them, you may learn it below). The only exceptions are [phptherightway.com](http://www.phptherightway.com/#pdo_extension) and [hashphp.org](http://wiki.hashphp.org/PDO_Tutorial_for_MySQL_Developers), but they lack a lot of important information. As a result, half of PDO features remain in obscurity and never used by average PHP developer.

Unlike those, this tutorial is written by someone who used PDO for many years, dug through it, and answered thousands questions on Stack Overflow (the [sole gold PDO badge bearer](http://stackoverflow.com/help/badges/4220/pdo)), and thus was able to trace common delusions and patterns of misuse, as well as to reveal best practices and useful tricks. 

Mysql is used for all examples, but, save for specific cases, they applicable for any driver supported.

###Why PDO?#why

First things first. Why PDO at all? 

PDO is a [Database Abstraction Layer](https://en.wikipedia.org/wiki/Database_abstraction_layer). The abstraction, however, is two-fold: one is widely known but less significant, while another is obscure but of most importance.

Everyone knows that PDO offers unified interface to access [many different databases](http://php.net/manual/en/pdo.drivers.php). Although this feature is magnificent by itself, it doesn't make a big deal for the particular application, where only one database backend is used anyway. And, despite of some rumors, it is impossible to switch database backends by means of changing a single line in PDO config - due to different SQL flavors. To do so, one need to use an averaged language like [DQL](http://doctrine-orm.readthedocs.org/projects/doctrine-orm/en/latest/reference/dql-doctrine-query-language.html). Thus, for the average LAMP developer this point is rather insignificant, and to him PDO is just a more complicated version of familiar `mysql(i)_query()` function. But it is not. It is much, much more. 

PDO abstracts not only a database API, but also basic operations that otherwise have to be repeated hundreds of times in every application, making your code extremely <abbr title="Write Everything Twice">WET</abbr>. Unlike `mysql` and `mysqli`, both of which are low level bare API, not intended to be used directly (but only as a building material for some higher level abstraction layer) `PDO` *is* such an abstraction already. Still incomplete but at least usable. 

The real PDO benefits are:

- security (prepared statements are essential)
- usability (many helper functions to automate routine operations)
- reusability (unified API to access multitude of databases, from SQLite to Oracle)

###Connecting. DSN#dsn

PDO has its own fancy connection method called [DSN](https://en.wikipedia.org/wiki/Data_source_name). Nothing complicated though - instead of one plain and simple list of options, PDO asks you to put different configuration directives in three different places:

- database driver, host, db (schema) name and charset, as well as less frequently used port and
unix_socket go into DSN;
- username and password go to constructor;
- all other options go into options array.

where DSN is a semicolon-delimited string consists of param=value pairs. Here goes an example for mysql:
    $host = '127.0.0.1';
    $db   = 'test';
    $user = 'root';
    $pass = '';
    $charset = 'utf8';

    $dsn = "mysql:host=$host;dbname=$db;charset=$charset";
    $opt = array(
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
    );
    $pdo = new PDO($dsn, $user, $pass, $opt);

Having all aforementioned variables properly set, we will have proper PDO instance in `$pdo` variable.

Important notes for the late mysql extension users: 
  
1. Unlike old `mysql_*` functions, which can be used anywhere in the code, `PDO` instance is stored in a regular  variable, which means it can be inaccessible inside functions - so, one have to make it accessible, by means of passing it via function parameters of using more advanced tecniques like IoC container. 
2. Connection have to be done only once! No connects in every function. No connects in every class constructor. Connection have to be done only **once** and then single PDO instance have to be used all the way. Otherwise multiple connections will be created, which will eventually kill your database server.
3. It is very important to *set charset through DSN* - that's the only proper way. Forget about running `SET NAMES` query manually, either via `query()` or `PDO::MYSQL_ATTR_INIT_COMMAND`.

###Error handling. Exceptions#errors

Although there are several error handling modes in PDO, the only proper one is `PDO::ERRMODE_EXCEPTION`. So, one ought to always set it as a connection option as shown in the example above.

Note that despite of a widespread delusion, <b>you should never catch errors to report them</b>. A module (like a database layer) should not report its errors, this function have to be delegated to an application-wide handler. All we need is to raise an error (in the form of exception) - so we already did. That's all. Nor should you "*always wrap your PDO operations in a `try/catch`*" like the most popular tutorial from tutsplus says.

In fact, there is nothing special about PDO exceptions, they are errors all the same. Thus you have to treat them exactly the same way as other errors. If you had an error handler before, then you shouldn't create a dedicated one for PDO. If you didn't care - it's all right too, PHP is good with basic error handling and will conduct PDO exceptions all right. 

Exceptions handling is one of the problems with PDO tutorials. Being acquainted with exceptions for the first time when started with PDO, authors consider exceptions dedicated to this library. And start diligently (but improperly) handling exceptions for PDO only. Which is utter nonsense. If one paid no special attention to any exceptions before, they shouldn't have changed their habit for PDO. If one didn't use try..catch before, they should keep with that, eventually learning how to use exceptions and when to use try..catch. 

You may want to catch PDO errors only in two cases:

1. If you are writing a wrapper for PDO, and you want to augment the error info with some additional data, like query string. In this case catch the exception, gather the required information, and re-throw another Exception.
2. If you have a certain scenario for handling errors in particular part of code. Some examples are: 

 - if error can be bypassed, you can use try..catch for this. But do not make it a habit. Empty catch in every aspect work as error suppressing operator, and so as evil it is.
 - if there is some action that have to be taken in case of failure, i.e. transaction rollback. 
 - if you are waiting for a particular error to handle. In this case catch the exception, see if the error is one you're looking for and then handle it. Otherwise re-throw it again so it can bubble up to the handler usual way.

E.g.:

    try {
        $pdo->prepare("INSERT INTO users VALUES (NULL,?,?,?,?)")->execute($data);
    } catch (\PDOException $e) {
        if ($e->getCode() == 1062) {
            // Take some action if there is a key constraint violation, i.e. duplicate name
        } else {
            throw $e;
        }
    }

But in general no dedicated treatment for PDO exceptions ever needed.

In short, to have PDO errors properly reported:

1. Set PDO in exception mode.
2. Do not use `try..catch` to report errors. 
3. Configure PHP for proper error reporting (display on development and log on production)
4. (optional) Create an error handler using set_error_handler() function if you want more flexible error reporting. 
5. Use `try..catch` only to handle errors, in case something could be done beside just reporting.

And you'll be fine. <a href="/try-catch">Further reading.</a> 

###Running queries#query

There are two ways to run a query in PDO.  
In case there are no variables going into query, you can use `query()` method. It will run your query and return special object of `PDOStatement` class which can be roughly compared to resource returned by `mysql_query()`, especially in the way you can get actual rows out of it:

    $stmt = $pdo->query('SELECT name FROM users');
    while ($row = $stmt->fetch())
    {
        echo $row['name'] . "\n";
    }

Also, `query()` method allows us to use neat method chaining, which will be shown below.

###Prepared statements. Protection from SQL injections#prepared

This is the main and the only important reason why you were deprived from your beloved `mysql_query()` and thrown into harsh world of Data Objects: `PDO` has prepared statements support out of the box. And prepared statement is the only proper way to run queries, if any variables are going into query. 

So, for the every query you run, if at least one variable have to be used in it, you have to substitute it with <b>placeholder</b>, then prepare your query, and then execute it, passing variables separately.

Long story short, it is not as hard as it seems. In most cases you need only two functions - `prepare()` and `execute()`.

First of all you have to alter your query, adding placeholders in place of variables, Say, a code like this

    $sql = "SELECT name FROM users WHERE email = '$email' AND status='$status'";
will become
    $sql = 'SELECT name FROM users WHERE email = ? AND status=?';
or
    $sql = 'SELECT name FROM users WHERE email = :email AND status=:status';

Note that PDO supports positional (`?`) and named (`:name`) placeholders. 

Having a query with placeholders, you have to prepare it, using `PDO::prepare()` method. This function will return the same `PDOStatement` object we were talking before, but <i>without any data attached to it</i>. 

Finally, to get the query executed, you must run `execute()` method of this object, passing your variables in it, in the form of array, and after that you will be able to get the resulting data out of statement (if applicable):

    $stmt = $pdo->prepare('SELECT name FROM users WHERE email = ?');
    $stmt->execute(array($email));
    // or
    $stmt = $pdo->prepare('SELECT name FROM users WHERE email = :email');
    $stmt->execute(array('email' => $email));

    $name = $stmt->fetchColumn();

As you can see, for the positional placeholders you have to supply a regular array with values,  while for the named placeholders it have to be associative array where keys have to have the same names as placeholders in the query. You cannot mix positional and named placeholders in one query. 

Note that positional placeholders let you write shorter code but sensitive to the order of arguments, which have to be exactly the same as the order of the corresponding placeholders in the query. While named placeholders make your code more verbose but allow random binding order. 

Also note that despite of a widespread delusion, no "`:`" in the keys is required. 

Passing data into `execute()` should be considered default and most convenient method. 
When this method is used, all values will be bound as **strings**, but don't worry: it's all right in every aspect, save for only one issue with `LIMIT` clause described [below](#limit).

Note that if a bound variable contains `NULL`, it will be sent to the query as is (i.e. as SQL `NULL` value). Thus a code

    $stmt = prepare('UPDATE users SET param = ? WHERE id = ?');
    $stmt->execute(array(NULL, 1));

Will produce a query

    UPDATE users SET param = NULL WHERE id = '1'

Although there are two more options for binding a variable to the query, `bindValue()` and `bindParam()` namely, they are less convenient, bloating the code with no apparent reason, as they bind variables to a query one by one. Note that `bindParam()` is binding variables by reference, so, use this function only if you have an idea what does it mean.

###Prepared statements. Multiple execution#multiple

Sometimes you can use prepared statements for the multiple execution of a prepared query. It is slightly faster than repeating the same query, as it does query parsing only once. 
This feature would have been more useful if it was possible to execute a statement prepared in another PHP instance. But alas - it is not. So, you are limited to repeating the same query only within the same instance, which is seldom needed in regular PHP scripts. Which is limiting the use of this feature to repeated inserts or updates:

    $data = array(
        1 => 1000,
        5 =>  300,
        9 =>  200,
    );
    $stmt = $pdo->prepare('UPDATE users SET bonus = bonus + ? WHERE id = ?');
    foreach ($data as $id => $bonus)
    {
        $stmt->execute([$id,$bonus]);
    }

Note that this feature is too much overrated. Not only it is needed too seldom to talk about, but the performance gain is hardly measurable - query parsing is <i>real</i> fast these times. 

###Running SELECT INSERT, UPDATE, or DELETE statements#dml

Come on guys.
There is absolutely nothing special in these queries. To PDO they all the same. It doesn't matter which query you are running. All you need is to prepare a query (any query!) with placeholders, and to execute it, sending variables separately. Either for `DELETE` and `SELECT` query the process is essentially the same. You can run any query with PDO, all the same.

###Getting data out of statement. foreach()#foreach

The most basic and direct way to get multiple rows from a statement would be `foreach()` loop. Thanks to Traversable interface, `PDOStatement` can be iterated over by using `foreach()` operator:

    $stmt = $pdo->query('SELECT name FROM users');
    foreach ($stmt as $row)
    {
        echo $row['name'] . "\n";
    }

Note that this method is memory-friendly, as it doesn't load all the resulting rows in the memory but delivers them one by one (though keep in mind this [issue](#mysqlnd)).

###Getting data out of statement. fetch()#fetch

We have seen this function already, but let's take a closer look. It fetches a single row from database, and moves the internal pointer in the result set, so consequent calls to this function will return all the resulting rows one by one. which makes this method a rough analogue to `mysq_fetch_array()` but it works slightly different way: instead of many separate functions (`mysql_fetch_assoc()`, `mysql_fetch_row()`, etc), there is only one, but its behavior can be changed by a parameter. There are many fetch modes in PDO, and we will discuss them later, but here are few for starter:

- `PDO::FETCH_NUM` returns enumerated array
- `PDO::FETCH_ASSOC` returns associative array
- `PDO::FETCH_BOTH` - both of the above
- `PDO::FETCH_OBJ` returns object
- `PDO::FETCH_LAZY` allows all three (numeric associative and object) methods without memory overhead. 

From the above you can tell that this function have to be used in two cases:

1. When only one row is expected - to get that only row.
2. When we need to process the returned data somehow before use. In this case it have to be run through usual while loop

Example:

    $row = $stmt->fetch(PDO::FETCH_ASSOC);

Will give you single row from the statement, in the form of associative array. 

Another useful mode is `PDO::FETCH_CLASS`, which can create an object of particular class

    $news = $pdo->query('SELECT * FROM news')->fetchAll(PDO::FETCH_CLASS, 'News');

will produce an array filled with objects of News class. Note that this mode is filling private properties as well, which is a bit unexpected yet quite handy.

Note that default mode is `PDO::FETCH_BOTH`, but you can change it using PDO::ATTR_DEFAULT_FETCH_MODE configuration option as shown in the connection example. Thus, once set, it can be omitted most of time.

###Getting data out of statement. fetchColumn()#fetchcolumn

A neat helper function that returns value of the singe field of returned row. Very handy when we are selecting only one field:

    $stmt = $pdo->prepare("SELECT name FROM table WHERE id=?");
    $stmt->execute(array($id));
    $name = $stmt->fetchColumn();

    $count = $pdo->query("SELECT count(*) FROM table")->fetchColumn();

###Getting data out of statement. fetchAll()#fetchall

That's most interesting function, with most astonishing features. 
Mostly thanks to its existence one can call PDO a wrapper, as this function can automate many operations otherwise performed manually. 

`PDOStatement::fetchAll()` returns an array that consists of <b>all the rows</b> returned by the query.
From this fact we can make two conclusions:

1. This function should not be used, if many rows has been selected. In such a case conventional while loop ave to be used, fetching rows one by one instead of getting them all into array at once. "Many" means more than it is suitable to be shown on the average web page. 
2. This function is mostly useful in a modern web application that never outputs data right during fetching, but rather passes it to template.

You'd be amazed, how many different formats this function can return data in (and how little an average PHP user knows of them), all controlled by `PDO::FETCH_*` variables. 
Some of them are:

####Getting plain array.

By default, this function will return just simple enumerated array consists of all the returned rows. Row formatting constants, such as `PDO::FETCH_NUM`, `PDO::FETCH_ASSOC`, `PDO::FETCH_OBJ` etc can change the row format.

    $data = $pdo->query('SELECT name FROM users')->fetchAll();
    var_export($data);
    /*
    array (
      0 => array('John'),
      1 => array('Mike'),
      2 => array('Mary'),
      3 => array('Kathy'),
    )*/

####Getting a column.

It is often very handy to get plain one-dimensional array right out of the query, if only one column out of many rows being fetched. Here you go: 

    $data = $pdo->query('SELECT name FROM users')->fetchAll(PDO::FETCH_COLUMN);
    /* array (
      0 => 'John',
      1 => 'Mike',
      2 => 'Mary',
      3 => 'Kathy',
    )*/

####Getting key-value pairs.

Also extremely useful format, when we need to get the same column, but indexed not by numbers in order byt by another field. Here goes `PDO::FETCH_KEY_PAIR` constant:

    $data = $pdo->query('SELECT id, name FROM users')->fetchAll(PDO::FETCH_KEY_PAIR);
    /* array (
      104 => 'John',
      110 => 'Mike',
      120 => 'Mary',
      121 => 'Kathy',
    )*/

Note that you have to select only two columns for this mode, first of which have to be unique.

####Getting rows indexed by unique field

Same as above, but getting not one column but full row, yet indexed by an unique field, thanks to 
`PDO::FETCH_UNIQUE` constant:

    $data = $pdo->query('SELECT * FROM users')->fetchAll(PDO::FETCH_UNIQUE);
    /* array (
      104 => array (
        'name' => 'John',
        'car' => 'Toyota',
      ),
      110 => array (
        'name' => 'Mike',
        'car' => 'Ford',
      ),
      120 => array (
        'name' => 'Mary',
        'car' => 'Mazda',
      ),
      121 => array (
        'name' => 'Kathy',
        'car' => 'Mazda',
      ),
    )*/

Note that first column selected have to be unique (in this query it is assumed that first column is id, but to be sure better list it explicitly). 

####Getting rows grouped by some field

 `PDO::FETCH_GROUP` will group rows into a nested array, where indexes will be unique values from the first columns, and values will be arrays similar to ones returmed by regular `fetchAll()`. The following code, for example, will separate boys from girls and put them into different arrays:
 
    $data = $pdo->query('SELECT sex, name, car FROM users')->fetchAll(PDO::FETCH_GROUP);
    array (
      'male' => array ( 
        0 => array (
          'name' => 'John',
          'car' => 'Toyota',
        ),
        1 => array (
          'name' => 'Mike',
          'car' => 'Ford',
        ),
      ),
      'female' => array (
        0 => array (
          'name' => 'Mary',
          'car' => 'Mazda',
        ),
        1 => array (
          'name' => 'Kathy',
          'car' => 'Mazda',
        ),
      ),
    )

So, this is the ideal solution for such a popular demand like "group events by date" or "group goods by category".

More modes are coming soon.

###Getting row count with PDO#count

You don't needed it.

Although PDO offers a function for returning the number of rows found by the query, `PDOstatement::rowCount()`, you scarcely need it. Really.

If you think it over, you will see that this is a most misused function in the web. Most of time it is used not to <i>count</i> anything, but as a mere flag - just to see if there was any data returned. But for such a case you have the data itself! Just get your data, using either `fetch()` or `fetchAll()` - and it will serve as such a flag all right! Say, to see if there is any user with such a name, just select a row:

    $stmt = $pdo->prepare("SELECT 1 FROM users WHERE name=?");
    $stmt->execute([$name]);
    $userExists = $stmt->fetchColumn();

Remember that here you don't need the <i>count</i>, the actual number of rows, but rather a boolean flag. So you got it.

Not to mention that the second most popular use case for this function should never be used at all. One should never use the `rowCount()` to count rows in database! Instead, one have to ask a database to count them, and return the result in a <b>single</b> row:

    $count = $pdo->query("SELECT count(1) FROM t")->fetchColumn();

is the only proper way.

In essence:

- if you need to know how many rows in the table, use SELECT COUNT(*) query.
- if you need to know if your query returned any data - check that data.
- if you still need to know how many rows has been returned by some query (though I hardly can imagine a case), then you can use `rowCount()` or simply call count() on the array returned by `fetchAll()` (if applicable).

Thus you could tell that the <a href="http://stackoverflow.com/a/883382/285587/">top answer for this question on Stack Overflow</a> is essentially pointless and harmful - a call to `rowCount()` could be never substituted with `SELECT count(*)` query - their purpose is essentially different, while running an extra query only to get the number of rows returned by other query makes absolutely no sense. 

###Affected rows and insert id#affected

PDO is using the same function for returning both number of rows returned by SELECT statement and number of rows affected by <abbr title="Data Manipulation Language, INSERT, UPDATE and DELETE queries">DML</abbr> queries - `PDOstatement::rowCount()`. Thus, to get the number of rows affected, just call this function after performing a query. 

Another frequently asked question is caused by the fact that mysql won't update the row, if new value is the same as old one. Thus number of rows affected could differ from the number of rows matched by the WHERE clause. Sometimes it is required to know this latter number. 

Although you can tell `rowCount(`) to return the number of rows matched instead of rows affected by setting `PDO::MYSQL_ATTR_FOUND_ROWS` option to TRUE, but, as this is a connection-only option and thus you cannot change it's behavior during runtime, you will have to stick to only one mode for the application, which could be not very convenient. 

Unfortunately, there is no PDO counterpart for the `mysql(i)_info()` function which output can be easily parsed and desired number found. This is one of minor PDO drawbacks.

An auto-generated identifier from a sequence or auto_inclement field in mysql can be obtained from the `PDO::insertId` function. An answer to a frequently asked question, "whether this function is safe to use in concurrent environment?" is positive: yes, it is safe. Being just interface to MySQL C API <a href="http://dev.mysql.com/doc/refman/5.7/en/mysql-insert-id.html">`mysql_insert_id()`</a> function it's perfectly safe. 

###Prepared statements and LIKE clause#like

Despite of the PDO's overall ease of use, there are some gotchas anyway, and I am going to explain some.

One of them is using placeholders with `LIKE` SQL clause. At first one would think that such a query will do:

    $stmt = $pdo->prepare("SELECT * FROM table WHERE name LIKE '%?%'");

but soon they will learn that it will produce an error. To understand its nature one have to understand that placeholder have to <b>represent a complete data literal only</b> - a string or a number namely. And by no means can it represent either a part of literal or some arbitrary SQL clause. So, when working with LIKE, we have to prepare our complete literal first, and then send it to the query the usual way:

    $search = "%$search%";
    $stmt  = $pdo->prepare("SELECT * FROM table WHERE name LIKE ?");
    $stmt->execute([$search]);
    $data = $stmt->fetchAll();

###Prepared statements and IN clause#in

Just like it was said above, it is impossible to substitute an arbitrary query part with a placeholder. Thus, for a comma-separated placeholders, like for `IN()` SQL operator, one must create a set of `?`s manually and put them into the query:

    $arr = array(1,2,3);
    $in  = str_repeat('?,', count($arr) - 1) . '?';
    $sql = "SELECT * FROM table WHERE column IN ($in)";
    $stm = $db->prepare($sql);
    $stm->execute($arr);
    $data = $stm->fetchAll();

###Prepared statements and table names#identifiers

PDO has no placeholder for identifiers (table and field names), so a developer must manually format them. To properly format an identifier, follow these two rules:

- Enclose identifier in backticks.
- Escape backticks inside by doubling them.

The code would be:

    $table = "`".str_replace("`","``",$table)."`";

After such formatting, it is safe to insert the `$table` variable into query.

It is also important to always check dynamic identifiers against a list of allowed values. Here is a brief example:

    $orders  = array("name","price","qty"); //field names
    $key     = array_search($_GET['sort'],$orders); // see if we have such a name
    $orderby = $orders[$key]; //if not, first one will be set automatically. smart enuf :)
    $query   = "SELECT * FROM `table` ORDER BY $orderby"; //value is safe

###A problem with LIMIT clause#limit

Another problem related to the SQL `LIMIT` clause. When in emulation mode (which is on by default), PDO substitutes placeholders with actual data, instead of sending it separately. And with "lazy" binding (using array in `execute()`), PDO treats every parameter as a string. As a result, the prepared `LIMIT ?,?` query becomes `LIMIT '10', '10'` which is invalid syntax that causes query to fail.

There are two solutions:

One is [turning emulation off](#emulation) (as MySQL can sort all placeholders out properly). To do so one can run this code:

    $conn->setAttribute( PDO::ATTR_EMULATE_PREPARES, false );

And parameters can be kept in execute():

    $conn->setAttribute( PDO::ATTR_EMULATE_PREPARES, false );
    $stmt = $pdo->prepare('SELECT * FROM table LIMIT ?, ?');
    $stmt->execute([$offset, $limit]);
    $data = $stmt->fetchAll();

Another way would be to bind these variables explicitly while setting the proper param type:

    $stmt = $pdo->prepare('SELECT * FROM table LIMIT ?, ?');
    $stmt->bindParam(1, $offset,PDO::PARAM_INT);
    $stmt->bindParam(2, $limit,PDO::PARAM_INT);
    $stmt->execute();
    $data = $stmt->fetchAll();

One peculiar thing about `PDO::PARAM_INT`: for some reason it does not enforce the type casting. Thus, using it on a number that has a string type will cause the aforementioned error: 

    $stmt = $pdo->prepare("SELECT 1 LIMIT ?");
    $stmt->bindValue(1, "1", PDO::PARAM_INT);
    $stmt->execute();

But change `"1"` in the example to `1` - and everything will go smooth.

###Optional criteria. Faceted search#faceted

Another frequently asked question regarding PDO is how to add optional criteria to the query. There are two general ways to do so:

One is just traditional approach of conditional building.

Another is a smart one, 

    SELECT * FROM people 
    WHERE (:fname is null OR fname = :fname)
    AND (:lname is null OR lname = :lname)
    AND (:age is null OR age = :age)
    AND (:sex is null OR sex = :sex )

###Calling stored procedures in PDO#call

There is one thing about stored procedures any programmer stumbles upon at first: every stored procedure always return <b>one extra result set</b>: one (or many) results with actual data and one just empty. Which means if you try call it as a regular query and then proceed to another query, then <b>"Cannot execute queries while other unbuffered queries are active"</b> error will occur, because you have to clear that extra empty result first. Thus, after calling a stored procedure that is intended to return only one result set, just call <a href="http://php.net/manual/en/pdostatement.nextrowset.php">`PDOStatement::nextRowset()`</a> once (of course after fetching all the returned data from statement, or it will be discarded):

    $stmt = $pdo->prepare("CALL bar()");
    $stmt->execute();
    $data = $stmt->fetchAll();
    $stmt->nextRowset();
 
While for the stored procedures returning many result sets the behavior is more intuitive: 

    $stmt = $pdo->prepare("CALL foo()");
    $stmt->execute();
    do {
        $data = $stmt->fetchAll();
        var_dump($data);
    } while ($stmt->nextRowset() && $stmt->columnCount());

However, as you can see here is another trick have to be used: remember that extra result set? It  is so essentially <i>empty</i> that even an attempt to fetch from it will produce an error. So, we cannot use just `while ($stmt->nextRowset())`. Instead, we have to check also for empty result. For which purpose `PDOStatement::columnCount()` is just excellent.

Note that this is one of essential differences between old mysql ext and modern libraries: after calling a stored procedure with `mysql_query()` there was no way to continue working with the same connection, because there is no `nextResult(`) function for `mysql ext`. One had to close the connection and then open a new one again in order to run other queries after calling a stored procedure. 

###Running multiple queries with PDO#multiquery

When in emulation mode, PDO can run mutiple queries in the same statement, either via query() or `prepare()/execute()`. To access the result of the other queries one have to use <a href="http://php.net/manual/en/pdostatement.nextrowset.php">`PDOStatement::nextRowset()`</a>:  

    $stmt = $pdo->prepare("SELECT ?;SELECT ?");
    $stmt->execute([1,2]);
    do {
        $data = $stmt->fetchAll();
        var_dump($data);
    } while ($stmt->nextRowset());

Within this loop you'll be able to gather all the related information from the every query, like affected rows, auto-generated id or errors occurred. 

It is important to understand that at the point of `execute()` PDO will report error **for the first query only**. But if error occurred at any of consequent queries, to get that error one have to iterate over results. Despite of some <a href="https://bugs.php.net/bug.php?id=61613">ignorant opinions</a>, PDO can not and should not report all the errors at once. Some people just cannot grasp the problem at whole, and don't understand that error message is not the only outcome from the query. There could be a dataset returned, or some metadata like insert id. To get these, one have to iterate over resultsets, one by one. But to be able to throw an error immediately, PDO would have to iterate automatically, and thus **discard some results**. Which would be a plain nonsense.

Unlike `mysqli_multi_query()` PDO doesn't make an asynchronous call, so you can't  "fire and forget" - send bulk of queries to mysql and close connection, PHP will wait until last query gets executed.

###Emulation mode. PDO::ATTR_EMULATE_PREPARES#emulation

One of the most controversial PDO configuration options is `PDO::ATTR_EMULATE_PREPARES`. What does it do? PDO can run your queries in two ways: 

1. It can use a <b>real</b> prepared statement:  
When prepare() is called, your query with placeholders gets sent to mysql as is, with all the question marks you put in (in case named placeholders are used, they are substituted with ?s as well), while actual data goes later, when execute() is called. 
2. It can use <b>emulated</b> prepared statement, when your query is sent to mysql as proper SQL, with all the data in place, <b>properly formatted</b>. In this case only one roundtrip to database happens, with `execute()` call.

So you can conclude that aforementioned directive changes the behavior described above. Note that for mysql driver emulation mode is `ON` by default.

Both methods has their drawbacks and advantages but, and - I have to stress on it - both being <b>equally secure</b>, if used properly. Despite of rather blatant tone of the popular <a href="http://stackoverflow.com/a/12202218/285587">article on Stack Overflow</a>, in the end it says that <b>if you are using supported versions of PHP and MySQL properly, you are 100% safe</b>. It is frowned upon when a security expert makes such a racket on the bugs that has been already fixed long time ago.  
All you have to do is to set encoding in the DSN, as it shown in the [example above](#dsn), and your emulated prepared statements will be as secure as real ones. 

Other issues with emulation mode as follows:

When emulation mode is `ON`, one can use a handy feature of named prepared statements - a placeholder with same name could be used any number of times in the same query, while corresponding variable have to be bound only once. For some obscure reason this functionality is disabled when emulation mode is off:

    $stmt = $pdo->prepare("SELECT * FROM t WHERE foo LIKE :search OR bar LIKE :search");
    $stmt->execute(['search'] => "%$search%");`

Also, when emulation is `ON`, PDO is able to run multiple queries in one prepared statement.

Also, as native prepared statements support only certain query types, you can run some queries with prepared statements only when emulation is `ON`. The following code will return table names in emulation mode and error otherwise:

    $stmt = $pdo->prepare("SHOW TABLES LIKE ?");
    $stmt->execute(["%$name%"]);
    var_dump($stmt->fetchAll());

On the other hand, when emulation mode is `OFF`, one could bother not with parameter types, as mysql will sort types properly. Thus, even string can be bound to LIMIT parameters, as it was noted in the corresponding chapter. 

It's hard to decide which mode have to be used, but for usability sake I would rather turn it `OFF`, to avoid a hassle with `LIMIT` clause. Other issues could be considered negligible in comparison. 

###Mysqlnd and buffered queries. Huge datasets. #mysqlnd

There is one thing called [buffered queries](http://php.net/manual/en/mysqlinfo.concepts.buffering.php). Although you probably didn't notice it, you were using them all the way. Unfortunately, here are bad news for you: unlike old PHP versions, where you were using buffered queries for free, modern versions built upon [mysqlnd driver](http://jpauli.github.io/2014/07/21/php-and-mysql-communication-mysqlnd.html) won't let you to do that anymore:

>When using libmysqlclient as library PHP's memory limit won't count the memory used for result sets unless the data is fetched into PHP variables. **With mysqlnd the memory accounted for will include the full result set.**

which means when mysqlnd is used, **buffered queries burden up the script memory**:

    $pdo->setAttribute(PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, FALSE);
    $stmt = $pdo->query("SELECT * FROM Board");
    $mem = memory_get_usage();
    while($row = $stmt->fetch());
    echo "Memory used: ".round((memory_get_usage() - $mem) / 1024 / 1024, 2)."M\n";

    $pdo->setAttribute(PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, TRUE);
    $stmt = $pdo->query("SELECT * FROM Board");
    $mem = memory_get_usage();
    while($row = $stmt->fetch());
    echo "Memory used: ".round((memory_get_usage() - $mem) / 1024 / 1024, 2)."M\n";

will give you (for my data)

    Memory used: 0.02M
    Memory used: 2.39M

which means that with buffered query the memory is consumed **even if you're fetching rows one by one!** 

So, keep in mind that if you are selecting a really huge amount of data, always set `PDO::MYSQL_ATTR_USE_BUFFERED_QUERY` to `FALSE`. 

Of course, there are some drawbacks, though minor ones: with unbuffered query you can't use `rowCount()` method (which is useless, as we learned [above](#count)) and moving (seeking) the current result pointer which is useless too.

###Making things simpler#func

There is a neat thing called method chaining. If some function returns an object, we can call this object's method, just by chaining it to previous:

    $ids = $pdo->query("SELECT id FROM t WHERE foo=1")->fetchAll(PDO::FETCH_COLUMN);

Neat, simple and dry! 

The same approach could be also used with prepare() and DML queries (which do not return the resultset): 

    $pdo->prepare("INSERT INTO t VALUES (?,?,?,?)")->execute($data);

- again, neat one-liner!

To my grief, you can't do it with `SELECT` queries. Just because `execute()` doesn't return an object but, silly boolean. In case you are using exceptions it's just a nonsense! There is not a single reason for execute to return a boolean! In case of error an exception would be raised and there will be no need to check `execute()` result manually. So, it can return the same object all right. But at the moment it doesn't :'(

Anyway, being programmers, we can overcome any inconvenience. For this we can write a function to make things even neater and shorter

    function pdo($sql, $data=[]) 
    {
        global $pdo; // you can add a call to your favorite IoC here.
        $stmt = $pdo->prepare($sql);
        $stmt->execute($data);
        return $stmt;
    }

although so simple, this function can greatly improve your experience. Say, we need a user name from id. With raw PDO three lines of code will be inevitable:

    $stmt = $pdo->prepare("SELECT email FROM users where id=?");
    $stmt->execute([$id]);
    $name = $stmt->fetchColumn();

but with our function it's one short line:

    $name = pdo("SELECT email FROM users where id=?", [$id])->fetchColumn();

If you aren't satisfied with such a primitive approach, there is a more sophisticated solution, <a href="http://phpdelusions.net/pdo/pdo_wrapper">Simple yet efficient PDO wrapper</a>

###What's Wrong with other guides#guides

The problem with these "tutorials" is that they are written by people who never laid hands on PDO in reality, nor used PDO for a long time. In essence they are just trying to retell PHP manual in other words, keeping all the mistakes and superstitions in place, and failing to highlight most important parts. Their essential fallacies are:

- overall dullness and uselessness. A tutorial from phpro is the best example. They fill vast spaces with just copy-pasted connection methods for different drivers, which is essentially useless for learning PDO, as for a particular user only one connector is needed. They just don't understand the difference between a tutorial and a reference book. They also spend a lot of time explaining useless things like error modes other than Exception and functions like bindValue() or exec()
- recitation of old superstitions, namely telling you to catch every single error and to echo it out, or to substitute a rowCount with <b>extra query(!!!)</b> to get the number of rows.
- a total failure in explaining PDO benefits.  
- a total failure in explaining PDO drawbacks as well as possible solutions for them. 
