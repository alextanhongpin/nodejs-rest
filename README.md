# NodeJS + ES6

### Code structure for REST API with NodeJS + Express, written in ES6

I've written a few different implementations, and have concluded on what I want to achieve:
+ avoid library or framework - focus more on the architecture design
+ consistent way of writing codes, avoid redundancy
+ clean codes, something that others can read and understand immediately
+ predictable flow, from the input to output
+ separation of concerns - avoid deep integration of database query in routes etc
+ centralized model parser
+ clear definition of business logic
+ able to combine multiple calls and return a combined JSON (example when calling a list, and getting total count/pagination)

I apologize if I did not make my point clear enough. 

TODO: Find a better way to explain this concept.
Here's the usual way of writing an Express routes:

For each API calls...
```javascript
// /api/followers.js
var Follower = require('../models/follower');
app.get('/v1/followers', function (req, res) {
  
  var request = {
    id: req.params.id
  }
  // callback style
  Follower.find(request, function (err, results) {
    // if (err) { handle error }
    // else handle results
    
    res.status(200).json({
      // ...object to be returned
    });
  });
});
```

The problem with this is:
+ repetitive, you will end up writing app.get('...')/app.post(...) over and over again.
+ callback hell, calling another service means you have to nest the callback together

I've resolved to this way of writing my code: 
```javascript
// /api/followers/actions.js
// Actions holds the endpoint of the REST uri and at the same time tells us what to do 
// when users call the routes.

// Honestly, I prefer writing the request URL this way then using Express Router()
const GET_FOLLOWERS = '/api/v1/followers';
// Example for getting a follower
// const GET_FOLLOWER = '/api/v1/followers/:id';

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
// welcome promises :)
const Services = (req, res) => {
	const json =  Promise.resolve(entity(req, res)) // 1
	.then(store) // 2
	.then(graph) // 3
	
	return json;// 4
}
// 1
const entity = (req, res) => {
	return {
		_id: req.params.id
	}
}
// 2
const store = (param) => {
	return Follower.find(param);
}

// 3
// can be abstracted if other routes share the same response schema
const graph = (results) => {
	return results.map((model) => {
		// do something
	})
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
// Scenario: What if you need to call multiple services?
Api.getFollowersWithCount = (req, res) => {

	Promise.all([
		getFollowersService(req, res),
		getFollowersCountService(req, res)
	]).then((data) => {
		// destructure
		const [ followers, count ] = data;
		// do something...parse/modify response
		return response;
	})
	.then(RestHelper.success(req, res)) // a helper to return the json and the correct status code
	.catch(RestHelper.error(req, res)); // a helper to return the error and the error description
}
// Scenario: What if you need to call other services (Notifications etc)?
Api.sendFollowRequest = (req, res) => {

	
	sendFollowRequest(req, res)
	.then(RestHelper.success(req, res)) // a helper to return the json and the correct status code
	.catch(RestHelper.error(req, res)) // a helper to return the error and the error description
	.then(() => {
		// always called in a promise
		// import notification service and call them
		notificationService();
	})
}

// Curry this so that you can pass other dependency if you need to (socket, passport)
module.exports = function follows(param) {
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
    // with this, you don't need to call app.get(...)/app.post(...) all the time
    // the calls are made into configurable properties that can be passed in
  });
  
});

```
