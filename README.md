# firebase-tips

A repository for keeping the notes of everything I learn about Firebase on my job @Growthfile.

## Cloud Functions

### Use Admin SDK in multiple modules in your project

* While using the Admin SDK for Firebase cloud functions, make sure that you `initalizeApp()` only __once__. Multiple instances of the Admin SDK can't be invoked and will result in an error.

  ```javascript
  const functions = require('firebase-functions');
  const admin = require('firebase-admin');

  // initializing the admin sdk using this line in more than once place in the cloud function will result in an error.
  admin.initializeApp(functions.config().firebase); // not allowed more than once
  ```

* If you are modularizing your project (which you obviously should be doing), chances are that you'd need to use Admin SDK's features in multiple files.

* But, as we discussed above, you can't initialize it in multiple places. A very simple solution to this is that you can initialize the Admin SDK in a seperate dedicated file and `require('admin');` it in all of your files where you want to use it.

* Initialize the Admin and authorization constructors in the same `admin-init.js` file so that you can simply use the references to the collections in your Database/Firestore/Auth as follows:

  ```javascript
  // admin-init.js
  const admin = require('./admin/admin');

  const db = admin.firestore();
  const auth = admin.auth();
  ```

* Now, to use these anywhere within your project, just `require()` the file and use it as follows:

  ```javascript
  const admin = require('./admin/admin');
  const usersRef = admin.db.collection('Users');
  //... do something with the Users collection
  ```

* Another benefit you get from this is that you won't need to `require()` the `firebase-functions` at the top of each of your files.

### Always return a promise from your cloud functions

* Always make sure that you are returning a `Promise` in return to exit the cloud function as explained in the [Google's guide on Sync, Async, and Promises](https://firebase.google.com/docs/functions/terminate-functions).

* If you don't, then chances are that the resources allocated to your cloud function will be released before the function actually resolves.

* So, after completing all the operations you that you intended the cloud function to perform, return a *promise*.

  ```javascript
  const admin = require('../admin/admin');
  // a function which executes whenever someone signs up on the Firebase platform for your app by signing up.

  exports.app = functions.auth.user().onCreate((event) => {
    const user = event.data;
    // if you are not doing anything that might use the Promises API, then simply use `Promise.resolve(true);` to make sure that the Firebase backend receives a Promise in return.
    return admin.db.collection('Users').doc(user.uid).set({
      // ... do stuff here...
    }).catch(console.log);
  });
  ```

### Multi-document writes

* If you are performing writes of multiple documents in a single shot, __always__ use `Batch()`. Don't use `documentRef.set()` for each document within your cloud function.

  ```javascript
  // bad
  const admin = require('../admin/admin');
  admin.db.collection('Users').doc('t7Fd9RbBAfgmw7Qx4VbL5WQtJkD3').set({
    // write data for the first document
  }).catch(console.log);

  admin.db.collection('Users').doc('RV0BSGEFlJQI5RwSsMFxb7vuWM03').set({
    // write data for the second document
  }).catch(console.log);
  ```

  ```javascript
  // good
  const admin = require('../admin/admin');
  const batch = admin.firestore().batch();

  batch.set(admin.db.collection('Users').doc('t7Fd9RbBAfgmw7Qx4VbL5WQtJkD3'), {
    // first document data here...
  });

  batch.set(admin.db.collection('Users').doc('RV0BSGEFlJQI5RwSsMFxb7vuWM03'), {
    // second document data here...
  });

  // handle errors in only a single place and not for every document write.
  batch.commit().catch(console.log);
  ```

* This will reduce the number of times you'll need to `catch()` errors within your code along with making the write operation atomic.

###  Get Firebase server time

* To get the server timestamp anytime from the Firebase servers, you can use the an HTTPS `onRequest()` trigger. Here's the minimal code that you can use.

  ```javascript
  exports.getServerTimestamp = functions.https.onRequest((req, res) => {
    res.status(200).send(new Date());
  });
  ```
