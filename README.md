IRMA Javascript client
======================

This is a Javascript client of the RESTful JSON API offered by the [IRMA API server](https://github.com/privacybydesign/irma_api_server). It offers a library that allows you to use the API server to:

 * Verify IRMA attributes. You specify which attributes, the library handles the user interaction and the communication with the API server and the IRMA token (such as the [IRMA Android app](https://github.com/privacybydesign/irma_android_cardemu)). Finally, the API server signes a JWT (JSON Web Token) containing the attributes (provided the attributes are valid), after which the library returns this signed JWT.
 * Issue IRMA attributes. You put the credentials to be issued in a signed JWT, the library handles the user interaction and the communication with the API server. (The API server does need to allow you to issue these credentials).
 * Create IMRA attribute-based signatures (experimental): signature on a string to which IRMA attributes are verifiably attached.

The flow of the various interactions in a typical IRMA session is shown [here](https://credentials.github.io/#irma-session-flow).

This javascript package contains two parts:

 * `client`: The IRMA Javascript library that you can use on your website if you want to integrate IRMA verification.
 * `examples`: Example pages that use various aspects of the IRMA Javascript library.

# Getting started

In case you quickly want to get started with setting up IRMA verification for your website please proceed as follows. First, you need an API server. For testing, feel free to user our API server. It is hosted at `https://demo.irmcard.org/tomcat/irma_api_server/`. (Please note that it only allows verifies and issues credentials in the demo domain.)

First, we load the IRMA client-side Javascript library and css:

```html
<link rel="stylesheet" href="https://path/to/webserver/client/irma.css">
<script src="https://path/to/webserver/client/irma.js" type="text/javascript"></script>
```

Next, we tell `irma.js` to initialize, and where to find the API server

```javascript
IRMA.init("https://demo.irmacard.org/tomcat/irma_api_server/api/v2/");
```

## A simple verification

We can now use `irma.js` for a simple verification. First, we specify the verification request:

```javascript
var sprequest = {
    "validity": 60,
    "request": {
        "content": [
            {
                "label": "over12",
                "attributes": ["irma-demo.MijnOverheid.ageLower.over12"]
            },
        ]
    }
};
```

in this case we would like to check that the user is over 12 years old, and we only accept the `irma-demo.MijnOverheid.ageLower.over12` attribute prove this. Next, you need to put the request in a JWT, that depending on the server's configuration, may need to be signed. If the server accepts unsigned JWT's, you can use `irma.js` to create one:


```javascript
var jwt = IRMA.createUnsignedVerificationJWT(sprequest);
```

After that you call `IRMA.verify` when you want to verify the attribute:

```javascript
IRMA.verify(jwt, success, warning, error);
```

Since this causes a popup please make sure to bind this to a user action, for example a button, and specify the callbacks if you want to see any result from the call

```javascript
var success = function(jwt) { console.log(jwt); }
var warning = function() { console.log("Warning:", arguments); }
var error = function() { console.log("Error:", arguments); }
var btn = document.getElementById("myButtonId");
btn.addEventListener("click", function() {
    console.log("Button clicked");
    var jwt = IRMA.createUnsignedVerificationJWT(sprequest);
    IRMA.verify(jwt, success, warning, error);
});
```

The `success`, `warning`, `error` callbacks are only examples, you may wish to redefine them based on your scenario. Most of this code was taken from the `verify` example in `examples`.

A full working minimal example is:

```HTML
<html>
    <head>
        <link rel="stylesheet" href="https://path/to/webserver/client/irma.css">
        <script src="https://path/to/webserver/client/irma.js" type="text/javascript"></script>
        <script type="text/javascript">
            IRMA.init("https://demo.irmacard.org/tomcat/irma_api_server/api/v2/");
            var sprequest = {
                "request": {
                    "content": [
                        {
                            "label": "18+",
                            "attributes": ["irma-demo.MijnOverheid.ageLower.over18"]
                        },
                    ]
                }
            };
            var success = function(jwt) { console.log("Success:", jwt); alert("Success"); }
            var warning = function() { console.log("Warning:", arguments); }
            var error = function() { console.log("Error:", arguments); }
            var clicked = function() {
                console.log("Clicked");
                var jwt = IRMA.createUnsignedVerificationJWT(sprequest);
                IRMA.verify(jwt, success, warning, error);
            };
        </script>
    </head>
    <body>
        <button onclick="clicked();">ClickMe</button>
    </body>
</html>
```

For a more complete scenario, see `examples/multiple-attributes` ([live demo](https://demo.irmacard.org/tomcat/irma_api_server/examples/multiple-attributes.html)).

### Using attributes in your backend

Behind the scenes, the IRMA API server produces a signed JWT containing the attributes you requested. `irma_js` returns this JWT to you to further handle these verified data in your backend. Naturally, you should always verify this JWT there too. If you want to rely on our server for now, you'll also need its [public key](https://demo.irmacard.org/v2/data/pk.pem). [This page](https://demo.irmacard.org/tomcat/irma_api_server/examples/verify.html) explicitly shows the JWT and its content.

## A simple issuance

Issuing a credential is equally easy. First we build a JSON object describing the credential(s) to issue.

```javascript
var iprequest = {
    timeout: 60,
    request: {
        "credentials": [
            {
                "credential": "irma-demo.MijnOverheid.address",
                "attributes": {
                    "country": "The Netherlands",
                    "city": "Nijmegen",
                    "street": "Toernooiveld 212",
                    "zipcode": "6525 EC"
                }
            }
        ],
    }
};
```

Then your backend application creates a signed JWT using this request so that the API server can check that you are indeed eligible to issue this credential. Our testing API server is very permissive (only for credentials in the demo domain), but it does require a JWT. As in the case of verification requests, `irma.js` offers a convenience function for creating an unsigned JWT. After you have created one, just make a call to `IRMA.issue` to issue the credentials:

```
var jwt = IRMA.createUnsignedIssuanceJWT(iprequest);
IRMA.issue(jwt, success, warning, error);
```

For a full example, see `examples\issue` ([live demo](https://demo.irmacard.org/tomcat/irma_api_server/examples/issue.html)).

## A simple IRMA signature

IRMA signatures are almost the same as verification. We first need to create a JSON object with a signature request:

```javascript
var sigrequest = {
    "validity": 60,
    "request": {
        "messageType" : "STRING",
        "message" : "a test message",
        "content": [
            {
                "label": "over12",
                "attributes": [ "irma-demo.MijnOverheid.ageLower.over12" ],
            },
        ]
    }
};
```

`messageType` must be "STRING", and `message` contains the message you want to sign and content contains the attributes that are used and included in the signature. Next, call `IRMA.sign` when you want to obtain the signature from the client:

```javascript
IRMA.sign(sprequest, success, warning, error);
```
This will start the protocol in the same way as in the verify case. Note that the web token that will be returned in the `success` case contains the IRMA signature.

## Multilanguage support

The IRMA javascript client supports localisation of all its interface elements to other languages. Translations are provided in the `client/languages` folder. During the call to `IRMA.init()`, a second argument can be provided to localize the interface to a different language. For example, initializing using

```javascript
IRMA.init("<URL>", "nl");
```

localizes the interface to the dutch language. When not provided, the IRMA javascript client defaults to english. The interface language can also be changed at runtime using `IRMA.setLang("<language>")`.

# Development

Some notes on development

## Quick start

First setup the development environment:

    npm install -g grunt-cli
    npm install compass     # unchecked

by installing grunt. Then run

    npm install

to install the node dependencies and JavaScript libraries. Finally run

    grunt build

to build the libraries and examples. See below for how to setup server URLs for a remote verification server or a local verification server. Alternatively, you can just run

    grunt build watch

to keep rebuilding the files as they change. If you specify neither `build` nor `watch`, `build watch` is implied.

Using the flags `--client` and `--examples` you can specify which of the subfolders should be built. If you specify none of these, all will be built.

## URL for the api

This project relies on one URL for verification: The client-side library needs to know the location of the API backend to perform the actual verifications and issuances. This can be specified in your webpage through the call to `IRMA.init`.

For the examples in this package, an url is required during the build proces. The default is to use the demo server at `https://demo.irmacard.org/tomcat/irma_api_server/`. To use a local verification server, one can specify the server address using the `--api_server_url=<URL>` option. For this, you might also be interested in the shortcut explained below.

## Running a local verification server

If you are running a local api server using the `irma_api_server` project you might as well use it to host the web pages as well (and thus avoid CORS problems). First, make sure that the assembled output is written to the `webapp` directory. If `irma_api_server` is in the same directory is `irma_js` run:

    ln -s ../irma_api_server/src/main/webapp/ build

(Note: if you have already run `grunt` before creating this symlink, then a directory named `build` already exists. Be sure to remove it first!)
Then simply specify the root of the servlet when running grunt:

     grunt --api-server_url="http://<HOST>:8088/irma_api_server/"

If you want to test your application using an external token, make sure that `<HOST>` is either is an ip address that the token can reach, or is resolvable to one by the token. You can then run the example by visiting

    http://<HOST>:8088/irma_api_server/examples/custom-button.html

or

    http://<HOST>:8088/irma_api_server/examples/issue.html

or

    http://<HOST>:8088/irma_api_server/examples/multiple-attributes.html
