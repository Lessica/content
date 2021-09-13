---
title: Using IndexedDB
slug: Web/API/IndexedDB_API/Using_IndexedDB
tags:
  - API
  - Advanced
  - Database
  - Guide
  - IndexedDB
  - Storage
  - Tutorial
  - jsstore
---
<p>{{DefaultAPISidebar("IndexedDB")}}</p>

<p>IndexedDB is a way for you to persistently store data inside a user's browser. Because it lets you create web applications with rich query abilities regardless of network availability, your applications can work both online and offline. </p>

<h2 id="About_this_document">About this document</h2>

<p>This tutorial walks you through using the asynchronous API of IndexedDB. If you are not familiar with IndexedDB, you should first read the <a href="/en-US/docs/Web/API/IndexedDB_API/Basic_Terminology">IndexedDB key characteristics and basic terminology</a> article.</p>

<p>For the reference documentation on the IndexedDB API, see the <a href="/en-US/docs/Web/API/IndexedDB_API">IndexedDB API</a> article and its subpages. This article documents the types of objects used by IndexedDB, as well as the methods of the asynchronous API (the synchronous API was removed from spec).  </p>

<h2 id="pattern">Basic pattern</h2>

<p>The basic pattern that IndexedDB encourages is the following:</p>

<ol>
 <li>Open a database.</li>
 <li>Create an object store in the database. </li>
 <li>Start a transaction and make a request to do some database operation, like adding or retrieving data.</li>
 <li>
  <div>Wait for the operation to complete by listening to the right kind of DOM event.</div>
 </li>
 <li>
  <div>Do something with the results (which can be found on the request object).</div>
 </li>
</ol>

<p>With these big concepts under our belts, we can get to more concrete stuff.</p>

<h2 id="open">Creating and structuring the store</h2>

<h3 id="Using_an_experimental_version_of_IndexedDB">Using an experimental version of IndexedDB</h3>

<p>In case you want to test your code in browsers that still use a prefix, you can use the following code:  </p>

<pre class="brush: js">// In the following line, you should include the prefixes of implementations you want to test.
window.indexedDB = window.indexedDB || window.mozIndexedDB || window.webkitIndexedDB || window.msIndexedDB;
// DON'T use "var indexedDB = ..." if you're not in a function.
// Moreover, you may need references to some window.IDB* objects:
window.IDBTransaction = window.IDBTransaction || window.webkitIDBTransaction || window.msIDBTransaction || {READ_WRITE: "readwrite"}; // This line should only be needed if it is needed to support the object's constants for older browsers
window.IDBKeyRange = window.IDBKeyRange || window.webkitIDBKeyRange || window.msIDBKeyRange;
// (Mozilla has never prefixed these objects, so we don't need window.mozIDB*)</pre>

<p>Beware that implementations that use a prefix may be buggy, or incomplete, or following an old version of the specification. Therefore, it is not recommended to use it in production code. It may be preferable to not support a browser than to claim to support it and fail:</p>

<pre class="brush: js">if (!window.indexedDB) {
    console.log("Your browser doesn't support a stable version of IndexedDB. Such and such feature will not be available.");
}
</pre>

<h3 id="Opening_a_database">Opening a database</h3>

<p>We start the whole process like this:</p>

<pre class="brush: js">// Let us open our database
var request = window.indexedDB.open("MyTestDatabase", 3);
</pre>

<p>See that? Opening a database is just like any other operation — you have to "request" it.</p>

<p>The open request doesn't open the database or start the transaction right away. The call to the <code>open()</code> function returns an <a href="/en-US/docs/Web/API/IDBOpenDBRequest"><code>IDBOpenDBRequest</code></a> object with a result (success) or error value that you handle as an event. Most other asynchronous functions in IndexedDB do the same thing - return an <a href="/en-US/docs/Web/API/IDBRequest"><code>IDBRequest</code></a> object with the result or error. The result for the open function is an instance of an <code><a href="/en-US/docs/Web/API/IDBDatabase">IDBDatabase</a>.</code></p>

<p>The second parameter to the open method is the version of the database. The version of the database determines the database schema — the object stores in the database and their structure. If the database doesn't already exist, it is created by the <code>open</code> operation, then an <code>onupgradeneeded</code> event is triggered and you create the database schema in the handler for this event. If the database does exist but you are specifying an upgraded version number, an <code>onupgradeneeded</code> event is triggered straight away, allowing you to provide an updated schema in its handler. More on this later in <a href="#updating_the_version_of_the_database">Updating the version of the database</a> below, and the {{ domxref("IDBFactory.open") }} reference page. </p>

<div class="warning">
<p><strong>Warning:</strong> The version number is an <code>unsigned long long</code> number, which means that it can be a very big integer. It also means that you can't use a float, otherwise it will be converted to the closest lower integer and the transaction may not start, nor the <code>upgradeneeded</code> event trigger. So for example, don't use 2.4 as a version number:<br>
 <code>var request = indexedDB.open("MyTestDatabase", 2.4); // don't do this, as the version will be rounded to 2</code></p>
</div>

<h4 id="Generating_handlers">Generating handlers</h4>

<p>The first thing you'll want to do with almost all of the requests you generate is to add success and error handlers:</p>

<pre class="brush: js">request.onerror = function(event) {
  // Do something with request.errorCode!
};
request.onsuccess = function(event) {
  // Do something with request.result!
};</pre>

<p>Which of the two functions, <code>onsuccess()</code> or <code>onerror()</code>, gets called? If everything succeeds, a success event (that is, a DOM event whose <code>type</code> property is set to <code>"success"</code>) is fired with <code>request</code> as its <code>target</code>. Once it is fired, the <code>onsuccess()</code> function on <code>request</code> is triggered with the success event as its argument. Otherwise, if there was any problem, an error event (that is, a DOM event whose <code>type</code> property is set to <code>"error"</code>) is fired at <code>request</code>. This triggers the <code>onerror()</code> function with the error event as its argument.</p>

<p>The IndexedDB API is designed to minimize the need for error handling, so you're not likely to see many error events (at least, not once you're used to the API!). In the case of opening a database, however, there are some common conditions that generate error events. The most likely problem is that the user decided not to give your web app permission to create a database. One of the main design goals of IndexedDB is to allow large amounts of data to be stored for offline use. (To learn more about how much storage you can have for each browser, see <a href="/en-US/docs/Web/API/IndexedDB_API/Browser_storage_limits_and_eviction_criteria#storage_limits">Storage limits</a>.)  </p>

<p>Obviously, browsers do not want to allow some advertising network or malicious website to pollute your computer, so browsers used to prompt the user the first time any given web app attempts to open an IndexedDB for storage. The user could choose to allow or deny access. Also, IndexedDB storage in browsers' privacy modes only lasts in-memory until the incognito session is closed (Private Browsing mode for Firefox and Incognito mode for Chrome, but in Firefox this is <a href="https://bugzilla.mozilla.org/show_bug.cgi?id=781982">not implemented yet</a> as of May 2021 so you can't use IndexedDB in Firefox Private Browsing at all).</p>

<p>Now, assuming that the user allowed your request to create a database, and you've received a success event to trigger the success callback; What's next? The request here was generated with a call to <code>indexedDB.open()</code>, so <code>request.result</code> is an instance of <code>IDBDatabase</code>, and you definitely want to save that for later. Your code might look something like this:</p>

<pre class="brush: js">var db;
var request = indexedDB.open("MyTestDatabase");
request.onerror = function(event) {
  console.log("Why didn't you allow my web app to use IndexedDB?!");
};
request.onsuccess = function(event) {
  db = event.target.result;
};
</pre>

