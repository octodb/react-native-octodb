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
// using npm
npm install --save react-native-octodb

// using yarn
yarn add react-native-octodb
```

Then follow the instructions for your platform to link `react-native-octodb` to your project


## iOS

Run:

```
cd ios && pod install && cd ..
```

For React Native 0.59 and below, please follow these [instructions](instructions/INSTALL.md)


## Android

There are no extra steps for React Native 0.60 and above

For React Native 0.59 and below, please follow these [instructions](instructions/INSTALL.md)


## How to Use

Add this line to access the module on your `App.js`:

```javascript
var SQLite = require('react-native-octodb')
```

Then add code to use the SQLite API. Here is a sample code:

```javascript
function on_error(err) {
  console.log("Error:", err);
}

on_success = function(){
  console.log("SQL executed");
}

on_db_open = () => {
  console.log("The database was opened");
}

// open the database
var uri = "file:test.db?node=secondary&connect=tcp://server:port";
var db = SQLite.openDatabase({name: uri}, on_db_open, on_error);

db.on('error', on_error);  // this should be the first

db.on('not_ready', () => {
  // show the signup/login screen
  ...
});

db.on('ready', () => {
  // login successful, show the main screen
  ...
});

db.on('sync', () => {
  show_items();
});

insert_items = () => {
  db.transaction((tx) => {
    //tx.executeSql("CREATE TABLE IF NOT EXISTS tasks (id INTEGER PRIMARY KEY, name, done, row_owner)", [], this.on_success, this.on_error);
    tx.executeSql("INSERT INTO tasks (name,done) VALUES ('Learn React Native',1)", []);
    tx.executeSql("INSERT INTO tasks (name,done) VALUES ('Use SQLite',1)", []);
    tx.executeSql("INSERT INTO tasks (name,done) VALUES ('Test OctoDB',0)", []);
  }, () => {  // success callback = transaction committed
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

### Working examples

* [Using callbacks](test/index.callback.js)
* [Using Promises](test/index.promise.js)

These sample apps can be used with the AwesomeProject generated by React Native. All you have to do is to copy one of those files into your AwesomeProject replacing the `index.js`


## Opening a database

Opening a database is slightly different between iOS and Android. Where as on Android the location of the database file is fixed, there are three choices of where the database file can be located on iOS. The 'location' parameter you provide to `openDatabase` call indicated where you would like the file to be created. This parameter is neglected on Android.

The default location on iOS is a no-sync location as mandated by Apple

To open a database in default no-sync location (affects iOS *only*):

```js
SQLite.openDatabase({name: uri, location: 'default'}, successcb, errorcb);
```

To specify a different location (affects iOS *only*):

```js
SQLite.openDatabase({name: uri, location: 'Library'}, successcb, errorcb);
```

where the `location` option may be set to one of the following choices:

- `default`: `Library/LocalDatabase` subdirectory - *NOT* visible to iTunes and *NOT* backed up by iCloud
- `Library`: `Library` subdirectory - backed up by iCloud, *NOT* visible to iTunes
- `Documents`: `Documents` subdirectory - visible to iTunes and backed up by iCloud
- `Shared`:  app group's shared container - *see next section*

The original webSql style `openDatabase` still works and the location will implicitly default to 'default' option:

```js
SQLite.openDatabase(uri, "1.0", "Demo", -1);
```

## Opening a database in an App Group's Shared Container (iOS)

If you have an iOS app extension which needs to share access to the same DB instance as your main app, you must use the shared container of a registered app group.

Assuming you have already set up an app group and turned on the "App Groups" entitlement of both the main app and app extension, setting them to the same app group name, the following extra steps must be taken:

#### Step 1 - supply your app group name in all needed `Info.plist`s

In both `ios/MY_APP_NAME/Info.plist` and `ios/MY_APP_EXT_NAME/Info.plist` (along with any other app extensions you may have), you simply need to add the `AppGroupName` key to the main dictionary with your app group name as the string value:

```xml
<plist version="1.0">
<dict>
  <!-- ... -->
  <key>AppGroupName</key>
  <string>MY_APP_GROUP_NAME</string>
  <!-- ... -->
</dict>
</plist>
```

#### Step 2 - set shared database location

When calling `SQLite.openDatabase` in your React Native code, you need to set the `location` param to `'Shared'`:

```js
SQLite.openDatabase({name: uri, location: 'Shared'}, successcb, errorcb);
```

## Importing a pre-populated database

This is **NOT** supported if the database uses OctoDB, because the database will be downloaded from the primary node(s) at the first run.

But as this library also supports normal SQLite databases, you can import an existing pre-populated database file into your application. 

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


### Promises

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
