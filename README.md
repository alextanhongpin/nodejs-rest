# NodeJS + ES6

### Structuring a REST api with NodeJS and ExpressJS in ES6.

The goal is to achieve the following:
+ consistent code
+ predictable flow
+ simple

// Todo, create a better way to visualize them
The old way of writing:

For each API calls...
```javascript
// Old way of writing
// /api/followers.js
var Follower = require('../models/follower');
app.get('/v1/followers', function (req, res) {
  
  var request = {
    id: req.params.id
  }
  
  Follower.find(request, function (err, results) {
    // if (err) { handle error }
    // else handle results
    
    res.status(200).json({
      // ...object to be returned
    });
  });
});
```

However, this will get pretty repetitive soon. You will writing app.get('...') all the time.
The workaround is to loop through the routes.

The new way of writing: 
```javascript
// /api/followers/actions.js
// Actions holds the endpoint of the REST uri and at the same time tells us what to do 
// when users call the routes.

// Honestly, I prefer writing the request URL this way then using Express Router()
const GET_FOLLOWERS = '/api/v1/followers';

const GetFollowers = {
  method: 'get',
  route: GET_FOLLOWERS,
  dispatch: 'getFollowers', // dispatch a command 
}

// Return an array of Actions
module.exports = [ GetFollowers ]
```

```javascript
// /api/followers/services/get-followers.js

// Services are merely pure functions.
// They carry out the following
// 1. parse the request object
// 2. query from database
// 3. parse the response object
// 4. returns a Promise that contains the parsed response. 
// Note that at this point the data is not returned to the users that calls the API route yet.

// entity, store and graph are just pure functions written in the same file
const Services = (req, res) => {
const json =  Promise.resolve(entity(req, res)) // 1
.then(store) // 2
.then(graph) // 3

return json;// 4
}
```

```javascript
// /api/followers/command.js
// Commands are responsible for tying the Action to the Services. 
// It it possible to call more than one services in the Command.

// Import the get followers service
const getFollowersService 	= require('./services/get-followers');

let Api = {}
Api.getFollowers = (req, res) => {
	getFollowersService(req, res)
	.then(RestHelper.success(req, res)) // a helper to return the json and the correct status code
	.catch(RestHelper.error(req, res)); // a helper to return the error and the error description
}
module.exports = function follows() {
	return Api;
}
```

```javascript
// /routes
// Tie the Action to Commands
const routes = [
	'followers', // pathname to the file directory
]

routes.map((route) => {
  const command = `./api/${route}/command`;
  const action = `./api/${route}/action`;
  
  // merge both actions and commands together
  const apis = ApiFactory.mergeActionsAndCommands(require(action), require(command)(passport));
  
  // you will now need to iterate through each routes and connect then to the app
  apis.map((api) => {
    app[api.method](api.route, api.dispatch);
  });
  
});

```