<h4 id="Handling_Errors">Handling Errors</h4>

<p>As mentioned above, error events bubble. Error events are targeted at the request that generated the error, then the event bubbles to the transaction, and then finally to the database object. If you want to avoid adding error handlers to every request, you can instead add a single error handler on the database object, like so:</p>

<pre class="brush: js">db.onerror = function(event) {
  // Generic error handler for all errors targeted at this database's
  // requests!
  console.error("Database error: " + event.target.errorCode);
};
</pre>

<p>One of the common possible errors when opening a database is <code>VER_ERR</code>. It indicates that the version of the database stored on the disk is <em>greater</em> than the version that you are trying to open. This is an error case that must always be handled by the error handler.</p>

<h3 id="Creating_or_updating_the_version_of_the_database">Creating or updating the version of the database</h3>

<p>When you create a new database or increase the version number of an existing database (by specifying a higher version number than you did previously, when {{ anch("Opening a database") }}), the <code>onupgradeneeded</code> event will be triggered and an <a href="/en-US/docs/Web/API/IDBVersionChangeEvent">IDBVersionChangeEvent</a> object will be passed to any <code>onversionchange</code> event handler set up on <code>request.result</code> (i.e., <code>db</code> in the example). In the handler for the <code>upgradeneeded</code> event, you should create the object stores needed for this version of the database:</p>

<pre class="brush:js;">// This event is only implemented in recent browsers
request.onupgradeneeded = function(event) {
  // Save the IDBDatabase interface
  var db = event.target.result;

  // Create an objectStore for this database
  var objectStore = db.createObjectStore("name", { keyPath: "myKey" });
};</pre>

<p>In this case, the database will already have the object stores from the previous version of the database, so you do not have to create these object stores again. You only need to create any new object stores, or delete object stores from the previous version that are no longer needed. If you need to change an existing object store (e.g., to change the <code>keyPath</code>), then you must delete the old object store and create it again with the new options. (Note that this will delete the information in the object store! If you need to save that information, you should read it out and save it somewhere else before upgrading the database.)</p>

<p>Trying to create an object store with a name that already exists (or trying to delete an object store with a name that does not already exist) will throw an error. </p>

<p>If the <code>onupgradeneeded</code> event exits successfully, the <code>onsuccess</code> handler of the open database request will then be triggered. </p>

<h3 id="Structuring_the_database">Structuring the database</h3>

<p>Now to structure the database. IndexedDB uses object stores rather than tables, and a single database can contain any number of object stores. Whenever a value is stored in an object store, it is associated with a key. There are several different ways that a key can be supplied depending on whether the object store uses a <a href="/en-US/docs/Web/API/IndexedDB_API/Basic_Terminology#key_path">key path</a> or a <a href="/en-US/docs/Web/API/IndexedDB_API/Basic_Terminology#key_generator">key generator</a>.</p>

<p>The following table shows the different ways the keys are supplied:</p>

<table class="no-markdown">
 <thead>
  <tr>
   <th scope="col">Key Path (<code>keyPath</code>)</th>
   <th scope="col">Key Generator (<code>autoIncrement</code>)</th>
   <th scope="col">Description</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td>No</td>
   <td>No</td>
   <td>This object store can hold any kind of value, even primitive values like numbers and strings. You must supply a separate key argument whenever you want to add a new value.</td>
  </tr>
  <tr>
   <td>Yes</td>
   <td>No</td>
   <td>This object store can only hold JavaScript objects. The objects must have a property with the same name as the key path.</td>
  </tr>
  <tr>
   <td>No</td>
   <td>Yes</td>
   <td>This object store can hold any kind of value. The key is generated for you automatically, or you can supply a separate key argument if you want to use a specific key.</td>
  </tr>
  <tr>
   <td>Yes</td>
   <td>Yes</td>
   <td>This object store can only hold JavaScript objects. Usually a key is generated and the value of the generated key is stored in the object in a property with the same name as the key path. However, if such a property already exists, the value of that property is used as key rather than generating a new key.</td>
  </tr>
 </tbody>
</table>

<p>You can also create indices on any object store, provided the object store holds objects, not primitives. An index lets you look up the values stored in an object store using the value of a property of the stored object, rather than the object's key.</p>

<p>Additionally, indexes have the ability to enforce simple constraints on the stored data. By setting the unique flag when creating the index, the index ensures that no two objects are stored with both having the same value for the index's key path. So, for example, if you have an object store which holds a set of people, and you want to ensure that no two people have the same email address, you can use an index with the unique flag set to enforce this.</p>

<p>That may sound confusing, but this simple example should illustrate the concepts. First, we'll define some customer data to use in our example:</p>

<pre class="brush: js">// This is what our customer data looks like.
const customerData = [
  { ssn: "444-44-4444", name: "Bill", age: 35, email: "bill@company.com" },
  { ssn: "555-55-5555", name: "Donna", age: 32, email: "donna@home.org" }
];
</pre>

<p>Of course, you wouldn't use someone's social security number as the primary key to a customer table because not everyone has a social security number, and you would store their birth date instead of their age, but let's ignore those unfortunate choices for the sake of convenience and move along.</p>

<p>Now let's look at creating an IndexedDB to store our data:</p>

<pre class="brush: js">const dbName = "the_name";

var request = indexedDB.open(dbName, 2);

request.onerror = function(event) {
  // Handle errors.
};
request.onupgradeneeded = function(event) {
  var db = event.target.result;

  // Create an objectStore to hold information about our customers. We're
  // going to use "ssn" as our key path because it's guaranteed to be
  // unique - or at least that's what I was told during the kickoff meeting.
  var objectStore = db.createObjectStore("customers", { keyPath: "ssn" });

  // Create an index to search customers by name. We may have duplicates
  // so we can't use a unique index.
  objectStore.createIndex("name", "name", { unique: false });

  // Create an index to search customers by email. We want to ensure that
  // no two customers have the same email, so use a unique index.
  objectStore.createIndex("email", "email", { unique: true });

  // Use transaction oncomplete to make sure the objectStore creation is
  // finished before adding data into it.
  objectStore.transaction.oncomplete = function(event) {
    // Store values in the newly created objectStore.
    var customerObjectStore = db.transaction("customers", "readwrite").objectStore("customers");
    customerData.forEach(function(customer) {
      customerObjectStore.add(customer);
    });
  };
};
</pre>

<p>As indicated previously, <code>onupgradeneeded</code> is the only place where you can alter the structure of the database. In it, you can create and delete object stores and build and remove indices.</p>

<div>Object stores are created with a single call to <code>createObjectStore()</code>. The method takes a name of the store, and a parameter object. Even though the parameter object is optional, it is very important, because it lets you define important optional properties and refine the type of object store you want to create. In our case, we've asked for an object store named "customers" and defined a <code>keyPath</code>, which is the property that makes an individual object in the store unique. That property in this example is "ssn" since a social security number is guaranteed to be unique. "ssn" must be present on every object that is stored in the <code>objectStore</code>. </div>

<p>We've also asked for an index named "name" that looks at the <code>name</code> property of the stored objects. As with <code>createObjectStore()</code>, <code>createIndex()</code> takes an optional <code>options</code> object that refines the type of index that you want to create. Adding objects that don't have a <code>name</code> property still succeeds, but the objects won't appear in the "name" index.</p>

<p>We can now retrieve the stored customer objects using their <code>ssn</code> from the object store directly, or using their name by using the index. To learn how this is done, see the section on <a href="#using_an_index">using an index</a>.</p>

<h3 id="Using_a_key_generator">Using a key generator</h3>

<p>Setting up an <code>autoIncrement</code> flag when creating the object store would enable the key generator for that object store. By default this flag is not set.</p>

