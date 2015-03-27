Ajax requests can easily go wrong. You can't guarantee the connection and the server always work correctly. They are also often used to send user input to the server and back, so it's vital the data is handled correctly.

But testing them can be tricky. Per definition, UNIT testing should focus on each UNIT at a time, which means that the component under test must be tested out apart from any other components it might relate to. In order to achieve such level of isolation, those components should be mocked out, in order to provide a reliable and consistent environment for stressing out every capability of our component without external interferences.

Then, let's consider unit testing the component which fires off Ajax requests. They are asynchronous, and a good unit test must be isolated, so how can we do that when the code talks to the server?

Let's take a look at some examples to learn how to test Ajax requests with ease.

## Setup

Before we begin, we need to set up the necessary tools and things. 

- Create a directory where all the necessary files will be placed
- Install Mocha, Chai and Sinon using `npm install mocha chai sinon`

### Test runner

To keep things simple, I'm going to run the tests directly in the browser. Don't worry if you would prefer using a console based runner - the tests themselves will work exactly the same. 

Below is the test runner file we will use. I'm going to call it testrunner.html. You can download it from [here](https://gist.githubusercontent.com/jhartikainen/20ed32a92ae825b48f00/raw/2296d267a1a2bf22eafc7686843483ad9770b420/testrunner.html)

```markup
<!DOCTYPE html>
<html>
	<head>
		<title>Mocha Tests</title>
		<link rel="stylesheet" href="node_modules/mocha/mocha.css">
	</head>
	<body>
		<div id="mocha"></div>
		<script src="node_modules/mocha/mocha.js"></script>
		<script src="node_modules/sinon/pkg/sinon-1.12.2.js"></script>
		<script src="node_modules/chai/chai.js"></script>
		<script>mocha.setup('bdd')</script>
		<script src="myapi.js"></script>
		<script src="test.js"></script>
		<script>
			mocha.run();
		</script>
	</body>
</html>
```

Note the paths for `mocha.css`, `mocha.js`, `sinon-1.12.2.js` and `chai.js`. Since we installed them using `npm`, they are within the `node_modules` directory. For Sinon, you may need to adjust the filename to match the installed version.

Also note the `myapi.js` and `test.js` files. These are our example module and test case, which I'll introduce next.


### Example module

Below I've created a basic module which does some Ajax requests. I'll use this to show you the techniques for testing Ajax code.

I'm calling this file `myapi.js`, you can [grab the file here](https://gist.githubusercontent.com/jhartikainen/b8f206051d7726753ed4/raw/110e321dae40e9429c6cb43521169dca83c3a74d/myapi.js)
```javascript
var myapi = {
	get: function(callback) {
		var xhr = new XMLHttpRequest();
		xhr.open('GET', 'http://jsonplaceholder.typicode.com/posts/1', true);

		xhr.onreadystatechange = function() {
			if(xhr.readyState == 4) {
				if(xhr.status == 200) {
					callback(null, JSON.parse(xhr.responseText));
				}
				else {
					callback(xhr.status);
				}
			}
		};

		xhr.send();
	},

	post: function(data, callback) {
		var xhr = new XMLHttpRequest();
		xhr.open('POST', 'http://jsonplaceholder.typicode.com/posts', true);

		xhr.onreadystatechange = function() {
			if(xhr.readyState == 4) {
				callback();
			}
		};

		xhr.send(JSON.stringify(data));
	}
};
```

This should look familiar. We have two functions, one for fetching data, another for posting data. I'm using the [JSONPlaceholder API](http://jsonplaceholder.typicode.com/), which is nice for quick testing.


### Test case skeleton

Let's create a skeleton where we can add tests as we go through each scenario. I'm calling this file `test.js`
```javascript
chai.should();

describe('MyAPI', function() {
  //Tests etc. go here
});
```
Mocha uses `describe` to create a test case. This is where we'll add our tests.

`chai.should()` enables "should style" assertions. This means we can easily verify our test results - or in other words, create assertions - by using a syntax like `someValue.should.equal(12345)`.

## Testing a GET request

The example module has a `get` function that can be used to load some data. We'll start by creating a test to verify the fetched data is parsed from JSON correctly. But the function is using XMLHttpRequest, how can we avoid sending the requests?

We included the Sinon library in our test runner. With Sinon, we can create *test-doubles*. We can replace the XMLHttpRequest with a test-double, which we can easily control in the test itself.

We need to update our test case skeleton a bit. Below you'll see the updated version, which you can download [here](https://gist.githubusercontent.com/jhartikainen/cc05a86fd87b4b9aa17d/raw/2b79943e41baf56aa741a33851d0047a3c1b0cf6/test.js)

