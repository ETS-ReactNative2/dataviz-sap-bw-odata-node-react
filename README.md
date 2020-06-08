# dataviz-sap-bw-odata-node-react
A simple data visualization application on Node.js and React.js requesting data from backend SAP BW system.

# Intro
This document describes three steps required to build a web application on Node.js and React.js with the purpose of analysis and visualization of data sourced from a backend SAP Business Warehouse (BW) system.

# Table of contents:
1. [Create an oData service to expose data using a BW query from backend SAP BW system](#create-an-odata-service-to-expose-data-from-backend-sap-bw-system)
    1. [Resources](#resources)
    1. [SAP Gateway configuration](#sap-gateway-configuration)
    1. [Check & setup settings for EQ and oData](#check-and-setup-settings-for-eq-and-odata)
    1. [Create BW query for oData service](#create-bw-query-for-odata-service)
    1. [Generate native Gateway service](#generate-native-gateway-service)
    1. [Create and publish oData service for a query](#create-and-publish-odata-service-for-a-query)
    1. [Get oData URL](#get-odata-url)
    1. [Metadata](#metadata)
    1. [Results](#results)
    1. [Authorization](#authorization)
    1. [Transport collection](#transport-collection)
1. [Set up server environment on Node.js](#set-up-server-environment-on-nodejs)
    1. [Resources](#server-resources)
    1. [Purpose](#purpose)
    1. [Server folder structure](#server-folder-structure)
    1. [Set up index.js and app.js](#set-up-index-and-app)
    1. [Create object mapping file](#create-object-mapping-file)
    1. [Credentials](#credentials)
    1. [oData URL manipulation class](#odata-url-manipulation-class)
    1. [Routes](#routes)
1. [Set up client React app](#set-up-react-client-app)
    1. [D3 data viz components](#d3-data-viz-components)

Step number three which describes setting up the client and building react data visualization components is applicable to any source system.

# Project

## Create an oData service to expose data from backend SAP BW system

### Resources
This part of this documentation was written based on below resources as well as own experiences:
* Resource 1: https://blogs.sap.com/2019/02/19/how-to-do-odata-services-from-bex-query/ 
* Resource 2: https://wiki.scn.sap.com/wiki/display/BI/Steps+to+Create+an+ODATA+service+for+a+BW+Query
* Resource 3: https://wiki.scn.sap.com/wiki/display/BI/BW+OData+Queries
* Resource 4: https://help.sap.com/viewer/64e2cdef95134a2b8870ccfa29cbedc3/7.4.19/en-US/c9384c774bcc4837b84bee3679520fb4.html
* Resource 5: https://launchpad.support.sap.com/#/notes/2424613 
* Resource 6: https://blogs.sap.com/2016/03/21/how-to-change-dev-class-tmp-for-the-repository-objects-of-an-odata-service/

### SAP Gateway configuration
This step is only required the first time an oData service is created in the system. 

Go to transaction `SPRO` and select: 
*SAP NetWeaver – SAP Gateway - OData channel – Configuration – Connection Settings – SAP Gateway to SAP System – Manage SAP System Aliases*

<img src="/public/images/spro-system-alias.png" width="500"/>

Click on *Create System Alias* icon and enter below details:
* System Alias: *LOCAL*
* RFC Destination: *NONE*
* Local Gateway: *Enabled*
* For Local App: *Disabled*
* OData on Backend: *Disabled*
* Target System ID: *Leave blank*
* Target Client: *Leave blank*
* Software Version: *DEFAULT*
* Web Service Provider *System: Leave blank*
* Description: *System Alias Local*

<img src="/public/images/system-alias.png" width="500"/>

### Check and setup settings for EQ and oData
This step is also only required the first time an oData service is created in the system. 
Go to transaction `SE38` and run program *EQ_RS_AUTOSETUP*

### Create BW query for oData service
Create a BW query and tick the option *Remote Access By oData*.

<img src="/public/images/query.png" width="500"/>

**Limitations and query functionalities**
* There are some limitations when building a query for oData service, for example: no hierarchies and no structures allowed
* Majority of functionalities, such as: currency conversion, display settings and conditions, are processed correctly (see SAP note [2367553](https://launchpad.support.sap.com/#/notes/2367553) ODataQuery features and limitation).

**Query variables**
* oData service allows to set values in query variables but not all types can be made use of. 
* Only mandatory single values and intervals can be passed to the oData URL. 
* There is no way to pass multiple selections or multiple single values in the URL. It is possible to do so using filter URL parameters.
* Using optional variables in the URL string will generate an error. To optionally filter the data use the `$filter` URL parameter.

**Query (fixed/default) filters**
* Both filters and default values are applied

**Dimensions**
* All dimensions from rows and free characteristics will be available in the output. It does not really make a difference in which place they are added. 
* Dimension display settings are important, especially:
    * *Display* setting *Key / Text / Key and Text* has impact on the number of generated key-value pairs. Selecting *Key and Text* will generate two separate object properties while only *Key* or only *Text*, one.
    * *Text Output Format* setting *Short/Medium/Long Text* is reflected in the JSON output
    * Setting *Show Result Rows* to *Always* will generate objects with result rows which could be desired, but could also lead to duplication of numbers in the client app, depending on how the data is used there. Think ahead and make sure you use the right setting here.

**Key Figure settings**
*	Measures generate two JSON properties for the value (one raw and one formatted) and optionally one for the measure unit. You can make use of measure formatting from the query such as decimals or scaling factor.  

### Generate native Gateway service
It might be necessary to run this standard SAP function module which performs *generate native Gateway service* functionality, if the service is not available automatically after creating a query.
Go to transaction `SE37` and execute function module *RSEQ_NAT_GENERATION*. Enter query technical name in `I_S_QUERY` field and click on the *execute* icon.

<img src="/public/images/generate-service.png" width="500"/>

This should generate three messages:
* SAP Gateway Model '<model_name>' Version '0001' created
* SAP Gateway Service '<service_name>' Version '0001' created
* SAP GW Model '<service_name> 'Version '0001' assigned to SAP GW Service '' Version ''

<img src="/public/images/generate-service-messages.png" width="500"/>
<img src="/public/images/generate-service-messages-2.png" width="500"/>

### Create and publish oData service for a query
Go to transaction `/n/IWFND/MAINT_SERVICE` and click on *Add Service*.

<img src="/public/images/add-service.png" width="500"/>

Enter System Alias *LOCAL* and click on *Get Services*.

<img src="/public/images/get-services.png" width="500"/>

A list of available services should appear. Search for the newly created service and click on it. Select the right package and press OK. The new service is created.

### Get oData URL
Find the added service and click on it. You should see an active ICF node in the bottom left corner. If for some reason the service status is not green, you can activate it there (click on ICF Node – Activate). To get the URL click on *Call Browser* button.

<img src="/public/images/call-browser.png" width="500"/>

The default URL will have format:

`http://<server>:<port>/sap/opu/odata/sap/service_name/?$format=xml`

In this example it would be:

`http://<server>:<port>/sap/opu/odata/sap/ T_ODATA_SRV /?$format=xml`

The first time you run the URL, you will be asked to select a certificate to authenticate yourself to the server and to enter your BW credentials. See the next point to see how to set up authorization for users to access the data.
Read [here](https://wiki.scn.sap.com/wiki/display/BI/BW+OData+Queries) and [here](https://help.sap.com/viewer/64e2cdef95134a2b8870ccfa29cbedc3/7.4.19/en-US/c9384c774bcc4837b84bee3679520fb4.html) how you can make use of all oData URL parameters to select desired dimensions and measures, filter data, order by dimension etc.

### Metadata
To see metadata add `/$metadata` parameter to the URL. This will generate query information such as the list of variables and columns available in the output (both technical names and descriptions). Interval and selection variables will generate two parameters in the URL, which you can check in the metadata output. Unfortunately, only available in `.xml` format.

`http://<server>:<port>/sap/opu/odata/sap/T_ODATA_SRV/$metadata/`

<img src="/public/images/results-metadata.png" width="500"/>

### Results
To see results in JSON format, add `/QuerynameResults?$format=json` to the URL. Be careful with running the URL’s without specifying which dimensions to select. If a query includes many dimensions on very granular level (like *Article* or *Order ID*), you might end up pulling too many records from the source system causing performance issues. Think of it as running a query in *Analysis for Excel*, selecting a full fiscal year and adding all available dimensions, incl. *Article* and *Order ID*, to rows or columns.

It is therefore advised to consciously request the dimensions using the *$select* parameter. Let’s say that we need to request *Amount* [0AMOUNT] aggregated by *Country* [0COUNTRY]. Our URL will look like this:

`http://<server>:<port>/sap/opu/odata/sap/T_ODATA_SRV/T_ODATAResults?$format=json&$select=0COUNTRY,A0AMOUNT,A0AMOUNT_E`

Results should look like below:

```
// yyyymmddhhmmss
// <url>

{
  "d": {
    "results": [
      {
        "__metadata": {
          "id": "<url>('V2.81.14.0_1.2.2.0%3A0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.9.0.0.0.0.0.0.X.0.0.0.0.0.0.1.0_IEQXXXXXXXX')",
          "uri": "<url>('V2.81.14.0_1.2.2.0%3A0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.9.0.0.0.0.0.0.X.0.0.0.0.0.0.1.0_IEQXXXXXXXX')",
          "type": "<service_name>.<query_name>Result"
        },
        "A0COUNTRY": "DE",
        "A0AMOUNT": "1234",
        "A0AMOUNT _E": "EUR"
      }
    ]
  }
}
```

It is best to assure proper error handling in your node application in order to prevent from accidental requests of too much data.

### Authorization
**What to consider**

To be able to request data from backend BW system, you need to assure that users have the right authorizations. 

Depending on the application that will be developed on top of the oData service, you can either decide to create one service user to communicate between BW and the client app or assign authorization to existing standard users. That decision purely depends on business requirements and agreements regarding data confidentiality within an organization.

The first option is preferable if the data, which will be shown in the web application, is not confidential and can be shown to users without applying additional filtering depending on users’ access rights. Here user credentials can be hardcoded in the client application. For example, a dashboard with produced quantites which is intended to be displayed on a wall in a warehouse.

The latter should be used if there is a need to make use of user authorization to InfoObjects, such as Reporting Units or Divisions, by applying authorization exit variables in BW query filters. Here we would need to implement a login form in our application and pass credentials in the server route. An example would be a dynamic web application with sales analysis by customers and products intended to be used globally by managers or analysts, who should be able to see only sales within their region or division.

Later in this document, you can find information how to handle both scenarios in the client application.

**Add authorization object S_SERVICE to BW role**

The authorization object required to access data via the oData service is `S_SERVICE` *Check at Start of External Service*. 
There are two authorizations required to access the data:
* SAP Gateway: Service Groups Metadata
* SAP Gateway Business Suite Enablement – Service

To investigate what exactly needs to be added to a BW role, first create a test user with standard user authorizations. Use its credentials to log in when running the oData URL. You should see an error which looks like this:

```
{
    "error": {
        "code": "/IWFND/CM_CONSUMER/101",
        "message": {
            "lang": "en",
            "value": "No authorization to access Service 'ZT_ODATA_SRV_0001'"
        },
        "innererror": {
            "application": {
                "component_id": "",
                "service_namespace": "/SAP/",
                "service_id": "T_ODATA_SRV",
                "service_version": "0001"
            },
            "transactionId": "XXX",
            "timestamp": "YYYMMDD",
            "Error_Resolution": {
                "SAP_Transaction": "For backend administrators: run transaction /IWFND/ERROR_LOG on SAP Gateway hub system and search for entries with the timestamp above for more details",
                "SAP_Note": "See SAP Note 1797736 for error analysis (https://service.sap.com/sap/support/notes/1797736)"
            },
            "errordetails": [

            ]
        }
    }
}
```

Go to transaction `SU53` and display error messages for the test user you used to access oData URL. You should see the details of missing authorizations. See the authorization object details and service ID.

<img src="/public/images/su53-1.png" width="500"/>

Go to transaction `PFCG` and edit the role to which you want to add the authorization. Go to Authorizations tab and click on the pencil icon by *Edit Authorization Data and Generate Profiles*.

<img src="/public/images/pfcg-1.png" width="500"/>

Click on *Add Manually*, add `S_SERVICE` and click *OK*.

<img src="/public/images/s-service.png" width="500"/>

Here you need to make a decision whether you want to control precisely which services are added to the role or if you want to add authorizations to all created services at one go. That decision again depends on how strictly the organization’s authorization concept is implemented in the system.

To add all services at one go just add * *All Values* in *Program, transaction or function*. 

<img src="/public/images/s-service-2.png" width="500"/>

If you want to control access to each created oData query (service), select *TADIR Service* and add the ID of the service which you can copy from the error log in `SU53` (*SAP Gateway: Service Groups Metadata*). 
To paste the ID switch from *Object Name* to *Technical Name* by clicking on the button pointed by the arrow on the screenshot below. Paste the id in the Name column and press enter. 

<img src="/public/images/s-service-3.png" width="500"/>

Click *Save* and then *Generate* icons.

<img src="/public/images/s-service-4.png" width="500"/>

Then repeat the process again to find the ID for the second authorization *SAP Gateway Business Suite Enablement – Service*.
Run the URL using the same test user credentials. This time the error will look like this:

```
{
    "error": {
        "code": "/IWBEP/CM_MGW_RT/000",
        "message": {
            "lang": "en",
            "value": "User does not have the sufficient authorizations"
        },
        "innererror": {
            "application": {
                "component_id": "",
                "service_namespace": "/SAP/",
                "service_id": "T_ODATA_SRV",
                "service_version": "0001"
            },
            "transactionId": "XXX",
            "timestamp": "YYYMMDD",
            "Error_Resolution": {
                "SAP_Transaction": "For backend administrators: run transaction /IWFND/ERROR_LOG on SAP Gateway hub system and search for entries with the timestamp above for more details",
                "SAP_Note": "See SAP Note 1797736 for error analysis (https://service.sap.com/sap/support/notes/1797736)"
            },
            "errordetails": [

            ]
        }
    }
}
```

Refresh the error log in `SU53`. Copy the new ID and add it to the role in the same place as before.

<img src="/public/images/su53-2.png" width="500"/>

Role authorization settings should look like below.

<img src="/public/images/pfcg-2.png" width="500"/>

Now user has sufficient authorizations to see the data. Run the URL and you should see the results.

### Transport collection

See SAP note [2424613](https://launchpad.support.sap.com/#/notes/2424613) for information.

Three things should be transported in below sequence:
* System Alias LOCAL (Customizing)
* Query with oData enabled (Workbench)
* Service for above query together with its model and active ICF node (Customizing)
Make sure that the system alias added to the service is marked as *Default System*.

<img src="/public/images/default-system.png" width="500"/>

It might happen that after transporting from development environment to QA or Production, you will see some errors because usually not all objects are added to the transport correctly.
See this [blog post](https://blogs.sap.com/2016/03/21/how-to-change-dev-class-tmp-for-the-repository-objects-of-an-odata-service/) which explains how to add required objects to the transport. 

Some errors you might see:

**Error:** Not Found
```
<error xmlns:xsi="http://www.w3.org/2001/XMLSchema-Instance">
<code>HTTP/404/E/Not Found</code>
<message> Service /sap/opu/odata/sap/FCP_U034_OQ001_SRV/ call was terminated because the corresponding service is not available.The termination occurred in system BWD with error code 404 and for the reason Not found. Please select a valid URL. If it is a valid URL, check whether service /sap/opu/odata/sap/FCP_U034_OQ001_SRV/ is active in transaction SICF. If you do not yet have a user ID, contact your system administrator. </message>
</error>
```

**Solution:** Check in `/n/IWFND/MAINT_SERVICE` if your ICF Node is active. If not, activate it.

<img src="/public/images/activate-icf-node.png" width="250"/>

<img src="/public/images/activate-icf-node-2.png" width="250"/>


**Error:** No System Alias found for Service 'XXX' and user 'X'

**Solution:** Check in `/n/IWFND/MAINT_SERVICE` if your service was transported with a system alias added. If not, go to the service in Development environment, remove the system alias and add it again. Make sure you mark it as Default System. Write the changes in transport and transport again.


**Error:** No service found for namespace …
```
{
  "error": {
    "code": "/IWBEP/CM_MGW_RT/026",
    "message": {
      "lang": "en",
      "value": "No service found for namespace /SAP/, name FCP_U034_OQ001_SRV, version 0001"
    },
    "innererror": {
      "application": {
        "component_id": "",
        "service_namespace": "/SAP/",
        "service_id": "FCP_U034_OQ001_SRV",
        "service_version": "0001"
      },
      "transactionid": "16C1FD27D54D00C0E005ED8CAAAB83A0",
      "timestamp": "20200604105347.5709890",
      "Error_Resolution": {
        "SAP_Transaction": "For backend administrators: run transaction /IWFND/ERROR_LOG on SAP Gateway hub system and search for entries with the timestamp above for more details",
        "SAP_Note": "See SAP Note 1797736 for error analysis (https://service.sap.com/sap/support/notes/1797736)"
      },
      "errordetails": [
        
      ]
    }
  }
}

```

**Solution:** Check if the system alias is marked as *Default System*. If not then go to `/n/IWFND/MAINT_SERVICE` in development environment, remove the alias and add again. This time make sure you mark the alias as Default System. Transport to QA/Prod.
If that doesn’t help then check if the service and model were transported correctly.
-	Go to transaction `/n/IWBEP/REG_SERVICE` and search for your service name. If it is not found, it was not transported from Development
-	Go to transaction `/n/IWBEP/REG_MODEL` and search for your service model. 

To collect the service and model follow the steps described in this [blog post](https://blogs.sap.com/2016/03/21/how-to-change-dev-class-tmp-for-the-repository-objects-of-an-odata-service/)


## Set up server environment on Nodejs

### Server Resources
* Resource 1: https://www.acorel.nl/2016/12/consuming-sap-odata-services-from-angularjs-and-or-node-js/

### Purpose
Once we have completed the first part, which is setting up the way data is exposed from a backend SAP BW system, we need to set up our application’s server. The server is built on Node.js using the express framework. In the future the application will be extended with a MongoDB for writing additional data, such as comments and users.

There are a few reasons why we need to set up a server.

**CORS policy**

Pulling data directly to a client application will be blocked by cross-origin errors. To avoid this issue we will set up the server and add a proxy between the server and the client to enable data fetching.

**Re-structuring of the data**

A data warehouse (structured database), specifically a SAP BW, has its own specific way of exposing data, which is not entirely convenient to use in a JavaScript application. The JSON API accessed via the oData URL contains properties with very long and not intuitive names, especially for a developer with a limited knowledge of SAP systems. 

**Building an oData URL manipulation class/library**

The use of URL parameters in oData from SAP BW can be very powerful but also a bit confusing for a developer who is not familiar with SAP systems. We will build our own JavaScript class or mini library on the server to easily pass correct parameters and values to the oData URL.


### Server folder structure
* -**server** – app, index, config, package.json, seeds data
* --**common** – helper methods, constants, etc.
* ---**sap_constants** – excluded from git repository as it might include confidential data. Stores sap credentials and query information file
* --**routes**
* --**testing**

### Set up index and app
**index.js**

The `index.js` script should start the server and should be kept as simple as possible.

```
const app = require('./app');
const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
    console.log(`Server has started on port ${PORT}`);
});
```

**app.js**

The `app.js` file exposes the app as a library. Here we use the express framework.

In the server folder, install express, body-parser and request. Also add nodemon globally and as a devdependency in order to automatically refresh the application after saving changes.

With npm:

`npm i express body-parser request –save`

`npm i –g nodemon –save-dev`

Or with yarn:

`yarn add express body-parser request`

`yarn add nodemon –dev`


See `package.json` to view dependencies and add scripts.

```
{
  "name": "dataviz-sap-bw-odata-node-express",
    "version": "1.0.0",
    "description": "A server api fetching and exposing data from a SAP BW system",
    "main": "index.js",
    "scripts": {
        "start": "nodemon index.js"
    },
    "keywords": [],
    "author": "",
    "license": "ISC",
  "dependencies": {
    "body-parser": "^1.19.0",
    "express": "^4.17.1"
  },
  "devDependencies": {
    "nodemon": "^2.0.4"
  }
}

```

The application can be run by executing the `nodemon index.js` script. Before it is possible to do so, it is required to set up the app.js and routes.

The `app.js` initializes the express application and handles routes. Route files are stored in a separate folder and imported in the `app.js`.

```
var express = require('express'),
    app = express(),
    bodyParser = require('body-parser');

// Middleware for parsing json objects
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());

// Production setup
if (process.env.NODE_ENV === 'production') {
    app.use(express.static('reactApp/build'));

    const path = require('path');
    app.get('*', (req, res) => {
        res.sendFile(path.resolve(__dirname, 'reactApp', 'build', 'index.html'));
    });
}

// Handle BW API routes
require('./routes/sales')(app);

module.exports = app;
```

### Create object mapping file
In BW, each object has usually a key and a text property, and each measure has a value and (optionally) a unit (currency, pieces, m2 etc.). Therefore, it is useful to group all properties of one object together. This way it is later easier to use the whole object with its properties in the client app.

In order to restructure data, it is necessary to build a mapping file with query details and a list of expected dimensions and measures. Such file will also allow to keep all metadata in one place, which can be useful especially when the data source structure changes. There are often multiple objects for the same thing in BW and depending on the business use, it might be necessary at some point to use another technical name for the same object. Creation of a mapping file will allow us to do such change only in one place, instead of everywhere in our code. Also, thanks to the restructuring we will be able to access the data using standard JavaScript object names, instead of long technical names from BW. For example to get a *Country* name from the data we can simply refer to `results.country.text`, instead of `results[“A0COUNTRY_T”]`. Using strings everywhere in the code is generally evil.

The mapping file is located in `server/common/sap_constants/queryInfo.js` and contains details as seen below:

```
const queryInfo = {
    sales: {
        server: 'abc.def.com',
        port:  '1000',
        service: 'T_ODATA_SRV',
        query: 'T_ODATA',
        variables: {
            country: 'VAR_COUNTRY',
        },
        dimensions: {
            country: {
                key: "0COUNTRY ",
                text: "0COUNTRY_T",
            },
		division: {
                key: "0MATERIAL__0DIVISION",
                text: "0MATERIAL__0DIVISION_T",
            },

            period: {
                key: '0FISCPER',
                text: '0FISCPER_T',
            },
        },
        measures: {
            qty: {
                value: "AAAABBBBCCCCDDDDEEEEFFFF1",
            },
            sales: {
                value: "A0AMOUNT",
                unit: "A0AMOUNT_E",
            },
        },
    },
}

module.exports = queryInfo;
```

The aim here is to re-structure the data from BW to more JavaScript-like API. So to visualize, to change this BW output:

```
// yyyymmddhhmmss
// <url>

{
  "d": {
    "results": [
      {
        "__metadata": {
          "id": "<url>('V2.81.14.0_1.2.2.0%3A0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.9.0.0.0.0.0.0.X.0.0.0.0.0.0.1.0_IEQXXXXXXXX')",
          "uri": "<url>('V2.81.14.0_1.2.2.0%3A0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.9.0.0.0.0.0.0.X.0.0.0.0.0.0.1.0_ IEQXXXXXXXX')",
          "type": "<service_name>.<query_name>Result"
        },
        "A0COUNTRY": "DE",
        "A0AMOUNT": "1234",
        " A0AMOUNT _E": "EUR"
      }
    ]
  }
}
```

To a clean JSON which looks like this:

```
[
    {
        country: {
            key: "DE",
            text: “Germany”,
        }
        amount: {
            value: "1234",
            unit: “EUR”,
        }
    }
]
```
See [Routes](#routes) to see which methods are responsible for this change of JSON structure.

### Credentials
To access the data the application needs to be connected to the internal network and use credentials of a BW user with sufficient authorizations. Credentials are passed in `Base64` encoded format `username:password`. Use `process.env` to securely store credentials in the deployment environment. 
In a working version of the app you can use a file stored in the `server/common/sap_constants/credentials.js`, BUT MAKE SURE YOU ADD IT TO THE GITIGNORE FILE. 

### oData URL manipulation class
Select data using chainable methods of a class [`oDataURL`](https://github.com/kxkaro/dataviz-sap-bw-odata-node-react/blob/master/server/common/oDataURL.js) which is defined in `server/common/`

### Routes
See method [`getBWData.js`](https://github.com/kxkaro/dataviz-sap-bw-odata-node-react/blob/master/server/common/getBWData.js) from `server/common/` which is a general method to pull data from a BW query using oData. See [this blog post](https://www.acorel.nl/2016/12/consuming-sap-odata-services-from-angularjs-and-or-node-js/) for a demo presenting how to pull data to a node application using request. 

This method reuses the structure from the `queryInfo.dimensions` and `queryInfo.measures` and replaces the technical names from BW, which were placed as values, with the actual values from the oData results. It should be used for all routes leading to SAP BW data.

In the route file, which is directly used in the `app.js`, we can define the routes, and therefore selections, which we need for the client app. For example, we can create a get route `/api/sales` which selects *Division* and *Quantity* and *Sales* amounts.

```
const routes = app => {
    getBWData(
        app,
        '/api/sales',
        URL,
        credentials,
        { division },
        { qty, sales }
    );
}
```

See the definition of [`getBWData`](https://github.com/kxkaro/dataviz-sap-bw-odata-node-react/blob/master/server/common/getBWData.js) method in `server/common`. I will not go into details here but as a result this structure from the `queryInfo.js`:
```
dimensions: {
    country: {
        key: "0COUNTRY ",
        text: "0COUNTRY_T",
    },
    division: {
         key: "0MATERIAL__0DIVISION",
        text: "0MATERIAL__0DIVISION_T",
    },git 
},
measures: {
    qty: {
        value: "AAAABBBBCCCCDDDDEEEEFFFF1",
     },
     sales: {
         value: "A0AMOUNT",
         unit: "A0AMOUNT_E",
    },
},
```

...will be translated to the final API to be fetched by client:
```
// 20200608194735
// http://localhost:5000/api/sales/

{
  "info": "This is dummy data for demo purposes",
  "results": [
    {
      "country": {
        "key": "DE",
        "text": "Germany"
      },
      "division": {
        "key": "A",
        "text": "A Products"
      },
      "qty": {
        "value": 123
      },
      "sales": {
        "value": 2345.25,
        "unit": "EUR"
      }
    },
    {
      "country": {
        "key": "DE",
        "text": "Germany"
      },
      "division": {
        "key": "B",
        "text": "B Products"
      },
      "qty": {
        "value": 346
      },
      "sales": {
        "value": 6378.5,
        "unit": "EUR"
      }
    }
  ]
}
```

## Set up React client app

### D3 Data viz components