<p>With the key generator, the key would be generated automatically as you add the value to the object store. The current number of a key generator is always set to 1 when the object store for that key generator is first created. Basically the newly auto-generated key is increased by 1 based on the previous key. The current number for a key generator never decreases, other than as a result of database operations being reverted, for example, the database transaction is aborted. Therefore deleting a record or even clearing all records from an object store never affects the object store's key generator.</p>

<p>We can create another object store with the key generator as below:</p>

<pre class="brush: js">// Open the indexedDB.
var request = indexedDB.open(dbName, 3);

request.onupgradeneeded = function (event) {

    var db = event.target.result;

    // Create another object store called "names" with the autoIncrement flag set as true.
    var objStore = db.createObjectStore("names", { autoIncrement : true });

    // Because the "names" object store has the key generator, the key for the name value is generated automatically.
    // The added records would be like:
    // key : 1 =&gt; value : "Bill"
    // key : 2 =&gt; value : "Donna"
    customerData.forEach(function(customer) {
        objStore.add(customer.name);
    });
};</pre>

<p>For more details about the key generator, please see <a href="https://www.w3.org/TR/IndexedDB/#key-generator-concept">"W3C Key Generators"</a>.</p>

<h2 id="Adding_retrieving_and_removing_data">Adding, retrieving, and removing data</h2>

<p>Before you can do anything with your new database, you need to start a transaction. Transactions come from the database object, and you have to specify which object stores you want the transaction to span. Once you are inside the transaction, you can access the object stores that hold your data and make your requests. Next, you need to decide if you're going to make changes to the database or if you just need to read from it. Transactions have three available modes: <code>readonly</code>, <code>readwrite</code>, and <code>versionchange</code>.</p>

<p>To change the "schema" or structure of the database—which involves creating or deleting object stores or indexes—the transaction must be in <code>versionchange</code> mode. This transaction is opened by calling the {{domxref("IDBFactory.open")}} method with a <code>version</code> specified.</p>

<p>To read the records of an existing object store, the transaction can either be in <code>readonly</code> or <code>readwrite</code> mode. To make changes to an existing object store, the transaction must be in <code>readwrite</code> mode. You open such transactions with {{domxref("IDBDatabase.transaction")}}. The method accepts two parameters: the <code>storeNames</code> (the scope, defined as an array of object stores that you want to access) and the <code>mode</code> (<code>readonly</code> or <code>readwrite</code>) for the transaction. The method returns a transaction object containing the {{domxref("IDBIndex.objectStore")}} method, which you can use to access your object store. By default, where no mode is specified, transactions open in <code>readonly</code> mode.</p>

<div class="note">
<p><strong>Note:</strong> As of Firefox 40, IndexedDB transactions have relaxed durability guarantees to increase performance (see {{Bug("1112702")}}.) Previously in a <code>readwrite</code> transaction {{domxref("IDBTransaction.oncomplete")}} was fired only when all data was guaranteed to have been flushed to disk. In Firefox 40+ the <code>complete</code> event is fired after the OS has been told to write the data but potentially before that data has actually been flushed to disk. The <code>complete</code> event may thus be delivered quicker than before, however, there exists a small chance that the entire transaction will be lost if the OS crashes or there is a loss of system power before the data is flushed to disk. Since such catastrophic events are rare most consumers should not need to concern themselves further. If you must ensure durability for some reason (e.g. you're storing critical data that cannot be recomputed later) you can force a transaction to flush to disk before delivering the <code>complete</code> event by creating a transaction using the experimental (non-standard) <code>readwriteflush</code> mode (see {{domxref("IDBDatabase.transaction")}}.</p>
</div>

<p>You can speed up data access by using the right scope and mode in the transaction. Here are a couple of tips:</p>

<ul>
 <li>When defining the scope, specify only the object stores you need. This way, you can run multiple transactions with non-overlapping scopes concurrently.</li>
 <li>Only specify a <code>readwrite</code> transaction mode when necessary. You can concurrently run multiple <code>readonly</code> transactions with overlapping scopes, but you can have only one <code>readwrite</code> transaction for an object store. To learn more, see the definition for <a href="/en-US/docs/Web/API/IndexedDB_API/Basic_Terminology#transaction">transaction</a> in the <a href="/en-US/docs/Web/API/IndexedDB_API/Basic_Terminology">IndexedDB key characteristics and basic terminology</a> article.</li>
</ul>

<h3 id="Adding_data_to_the_database">Adding data to the database</h3>

<p>If you've just created a database, then you probably want to write to it. Here's what that looks like:</p>

<pre class="brush:js;">var transaction = db.transaction(["customers"], "readwrite");
// Note: Older experimental implementations use the deprecated constant IDBTransaction.READ_WRITE instead of "readwrite".
// In case you want to support such an implementation, you can write:
// var transaction = db.transaction(["customers"], IDBTransaction.READ_WRITE);</pre>

<p>The <code>transaction()</code> function takes two arguments (though one is optional) and returns a transaction object. The first argument is a list of object stores that the transaction will span. You can pass an empty array if you want the transaction to span all object stores, but don't do it because the spec says an empty array should generate an InvalidAccessError. If you don't specify anything for the second argument, you get a read-only transaction. Since you want to write to it here you need to pass the <code>"readwrite"</code> flag.</p>

<p>Now that you have a transaction you need to understand its lifetime. Transactions are tied very closely to the event loop. If you make a transaction and return to the event loop without using it then the transaction will become inactive. The only way to keep the transaction active is to make a request on it. When the request is finished you'll get a DOM event and, assuming that the request succeeded, you'll have another opportunity to extend the transaction during that callback. If you return to the event loop without extending the transaction then it will become inactive, and so on. As long as there are pending requests the transaction remains active. Transaction lifetimes are really very simple but it might take a little time to get used to. A few more examples will help, too. If you start seeing <code>TRANSACTION_INACTIVE_ERR</code> error codes then you've messed something up.</p>

<p>Transactions can receive DOM events of three different types: <code>error</code>, <code>abort</code>, and <code>complete</code>. We've talked about the way that <code>error</code> events bubble, so a transaction receives error events from any requests that are generated from it. A more subtle point here is that the default behavior of an error is to abort the transaction in which it occurred. Unless you handle the error by first calling <code>stopPropagation()</code> on the error event then doing something else, the entire transaction is rolled back. This design forces you to think about and handle errors, but you can always add a catchall error handler to the database if fine-grained error handling is too cumbersome. If you don't handle an error event or if you call <code>abort()</code> on the transaction, then the transaction is rolled back and an <code>abort</code> event is fired on the transaction. Otherwise, after all pending requests have completed, you'll get a <code>complete</code> event. If you're doing lots of database operations, then tracking the transaction rather than individual requests can certainly aid your sanity.</p>

<p>Now that you have a transaction, you'll need to get the object store from it. Transactions only let you have an object store that you specified when creating the transaction. Then you can add all the data you need.</p>

<pre class="brush: js">// Do something when all the data is added to the database.
transaction.oncomplete = function(event) {
  console.log("All done!");
};

transaction.onerror = function(event) {
  // Don't forget to handle errors!
};

var objectStore = transaction.objectStore("customers");
customerData.forEach(function(customer) {
  var request = objectStore.add(customer);
  request.onsuccess = function(event) {
    // event.target.result === customer.ssn;
  };
});</pre>

<p>The <code>result</code> of a request generated from a call to <code>add()</code> is the key of the value that was added. So in this case, it should equal the <code>ssn</code> property of the object that was added, since the object store uses the <code>ssn</code> property for the key path. Note that the <code>add()</code> function requires that no object already be in the database with the same key. If you're trying to modify an existing entry, or you don't care if one exists already, you can use the <code>put()</code> function, as shown below in the {{ anch("Updating an entry in the database") }} section.</p>

<h3 id="Removing_data_from_the_database">Removing data from the database</h3>

<p>Removing data is very similar:</p>

<pre class="brush: js">var request = db.transaction(["customers"], "readwrite")
                .objectStore("customers")
                .delete("444-44-4444");
request.onsuccess = function(event) {
  // It's gone!
};</pre>

<h3 id="Getting_data_from_the_database">Getting data from the database</h3>

<p>Now that the database has some info in it, you can retrieve it in several ways. First, the simple <code>get()</code>. You need to provide the key to retrieve the value, like so:</p>

<pre class="brush: js">var transaction = db.transaction(["customers"]);
var objectStore = transaction.objectStore("customers");
var request = objectStore.get("444-44-4444");
request.onerror = function(event) {
  // Handle errors!
};
request.onsuccess = function(event) {
  // Do something with the request.result!
  console.log("Name for SSN 444-44-4444 is " + request.result.name);
};</pre>

<p>That's a lot of code for a "simple" retrieval. Here's how you can shorten it up a bit, assuming that you handle errors at the database level:</p>

<pre class="brush: js">db.transaction("customers").objectStore("customers").get("444-44-4444").onsuccess = function(event) {
  console.log("Name for SSN 444-44-4444 is " + event.target.result.name);
};</pre>

<p>See how this works? Since there's only one object store, you can avoid passing a list of object stores you need in your transaction and just pass the name as a string. Also, you're only reading from the database, so you don't need a <code>"readwrite"</code> transaction. Calling <code>transaction()</code> with no mode specified gives you a <code>"readonly"</code> transaction. Another subtlety here is that you don't actually save the request object to a variable. Since the DOM event has the request as its target you can use the event to get to the <code>result</code> property.</p>

<p>Note that you can speed up data access by limiting the scope and mode in the transaction. Here are a couple of tips:</p>

<ul>
 <li>When defining the <a href="#scope">scope</a>, specify only the object stores you need. This way, you can run multiple transactions with non-overlapping scopes concurrently.</li>
 <li>Only specify a readwrite transaction mode when necessary. You can concurrently run multiple readonly transactions with overlapping scopes, but you can have only one readwrite transaction for an object store. To learn more, see the definition for <a href="/en-US/docs/Web/API/IndexedDB_API/Basic_Terminology#transaction">transaction</a> in the <a href="/en-US/docs/Web/API/IndexedDB_API/Basic_Terminology">IndexedDB key characteristics and basic terminology</a> article.</li>
</ul>

<h3 id="Updating_an_entry_in_the_database">Updating an entry in the database</h3>

<p>Now we've retrieved some data, updating it and inserting it back into the IndexedDB is pretty simple. Let's update the previous example somewhat:</p>

<pre class="brush: js">var objectStore = db.transaction(["customers"], "readwrite").objectStore("customers");
var request = objectStore.get("444-44-4444");
request.onerror = function(event) {
  // Handle errors!
};
request.onsuccess = function(event) {
  // Get the old value that we want to update
  var data = event.target.result;

  // update the value(s) in the object that you want to change
  data.age = 42;

  // Put this updated object back into the database.
  var requestUpdate = objectStore.put(data);
   requestUpdate.onerror = function(event) {
     // Do something with the error
   };
   requestUpdate.onsuccess = function(event) {
     // Success - the data is updated!
   };
};</pre>

<p>So here we're creating an <code>objectStore</code> and requesting a customer record out of it, identified by its ssn value (<code>444-44-4444</code>). We then put the result of that request in a variable (<code>data</code>), update the <code>age</code> property of this object, then create a second request (<code>requestUpdate</code>) to put the customer record back into the <code>objectStore</code>, overwriting the previous value.</p>

<div class="note">
<p><strong>Note:</strong> In this case we've had to specify a <code>readwrite</code> transaction because we want to write to the database, not just read from it.</p>
</div>

<h3 id="Using_a_cursor">Using a cursor</h3>

<p>Using <code>get()</code> requires that you know which key you want to retrieve. If you want to step through all the values in your object store, then you can use a cursor. Here's what it looks like:</p>

<pre class="brush: js">var objectStore = db.transaction("customers").objectStore("customers");

objectStore.openCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    console.log("Name for SSN " + cursor.key + " is " + cursor.value.name);
    cursor.continue();
  }
  else {
    console.log("No more entries!");
  }
};</pre>

