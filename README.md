# algolia-sample-todo-app

This is a TODO management application sample using Nuxt.js and Algolia.
You can learn basic usage of Algolia shown below from this repository.

- create, update and delete operation
- full text search
- highlight hitted word of search result


However, **Please do not use this code to production as it is!!!**


This sample use admin api key to create, update and delete records from front end for simplicity.
So, if you make this application public, users can get admin key and then they can do all the operation to your index.

If you create a application for production, frontend application must use readonly api key and all the update logic must to move to backend.


## Build Setup

Firstly, please sign up or sign in for [Algolia](https://www.algolia.com/).

Then, you can get your application id and admin api key from console (see API keys tab to get them).

Next, create a file named `algolia.config.js` in project root folder (same level as `package.json`) and paste your application id and api key like this.

``` javascript:algolia.config.js
export default {
  appId: '<YOUR APPLICATION ID>',
  apiKey: '<YOUR ADMIN API KEY>'
}
```

Then, install libraries and enjoy :)
``` bash
# install dependencies
$ npm install

# serve with hot reload at localhost:3000
$ npm run dev
```
