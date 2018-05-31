![FIWARE Banner](https://fiware.github.io/tutorials.Context-Providers/img/fiware.png)

[![NGSI v1](https://img.shields.io/badge/NGSI-v1-ff69b4.svg)](https://forge.fi-ware.org/docman/view.php/7/3213/FI-WARE_NGSI_RESTful_binding_v1.0.zip)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](http://fiware.github.io/context.Orion/api/v2/stable/)


This tutorial teaches FIWARE users about context data and context providers.
The tutorial builds on the **Store** entity created in the previous [stock management example](https://github.com/Fiware/tutorials.CRUD-Operations/) and enables a user to 
retrieve data about a store which is not held directly within the Orion Context Broker.

The tutorial uses [cUrl](https://ec.haxx.se/) commands throughout, but is also available as [Postman documentation](http://fiware.github.io/tutorials.Context-Providers/).

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/7c9bed4bd2ce5213a80b)

* このチュートリアルは[日本語](README.ja.md)でもご覧いただけます。

# Contents

- [Context Data and Context Providers](#context-data-and-context-providers)
  * [Entities within a stock management system](#entities-within-a-stock-management-system)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
  * [Docker](#docker)
  * [Cygwin](#cygwin)
  * [Context Provider NGSI proxy](#context-provider-ngsi-proxy)
- [Start Up](#start-up)
- [Using a Context Provider](#using-a-context-provider)
  * [Health Checks](#health-checks)
    + [Static Data Context Provider (Health Check)](#static-data-context-provider-health-check)
    + [Random Data Context Provider (Health Check)](#random-data-context-provider-health-check)
    + [Twitter API Context Provider (Health Check)](#twitter-api-context-provider-health-check)
    + [Weather API Context Provider (Health Check)](#weather-api-context-provider-health-check)
  * [Accessing the NGSI v1 QueryContext Endpoint](#accessing-the-ngsi-v1-querycontext-endpoint)
    + [Retrieving a Single Attribute Value](#retrieving-a-single-attribute-value)
    + [Retrieving Multiple Attribute Values](#retrieving-multiple-attribute-values)
  * [Context Provider Registration Actions](#context-provider-registration-actions)
    + [Registering a new Context Provider](#registering-a-new-context-provider)
    + [Read a registered Context Provider](#read-a-registered-context-provider)
    + [List all registered Context Providers](#list-all-registered-context-providers)
    + [Remove a registered Context Provider](#remove-a-registered-context-provider)
- [Next Steps](#next-steps)

# Context Data and Context Providers

> "Knowledge is of two kinds. We know a subject ourselves, or we know where we can find information upon it."
>
> — Samuel Johnson (Boswell's Life of Johnson)


Within the FIWARE platform, an entity represents the state of a physical or conceptural object which exists in the real world. 
For example, a **Store** is a real world bricks and mortar building.

The context data of that entity defines the state of that real-world object at a given moment in time. 

In all of the tutorials so far, we are holding all of the context data for our **Store** entities directly within the Orion 
Context Broker, for example stores would have attributes such as:

* A unique identifier for the store e.g. `urn:ngsi-ld:Store:002`
* The name of the store e.g. "Checkpoint Markt"
* The address "Friedrichstraße 44, 10969 Kreuzberg, Berlin"
* A physical location  e.g. *52.5075 N, 13.3903 E*

As you can see, most of these attributes are completely static (such as the location) and the others are unlikely to be
changed on a regular basis - though a street could be renamed, or the store name could be rebranded.

There is however another class of context data about the **Store** entity which is much more dynamic, information such as:

* The current temperature at the store location
* The current relative humidity at the store location
* Recent social media tweets regarding the store 

This information is always changing, and if it were held in a database, the data would always be out-of-date. To keep the context
data fresh, and to be able to retrieve the current state of the system on demand, new values for these dynamic data attributes will 
need to be retrieved whenever the entity context is requested.

Smart solutions are designed to react on the current state of the real-world. They are "aware" since they rely on dynamic data readings from 
external sources (such social media, IoT sensors, user inputs). The FIWARE platform makes the gathering and presentation of real-time 
context data transparent, since whenever an [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) request is made to the Orion Context
Broker it will always return the latest context by combining the data held within its database along with real-time data readings from 
any registered external context providers.

In order to be able to fulfil these requests, the Orion Context Broker, must first be supplied with two types of information:

* The static context data held within Orion itself  (*Entities that Orion "knows" about*) 
* Registered external context providers associated with existing entities (*Entities that Orion can "find information" about*) 


## Entities within a stock management system


Within our simple stock management system, our **Store** entity currently returns `id`, `name`,  `address` and `location` attributes. 
We will augment this with additional real-time context data from the following free publicly available data sources:

* The temperature and relative humidity from the [Weather Underground API](https://www.wunderground.com/weather/api/d/docs?MR=1)
* Recent social media tweets regarding the store from the [Twitter API](https://developer.twitter.com/)

The relationship between our entities is defined as shown:

![](https://fiware.github.io/tutorials.Context-Providers/img/entities.png)

# Architecture

This application will only make use of one FIWARE component - the [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/). Usage of the Orion Context Broker is sufficient for an application to qualify as *“Powered by FIWARE”*.

Currently, the Orion Context Broker relies on open source [MongoDB](https://www.mongodb.com/) technology to keep persistence of the context data it holds. 
To request context data from external sources, we will now need to add a simple Context Provider NGSI proxy.


Therefore, the architecture will consist of three elements:

* The [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
* The underlying [MongoDB](https://www.mongodb.com/) database :
  + Used by the Orion Context Broker to hold context data information such as data entities, subscriptions and registrations
* The **Context Provider NGSI proxy** which will will:
  + receive requests using [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
  + makes requests to publicly available data sources using their own APIs in a proprietory format 
  + returns context data back to the Orion Context Broker in [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) format.

Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run from exposed ports. 

![](https://fiware.github.io/tutorials.Context-Providers/img/architecture.png)

The necessary configuration information for the **Context Provider NGSI proxy** can be seen in the services section the of the associated `docker-compose.yml`  file:

```yaml
  context-provider:
    image: fiware/cp-web-app:latest
    hostname: context-provider
    container_name: context-provider
    networks:
        - default
    expose:
        - "3000"
    ports:
        - "3000:3000"
    environment:
        - "DEBUG=proxy:*"
        - "PORT=3000" 
        - "CONTEXT_BROKER=http://orion:1026/v2" 
        - "WUNDERGROUND_KEY_ID=<ADD_YOUR_KEY_ID>"
        - "TWITTER_CONSUMER_KEY=<ADD_YOUR_CONSUMER_KEY>"
        - "TWITTER_CONSUMER_SECRET=<ADD_YOUR_CONSUMER_SECRET>"
```

The `context-provider` container is driven by environment variables as shown:

| Key |Value|Description|
|-----|-----|-----------|
|DEBUG|`proxy:*`| Debug flag used for logging |
|PORT|`3000`|Port used by the content provider proxy and web-app for viewing data |
|CONTEXT_BROKER|`http://orion:1026/v2`| URL of the context broker to  connect to update context|
|WUNDERGROUND_KEY_ID|`<ADD_YOUR_KEY_ID>`| A consumer key used to obtain access to the Weather Underground API|
|TWITTER_CONSUMER_KEY|`<ADD_YOUR_CONSUMER_KEY>`| A consumer key used to obtain access to the Twitter API|
|TWITTER_CONSUMER_SECRET|`<ADD_YOUR_CONSUMER_SECRET>`| A user key used to obtain access to the Twitter API |

The other `context-provider` container configuration values described in the YAML file are not used in this tutorial.


The configuration information for MongoDB and the Orion Context Broker
has been described in a [previous tutorial](https://github.com/Fiware/tutorials.Entity-Relationships/)


# Prerequisites

## Docker

To keep things simple both components will be run using [Docker](https://www.docker.com). **Docker** is a container technology which allows
to different components isolated into their respective environments. 

* To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
* To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
* To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A 
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml) is used configure the required
services for the application. This means all container sevices can be brought up in a single commmand. Docker Compose is installed by default 
as part of Docker for Windows and  Docker for Mac, however Linux users will need to follow the instructions found 
[here](https://docs.docker.com/compose/install/)

## Cygwin 

We will start up our services using a simple bash script. Windows users should download [cygwin](http://www.cygwin.com/) to provide a command line functionality similar to a Linux distribution on Windows. 

## Context Provider NGSI proxy

A simple [nodejs](https://nodejs.org/) [Express](https://expressjs.com/) application has been bundled as part of the repository. The application offers an NGSI v1 interface for four different context providers - the Weather Underground API, the Twitter Search API and two dummy data context providers - a static data provider (which always returns the same data) and a random data context provider (which will change every time it is invoked). 

More information about the proxy endpoints can be found [here](https://github.com/Fiware/tutorials.Context-Providers/blob/master/proxy/README.md)

* In order to access the Weather Underground API, you will need to sign up for a key at https://www.wunderground.com/weather/api/ 
* In order to access the Twitter Search API, you will have to create an app in Twitter via  https://apps.twitter.com/app/new to obtain a 
Consumer Key & Consumer Secret. 


Replace the placeholders in `docker-compose.yml` in the root of the repository with the values you obtain for your application:

```yaml
    environment:
        - "DEBUG=proxy:*"
        - "CONTEXT_BROKER=http://orion:1026/v2"
        - "WUNDERGROUND_KEY_ID=<ADD_YOUR_KEY_ID>"
        - "TWITTER_CONSUMER_KEY=<ADD_YOUR_CONSUMER_KEY>"
        - "TWITTER_CONSUMER_SECRET=<ADD_YOUR_CONSUMER_SECRET>"
```

If you do not wish to sign-up for an API key, you can use data from the random data context provider instead.


# Start Up

All services can be initialised from the command line by running the bash script provided within the repository:

```console
./services create; ./services start;
```

This command will also import seed data from the previous [Stock Management example](https://github.com/Fiware/tutorials.CRUD-Operations) on startup.

>:information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
>```console
>./services stop
>``` 
>

# Using a Context Provider

>:information_source: **Tip** You can also watch the status of recent requests yourself by following the container logs or 
>viewing information on `localhost:3000/app/monitor` on a web browser.
>
>![FIWARE Monitor](https://fiware.github.io/tutorials.Context-Providers/img/monitor.png)

## Health Checks

The nodejs proxy application offers a `health` endpoint for each of the four context providers. Making a request to the appropriate endpoint will check that the provider is running and external data can be received. The application runs on port `3000`. 


### Static Data Context Provider (Health Check)

This example returns the health of the Static Data Context Provider endpoint.

A non-error response shows that an NGSI proxy is available on the network and returning values.
Each Request will return the same data.

#### Request:

```console
curl -X GET \
  'http://localhost:3000/health/static'
```

#### Response:

```json
{
    "array": [
        "Arthur",
        "Dent"
    ],
    "boolean": true,
    "number": 42,
    "structuredValue": null,
    "text": "I never could get the hang of thursdays"
}
```

### Random Data Context Provider (Health Check)

This example returns the health of the Random Data Generator Context Provider endpoint.

A non-error response shows that an NGSI proxy is available on the network and returning values.
Each Request will return some random dummy data.

#### Request:

```console
curl -X GET \
  'http://localhost:3000/health/random'
```

#### Response:

```json
{
    "array": [
        "sit", "consectetur", "sint", "excepteur"
    ],
    "boolean": false,
    "number": 4,
    "structuredValue": null,
    "text": " nisi reprehenderit pariatur. Aute ea"
}
```


### Twitter API Context Provider (Health Check)

This example returns the health of the Twitter API Context Provider endpoint.

A non-error response shows that an NGSI proxy for the Twitter API is available on the network and returning values.

If the proxy is correctly configured to connect to the Twitter API, a series of Tweets will be
returned.

The Twitter API uses OAuth2: 
* To get Consumer Key & Consumer Secret for the Twitter API, you have to create an app in Twitter 
  via [https://apps.twitter.com/app/new](https://apps.twitter.com/app/new). Then you'll be taken to a page 
  containing Consumer Key & Consumer Secret.
* For more information see: [https://developer.twitter.com/](https://developer.twitter.com/)

#### Request:

```console
curl -X GET \
  'http://localhost:3000/health/twitter'
```

#### Response:

The response will contain a series of 15 tweets about FIWARE. The full response is rather long, but a snippet can be seen below:

```json
{
	"statuses": [
	{
		"created_at": "Mon Apr 23 13:08:35 +0000 2018",
		"id": 988404265227038700,
		"id_str": "988404265227038721",
		"text": "@FIWARE: Full house today during the Forum Industrie 4.0 at @Hannover_Messe as #FIWARE Foundation CEO ...",
		"truncated": false,
		"entities": {
			... ETC
		},
		"metadata": {
	        ... ETC
	    }
	    ... ETC
	}
	... ETC

    ],
    "search_metadata": {
        "completed_in": 0.089,
        "max_id": 988407193497108500,
        "max_id_str": "988407193497108481",
        "next_results": "?max_id=988373340074242048&q=FIWARE&include_entities=1",
        "query": "FIWARE",
        "refresh_url": "?since_id=988407193497108481&q=FIWARE&include_entities=1",
        "count": 15,
        "since_id": 0,
        "since_id_str": "0"
    }

}
```

As you can see details the `text` of each tweet is available within the `statuses` array.

### Weather API Context Provider (Health Check)

This example returns the health of the Static Data Context Provider endpoint.

A non-error response shows that an NGSI proxy is available on the network and returning values.
Each Request will return the same data.

#### Request:

```console
curl -X GET \
  'http://localhost:3000/health/weather'
```

#### Response:

The response will contain a data about the current weather in Berlin. The full response is rather long, but a snippet can be seen below:

```json
{
    "response": {
        "version": "0.1",
        "termsofService": "http://www.wunderground.com/weather/api/d/terms.html",
        "features": {
            "conditions": 1
        }
    },
    "current_observation": {
        "image": {
            ... ETC
        },
        "display_location": {
            "full": "Berlin, Germany",
            ... ETC
        },
        "observation_location": {
            ... ETC
        },
        ... ETC
        "temp_f": 71.4,
        "temp_c": 21.9,
        "relative_humidity": "65%",
        "wind_string": "From the SW at 2.5 MPH Gusting to 5.6 MPH",
        "wind_dir": "SW",
        ... ETC
    }
}
```


As you can see details of the current temperature and relative humidity are available within the attributes of the `current_observation`


## Accessing the NGSI v1 QueryContext Endpoint

Because the `3000` port of the Context Provider has been exposed outside of the Docker container, it is possible for curl to make requests directly to the Context Provider - this simulates the requests that would have been made by the Orion Context Broker. You can also simulate making the requests as part of the docker container network by running the `appropriate/curl` Docker image.

Firstly obtain the name of the network used within the Docker containers by running

```console
docker network ls
```

Then run the following curl command including the `--network` parameter:

```console
docker run --network fiware_default --rm appropriate/curl -X GET http://context-provider:3000/health/random
```

As you can see, within the network, the host name of the Context Provider is `context-provider`.


### Retrieving a Single Attribute Value

This example uses the NGSI v1 `queryContext` endpoint to request a `temperature` reading from the  Static Data Generator Context Provider. The requested attributes are found within the `attributes` array of the POST body.

The Orion Context Broker will make similar requests to this `queryContext` endpoint once a context provider has been registered.

#### Request:

```console
curl -X POST \
  'http://localhost:3000/proxy/static/temperature/queryContext' \
  -H 'Content-Type: application/json' \
  -d '{
    "entities": [
        {
            "type": "Store",
            "isPattern": "false",
            "id": "urn:ngsi-ld:Store:001"
        }
    ],
    "attributes": [
        "temperature"
    ]
} '
```

#### Response:

The response will be in NGSI v1 response format as shown. The `attributes` element holds the data returned - an object of `type:Number` with the `value:42`.

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "temperature",
                        "type": "Number",
                        "value": 42
                    }
                ],
                "id": "urn:ngsi-ld:Store:001",
                "isPattern": "false",
                "type": "Store"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```




### Retrieving Multiple Attribute Values

It is possible for the Orion Context Broker to make a request for multiple data values . This example uses the NGSI v1 `queryContext` endpoint to request  `temperature` and `relativeHumidity` readings from the Random Data Generator Context Provider. The requested attributes are found within the `attributes` array of the POST body.

#### Request:

```console
curl -X POST \
  'http://localhost:3000/proxy/v1/random/weatherConditions/queryContext' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 2ae9e6d6-802b-4a62-a561-5c7739489fb3' \
  -d '{
    "entities": [
        {
            "type": "Store",
            "isPattern": "false",
            "id": "urn:ngsi-ld:Store:001"
        }
    ],
    "attributes": [
        "temperature",
        "relativeHumidity"
    ]
}''
```

#### Response:

The response will be in NGSI v1 response format as shown. The `attributes` element holds the data returned

```json
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "temperature",
                        "type": "Number",
                        "value": 27
                    },
                    {
                        "name": "relativeHumidity",
                        "type": "Number",
                        "value": 21
                    }
                ],
                "id": "urn:ngsi-ld:Store:001",
                "isPattern": "false",
                "type": "Store"
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
```


## Context Provider Registration Actions

All Context Provider Registration actions take place on the `v2/registrations` endpoint. The standard CRUD mappings apply:

* Creation is mapped to the HTTP POST
* Reading/Listing registrations to HTTP GET verb
* Deletion is mapped to HTTP DELETE

### Registering a new Context Provider
This example registers the Random Data Context Provider with the Orion Context Broker.

The body of the request states that: *"The URL* `http://context-provider:3000/proxy/v1/random/weatherConditions` *is capable of providing* `relativeHumidity`  and `temperature` *data for the entity called* `id=urn:ngsi-ld:Store:001`.*"*

The values are **never** held within Orion, it is always requested on demand from the registered context provider. Orion merely holds the registration information about which context providers can offer context data.

The presence of the flag `"legacyForwarding": true` indicates that the registered context provider offers an NGSI v1 interface - therefore Orion  will make POST request for data on `http://context-provider:3000/proxy/v1/random/weatherConditions/queryContext` using the NGSI v1 format for the body, and expect to receive data in the NGSI v1 format in return.

>*Note:* if you have registered with the Weather API, you can retrieve live values for `temperature` and `relativeHumidity` in Berlin by placing the following `url` in the `provider`:
>
> * `http://context-provider:3000/proxy/v1/weather/weatherConditions`
>

This request will return with a **201 - Created** response code. The `Location` Header of the response contains a path to the registration record held in Orion

#### Request:

```console
curl -X POST \
  'http://localhost:1026/v2/registrations' \
  -H 'Content-Type: application/json' \
  -d '{
  "description": "Random Weather Conditions",
  "dataProvided": {
    "entities": [
      {
        "id": "urn:ngsi-ld:Store:001",
        "type": "Store"
      }
    ],
    "attrs": [
      "relativeHumidity"
    ]
  },
  "provider": {
    "http": {
      "url": "http://context-provider:3000/proxy/v1/random/weatherConditions"
    },
     "legacyForwarding": true
  }
}'
```


Once a Context Provider has been registered, the new context data will be included if the context of the **Store** entity `urn:ngsi-ld:Store:001` is requested using the `/entities/<entity-id>` endpoint:

#### Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Store:001
```

#### Response:

```json
{
    "id": "urn:ngsi-ld:Store:001",
    "type": "Store",
    "address": {
        "type": "PostalAddress",
        "value": {
            "streetAddress": "Bornholmer Straße 65",
            "addressRegion": "Berlin",
            "addressLocality": "Prenzlauer Berg",
            "postalCode": "10439"
        },
        "metadata": {}
    },
    "location": {
        "type": "geo:json",
        "value": {
            "type": "Point",
            "coordinates": [
                13.3986,
                52.5547
            ]
        },
        "metadata": {}
    },
    "name": {
        "type": "Text",
        "value": "Bösebrücke Einkauf",
        "metadata": {}
    },
    "temperature": {
        "type": "Number", "value": "22.6", "metadata": {}
    },
    "relativeHumidity": {
        "type": "Number",
        "value": "58%",
        "metadata": {}
    }
}
```


Similarly, a single attribute can be obtained by making a request to the `/entities/<entity-id>/attrs/<attribute>`

#### Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Store:001/attrs/relativeHumidity/value'
```

#### Response:

```
"58%"
```


### Read a registered Context Provider

This example reads the registration data with the id 5addeffd93e53f86d8264521 from the context.

Registration data can be obtained by making a GET request to the `/v2/registrations/<entity>` endpoint.

```console
curl -X DELETE \
  'http://localhost:1026/v2/registrations/5ad5b9435c28633f0ae90671'
```


### List all registered Context Providers

This example lists all registered context providers

Full context data  for a specified entity type can be retrieved by making a GET request to the `/v2/registrations/` endpoint.

#### Request:

```console
curl -X GET \
  'http://localhost:1026/v2/registrations'
```

#### Response:

```json
[
    {
        "id": "5addeffd93e53f86d8264521",
        "description": "Random Weather Conditions",
        "dataProvided": {
            "entities": [
                {
                    "id": "urn:ngsi-ld:Store:002",
                    "type": "Store"
                }
            ],
            "attrs": [
                "temperature",
                "relativeHumidity"
            ]
        },
        "provider": {
            "http": {
                "url": "http://context-provider:3000/proxy/v1/random/weatherConditions"
            },
            "supportedForwardingMode": "all",
            "legacyForwarding": true
        },
        "status": "active"
    }
]
```


### Remove a registered Context Provider

Registrations can be deleted by making a DELETE request to the `/v2/registrations/<entity>` endpoint.

```console
curl -X DELETE \
  'http://localhost:1026/v2/registrations/5ad5b9435c28633f0ae90671'
```

# Next Steps

Want to learn how to add more complexity to your application by adding advanced features?
You can find out by reading the other tutorials in this series:

&nbsp; 101. [Getting Started](https://github.com/Fiware/tutorials.Getting-Started)<br/>
&nbsp; 102. [Entity Relationships](https://github.com/Fiware/tutorials.Entity-Relationships/)<br/>
&nbsp; 103. [CRUD Operations](https://github.com/Fiware/tutorials.CRUD-Operations/)<br/>
&nbsp; 104. [Context Providers](https://github.com/Fiware/tutorials.Context-Providers/)<br/>
&nbsp; 105. [Altering the Context Programmatically](https://github.com/Fiware/tutorials.Accessing-Context/)<br/> 
&nbsp; 106. [Subscribing to Changes in Context](https://github.com/Fiware/tutorials.Subscriptions/)<br/>

&nbsp; 201. [Introduction to IoT Sensors](https://github.com/Fiware/tutorials.IoT-Sensors/)<br/>
&nbsp; 202. [Provisioning an IoT Agent](https://github.com/Fiware/tutorials.IoT-Agent/)<br/>