<p>The <code>openCursor()</code> function takes several arguments. First, you can limit the range of items that are retrieved by using a key range object that we'll get to in a minute. Second, you can specify the direction that you want to iterate. In the above example, we're iterating over all objects in ascending order. The success callback for cursors is a little special. The cursor object itself is the <code>result</code> of the request (above we're using the shorthand, so it's <code>event.target.result</code>). Then the actual key and value can be found on the <code>key</code> and <code>value</code> properties of the cursor object. If you want to keep going, then you have to call <code>continue()</code> on the cursor. When you've reached the end of the data (or if there were no entries that matched your <code>openCursor()</code> request) you still get a success callback, but the <code>result</code> property is <code>undefined</code>.</p>

<p>One common pattern with cursors is to retrieve all objects in an object store and add them to an array, like this:</p>

<pre class="brush: js">var customers = [];

objectStore.openCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    customers.push(cursor.value);
    cursor.continue();
  }
  else {
    console.log("Got all customers: " + customers);
  }
};</pre>

<div class="note">
<p><strong>Note:</strong> Alternatively, you can use <code>getAll()</code> to handle this case (and <code>getAllKeys()</code>) . The following code does precisely the same thing as above:</p>

<pre class="brush: js">objectStore.getAll().onsuccess = function(event) {
  console.log("Got all customers: " + event.target.result);
};</pre>

<p>There is a performance cost associated with looking at the <code>value</code> property of a cursor, because the object is created lazily. When you use <code>getAll()</code> for example, the browser must create all the objects at once. If you're just interested in looking at each of the keys, for instance, it is much more efficient to use a cursor than to use <code>getAll()</code>. If you're trying to get an array of all the objects in an object store, though, use <code>getAll()</code>.</p>
</div>

<h3 id="Using_an_index">Using an index</h3>

<p>Storing customer data using the SSN as a key is logical since the SSN uniquely identifies an individual. (Whether this is a good idea for privacy is a different question, and outside the scope of this article.) If you need to look up a customer by name, however, you'll need to iterate over every SSN in the database until you find the right one. Searching in this fashion would be very slow, so instead you can use an index.</p>

<pre class="brush: js">// First, make sure you created index in request.onupgradeneeded:
// objectStore.createIndex("name", "name");
// Otherwise you will get DOMException.

var index = objectStore.index("name");

index.get("Donna").onsuccess = function(event) {
  console.log("Donna's SSN is " + event.target.result.ssn);
};
</pre>

<p>The "name" index isn't unique, so there could be more than one entry with the <code>name</code> set to <code>"Donna"</code>. In that case you always get the one with the lowest key value.</p>

<p>If you need to access all the entries with a given <code>name</code> you can use a cursor. You can open two different types of cursors on indexes. A normal cursor maps the index property to the object in the object store. A key cursor maps the index property to the key used to store the object in the object store. The differences are illustrated here:</p>

<pre class="brush: js">// Using a normal cursor to grab whole customer record objects
index.openCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // cursor.key is a name, like "Bill", and cursor.value is the whole object.
    console.log("Name: " + cursor.key + ", SSN: " + cursor.value.ssn + ", email: " + cursor.value.email);
    cursor.continue();
  }
};

