[![build status](https://secure.travis-ci.org/bloomberg/blpapi-node.svg)](http://travis-ci.org/bloomberg/blpapi-node)
# blpapi-node


[Bloomberg Open API] binding for [Node.js].

Find source code in the [Github repository].

[Bloomberg Open API]: http://www.bloomberglabs.com/api/about/
[Node.js]: http://nodejs.org
[Github repository]: https://github.com/bloomberg/blpapi-node

**Note:** This repository was renamed from `node-blpapi` to `blpapi-node`.

# Dependencies

This module requires:

+ Node.js version >= **0.12.x** (io.js >= **1.0.x** is supported)
+ Linux, Windows, or Mac OS X (32 or 64-bit)
+ GCC (Linux), MSVC++ (Windows), or Xcode (Mac OS X)
+ Bloomberg Desktop API (DAPI), Server API (SAPI), or [B-PIPE] subscription

This module includes:

+ [Bloomberg BLPAPI C++ SDK] v3.7.9.1 (Linux/Windows)
+ [Bloomberg BLPAPI C++ SDK] v3.8.1.1 (Mac OS X)

**Note:** Mac OS X users can only connect to SAPI or [B-PIPE] products.

[Bloomberg BLPAPI C++ SDK]: http://www.bloomberglabs.com/api/libraries/
[B-PIPE]: http://www.bloomberg.com/enterprise/enterprise_products/data_optimization/data_feeds

# Installation

Clone the repo (in the Git Bash shell):

```
$ git clone https://github.com/tracksaw/blpapi-node.git
$ cd blpapi_node
```

## On Windows

The ./examples/node_modules directory contains an entry: **blpapi**.

On Unix this will be a symbolic link that refers to the path ../... One windows it will be a file containing the text '../..', which will not work. So run a windows CMD shell and do the following:

```
> cd ....\blpapi-node\examples\node_modules
> rm blpapi
> mklink /d blpapi ..\..
```

Now you should see the link points to the ...\blpapi-node directory

```
> cd ....\blpapi-node
> dir examples\node_modules\blpapi\package.json
   Volume in drive E is New Volume
   Volume Serial Number is DE1E-FE0D

   Directory of E:\GitHub\tracksaw\blpapi-node\examples\node_modules\blpapi

  10/05/2017  08:38 AM               960 package.json
                 1 File(s)            960 bytes
```

# Installation Continued

From your project directory, run:

```
$ npm install blpapi
```

To install directly from github source, run:

```
$ npm install git://github.com/bloomberg/blpapi-node.git
```

This will download and build `blpapi` in `node_modules/`.

**Note:** Windows users using the Express version of Visual Studio may not
have the 64-bit compiler platform installed. If errors are seen related
to the `x64` platform not being found, please force a 32-bit arch before
invoking `npm` by running from the command shell:

```
> set npm_config_arch="ia32"
```

# Usage

The module design closely follows the BLPAPI SDK design, with slight
modifications in syntax for easier consumption in Javascript.  The SDK
developer's guide should serve as the main guide for the module's
functionality.

Full examples contained in the `examples` directory demonstrate how to
use most SDK functionality.  Full descriptions of all available requests,
responses, and options are contained within the BLPAPI API
[Developer Guide](http://www.bloomberglabs.com/api/documentation/).


### Opening A Session ###

    var blpapi = require('blpapi');
    var session = new blpapi.Session({ host: '127.0.0.1', port: 8194 });

    session.on('SessionStarted', function(m) {
        // ready for work
    });

    ...

    session.start();

### Opening A Subscription Service ###

    var service_id = 1;

    session.on('SessionStarted', function(m) {
        session.openService('//blp/mktdata', service_id);
    });

    session.on('ServiceOpened', function(m) {
        // m.correlations[0].value == service_id
        // ready for subscriptions
    });

### Subscribing To Market Data ###

    var securities = [
        { security: 'AAPL US Equity', correlation: 0, fields: ['LAST_TRADE'] },
        { security: 'GOOG US Equity', correlation: 1, fields: ['LAST_TRADE'] }
    ];

    session.on('ServiceOpened', function(m) {
        if (m.correlations[0].value == service_id) {
            session.subscribe(securities);
        }
    });

    session.on('MarketDataEvents', function(m) {
        if (m.data.hasOwnProperty('LAST_TRADE')) {
            console.log(securities[m.correlations[0].value].security,
                        'LAST_TRADE', m.data.LAST_TRADE);
            // outputs:
            // AAPL US Equity LAST_TRADE 600.00
            // AAPL US Equity LAST_TRADE 601.00
            // GOOG US Equity LAST_TRADE 650.00
            // ...
        }
    });

### Creating An Authorized Identity ###

Some session configurations, for example when connecting to a B-PIPE, may
require calls to `request` and `subscribe` to specify an authorized Identity.
The `authorizeUser` function performs an `AuthorizationRequest` on the
`//blp/apiauth`. This function differs slightly from the BLPAPI SDK design in
two ways. First, rather than having separate response events for success and
failure, it emits the `AuthorizationResponse` event for both. Second, the
`Identity` object is returned via the response as `data.identity`. This is only
set for successful authorization, so its presence or absence can be used to
determine whether the `AuthorizationResponse` indicates success or failure.
`data.identity` is an opaque object representing the authorized user. Its only
use is to be passed to `request` and `subscribe`.

    var auth_service_id = 2;
    var token_correlation_id = 3;
    var identity_correlation_id = 4;

    session.on('SessionStarted', function(m) {
        session.openService('//blp/apiauth', auth_service_id);
    });

    session.on('ServiceOpened', function(m) {
        if (m.correlations[0].value == auth_service_id) {
            // Request a token to be sent to you via MSG.
            session.request('//blp/apiauth', 'AuthorizationTokenRequest',
                { uuid: 12345678, label: 'testApp' }, token_correlation_id);
        }
    });

    session.on('AuthorizationTokenResponse', function(m) {
        if (m.correlations[0].value == token_correlation_id) {
            // Request the identity
            session.authorizeUser({ uuid: 12345678, token: 'token from MSG' },
                identity_correlation_id);
        }
    });

    session.on('AuthorizationResponse', function(m) {
        if (m.correlations[0].value == identity_correlation_id) {
            if (m.data.hasOwnProperty('identity') {
                // Authorization successful;
                // Save m.data.identity for use with later requests.
            }
        }
    });

### Using An Authorized Identity To Make A Request ###

    var identity;  // Assumed to be set by a previous AuthorizationResponse

    var refdata_service_id = 5;
    var refdata_correlation_id = 6;

    session.on('SessionStarted', function(m) {
        session.openService('//blp/refdata', refdata_service_id);
    });

    session.on('ServiceOpened', function(m) {
        if (m.correlations[0].value == refdata_service_id) {
            session.request('//blp/refdata', 'ReferenceDataRequest',
                { securities: ['IBM US Equity'], fields: [PX_LAST'] },
                refdata_correlation_id, identity);
        }
    });

    session.on('ReferenceDataResponse', function(m) {
        if (m.correlations[0].value == refdata_correlation_id) {
             console.log(m.data);
        }
    });

# Error Handling

Exceptions thrown from the C++ SDK layer are translated into
JavaScript exceptions with the same type name. The JavaScript
exception types inherit from `Error` with the `message` property set
to the description obtained from the original C++ exception.
Refer to the C++ SDK documentation for additional information on
exceptions.

This is a list of the exception types:

+ DuplicateCorrelationIdException
+ InvalidStateException
+ InvalidArgumentException
+ InvalidConversionException
+ IndexOutOfRangeException
+ FieldNotFoundException
+ NotFoundException
+ UnknownErrorException
+ UnsupportedOperationException

# Running the FieldSearchRequest example

In Git bash shell:

```
$ cd .../blpapi-node
$  node examples/FieldSearchRequest.js 127.0.0.1:8194 PX
```

The output should look something like:

```
PX_LAST => Last Price
PX_BID => Bid Price
PX_ASK => Ask Price
PX_YEST_CLOSE => Yesterday Close Price
PX_MID => Mid Price
PX_CLOSE_1D => Closing Price 1 Day Ago
PX_VOLUME => Volume
YAS_BOND_PX => YAS Bond Price
PX_SETTLE => Settlement Price
PX_LOW => Low Price
PX_HIGH => High Price
PX_OPEN => Open Price
PX_TO_BOOK_RATIO => Price to Book Ratio
EQY_WEIGHTED_AVG_PX => VWAP (Vol Weighted Average Price)
OPT_STRIKE_PX => Strike Price
PX_ROUND_LOT_SIZE => Round Lot Size
PX_CLOSE_DT => Date Of Last Close
CRNCY_ADJ_PX_LAST => Currency Adjusted Last Price
PX_CLOSE_5D => Closing Price 5 Days Ago
PX_SETTLE_LAST_DT => Date of Last Settlement Price
```

# License

MIT license. See license text in [LICENSE](https://github.com/bloomberg/blpapi-node/blob/master/LICENSE).
