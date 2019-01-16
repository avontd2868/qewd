# Contents

- [QEWD Interactive Applications](#qewd-interactive-applications)
- [The QEWD WebSocket Client](#the-qewd-websocket-client)
  - [Loading *ewd-client* into the Browser](#loading-ewd-client-into-the-browser)
  - [Application Registration](#application-registration)
    - [The *start()* Method](#the-start()-method)
  - [Sending Messages from the Browser](#sending-messages-from-the-browser)
  - [Handling Messages Sent from the QEWD Back-end](#handling-messages-sent-from-the-qewd-back-end)
- [The QEWD Application Back-end](#the-qewd-application-back-end)
  - [Defining A QEWD Interactive Application](#defining-a-qewd-interactive-application)
  - [Locating Client-Side Code](#locating-client-side-code)
  - [Locating Server-Side Code](#locating-server-side-code)
  - [Message Handler Functions](#message-handler-functions)
    - [Arguments](#arguments)
    - [Message Handler Function Context](#message-handler-function-context)
    - [The Incoming Message Object](#the-incoming-message-object)
    - [Returning a Response from your Message Handler Function](#returning-a-response-from-your-message-handler-function)
    - [Accessing the User's QEWD Session](#accessing-the-users-qewd-session)
    - [Accessing the Integrated Persistent JSON/Document Database](#accessing-the-integrated-persistent-jsondocument-database)
    - [Intermediate Messages](#intermediate-messages)


# QEWD Interactive Applications

In addition to its REST support, QEWD provides in-build support for interactive, browser-based applications.  By default, these rely on WebSockets, rather than HTTP, for the messaging between the browser and the QEWD back-end.

WebSockets not only provide faster communication between browser and back-end, they also allow you to break free of the limitations of the request/response-based HTTP protocol.  When you first start an interactive QEWD application, it automatically creates for you a persistent WebSocket connection between the browser and QEWD back-end.  Once in place, rather than waiting for requests from the browser, the QEWD back-end can, at any time, send messages to the browser.  These trigger events in the browser to handle those incoming messages.

Conversely, if you send a message from the browser over the WebSocket connection, you won't necessarily expect and need to handle a response from the QEWD back-end.  *Fire and forget* messaging from the browser to the QEWD back-end is very simple to achieve.

That having been said, request/response messaging from the browser is also catered for and made very simple to use.

If you've used the *qewd-monitor* application to monitor your QEWD instances, you've already seen one such interactive, WebSocket-based application in action.  However, you can create your own applications which can run either as well as or instead of REST-based applications on a QEWD instance.

# The QEWD WebSocket Client

When you start QEWD-Up, you'll see that a new sub-folder - */www* is created within your QEWD-Up application folder, and within this you'll discover not only a symbolic link to the *qewd-monitor* front-end resources, but also a symbolic link to a file named *ewd-client.js* ([See here for details](https://github.com/robtweed/ewd-client)].

This pre-built client module is designed to be loaded into the browser, where it handles all the security of your application and the WebSocket messaging with your application's QEWD back-end.

The *ewd-client* module is designed to be compatible with any style of browser-based application, including:

- plain manually-created HTML/JavaScript web applications that do not use a JavaScrip framework
- web applications using a JavaScript framework (eg jQuery, React, Vue.js, Angular)
- Mobile applications using, for example, React Native

*ewd-client* requires a WebSocket module: the recommended on is [*socket.io*](https://github.com/socketio/socket.io).  *ewd-client* uses the client module, while QEWD uses the Node.js-based server module.  By default, QEWD uses *socket.io*.


There are four key aspects of *ewd-client* that you need to understand:

- loading *ewd-client* into the browser
- application registration
- sending messages from the browser and handling them in the QEWD back-end
- handling messages sent from the QEWD back-end to the browser

## Loading *ewd-client* into the Browser

In plain manually-created HTML/JavaScript applications, you'll load *ewd-client* into the browser using a *<script>* tag, eg:

      <script src="/socket.io/socket.io.js"></script>
      <script src="/ewd-client.js"></script>

**Note:** in the above example, both */socket.io/socket.io.js* and */ewd-client.js* are automatcally available to you on your QEWD-Up instance.


If you're using a framework such as React or Angular, you'll have *bundled* all the JavaScript resources into a single JS file - *ewd-client* and [*socket.io-client*](https://github.com/socketio/socket.io-client) are resources that you must include in your project.  For React applications, it's worth looking at the [*react-qewd*](https://github.com/wdbacker/react-qewd) module that has been created from *ewd-client*.


## Application Registration

Once loaded, the first step is to invoke the *ewd-client* module's *start()* method.  This performs the following steps which are known as *application registration*:

- *ewd-client* attempts to create a persistent WebSocket connection with the QEWD back-end
- one a WebSocket connection is established, *ewd-client* sends a message to the QEWD back-end, specifying the name of the QEWD Application it wants to use
- the QEWD back-end creates a new QEWD Session and returns a response message to the browser containing an opaque QEWD Session token
- *ewd-client* retains the QEWD Session token within its closure, and creates a *send()* method that you will use for all your application messaging
- *ewd-client* emits an *ewd-registered* event, denoting that it is now ready and safe for messages to be exhanged between the browser and QEWD back-end

As part of its security management, on successful registration, *ewd-client* deletes the *socket.io* interface so that it cannot be used by the browser user for any other purpose.  The *ewd-client send()* method, however, retains access to *socket.io*.

Furthermore, because *ewd-client* automatically adds the registered QEWD Session Token to your messages from within its closure, the QEWD back-end will always recognise your messages as requiring handling **only**by the registered application - the browser user cannot manipulate the messages to attempt to access any other QEWD application.

Additionally, as a result of the QEWD Session Token that the *send()* method attaches to all outgoing messages from the browser, those messages are locked to the QEWD Session that was created at registration.


Registration is triggered by invoking *ewd-client's start()* method, so let's now examine how it is used.

### The *start()* Method

The arguments for the *start()* method are as follows:

- **application:** The name of the QEWD Application that will handle messages from the browser
- **$**: (Optional) If you are using jQuery, you should set this to the loaded/imported jQuery library/module
- **io**: Set this to the loaded/imported WebSocket module (eg the instance of *socket.io*)
- **customAjaxFn**: (Optional) Your own custom Ajax handler module if:
  - you want to use Ajax messaging instead of WebSocket messaging; and
  - you don't want to use *ewd-client*'s built-in jQuery-based Ajax messaging function
- **url**: The URL of the back-end QEWD server, eg *http://qewd.example.com:8080*

You can either specify these arguments separately or as properties of a single argument object, eg:

      EWD.start('myQEWDApplication', $, io, null, 'http://qewd.example.com:8080')

or:

      EWD.start({
        application: 'myQEWDApplication',
        $: $,
        io: io,
        url: 'http://qewd.example.com:8080'
      });

The arguments you provide for the *start()* method will depend on your style of application - specifically whether:

- you loaded *ewd-client* into the browser using a *<script>* tag and sourced it from the same origin as the HTML page; or
- *ewd-client* was pre-bundled into the JavaScript (eg React, Angular, React Native)

In the former instance, you do not need to specify the URL - it is implicitly the Origin server.  For example:

    <script src="//ajax.googleapis.com/ajax/libs/jquery/2.2.0/jquery.min.js"></script>
    <script src="/socket.io/socket.io.js"></script>
    <script src="/ewd-client.js"></script>


    EWD.start('myQEWDApplication', $, io)


Note that in the above example, *$*, *io* and *EWD* will have been implicitly created by the libraries loaded in the *<script>* tags.


In the latter instance, you'll need to explicitly specify the *url*, eg:

      import io from 'socket.io-client'
      // import $ from 'jquery'
      import EWD from 'ewd-client'

      EWD.start({
        application: 'myQEWDApplication',
        io: io,
        //$: $,
        url: 'http://qewd.example.com:8080'
      });


**NOTE**: *ewd-client* does not have any explicit dependency on *jQuery*.  Specifying the *$* argument is optional and **ONLY** necessary if:

- you want to use Ajax instead of WebSocket messaging for communication between the browser and your QEWD instance; AND
- you want to use *ewd-client's* built-in *jQuery*-based Ajax handler function.

If you want to use WebSocket messaging (the recommended approach), you **DO NOT** need to specify the *$* argument.

If you want to use Ajax messaging **BUT** you want to use your framework's own Ajax handler, or another one such as that provided by *axios*, then you don't specify the *$* argument but you do define your Ajax handler function using the *customAjaxFn* argument.

The arguments for a *customAjaxFn* are:

- params: an object, created by *ewd-client* containing the message and relevant Ajax properties
- success: *ewd-client*'s success handler function
- fail: *ewd-client*'s fail handler function

For example:


      import axios from 'axios'

      EWD.start({
        application: 'myQEWDApplication',
        io: io,
        url: 'http://qewd.example.com:8080'
        customAjaxFn: function(params, success, fail) {
          let data = JSON.stringify(params.data)
          axios({
            url: params.url,
            method: 'post',
            headers: {
              'Content-Type': params.contentType
            },
            data,
            timeout: params.timeout
          })
            .then(function (response) {
              success(response.data)
            })
            .catch(function (error) {
              if (error.response) {
                success(error.response.data)
              } else {
                fail(error.message || 'unknown ajax error')
              }
            })
        }
      });


### The *ewd-registered* Event

Although usually a rapid process, *ewd-client* registration does take a finite amount of time, and until it has completed it is not safe for the browser to use the *send()* method.  Indeed, the *send()* method does not exist until registration is complete.

It is therefore important that the browser-side logic of your application listens for the *ewd-registered* event before commencing any activity that involves message exchange.

*ewd-client* provides you with an Event Handler function: **EWD.on()** which you should use for this purpose.

In a simple manually-created web application that uses jQuery, we could apply the following logic to controllably start an application:

      $(document).ready(function() {

        EWD.on('ewd-registered', function() {
          // OK the app is now ready for use!
          // commence the application's logic

          // Use EWD.send() to send messages to QEWD back-end

        });

        EWD.start({
          application: 'test-app', 
          io: io
        });

      });


This logic and approach is fairly simple to adapt for use with Angular.js


In applications built using the React framework, it becomes a bit more tricky to ensure that *ewd-client* registration has completed before letting the application rendering to properly take place.  However, you can use the [*react-qewd*](https://github.com/wdbacker/react-qewd) module which does all the hard work for you. 

Here's an example:

      import React from 'react';
      import { render } from 'react-dom';
      import io from 'socket.io-client';
      import { QEWD, QEWDProvider } from 'react-qewd';
      import App from 'myApp';

      let qewd = QEWD({
        application: 'test-app',
        url: 'http://localhost:8080',
        io: io
      });

      function AppContainer(props) {
        return (
          {
            props.qewdProviderState.registered ?
              <App qewd={qewd} />
            :
              <div>Please wait...</div>
          }
        )
      }

      render(
        <QEWDProvider qewd={qewd}>
          <AppContainer />
        </QEWDProvider>,
        document.getElementById('content')
      );


Note how the fully started-up instance of *ewd-client* is passed as a prop to your application:

      <App qewd={qewd} />

From within your application component(s), you can send your messages using *this.props.qewd.send()*


Similarly, for applications that use the Vue.js and Nuxt.js frameworks, you should consider using [*vue.qewd*](https://github.com/wdbacker/vue-qewd)


## Sending Messages from the Browser

Once *ewd-client* has registered your application, you can send messages to the QEWD back-end using its *send()* method.

This method has two arguments:

- **messageObj**: (Mandatory) Object defining the message to send to the QEWD back-end.  This object should be defined using three properties:
  - **type**: the *type* of message you want to define the message as.  Type names are up to you to define and can be any string value
  - **params**: an object containing the parameters you want to speficy for your message.  The content and structure of this object is up to you to define
  - **ajax**: (Optional).  If defined and set to *true*, then the message is sent using Ajax instead of via the QEWD WebSocket connection.  By default, messages are sent via the WebSocket connection.
- **callback**: (Optional) Callback function for handling the response.  This function has a single argument:
  - responseObj: Object containing the response message which has two key properties:
    - type: the type of the original request message
    - message: the response message object which is what you will create in your back-end message handler 

For example:

      var msg = {
        type: 'login',
        params: {
          username: 'rob',
          password: 'secret'
        }
      };
      EWD.send(msg, (responseObj) => {
        console.log('Response was ' + JSON.stringify(responseObj.message));
      });

The example above would send the *login* message as a WebSocket message.

To send it as an Ajax message, simply add *ajax: true* as a property:

      var msg = {
        type: 'login',
        params: {
          username: 'rob',
          password: 'secret'
        },
        ajax: true
      };
      EWD.send(msg, (responseObj) => {
        console.log('Response was ' + JSON.stringify(responseObj.message));
      });


See later for details on how to handle messages in the QEWD Back-end and return responses.


## Handling Messages Sent from the QEWD Back-end

If you send a message as an Ajax message (ie by setting the *ajax* message property to *true*), then you **must** expect a response **and** you will normally handle that response using the *send()* method's callback function as shown in the previous section.

If, however, you're using WebSockets, it's possible for the QEWD Back-end to send messages to the browser at any time, without an initiating request arriving from the browser.

The way to create messages in the QEWD back-end is described later.  

We've already seen in the previous section that if you are sending a WebSocket message from the client that results in a single response message being returned from QEWD, then you can handle the response by using *ewd-client's send()* method's callback function.

However, there are other circumstances where you'll want to handle incoming WebSocket messages from the back-end independently, including:

- QEWD back-end message handlers that return more than one response to an incoming message
- messages independently sent from the QEWD back-end, ie without a triggering request from the browser
- when using a framework such as React, you will often want to handle an incoming response in a Component that is higher up the Component hierarchy, so that handling the response triggers a re-rendering of that part of the sub-tree of Components.

In all three such situations, you should use *ewd-client's on()* event handler method:

      EWD.on(messageType, (messageObject) => {
        // handle the incoming message from the QEWD back-end
      });

If you are using *react-qewd*, you'll probably access it using:

      this.props.qewd.on(message, callback);


The arguments are:

- **messageType**: string value that identifies the type of message being sent from QEWD

- **callback**: Callback function that is triggered on receipt of an incoming message of the specified type.  The incoming message object is provided as its one and only argument.

For example:

      EWD.on('myTestMessage', (messageObj) => {
        console.log('handle incoming message: ' + JSON.stringify(messageObj));
      });

All incoming messages will have a *type* property (which, of course, is used to trigger the *on()* Event Handler function.  The rest of the message structure and content will depend on the QEWD back-end method that generated it.


# The QEWD Application Back-end

The back-end of an interactive QEWD application is defined in your QEWD-Up Application Directory.  Interactive applications are supported in all three QEWD-Up Modes:

- Native Monolith
- Docker Monolith
- Docker MicroServices

In all three cases, a QEWD-Up instance can support as many interactive applications as you like, and you can run interactive applications together with REST applications, or run a QEWD-Up instance with just interactive applications and no REST APIs at all.

In the case of the Docker Monolith mode, the Orchestrator and/or any of the MicroService QEWD Instances can run interactive QEWD applications.  The key proviso is that each QEWD instance that runs an interactive application is exposed via a host port, so that the browser can make a WebSocket connections to that port.


## Defining A QEWD Interactive Application
 
There are two parts to defining an interactive QEWD Application in QEWD-Up:

- creating a home for the browser/client-side code, including its HTML, CSS and JavaScript resources
- defining the QEWD/server-side code, which consists of handler methods for each message type sent from the browser by the client side of the code.

Each of your QEWD applications must be given a unique name - this can be any string value.  This name will be used to identify the application on both the client and server side.

## Locating Client-Side Code

[The earlier section above](#the-qewd-websocket-client) described how to use the *ewd-client* module in your client side code.  Having created that code, where you locate it will depend on whether:

- your QEWD-Up instances are sitting behind a reverse-proxy such as NGINX
- your QEWD Monolith instance or Orchstrator instance is directly exposed to the external users; or

### Proxied Set=up

If you are using a reverse-proxy such as NGINX, then your client-side resources should be placed in the NGINX Web Server root path, eg:

      /usr/share/nginx/html/myQEWDApplication

NGINX will then need to be configured to act as a proxy to your QEWD instance(s).  [Documented separately](#link-here)

### Directly-exposed Set-up

If your QEWD instance is acting directly as the externally-facing web server, you should place the QEWD Application code in the appropriate */www* sub-folder within your QEWD-Up application folder.

If you've previously started your QEWD-Up instance(s), you'll find a */www* subfolder already present.  If not, just create it and add your application code.  QEWD-Up will add its additional files to it when you next start it up.

For example, for an interactive QEWD Application named *myQEWDApplication*:


### Monolith

        ~/dockerExample
            |
            |_ configuration
            |            |
            |            |_ config.json
            |
            |_ www
            |    |
            |    |_ myQEWDApplication
            |            |
            |            |_ index.html
            |            |
            |            |_ app.js etc....
            |


### MicroService: Orchestrator

        ~/microserviceExample
            |
            |_ configuration
            |
            |_ orchestrator
            |    |
            |    |_ www
            |        |
            |        |_ myQEWDApplication
            |            |
            |            |_ index.html
            |            |
            |            |_ app.js etc....


### MicroService: Other MicroService

        ~/microserviceExample
            |
            |_ configuration
            |
            |_ login_service
            |    |
            |    |_ www
            |        |
            |        |_ myQEWDApplication
            |            |
            |            |_ index.html
            |            |
            |            |_ app.js etc....



When you start up the QEWD-Up instance, you'll see that symbolic links to additional files and folders are automatically added to the */www* folders by QEWD-Up, eg:


        ~/dockerExample
            |
            |_ configuration
            |            |
            |            |_ config.json
            |
            |_ www
            |    |
            |    |_ ewd-Client.js
            |    |
            |    |_ myQEWDApplication
            |    |       |
            |    |       |_ index.html
            |    |       |
            |    |       |_ app.js etc....
            |    |
            |    |_ qewd-monitor
            |    |       |
            |    |       |_ index.html
            |    |       |
            |    |       |_ bundle.js etc....


Please leave these additional generated links/files untouched.


## Locating Server-Side Code

The way in which you define the server-side code of a QEWD Interactive application is very similar to how QEWD-Up REST APIs are defined.

[You've seen earlier](#sending-messages-from-the-browser) how messages sent from the browser specify its *type*.  The server-side of a QEWD Interactive application consists mainly of *message handler functions*: functions you write that specify how each of these message types is to be handled.  What a *message handler function* does is completely up to you, provided:

- the function signature, in terms of its arguments, meets the QEWD requirements
- you use the methods provided to return any response messages and signal completion of your handler's logic

The first step is to create a sub-folder named *qewd-apps* in your QEWD-Up directory for your QEWD Interactive applications.  The location of this sub-folder depends on the mode of QEWD-Up application you're using.

### Monolith

        ~/dockerExample
            |
            |_ configuration
            |            |
            |            |_ config.json
            |
            |_ qewd-apps
            |    



### MicroService: Orchestrator

        ~/microserviceExample
            |
            |_ configuration
            |
            |_ orchestrator
            |    |
            |    |_ qewd-apps



### MicroService: Other MicroService

        ~/microserviceExample
            |
            |_ configuration
            |
            |_ login_service
            |    |
            |    |_ qewd-apps



Within the *qewd-apps* folder, you create a sub-folder for each application you want to make available.  The sub-folder name must match the name of the application.  For example, if you wanted to define an application with a name of *myQEWDApplication*:

### Monolith

        ~/dockerExample
            |
            |_ configuration
            |            |
            |            |_ config.json
            |
            |_ qewd-apps
            |       |
            |       |_ myQEWDApplication



### MicroService: Orchestrator

        ~/microserviceExample
            |
            |_ configuration
            |
            |_ orchestrator
            |    |
            |    |_ qewd-apps
            |       |
            |       |_ myQEWDApplication



### MicroService: Other MicroService

        ~/microserviceExample
            |
            |_ configuration
            |
            |_ login_service
            |    |
            |    |_ qewd-apps
            |       |
            |       |_ myQEWDApplication


You can now define the *message handler functions* that your application will require.  Create a sub-folder for each one, using the message *type* as the sub-folder name, and then, within that sub-folder, create the function as a module file named *index.js*

For example, for a message type of *login*:

        ~/microserviceExample
            |
            |_ configuration
            |
            |_ login_service
            |    |
            |    |_ qewd-apps
            |    |  |
            |    |  |_ myQEWDApplication
            |    |  |         |           
            |    |  |         |_ login
            |    |  |         |     |
            |    |  |         |     |- index.js



## Message Handler Functions

Each Message Handler Function *index.js* file must export a function with the following signature:

      module.exports = function(messageObj, session, send, finished) {
      };

or

      module.exports = (messageObj, session, send, finished) => {
      };


### Arguments

The arguments of a *message handler function* are:

- **messageObj**: The incoming message object, which will be identical to the object you sent from the browser using the [*ewd-client's send()*](#sending-messages-from-the-browser) method
- **session**: The QEWD Session for the incoming message instance.  QEWD uses the session token that was included in the message by the *ewd-client* module to automatically link your handler function to the user's QEWD Session
- **send**: a function provided by QEWD that you can use to send *intermediate* messages to the browser (see later)
- **finished**: a function provided by QEWD that you must use to return your handler's primary response (if any) and with which you signal to QEWD that you have finished using its Worker process (so that it can be returned to QEWD's available pool).

### Message Handler Function Context

The *this* object within your *message handler function* is the QEWD context which provides you access to, for example:

- **this.db.use**: the function to use to instantiate a *document node object*, which is how you access the integrated persistent JSON / document database
- **this.userDefined**: an object containing your QEWD configuration options *and* any custom properties that you defined at startup

### The Incoming Message Object

The first argument of a *message handler function* provides access to the incoming message object.  For example, suppose you used the *ewd-client's send()* method to send the following message from the browser:

      var msg = {
        type: 'login',
        params: {
          username: 'rob',
          password: 'secret'
        }
      };
      EWD.send(msg, (responseObj) => {
        // handle the reseponse returned by the QEWD message handler function
      });

You would handle this using a *message handler function* within a folder named *login*, and the *messageObj* argument would contain an exact copy of the message object you sent, ie:

      {
        type: 'login',
        params: {
          username: 'rob',
          password: 'secret'
        }
      }

So, your message handler logic for this example might look like this:

      module.exports = (messageObj, session, send, finished) => {
        var username = messageObj.params.username;
        var password = messageObj.params.password;
        // perform the appropriate logic to confirm the validity of the username and password
      };


### Returning a Response from your Message Handler Function

You return a response from your *message handler function* using the *finished()* method which has a single argument: *responseObject*.

The structure and content of the response object is up to you, but to return an error response, you should use the reserved response object structure:

      {error: error_message_text}

For example, extending the above example:

      module.exports = (messageObj, session, send, finished) => {
        var username = messageObj.params.username;
        var password = messageObj.params.password;
        // simple hard-coded validation by way of example:
        if (username !== 'rob' && password !== 'secret') {
          return finished({error: 'Invalid login attempt'});
        }
        finished({ok: true});
      };

The response object that you specify in your *finished()* method will be returned to the *ewd-client*, and will be contained in the *message* property of the response it receives.  So, for example, taking the *ewd-client* example we used above:

#### Successful login attempt

      var msg = {
        type: 'login',
        params: {
          username: 'rob',
          password: 'secret'
        }
      };
      EWD.send(msg, (responseObj) => {
        console.log('Response was ' + JSON.stringify(responseObj.message));
        // responseObj.message.ok = true
      });

#### Unsuccessful login attempt

      var msg = {
        type: 'login',
        params: {
          username: 'xxx',
          password: 'yyyyyy'
        }
      };
      EWD.send(msg, (responseObj) => {
        console.log('Response was ' + JSON.stringify(responseObj.message));
        // responseObj.message.error = 'Invalid login attempt'
      });

**IMPORTANT**: You must **ALWAYS** terminate your *message handler function*'s logic by invoking the *finished()* function.  Failure to do so will mean that the QEWD Worker process that invokes your *message handler function* will never be released back to QEWD's worker pool.  If a number of such message types are handled, you'll quickly run out of available Worker processes and QEWD will queue up subsequent messages until you manually force down the Worker processes using the *qewd-monitor* application or you restart QEWD (which will result in the loss of queued messages).

If your *message handler function* includes asynchronous logic, then you must make sure you invoke the *finished()* method from within the asynchronous logic's callback.  For example:

      module.exports = (messageObj, session, send, finished) => {
        setTimeout(() => {
          finished({ok: true});
        }, 5000);
      };

In the example above, the QEWD Worker process will not be released until after 5 seconds, when the *setTimeout* has triggered.


### Accessing the User's QEWD Session

You can use the QEWD Session to save and retrieve user-specific information that you want to exist for the duration of the user's session.  

A user's session starts when they load your applications's client-side resources into their browser and your code invoked the *ewd-client's start()* method.

In most situations, a user's session stops when it times out, through lack of activity.  By default a user session will expire after 5 minutes: this initial timeout value is set when *ewd-client* first registers the application.

You can reset the session timeout value from within any of your *message handler functions*.  For most applications that require a user authentication/login step, your *login message handler function* will be the normal place to do this.

The QEWD Session object is a *Document Node Object* (ie it is implemented using the integrated persistent JSON / document database), and is made available you via the 2nd argument of your *message handler function*.  It has a number of reserved properties and methods that you may use, but you can create and maintain your own custom information within its *data* property.

[See here for detailed documentation about the QEWD Session Object](#not-yet-documented).

Here's an example demonstrating typical use of the QEWD Session.

      module.exports = (messageObj, session, send, finished) => {
        var username = messageObj.params.username;
        var password = messageObj.params.password;
        // simple hard-coded validation by way of example:
        if (username !== 'rob' && password !== 'secret') {
          return finished({error: 'Invalid login attempt'});
        }

        // valid login, so flag the user's session as authenticated
        // and reset and update the session timeout

        session.authenticated = true;
        session.timeout = 3600; // 1 hour inactivity timeout
        session.updateExpiry(); // apply the new timeout immediately

        session.data.$('username').value = username; // add username to session

        finished({ok: true});
      };


Your other *message handler functions* can check the *session.authenticated* property to confirm that the user has logged in - you'll want to prevent unauthorised access by users who have not logged in!  They can also make use of or update the user's session information.  For example:


      module.exports = (messageObj, session, send, finished) => {
        if (!session.authenticated) {
          return finished({error: 'You have not logged in'});
        }
        // get the user's username with which they logged in:

        var username = session.data.$('username').value;

        // save some information that you sent from the browser into the user's session

        session.data.$('myNewInfo').setDocument(messageObj.params.newInfo);

        // return the username back to the browser

        finished({username: username});
      };


### Accessing the Integrated Persistent JSON/Document Database

You can access QEWD's integrated Persistent JSON Database from within your *message handler functions* and make use of it for whatever purposes you require.  The key first step is to use the *this.db.use()* function to instantiate what is known as a *Document Node Object*.  For example:

      // create a Document Node Object that references the topmost node - a physical Global
      var userDoc = this.db.use('Users');

      // then create a Document Node Object that references the former's 'administrator' child node

      var adminDoc = userDoc.$('admininstrators');


The latter Document Node Object could alternatively be created in one step:

      var adminDoc = this.db.use('Users', 'administrators');

From this point on, you can use and apply all the methods and techniques described in the training presentation slide decks listed below:

#### Introduction to Global Storage Databases

- [Modelling NoSQL Databases using Global Storage](https://www.slideshare.net/robtweed/ewd-3-training-course-part-18-modelling-nosql-databases-using-global-storage)
- [Basic Access to a Global Storage Database from JavaScript: the cache.node APIs](https://www.slideshare.net/robtweed/ewd-3-training-course-part-19-the-cachenode-apis)

#### JavaScript Abstraction of Global Storage

- [The DocumentNode Object](https://www.slideshare.net/robtweed/ewd-3-training-course-part-20-the-documentnode-object)
- [Persistent JavaScript Objects](https://www.slideshare.net/robtweed/ewd-3-training-course-part-21-persistent-javascript-objects)
- [Traversing Documents](https://www.slideshare.net/robtweed/ewd-3-training-course-part-22-traversing-documents-using-documentnode-objects)
- [Traversing a Range of Nodes](https://www.slideshare.net/robtweed/ewd-3-training-course-part-23-traversing-a-range-using-documentnode-objects)
- [Traversing a Document's Leaf Nodes](https://www.slideshare.net/robtweed/ewd-3-training-course-part-24-traversing-a-documents-leaf-nodes)
- [Global Storage as a Document Database](https://www.slideshare.net/robtweed/ewd-3-training-course-part-25-document-database-capabilities)
- [Event-Driven Indexing](https://www.slideshare.net/robtweed/ewd-3-training-course-part-26-eventdriven-indexing)

#### QEWD's Session Storage

In the previous section you saw examples of how to use the QEWD Session which makes use of this same JSON Database.  The user-defined custom storage part of the QEWD session is exposed as a *Document Node Object*, allowing, once again, all the above techniques and methods to be applied to your custom Session storage.  

[The QEWD Session is described in more detail here](https://www.slideshare.net/robtweed/ewd-3-training-course-part-27-the-ewd-3-session)


### Intermediate Messages

If you are using WebSocket messaging for your QEWD Interactive application, you are not limited to a single response message being returned from your *message handler function*.  QEWD provides your *message handler functions* with a function - *send()* - that allows you to send additional messages, known as *intermediate messages* from your handler function before you signal its completion with the *finished()* method.

Unlike the *finished()* method that can only be invoked once from within your *message handler function*, you can invoke the *send()* method as many times as you like.

The *send()* method takes a single argument: *messageObject*.  The content and structure of this object is up to you, but it **MUST** contain one reserved property: *type*.  The value of *type* is up to you to define and is a string value.  It's a good idea to use a different *type* than the one for the message your handler function is dealing with - this ensures that your client/browser-side *ewd-client* response handler doesn't get confused.

Here's an example of a *message handler function*, let's say for a mesage of type *intermediateTest*, that generates an intermediate and final response message provided the user is logged in:


      module.exports = (messageObj, session, send, finished) => {
        if (!session.authenticated) {
          return finished({error: 'You have not logged in'});
        }

        send({
          type: 'info',
          foo: 'bar'
        });

        //... etc

        finished({ok: true});
      };


The associated browser/client-side logic might look like this:


      EWD.on('info', (responseObj) => {
        // this will handle the intermediate message
        // responseObj will contain {"type": "info", "foo": "bar"}
      });

      var msg = {
        type: 'intermediateTest'
      };
      EWD.send(msg, (responseObj) => {
        // this will handle the response (or error) from the message handler function's finished() function
      }


In theory, you could create a *message handler function* that sent a series of intermediate messages using a timed event (eg using *setInterval*), but it would be a bad idea to do so - it would mean that the QEWD Worker process handling the *message handler function* would be tied up and not released back to the QEWD Available Worker Pool for the entire duration of the timed events.  See later for alternative techniques for this kind of scenario that avoid tying up a Worker process for long periods of time.