// Using a key cursor to grab customer record object keys
index.openKeyCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // cursor.key is a name, like "Bill", and cursor.value is the SSN.
    // No way to directly get the rest of the stored object.
    console.log("Name: " + cursor.key + ", SSN: " + cursor.primaryKey);
    cursor.continue();
  }
};</pre>

<h3 id="Specifying_the_range_and_direction_of_cursors">Specifying the range and direction of cursors</h3>

<p>If you would like to limit the range of values you see in a cursor, you can use an <code>IDBKeyRange</code> object and pass it as the first argument to <code>openCursor()</code> or <code>openKeyCursor()</code>. You can make a key range that only allows a single key, or one that has a lower or upper bound, or one that has both a lower and upper bound. The bound may be "closed" (i.e., the key range includes the given value(s)) or "open" (i.e., the key range does not include the given value(s)). Here's how it works:</p>

<pre class="brush: js">// Only match "Donna"
var singleKeyRange = IDBKeyRange.only("Donna");

// Match anything past "Bill", including "Bill"
var lowerBoundKeyRange = IDBKeyRange.lowerBound("Bill");

// Match anything past "Bill", but don't include "Bill"
var lowerBoundOpenKeyRange = IDBKeyRange.lowerBound("Bill", true);

// Match anything up to, but not including, "Donna"
var upperBoundOpenKeyRange = IDBKeyRange.upperBound("Donna", true);

// Match anything between "Bill" and "Donna", but not including "Donna"
var boundKeyRange = IDBKeyRange.bound("Bill", "Donna", false, true);

// To use one of the key ranges, pass it in as the first argument of openCursor()/openKeyCursor()
index.openCursor(boundKeyRange).onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // Do something with the matches.
    cursor.continue();
  }
};</pre>

<p>Sometimes you may want to iterate in descending order rather than in ascending order (the default direction for all cursors). Switching direction is accomplished by passing <code>prev</code> to the <code>openCursor()</code> function as the second argument:</p>

<pre class="brush: js">objectStore.openCursor(boundKeyRange, "prev").onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // Do something with the entries.
    cursor.continue();
  }
};</pre>

<p>If you just want to specify a change of direction but not constrain the results shown, you can just pass in null as the first argument:</p>

<pre class="brush: js">objectStore.openCursor(null, "prev").onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // Do something with the entries.
    cursor.continue();
  }
};</pre>

<p>Since the "name" index isn't unique, there might be multiple entries where <code>name</code> is the same. Note that such a situation cannot occur with object stores since the key must always be unique. If you wish to filter out duplicates during cursor iteration over indexes, you can pass <code>nextunique</code> (or <code>prevunique</code> if you're going backwards) as the direction parameter. When <code>nextunique</code> or <code>prevunique</code> is used, the entry with the lowest key is always the one returned.</p>

<pre class="brush: js">index.openKeyCursor(null, "nextunique").onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // Do something with the entries.
    cursor.continue();
  }
};</pre>

<p>Please see "<a href="/en-US/docs/Web/API/IDBCursor#constants">IDBCursor Constants</a>" for the valid direction arguments.</p>

<h2 id="Version_changes_while_a_web_app_is_open_in_another_tab">Version changes while a web app is open in another tab</h2>

<p>When your web app changes in such a way that a version change is required for your database, you need to consider what happens if the user has the old version of your app open in one tab and then loads the new version of your app in another. When you call <code>open()</code> with a greater version than the actual version of the database, all other open databases must explicitly acknowledge the request before you can start making changes to the database (an <code>onblocked</code> event is fired until they are closed or reloaded). Here's how it works:</p>

<pre class="brush: js">var openReq = mozIndexedDB.open("MyTestDatabase", 2);

openReq.onblocked = function(event) {
  // If some other tab is loaded with the database, then it needs to be closed
  // before we can proceed.
  console.log("Please close all other tabs with this site open!");
};

openReq.onupgradeneeded = function(event) {
  // All other databases have been closed. Set everything up.
  db.createObjectStore(/* ... */);
  useDatabase(db);
};

openReq.onsuccess = function(event) {
  var db = event.target.result;
  useDatabase(db);
  return;
};

function useDatabase(db) {
  // Make sure to add a handler to be notified if another page requests a version
  // change. We must close the database. This allows the other page to upgrade the database.
  // If you don't do this then the upgrade won't happen until the user closes the tab.
  db.onversionchange = function(event) {
    db.close();
    console.log("A new version of this page is ready. Please reload or close this tab!");
  };

  // Do stuff with the database.
}
</pre>

<p>You should also listen for <code>VersionError</code> errors to handle the situation where already opened apps may initiate code leading to a new attempt to open the database, but using an outdated version.</p>

<h2 id="Security">Security</h2>

<p>IndexedDB uses the same-origin principle, which means that it ties the store to the origin of the site that creates it (typically, this is the site domain or subdomain), so it cannot be accessed by any other origin.</p>

<p>Third party window content (e.g. {{htmlelement("iframe")}} content) cannot access IndexedDB if the browser is set to <a href="https://support.mozilla.org/en-US/kb/disable-third-party-cookies">never accept third party cookies</a> (see {{bug("1147821")}}.)</p>

<h2 id="Warning_about_browser_shutdown">Warning about browser shutdown</h2>

<p>When the browser shuts down (because the user chose the Quit or Exit option), the disk containing the database is removed unexpectedly, or permissions are lost to the database store, the following things happen:</p>

<ol>
 <li>Each transaction on every affected database (or all open databases, in the case of browser shutdown) is aborted with an <code>AbortError</code>. The effect is the same as if {{domxref("IDBTransaction.abort()")}} is called on each transaction.</li>
 <li>Once all of the transactions have completed, the database connection is closed.</li>
 <li>Finally, the {{domxref("IDBDatabase")}} object representing the database connection receives a {{event("close")}} event. You can use the {{domxref("IDBDatabase.onclose")}} event handler to listen for these events, so that you know when a database is unexpectedly closed.</li>
</ol>

<p>The behavior described above is new, and is only available as of the following browser releases: Firefox 50, Google Chrome 31 (approximately).</p>

<p>Prior to these browser versions, the transactions are aborted silently, and no {{event("close")}} event is fired, so there is no way to detect an unexpected database closure.</p>

<p>Since the user can exit the browser at any time, this means that you cannot rely upon any particular transaction to complete, and on older browsers, you don't even get told when they don't complete. There are several implications of this behavior.</p>

<p>First, you should take care to always leave your database in a consistent state at the end of every transaction. For example, suppose that you are using IndexedDB to store a list of items that you allow the user to edit. You save the list after the edit by clearing the object store and then writing out the new list. If you clear the object store in one transaction and write the new list in another transaction, there is a danger that the browser will close after the clear but before the write, leaving you with an empty database. To avoid this, you should combine the clear and the write into a single transaction. </p>

<p>Second, you should never tie database transactions to unload events. If the unload event is triggered by the browser closing, any transactions created in the unload event handler will never complete. An intuitive approach to maintaining some information across browser sessions is to read it from the database when the browser (or a particular page) is opened, update it as the user interacts with the browser, and then save it to the database when the browser (or page) closes. However, this will not work. The database transactions will be created in the unload event handler, but because they are asynchronous they will be aborted before they can execute.</p>

<p>In fact, there is no way to guarantee that IndexedDB transactions will complete, even with normal browser shutdown. See {{ bug(870645) }}. As a workaround for this normal shutdown notification, you might track your transactions and add a <code>beforeunload</code> event to warn the user if any transactions have not yet completed at the time of unloading.</p>

