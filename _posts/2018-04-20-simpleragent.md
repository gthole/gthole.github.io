---
layout: post
title:  "Introducing SimplerAgent"
date:   2018-04-20 17:00:00 -0400
categories: node http simpleragent
---

I published a small npm package that I've dubbed [simpleragent](
https://www.npmjs.com/package/simpleragent).  It's a NodeJS simplified version
of [SuperAgent.js](http://visionmedia.github.io/superagent/) for making JSON
HTTP requests.

### Why?
Superagent is great!  It lets you structure your requests cleanly and programmatically.

But it's also pretty heavy - it contains lots of features that aren't necessary
for making simple JSON requests, which are most of the requests I make most of
the time.  If I'm bundling up a small Lambda function, it's better to have a
super small library to provide the same functionality without all the extra
stuff.

### Features
- Can probably do most of what you need for API requests
- Has no production dependencies
- Is small: roughly 100 lines of code
- Is typed with Typescript
- Has good test coverage

### Installation

Easy enough:

```bash
$ npm install --save simpleragent
```

### What's Supported?
Basic JSON requests work just fine:

```javascript
const request = require('simpleragent');

// Auth and query strings
request
    .get('http://www.example.com')
    .auth('my-user', 'pass')
    .query({name: 'bananas'})
    .end((err, resp) => {
        // Do something
    });

// Promises, within an async function
const resp = await request
    .get('http://www.example.com/foo?bar=baz')
    .promise();

// Sending JSON bodies, HTTPS, and setting headers
request
    .post('https://api.example.com/v1/fruit')
    .set('Authorization', 'Bearer ' + myApiKey)
    .send({name: 'banana', type: 'peel'})
    .then((resp) => {
        // Do something
    });
```

That's it!  None of the other hefty features are supported, and that keeps the
library nice and small with no external dependencies.
