This is a fork of [migrated-nhs](https://bitbucket.org/MailOnline/migrated-nhs)

NHS - Node Health Service
=========================

A simple monitor and helpers to automate collection and reporting of errors in Node apps.

Installation
------------
	
	npm install node-health-service
	
Usage
=====
NHS is divided into two parts: Reporter, that can monitor and report via HTTP errors returned from your application or service, and Service that gathers reports from one or more sources and reports them bia HTTP.

Typically, in a micro-service patterned application, each micro-service will use Reporter to record any errors, and a separate micro-service will gather all of these into one convenient place.

Reporters
---------
In your micro-services you instatiate a Reporter via:

	var reporter = require('node-health-service').Reporter() ;
	...
	// Middleware to capture any HTTP responses with a status code>=400
	app.use(reporter.monitor) ; 
	
	// Route to report the last error via HTTP
	app.get('/healthcheck',reporter.lastError) ;
	
The Reporter() method accepts a single config argument with the following members:

* __errorTTL:__ The maximum time to report an error for (i.e. if no one looked within this time, stop reporting it as an error). The defualt value is 600000 (10 minutes).
* __errorThreshold:__ The minimum HTTP response code to consider as an error. The default value is 400.
	
That's it. You micro-service will now capture the last HTTP error generated and expose it via HTTP. The response generated by reporter.lastError is a JSON object in the format with a member called 'status', which is either "OK" or "ERROR", and a member called 'last' which reports the statusCode, the originating URL, the response generated by your service and the age of the error, in milliseconds.

There's a handy shortcut for the above too, if you have an Express or Connect app to monitor:

	var nhs = require('node-health-service') ;
	...
	nhs.reportOn(app) ;

Other errors
------------
What if you want to report on other, non-HTTP errors? You could generalise your solution using a custom Probe (see below), or arbitrarily set an error in general:

	var reporter = require('node-health-service').Reporter() ;
	...
	reporter.setLastError(some-object-describing-a-problem) ;
	
You can (if you want) plumb this into an existing error system, such as the console:

	var consoleError = console.error ;
	console.error = function() {
		reporter.setLastError(arguments) ;		// Save the last error for Reporting
		consoleError.apply(console,arguments) ;	// Log to the console as usual
	}

Services
--------

To aggregate and monitor your micro-services, or other external services like databases or cloud services, instantiate a Service:

	var service = require('node-health-service').Service ;
	...
	app.get('/healthcheck',service.route(config)) ;
	
The magic here is the 'config' parameter. This specifies what HTTP endpoints to monitor - both those you have implemented with 'Reporter' above, and any other HTTP endpoints that are exposed by third-party services. Note that you can also report on other (non-HTTP) services, but you'll need to build a 'probe' - see below.

The service config is a map of names you like, and specifications of where to find the status and how to interpret it. For example, to gather status from 3 sources - a service of our own that reports via the URL /healthcheck, a CouchDB and a RabbitMQ server:

	{
		"service-1":{
			"probe":"ping",
			"url":"http://localhost:8889/healthcheck"
		},
		"couch-documents":{
			"probe":"ping",
			"url":"http://user:pass@localhost:5984/documents"
		},
		"rabbit-imports":{
			"probe":"ping",
			"url":"http://user:pass@localhost:15672/api/queues/%2f/imports?columns=messages",
			"validation":"this.messages>100"
		}
	}

This declares we should monitor 3 HTTP end points. The fact that they are HTTP is defined by the "probe" member being set to "ping". "ping" is a built-in probe that knows about HTTP responses and JSON. See below for more about probes.

For the "ping" probe, we have to specify a URL. By default, a "ping" will only treat any response with an HTTP status < 400 as 'ok', and anything else as an error. This works fine for a lot of basic tests (is my DB up, can I see Amazon via HTTP), but is a bit coarse for some services. To help alleviate this, the "ping" probe has an optional "validation" expression that can turn an 'ok' response into an 'error'.

In the example above, this is used by the 'rabbit-imports' probe. Assuming it receives an 'ok' response in JSON, it then evaluates the test with 'this' set to the document returned by the service. In this case, we're testing that the queue length is less or equal to 100. If it's 100 or greater, we treat this as an error.

All the results are gathered up an reported as one big JSON document, with a top level member called 'status', which like Reporter is set to either 'OK' or 'ERROR'. The individual responses are included (using the same names as in your config object) so you can quickly and easily work out what caused any errors or generate status information.

Like with Reporter, there's a handy shortcut if you need a quick app to monitor lots of things, which doesn't need Express, Connect or anything else. The minimal server is:

	var nhs = require('node-health-service') ;
	var config = require('./config.json') ;
	nhs.listen(config.port,config.healthcheck) ;
	console.log("NHS Healthcheck on port",config.port) ;

Probes
------

The NHS Service can be given additional probes, as well as the built-in "ping" if you need to check the status of other resources.

You create additional types of probe by creating a function:

	var nhs = require('node-health-service') ;

	nhs.Service.probe.<nameOfProbe> = function(cfg) {
		return function(done) {
			var err = /* Do you testing */ ;
			if (err)
				this.error = err ;	/* Set a failure */
			else {
				this.ok = true ;	/* Set a success */
			}
			done() ;	/* Let NHS know we're done */
		}
	};

For example, to check for the presence of a file on the File system:

	var nhs = require('node-health-service') ;

	nhs.Service.probe.fileExists = function(cfg) {
		return function(done) {
			var result = this ;
			fs.exists(cfg.path,function(exists){
				if (!exists)
					result.error = cfg.path+" doesn't exist" ;
				else {
					result.ok = true ;
					result.detail = {path:cfg.path,lastChecked:new Date().toISOString()} ;
				}
				done() ;
			}) ;
		}
	};

Once this is done, we could add the following test to the config:

		"web-config":{
			"probe":"fileExists",
			"path":"/home/web/config.data"
		}

Obviously, this is a basic example. More useful tests might be for a maximum file size, minimum disk space, etc., but NHS is not designed to test everything (there are plenty of great Ops-focussed tools for that), rather it's a way to quickly check for 'live' errors in Node-based app servers.

Disclaimer
----------
The Node Health Service ('nhs') is not affiliated in any way with similarly named software or organisations.