<p>At least with the addition of the abort notifications and {{domxref("IDBDatabase.onclose")}}, you can know when this has happened.</p>

<h2 id="Locale-aware_sorting">Locale-aware sorting</h2>

<p>Mozilla has implemented the ability to perform locale-aware sorting of IndexedDB data in Firefox 43+. By default, IndexedDB didn’t handle internationalization of sorting strings at all, and everything was sorted as if it were English text. For example, b, á, z, a would be sorted as:</p>

<ul>
 <li>a</li>
 <li>b</li>
 <li>z</li>
 <li>á</li>
</ul>

<p>which is obviously not how users want their data to be sorted — Aaron and Áaron for example should go next to one another in a contacts list. Achieving proper international sorting therefore required the entire dataset to be called into memory, and sorting to be performed by client-side JavaScript, which is not very efficient.</p>

<p>This new functionality enables developers to specify a locale when creating an index using {{domxref("IDBObjectStore.createIndex()")}} (check out its parameters.) In such cases, when a cursor is then used to iterate through the dataset, and you want to specify locale-aware sorting, you can use a specialized {{domxref("IDBLocaleAwareKeyRange")}}.</p>

<p>{{domxref("IDBIndex")}} has also had new properties added to it to specify if it has a locale specified, and what it is: <code>locale</code> (returns the locale if any, or null if none is specified) and <code>isAutoLocale</code> (returns <code>true</code> if the index was created with an auto locale, meaning that the platform's default locale is used, <code>false</code> otherwise.)</p>

<div class="note">
<p><strong>Note:</strong> This feature is currently hidden behind a flag — to enable it and experiment, go to <code>about:config</code> and enable <code>dom.indexedDB.experimental</code>.</p>
</div>

<h2 id="Full_IndexedDB_example">Full IndexedDB example</h2>

<h3 id="HTML_Content">HTML Content</h3>

<pre class="brush: html">&lt;script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.8.3/jquery.min.js"&gt;&lt;/script&gt;

    &lt;h1&gt;IndexedDB Demo: storing blobs, e-publication example&lt;/h1&gt;
    &lt;div class="note"&gt;
      &lt;p&gt;
        Works and tested with:
      &lt;/p&gt;
      &lt;div id="compat"&gt;
      &lt;/div&gt;
    &lt;/div&gt;

    &lt;div id="msg"&gt;
    &lt;/div&gt;

    &lt;form id="register-form"&gt;
      &lt;table&gt;
        &lt;tbody&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-title" class="required"&gt;
                Title:
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-title" name="pub-title" /&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-biblioid" class="required"&gt;
                Bibliographic ID:&lt;br/&gt;
                &lt;span class="note"&gt;(ISBN, ISSN, etc.)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-biblioid" name="pub-biblioid"/&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-year"&gt;
                Year:
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="number" id="pub-year" name="pub-year" /&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
        &lt;/tbody&gt;
        &lt;tbody&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-file"&gt;
                File image:
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="file" id="pub-file"/&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-file-url"&gt;
                Online-file image URL:&lt;br/&gt;
                &lt;span class="note"&gt;(same origin URL)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-file-url" name="pub-file-url"/&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
        &lt;/tbody&gt;
      &lt;/table&gt;

      &lt;div class="button-pane"&gt;
        &lt;input type="button" id="add-button" value="Add Publication" /&gt;
        &lt;input type="reset" id="register-form-reset"/&gt;
      &lt;/div&gt;
    &lt;/form&gt;

    &lt;form id="delete-form"&gt;
      &lt;table&gt;
        &lt;tbody&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-biblioid-to-delete"&gt;
                Bibliographic ID:&lt;br/&gt;
                &lt;span class="note"&gt;(ISBN, ISSN, etc.)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-biblioid-to-delete"
                     name="pub-biblioid-to-delete" /&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="key-to-delete"&gt;
                Key:&lt;br/&gt;
                &lt;span class="note"&gt;(for example 1, 2, 3, etc.)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="key-to-delete"
                     name="key-to-delete" /&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
        &lt;/tbody&gt;
      &lt;/table&gt;
      &lt;div class="button-pane"&gt;
        &lt;input type="button" id="delete-button" value="Delete Publication" /&gt;
        &lt;input type="button" id="clear-store-button"
               value="Clear the whole store" class="destructive" /&gt;
      &lt;/div&gt;
    &lt;/form&gt;

    &lt;form id="search-form"&gt;
      &lt;div class="button-pane"&gt;
        &lt;input type="button" id="search-list-button"
               value="List database content" /&gt;
      &lt;/div&gt;
    &lt;/form&gt;

    &lt;div&gt;
      &lt;div id="pub-msg"&gt;
      &lt;/div&gt;
      &lt;div id="pub-viewer"&gt;
      &lt;/div&gt;
      &lt;ul id="pub-list"&gt;
      &lt;/ul&gt;
    &lt;/div&gt;
</pre>

<h3 id="CSS_Content">CSS Content</h3>

<pre class="brush: css">body {
  font-size: 0.8em;
  font-family: Sans-Serif;
}

form {
  background-color: #cccccc;
  border-radius: 0.3em;
  display: inline-block;
  margin-bottom: 0.5em;
  padding: 1em;
}

table {
  border-collapse: collapse;
}

input {
  padding: 0.3em;
  border-color: #cccccc;
  border-radius: 0.3em;
}

.required:after {
  content: "*";
  color: red;
}

.button-pane {
  margin-top: 1em;
}

#pub-viewer {
  float: right;
  width: 48%;
  height: 20em;
  border: solid #d092ff 0.1em;
}
#pub-viewer iframe {
  width: 100%;
  height: 100%;
}

#pub-list {
  width: 46%;
  background-color: #eeeeee;
  border-radius: 0.3em;
}
#pub-list li {
  padding-top: 0.5em;
  padding-bottom: 0.5em;
  padding-right: 0.5em;
}

#msg {
  margin-bottom: 1em;
}

.action-success {
  padding: 0.5em;
  color: #00d21e;
  background-color: #eeeeee;
  border-radius: 0.2em;
}

.action-failure {
  padding: 0.5em;
  color: #ff1408;
  background-color: #eeeeee;
  border-radius: 0.2em;
}

.note {
  font-size: smaller;
}

.destructive {
  background-color: orange;
}
.destructive:hover {
  background-color: #ff8000;
}
.destructive:active {
  background-color: red;
}
</pre>

<h3 id="JavaScript_Content">JavaScript Content</h3>