```javascript
chai.should();

describe('MyAPI', function() {
	beforeEach(function() {
		this.xhr = sinon.useFakeXMLHttpRequest();

		this.requests = [];
		this.xhr.onCreate = function(xhr) {
			this.requests.push(xhr);
		}.bind(this);
	});

	afterEach(function() {
		this.xhr.restore();
	});


    //Tests etc. go here
});
```

As their names might lead you to guess, `beforeEach` and `afterEach` let you run some code before and after each test. Before each test, we enable a fake XMLHttpRequest and put each created request into an array. By saving the values into `this.xhr` and `this.requests`, they are available to be used elsewhere within the test case.

After each test, we use `this.xhr.restore()` to restore the original XMLHttpRequest object back. 

Now we can write the first test:

```javascript
it('should parse fetched data as JSON', function(done) {
	var data = { foo: 'bar' };
	var dataJson = JSON.stringify(data);

	myapi.get(function(err, result) {
		result.should.deep.equal(data);
		done();
	});

	this.requests[0].respond(200, { 'Content-Type': 'text/json' }, dataJson);
});
```

First some data: An object and its JSON version. We define these to avoid having to repeat the values within the test. Next, we call `myapi.get`. In its callback, we use Chai's helper  `result.should.deep.equal` to make sure the result equals the expected data. We also call `done()`, which tells Mocha the asynchronous  test is complete. Notice `done` was a parameter on the test function.

Finally, we call `this.requests[0].respond`. Remember the `beforeEach` function? It added a listener to put all created fake XMLHttpRequests into `this.requests`. When our test calls `myapi.get`, it creates a request. The request gets put into the `this.requests` array, and we can then access it here. 

Normally XMLHttpRequests don't have a `respond` function. Here `respond` is a function in the fake, and its used to respond to the fake request. We set the status to `200`, indicating success. We set a `Content-Type` header as `text/json` because the data is JSON formatted. The last parameter is the response body, which we set to the `dataJson` variable we set up earlier.

The fake XMLHttpRequest pretends the response was sent by a web server and `myapi.get` sees it and calls the callback. Looking at the callback, notice we compare the result against the `data` variable:
```javascript
myapi.get(function(err, result) {
	result.should.deep.equal(data);
	done();
});
```

If the response is handled correctly, the response should be parsed into an object. Since we set up the `data` and `dataJson` variables to represent that earlier, we simply compare against that.

We can now run the tests in our browser of choice. Open the `testrunner.html` file, and you should see a message about the test passing.


## Testing a POST request

Our example module also has a `post` function which sends some data. The sent data needs to be encoded as JSON, and put into the request body. Let's write a test for this.

```javascript
it('should send given data as JSON body', function() {
	var data = { hello: 'world' };
	var dataJson = JSON.stringify(data);

	myapi.post(data, function() { });

	this.requests[0].requestBody.should.equal(dataJson);
});
```

Like before, we first define the test data. Then, we call `myapi.post`. This time we only need to verify the data is correctly converted into JSON, so we leave the callback empty.

The last line has an assertion to confirm the behavior. Same as before, we access the created fake XMLHttpRequest, but this time we need to verify it contains the correct data. We do this by comparing the `requestBody` property with the test data we defined earlier.

## Testing for failures

As the final example, let's test a failing request. This is important because network connections can have problems and the server can have problems, and we shouldn't leave the user wondering "What happened?". 

The example module uses node-style callbacks - It should pass errors into the callback as the first parameter. We can test it like below:

```javascript
it('should return error into callback', function(done) {
	myapi.get(function(err, result) {
		err.should.exist;
		done();
	});

	this.requests[0].respond(500);
});
```

This time we don't need any data. We call `myapi.get`, and verify an error parameter exists. The last line sends an erroneous response code, 500 Internal Server Error, to trigger the error handling code.


## Conclusion and next steps

Ajax requests are important to test. If you verify they work correctly, the rest of your application can trust it will not get bad values from them. That can save you a lot of headaches down the line.

What if you're using jQuery Ajax instead of plain XMLHttpRequest? No problem. You can use exact same methods shown here to test your code. You can use the same approach with Sinon's fake XMLHttpRequests, as behind the scenes, jQuery also uses the basic XMLHttpRequest object.

To get the most out of this, here's two things I recommend you should do:

- As an exercise, implement a test for error handling for the `myapi.post` function.
- Move your tests into a console based tool to simplify running them

The best thing about Mocha is you can use the same techniques I showed here to test NodeJS code too. If you want to learn how, see my article on [how to test NodeJS HTTP requests with Mocha](http://codeutopia.net/blog/2015/01/30/how-to-unit-test-nodejs-http-requests/)



