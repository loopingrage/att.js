# ATT.js Architecture

ATT.js is an event emitter framework, which allows for plugins to interact via events using ATT.js as a message bus.

The goal is that various user interface components can register handlers for standardized events, and not have to worry about the low level implemenation of the phone service or which vendor backend is used.

    UI Components (dial pad, call control bar, etc)
                        |
                        |
                      ATT.js
                        |
                        |
                Vendor Phone Plugin


By using the event pattern, multiple handlers may be registered for a single action, instead of a single callback function. This allows for multiple UI components to be easily kept in sync without coupling.


Including plugins is done by inserting additional `<script />` tags after the main `att.js` script:

    <script src="att.js" />
    <script src="att.me.js />
    <script src="att.phonenumber.js" />
    ...

## Creating a Plugin

An ATT.js plugin works similarly to a jQuery plugin, extending the core `ATT` object with new methods and properties. 


1. The basic plugin wrapper:

  An ATT.js plugin uses a closure to limit leaking variables into the global scope. The basic plugin looks like:

        (function (ATT) {
        })(ATT);

2. Adding a new method on the root `ATT` object.

  The field `ATT.fn` is equivalent to `ATT.prototype`, and can be extended to add new methods to the root `ATT` object.

        (function (ATT) {
            ATT.fn.rootMethod = function () {
                // useful things
            };
        })(ATT);

  Plugins should attempt to avoid root level methods as much as possible, so that only common, standardized methods are
  available from the root object.

3. Adding new namespaced methods to the `ATT` object.

  To prevent naming conflicts between plugins, new methods can be included in a custom namespace for the plugin.

        (function (ATT) {
            ATT.fn.myPlugin = {
                usefulFunction: function () {
                },
                another: function () {
                }
            };
        })(ATT);

  You can then access your plugin methods using:

        var att = new ATT({...});
        att.myPlugin.usefulFunction();

4. Handling events

  While adding new functionality to the `ATT` object is easily done by extending `ATT.fn`, handling events requires access to the actual object you instantiate when you call `var att = new ATT({...});`. To get access to the instantiated object, the `ATT.initPlugin` method is used.

        (function (ATT) {
            ATT.initPlugin(function (att) {
                att.on('init', function () {
                    // setup the plugin
                });
            });
        })(ATT);
      
  The `ATT.initPlugin` method is called during instantiation in `new ATT({...})`, allowing plugins to have direct access to the initialized object so that event handlers may be registered. Once all of the callback functions passed to `ATT.initPlugin` have been called, the `init` event will be emitted, signalling that the `ATT.js` framework has been loaded. If a plugin requires access to features provided by another plugin in order to complete its setup and configuration, it must wait for the `init` event to ensure that the other plugin has been loaded.

5. Creating a Spec file

  In order to automatically generate useful documentation for all att.js plugins, a plugin should provide a spec file describing the methods it provides, and any events it emits. An example spec file that uses all of the available features would look like:

        {
            "plugin": "my.plugin",
            "description": "A demo plugin for demonstrating spec files",
            "methods": {
                "runFoo": {
                    "description": "Calculates and returns foo directly, and via a callback.",
                    "parameters": [
                        {"name": "InitialValue", "type": "BarData"}
                    ],
                    "returns": "BarData",
                    "callbackArgs": [
                        {"name": "SomeOutputValue", "type": "BarData"}
                    ]
                }
            },
            "events": {
                "success": {
                    "description": "Things worked!",
                    "args": [
                        {"name": "ResultCode", "type": "string"}
                    ]
                },
                "failed": {
                    "description": "Things didn't work",
                    "args": [
                          {"name": "ResultCode", "type": "string"}
                    ]
                },
                "pong": {
                    "description": "Ping Pong!"
                }
            },
            "datatypes": {
                "BarData": {
                    "description": "A dictionary of data about bar",
                    "methods": {
                        "ping": {
                            "description": "Ping something"
                        }
                    },
                    "events": [
                        "pong"
                    ]
                }
            }
        }

6. Testing

  Testing is done via [QUnit](http://qunitjs.com/). Test suites are placed in `testing/tests` and then referenced in `testing/index.html`. 

  For example, a basic test suite looks like:

        module('foo');

        test("Ensure that runFoo returns BarData", function() {
            // How many assertions to expect during this test
            expect(2);

            var att = new ATT();

            att.on('success', function (result) {
                equal(result, 'OK');
            });

            att.runFoo({}, function (barData) {
                equal(barData, {a: 1});
            });
        });

  After saving the test suite in `testing/tests/foo.js` the `testing/index.html` file is updated to include both the plugin being tested, and the test suite:

        <!DOCTYPE html>
        <html>
            <head>
                <meta charset="utf-8">
                <title>att.js core tests</title>
                <link rel="stylesheet" href="qunit.css">
            </head>
            <body>
                <div id="qunit"></div>
                <div id="qunit-fixture"></div>
                <script src="qunit.js"></script>
                <script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
                <!-- the scripts we're testing -->
                <script src="../src/att.js"></script>
                ...
                <script src="../src/foo.js"></script><!-- The plugin to test -->
        
                <!-- the tests -->
                <script src="tests/core.js"></script>
                <script src="tests/phoneNumber.js"></script>
                ...
                <script src="tests/foo.js"></script><!-- The test suite -->
            </body>
        </html>       

  Running the entire collection of test suites is done by opening `testing/index.html` in the browser version you wish to test against.