<pre class="brush: js">(function () {
  var COMPAT_ENVS = [
    ['Firefox', "&gt;= 16.0"],
    ['Google Chrome',
     "&gt;= 24.0 (you may need to get Google Chrome Canary), NO Blob storage support"]
  ];
  var compat = $('#compat');
  compat.empty();
  compat.append('&lt;ul id="compat-list"&gt;&lt;/ul&gt;');
  COMPAT_ENVS.forEach(function(val, idx, array) {
    $('#compat-list').append('&lt;li&gt;' + val[0] + ': ' + val[1] + '&lt;/li&gt;');
  });

  const DB_NAME = 'mdn-demo-indexeddb-epublications';
  const DB_VERSION = 1; // Use a long long for this value (don't use a float)
  const DB_STORE_NAME = 'publications';

  var db;

  // Used to keep track of which view is displayed to avoid uselessly reloading it
  var current_view_pub_key;

  function openDb() {
    console.log("openDb ...");
    var req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onsuccess = function (evt) {
      // Equal to: db = req.result;
      db = this.result;
      console.log("openDb DONE");
    };
    req.onerror = function (evt) {
      console.error("openDb:", evt.target.errorCode);
    };

    req.onupgradeneeded = function (evt) {
      console.log("openDb.onupgradeneeded");
      var store = evt.currentTarget.result.createObjectStore(
        DB_STORE_NAME, { keyPath: 'id', autoIncrement: true });

      store.createIndex('biblioid', 'biblioid', { unique: true });
      store.createIndex('title', 'title', { unique: false });
      store.createIndex('year', 'year', { unique: false });
    };
  }

  /**
   * @param {string} store_name
   * @param {string} mode either "readonly" or "readwrite"
   */
  function getObjectStore(store_name, mode) {
    var tx = db.transaction(store_name, mode);
    return tx.objectStore(store_name);
  }

  function clearObjectStore() {
    var store = getObjectStore(DB_STORE_NAME, 'readwrite');
    var req = store.clear();
    req.onsuccess = function(evt) {
      displayActionSuccess("Store cleared");
      displayPubList(store);
    };
    req.onerror = function (evt) {
      console.error("clearObjectStore:", evt.target.errorCode);
      displayActionFailure(this.error);
    };
  }

  function getBlob(key, store, success_callback) {
    var req = store.get(key);
    req.onsuccess = function(evt) {
      var value = evt.target.result;
      if (value)
        success_callback(value.blob);
    };
  }

  /**
   * @param {IDBObjectStore=} store
   */
  function displayPubList(store) {
    console.log("displayPubList");

    if (typeof store == 'undefined')
      store = getObjectStore(DB_STORE_NAME, 'readonly');

    var pub_msg = $('#pub-msg');
    pub_msg.empty();
    var pub_list = $('#pub-list');
    pub_list.empty();
    // Resetting the iframe so that it doesn't display previous content
    newViewerFrame();

    var req;
    req = store.count();
    // Requests are executed in the order in which they were made against the
    // transaction, and their results are returned in the same order.
    // Thus the count text below will be displayed before the actual pub list
    // (not that it is algorithmically important in this case).
    req.onsuccess = function(evt) {
      pub_msg.append('&lt;p&gt;There are &lt;strong&gt;' + evt.target.result +
                     '&lt;/strong&gt; record(s) in the object store.&lt;/p&gt;');
    };
    req.onerror = function(evt) {
      console.error("add error", this.error);
      displayActionFailure(this.error);
    };

    var i = 0;
    req = store.openCursor();
    req.onsuccess = function(evt) {
      var cursor = evt.target.result;

      // If the cursor is pointing at something, ask for the data
      if (cursor) {
        console.log("displayPubList cursor:", cursor);
        req = store.get(cursor.key);
        req.onsuccess = function (evt) {
          var value = evt.target.result;
          var list_item = $('&lt;li&gt;' +
                            '[' + cursor.key + '] ' +
                            '(biblioid: ' + value.biblioid + ') ' +
                            value.title +
                            '&lt;/li&gt;');
          if (value.year != null)
            list_item.append(' - ' + value.year);

          if (value.hasOwnProperty('blob') &amp;&amp;
              typeof value.blob != 'undefined') {
            var link = $('&lt;a href="' + cursor.key + '"&gt;File&lt;/a&gt;');
            link.on('click', function() { return false; });
            link.on('mouseenter', function(evt) {
                      setInViewer(evt.target.getAttribute('href')); });
            list_item.append(' / ');
            list_item.append(link);
          } else {
            list_item.append(" / No attached file");
          }
          pub_list.append(list_item);
        };

        // Move on to the next object in store
        cursor.continue();

        // This counter serves only to create distinct ids
        i++;
      } else {
        console.log("No more entries");
      }
    };
  }

  function newViewerFrame() {
    var viewer = $('#pub-viewer');
    viewer.empty();
    var iframe = $('&lt;iframe /&gt;');
    viewer.append(iframe);
    return iframe;
  }

  function setInViewer(key) {
    console.log("setInViewer:", arguments);
    key = Number(key);
    if (key == current_view_pub_key)
      return;

    current_view_pub_key = key;

    var store = getObjectStore(DB_STORE_NAME, 'readonly');
    getBlob(key, store, function(blob) {
      console.log("setInViewer blob:", blob);
      var iframe = newViewerFrame();

      // It is not possible to set a direct link to the
      // blob to provide a mean to directly download it.
      if (blob.type == 'text/html') {
        var reader = new FileReader();
        reader.onload = (function(evt) {
          var html = evt.target.result;
          iframe.load(function() {
            $(this).contents().find('html').html(html);
          });
        });
        reader.readAsText(blob);
      } else if (blob.type.indexOf('image/') == 0) {
        iframe.load(function() {
          var img_id = 'image-' + key;
          var img = $('&lt;img id="' + img_id + '"/&gt;');
          $(this).contents().find('body').html(img);
          var obj_url = window.URL.createObjectURL(blob);
          $(this).contents().find('#' + img_id).attr('src', obj_url);
          window.URL.revokeObjectURL(obj_url);
        });
      } else if (blob.type == 'application/pdf') {
        $('*').css('cursor', 'wait');
        var obj_url = window.URL.createObjectURL(blob);
        iframe.load(function() {
          $('*').css('cursor', 'auto');
        });
        iframe.attr('src', obj_url);
        window.URL.revokeObjectURL(obj_url);
      } else {
        iframe.load(function() {
          $(this).contents().find('body').html("No view available");
        });
      }

    });
  }

  /**
   * @param {string} biblioid
   * @param {string} title
   * @param {number} year
   * @param {string} url the URL of the image to download and store in the local
   *   IndexedDB database. The resource behind this URL is subjected to the
   *   "Same origin policy", thus for this method to work, the URL must come from
   *   the same origin as the web site/app this code is deployed on.
   */
  function addPublicationFromUrl(biblioid, title, year, url) {
    console.log("addPublicationFromUrl:", arguments);

    var xhr = new XMLHttpRequest();
    xhr.open('GET', url, true);
    // Setting the wanted responseType to "blob"
    // http://www.w3.org/TR/XMLHttpRequest2/#the-response-attribute
    xhr.responseType = 'blob';
    xhr.onload = function (evt) {
      if (xhr.status == 200) {
        console.log("Blob retrieved");
        var blob = xhr.response;
        console.log("Blob:", blob);
        addPublication(biblioid, title, year, blob);
      } else {
        console.error("addPublicationFromUrl error:",
        xhr.responseText, xhr.status);
      }
    };
    xhr.send();

    // We can't use jQuery here because as of jQuery 1.8.3 the new "blob"
    // responseType is not handled.
    // http://bugs.jquery.com/ticket/11461
    // http://bugs.jquery.com/ticket/7248
    // $.ajax({
    //   url: url,
    //   type: 'GET',
    //   xhrFields: { responseType: 'blob' },
    //   success: function(data, textStatus, jqXHR) {
    //     console.log("Blob retrieved");
    //     console.log("Blob:", data);
    //     // addPublication(biblioid, title, year, data);
    //   },
    //   error: function(jqXHR, textStatus, errorThrown) {
    //     console.error(errorThrown);
    //     displayActionFailure("Error during blob retrieval");
    //   }
    // });
  }

  /**
   * @param {string} biblioid
   * @param {string} title
   * @param {number} year
   * @param {Blob=} blob
   */
  function addPublication(biblioid, title, year, blob) {
    console.log("addPublication arguments:", arguments);
    var obj = { biblioid: biblioid, title: title, year: year };
    if (typeof blob != 'undefined')
      obj.blob = blob;

    var store = getObjectStore(DB_STORE_NAME, 'readwrite');
    var req;
    try {
      req = store.add(obj);
    } catch (e) {
      if (e.name == 'DataCloneError')
        displayActionFailure("This engine doesn't know how to clone a Blob, " +
                             "use Firefox");
      throw e;
    }
    req.onsuccess = function (evt) {
      console.log("Insertion in DB successful");
      displayActionSuccess();
      displayPubList(store);
    };
    req.onerror = function() {
      console.error("addPublication error", this.error);
      displayActionFailure(this.error);
    };
  }

  /**
   * @param {string} biblioid
   */
  function deletePublicationFromBib(biblioid) {
    console.log("deletePublication:", arguments);
    var store = getObjectStore(DB_STORE_NAME, 'readwrite');
    var req = store.index('biblioid');
    req.get(biblioid).onsuccess = function(evt) {
      if (typeof evt.target.result == 'undefined') {
        displayActionFailure("No matching record found");
        return;
      }
      deletePublication(evt.target.result.id, store);
    };
    req.onerror = function (evt) {
      console.error("deletePublicationFromBib:", evt.target.errorCode);
    };
  }

  /**
   * @param {number} key
   * @param {IDBObjectStore=} store
   */
  function deletePublication(key, store) {
    console.log("deletePublication:", arguments);

    if (typeof store == 'undefined')
      store = getObjectStore(DB_STORE_NAME, 'readwrite');

    // As per spec http://www.w3.org/TR/IndexedDB/#object-store-deletion-operation
    // the result of the Object Store Deletion Operation algorithm is
    // undefined, so it's not possible to know if some records were actually
    // deleted by looking at the request result.
    var req = store.get(key);
    req.onsuccess = function(evt) {
      var record = evt.target.result;
      console.log("record:", record);
      if (typeof record == 'undefined') {
        displayActionFailure("No matching record found");
        return;
      }
      // Warning: The exact same key used for creation needs to be passed for
      // the deletion. If the key was a Number for creation, then it needs to
      // be a Number for deletion.
      var deleteReq = store.delete(key);
      deleteReq.onsuccess = function(evt) {
        console.log("evt:", evt);
        console.log("evt.target:", evt.target);
        console.log("evt.target.result:", evt.target.result);
        console.log("delete successful");
        displayActionSuccess("Deletion successful");
        displayPubList(store);
      };
      deleteReq.onerror = function (evt) {
        console.error("deletePublication:", evt.target.errorCode);
      };
    };
    req.onerror = function (evt) {
      console.error("deletePublication:", evt.target.errorCode);
    };
  }

  function displayActionSuccess(msg) {
    msg = typeof msg != 'undefined' ? "Success: " + msg : "Success";
    $('#msg').html('&lt;span class="action-success"&gt;' + msg + '&lt;/span&gt;');
  }
  function displayActionFailure(msg) {
    msg = typeof msg != 'undefined' ? "Failure: " + msg : "Failure";
    $('#msg').html('&lt;span class="action-failure"&gt;' + msg + '&lt;/span&gt;');
  }
  function resetActionStatus() {
    console.log("resetActionStatus ...");
    $('#msg').empty();
    console.log("resetActionStatus DONE");
  }

  function addEventListeners() {
    console.log("addEventListeners");

    $('#register-form-reset').click(function(evt) {
      resetActionStatus();
    });

    $('#add-button').click(function(evt) {
      console.log("add ...");
      var title = $('#pub-title').val();
      var biblioid = $('#pub-biblioid').val();
      if (!title || !biblioid) {
        displayActionFailure("Required field(s) missing");
        return;
      }
      var year = $('#pub-year').val();
      if (year != '') {
        // Better use Number.isInteger if the engine has EcmaScript 6
        if (isNaN(year))  {
          displayActionFailure("Invalid year");
          return;
        }
        year = Number(year);
      } else {
        year = null;
      }

      var file_input = $('#pub-file');
      var selected_file = file_input.get(0).files[0];
      console.log("selected_file:", selected_file);
      // Keeping a reference on how to reset the file input in the UI once we
      // have its value, but instead of doing that we rather use a "reset" type
      // input in the HTML form.
      //file_input.val(null);
      var file_url = $('#pub-file-url').val();
      if (selected_file) {
        addPublication(biblioid, title, year, selected_file);
      } else if (file_url) {
        addPublicationFromUrl(biblioid, title, year, file_url);
      } else {
        addPublication(biblioid, title, year);
      }

    });

    $('#delete-button').click(function(evt) {
      console.log("delete ...");
      var biblioid = $('#pub-biblioid-to-delete').val();
      var key = $('#key-to-delete').val();

      if (biblioid != '') {
        deletePublicationFromBib(biblioid);
      } else if (key != '') {
        // Better use Number.isInteger if the engine has EcmaScript 6
        if (key == '' || isNaN(key))  {
          displayActionFailure("Invalid key");
          return;
        }
        key = Number(key);
        deletePublication(key);
      }
    });

    $('#clear-store-button').click(function(evt) {
      clearObjectStore();
    });

    var search_button = $('#search-list-button');
    search_button.click(function(evt) {
      displayPubList();
    });

  }

  openDb();
  addEventListeners();

})(); // Immediately-Invoked Function Expression (IIFE)</pre>

