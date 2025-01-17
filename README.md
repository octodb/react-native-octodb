<p align="center"><img width="50%" src="https://github.com/octodb/docs/blob/master/images/octodb-logo.jpg" alt="OctoDB logo"></p>

<h1 align="center">React Native OctoDB</h1>

OctoDB plugin for React Native (Android and iOS)

This plugin works as a wrapper over the OctoDB native library. It can be used with both the free and the full versions

This is a fork of [react-native-sqlite-storage](https://github.com/andpor/react-native-sqlite-storage)

Main differences:

* Links to the OctoDB native library
* Query parameters are not converted to strings

Features:

* iOS and Android supported via identical JavaScript API
* SQL transactions
* JavaScript interface via callbacks or Promises
* Pre-populated SQLite database import from application bundle and sandbox (for dbs that do not use OctoDB)


# Installation

```
# using yarn
yarn add react-native-octodb react-native-udp

# using npm
npm install react-native-octodb react-native-udp
```

Then run:

```
cd ios && pod install && cd ..
```

For React Native 0.59 and below (manual linking) please follow these [instructions](instructions/INSTALL.md)


## Native Libraries

To install the free version of OctoDB native libraries, execute the following:

```
wget http://octodb.io/download/octodb.aar
wget http://octodb.io/download/octodb-free-ios-native-libs.tar.gz
tar zxvf octodb-free-ios-native-libs.tar.gz lib
mv lib node_modules/react-native-octodb/platforms/ios/
mv octodb.aar node_modules/react-native-octodb/platforms/android/
```

When moving to the full version just copy the libraries to the respective folders as done above,
replacing the existing files.


## How to Use

Here is an example code:

```javascript
var SQLite = require('react-native-octodb')

on_error = (err) => {
  console.log("Error:", err);
}

on_success = () => {
  console.log("SQL executed");
}

on_db_open = () => {
  console.log("The database was opened");
}

// open the database
var uri = "file:test.db?node=secondary&connect=tcp://server:port";
var db = SQLite.openDatabase({name: uri}, on_db_open, on_error);

db.on('error', on_error);  // this should be the first callback

db.on('not_ready', () => {
  // the user is not logged in. show the login screen (and do not access the database)
  ...
});

db.on('ready', () => {
  // the user is already logged in or the login was successful. show the main screen
  ...
});

db.on('sync', () => {
  // the db received an update. update the screen with new data
  show_items();
});

insert_items = () => {
  db.transaction((tx) => {
    // CREATE TABLE IF NOT EXISTS tasks (id INTEGER PRIMARY KEY, name, done, row_owner)
    tx.executeSql("INSERT INTO tasks (name,done) VALUES ('Learn React Native',1)", []);
    tx.executeSql("INSERT INTO tasks (name,done) VALUES ('Use SQLite',1)", []);
    tx.executeSql("INSERT INTO tasks (name,done) VALUES ('Test OctoDB',0)", []);
  }, () => {
    // success callback = transaction committed
    show_items();
  }, on_error);
}

show_items = () => {
  db.executeSql('SELECT * FROM tasks', [], (result) => {
    // Get rows with Web SQL Database spec compliance
    var len = result.rows.length;
    for (let i = 0; i < len; i++) {
      let task = result.rows.item(i);
      console.log(`Task: ${task.name}, done: ${task.done}`);
    }
    // Alternatively, you can use the non-standard raw method
    /*
    let tasks = result.rows.raw();  // shallow copy of the rows Array
    tasks.forEach(row => console.log(`Task: ${task.name}, done: ${task.done}`));
    */
  });
}
```

### Working Examples

* [Using callbacks](test/index.callback.js)
* [Using Promises](test/index.promise.js)

These example apps can be used with the AwesomeProject generated by React Native.
All you have to do is to copy one of those files into your AwesomeProject replacing the `index.js`


## Opening a Database

```js
SQLite.openDatabase({name: uri}, successcb, errorcb);
```


## Database Status

:warning:  The application should **NOT** access the database before it is ready for read and write!

The app should subscribe to database events to act according to its state

If the database is not ready, the `not_ready` event will be fired. This generally means that it is
the first time the app is being open and/or the user has not signed up or logged in yet:

```js
db.on('not_ready', () => {
  // the user is not logged in. show the login screen (and do not access the database)
  ...
});
```

If the user has already signed up or logged in, then the database will fire the `ready` event:

```js
db.on('ready', () => {
  // the user is already logged in or the login was successful. show the main screen
  ...
});
```

It is also possible to subscrive to `sync` events, so the app can read fresh data from the
database and update the screen:

```js
db.on('sync', () => {
  // the db received an update. update the screen with new data
  ...
});
```

To check the full status of the database we can use:

```js
db.executeSql("pragma sync_status", [], (result) => {
  if (result.rows && result.rows.length > 0) {
    var status = result.rows.item(0);
    console.log('sync status:', status.sync_status);
  } else {
    console.log('OctoDB is not active')
  }
}, (msg) => {
  console.log('could not run "pragma sync_status":', msg)
});
```


## User Sign Up & User Login 

When the user is accessing your app for the first time (in any device), it will be required to sign up

The app can capture user data and send it to the backend user authorization service using this command:

```
pragma user_signup='<user_data>'
```

If the user is already signed up on your backend and is now using a new device, it should use the login option:

```
pragma user_login='<user_data>'
```

You can use any user data you want (phone number, username, OAuth...). The data can be encoded in a text (JSON, YAML...)
or a binary format. If using a text format, each single quote must be doubled (replace `'` with `''`).
If using a binary format then it must be encoded in base64 or hex because the command only accepts strings.

Here is an example code using JSON:

```js
const user_signup = async () => {
  var passwd = await sha256(email + password)
  var info = JSON.stringify({email: email, passwd: passwd})
  var sql = "pragma user_signup='" + info.replace(/'/g, "''") + "'"
  db.executeSql(sql, [], (result) => {
    console.log('log in command sent' + JSON.parse(result))
  }, (msg) => {
    setWaiting(false)
    console.log('could not log in the user:', msg)
  });  
}
```

Notice that you must implement the [backend service](https://github.com/octodb/docs/blob/master/auth-service.md)
that handles these authorization requests.


## Multi-User App

To support multiple users in a single app installation your app can have a database for each user.

Your app will need to keep track of which database is used for each user.
An easy way is to convert the username or e-mail into hex format and then use it as the database name:

```js
var dbname = hex(email) + ".db"
var uri = "file:" + dbname + "?node=secondary&connect=tcp://111.222.33.44:1234"
```

### Sign Out

Just close the currently open database and display the signup & login screen

### Login

When the user enters its data (usually e-mail and password):

1. Get the database name based on the e-mail and open it
2. Wait for the db event. If not ready, then send signup/login info via the `pragma` command. If ready, then check if the password is correct


## Importing a pre-populated database

This is **NOT** supported if the database uses OctoDB, because the database will be downloaded from the primary node(s) at the first run.

But as this library also supports normal SQLite databases, you can import an existing pre-populated database file into your application when opening a normal SQLite database.

On this case follow the instructions at the [original repo](https://github.com/andpor/react-native-sqlite-storage)


## Attaching another database

SQLite3 offers the capability to attach another database to an existing database instance, i.e. for making cross database JOINs available.
This feature allows to SELECT and JOIN tables over multiple databases with only one statement and only one database connection.
To archieve this, you need to open both databases and to call the `attach()` method of the destination (or master) database to the other ones.

```js
let dbMaster, dbSecond;

dbSecond = SQLite.openDatabase({name: 'second'},
  (db) => {
    dbMaster = SQLite.openDatabase({name: 'master'},
      (db) => {
        dbMaster.attach( "second", "second", () => console.log("Database attached successfully"), () => console.log("ERROR"))
      },
      (err) => console.log("Error on opening database 'master'", err)
    );
  },
  (err) => console.log("Error on opening database 'second'", err)
);
```

The first argument of `attach()` is the name of the database, which is used in `SQLite.openDatabase()`. The second argument is the alias, that is used to query on tables of the attached database.

The following statement would select data from the master database and include the "second" database within a simple SELECT/JOIN statement:

```sql
SELECT * FROM user INNER JOIN second.subscriptions s ON s.user_id = user.id
```

To detach a database use the `detach()` method:

```js
dbMaster.detach( 'second', successCallback, errorCallback );
```

There is also Promise support for `attach()` and `detach()` as shown in the example application under the
[test](test) folder


## Promises

To enable promises, run:

```javascript
SQLite.enablePromise(true);
```


## Known Issues

1. React Native does not distinguish between integers and doubles. Only a Numeric type is available on the interface point. You can check [the original issue](https://github.com/facebook/react-native/issues/4141)

The current solution is to cast the bound value in the SQL statement as shown here:

```sql
INSERT INTO products (name,qty,price) VALUES (?, cast(? as integer), cast(? as real))
```