<p>{{ LiveSampleLink('Full_IndexedDB_example', "Test the online live demo") }}</p>

<div class="notecard note">
<p><strong>Note:</strong> <code>window.indexedDB.open()</code> is asynchronous; the method will finish running long before the <code>success</code> event is fired. This means that a function (e.g. <code>openDb()</code>) that calls <code>open()</code> and <code>onsuccess</code> will return before the <code>onsuccess</code> handler has run. This issue is also true of other request methods such as <code>transaction()</code> and <code>get()</code>.</p>
</div>

<h2 id="See_also">See also</h2>

<p>Further reading for you to find out more information if desired.</p>

<h3 id="Reference">Reference</h3>

<ul>
 <li><a href="/en-US/docs/Web/API/IndexedDB_API">IndexedDB API Reference</a></li>
 <li><a href="https://www.w3.org/TR/IndexedDB/">Indexed Database API Specification</a></li>
 <li>IndexedDB <a class="link-https" href="https://searchfox.org/mozilla-central/search?q=dom%2FindexedDB%2F.*%5C.idl&path=&case=false&regexp=true">interface files</a> in the Firefox source code</li>
</ul>

<h3 id="Tutorials_and_guides">Tutorials and guides</h3>

<ul>
 <li><a href="http://www.html5rocks.com/en/tutorials/indexeddb/uidatabinding/">Databinding UI Elements with IndexedDB</a></li>
 <li><a href="https://msdn.microsoft.com/en-us/scriptjunkie/gg679063.aspx">IndexedDB — The Store in Your Browser</a></li>
</ul>

<h3 id="Libraries">Libraries</h3>

<ul>
 <li><a href="https://localforage.github.io/localForage/">localForage</a>: A Polyfill providing a simple name:value syntax for client-side data storage, which uses IndexedDB in the background, but falls back to WebSQL and then localStorage in browsers that don't support IndexedDB.</li>
 <li><a href="http://www.dexie.org/">dexie.js</a>: A wrapper for IndexedDB that allows much faster code development via nice, simple syntax.</li>
 <li><a href="https://github.com/jakearchibald/idb">IDB</a>: A tiny library that mostly mirrors the IndexedDB API but with small usability improvements.</li>
 <li><a href="https://github.com/erikolson186/zangodb">ZangoDB</a>: A MongoDB-like interface for IndexedDB that supports most of the familiar filtering, projection, sorting, updating and aggregation features of MongoDB.</li>
 <li><a href="http://jsstore.net/">JsStore</a>: A simple and advanced IndexedDB wrapper having SQL like syntax. </li>
</ul>