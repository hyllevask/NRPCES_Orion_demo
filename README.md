# Walkthrough
This repo serves as a short example on how the orion context broker can be used. This example has been tested with the following configuration:
- Ubuntu 22.04.2 LTS
- Docker version 28.0.1, build 068a01e
- Python 3.12.9
- Flask 3.1.0
## Chat-GPT: Introduction to FIWARE & Orion Context Broker  
**This whole paragraph is genereated by chat-GPT** 
### FIWARE Overview  
- **FIWARE** is an open-source platform for building smart applications.  
- It provides **context management** capabilities, meaning it helps manage and share real-time data from various sources (IoT, applications, databases).  
- It follows **NGSI standards** for exchanging structured data.  

### Orion Context Broker (OCB)  
- The **core component** of FIWARE for handling real-time data.  
- Works like a **central hub**: collects, stores, updates, and distributes data.  
- Uses **entities, attributes, and metadata** to represent real-world objects.  
- Supports **subscriptions and notifications** to track changes in data.  

### Key Concepts & Nomenclature  
- **Entity**: A digital representation of a real-world object (e.g., a machine, sensor, or vehicle).  
- **Attributes**: Properties of an entity (e.g., temperature, status, location).  
- **Context**: The current state of an entity at a given time.  
- **Subscription**: A mechanism to notify external services when data changes.  
- **Context Provider**: A service that dynamically provides additional data when requested.  

### How Orion Works in Practice  
1. **Register entities**: Define objects (e.g., "Asset1" with temperature & power consumption).  
2. **Update entities**: Change values dynamically (e.g., sensor data updates).  
3. **Query context**: Fetch real-time data from Orion.  
4. **Subscribe to changes**: Get notifications when specific values change.  
5. **Integrate with external systems**: Connect databases, IoT devices, or analytics tools. 


# Orion context broker

The context brooker and its associated database is hosted as docker containers.
In this repository there is a docker-compose.yml file included, it reads:

```yaml
services:
  mongo:
    image: mongo:4.4
    container_name: fiware-orion-mongo
    command: --nojournal
    ports:
      - "27017:27017"
    networks:
      - fiware

  orion:
    image: fiware/orion
    container_name: fiware-orion
    depends_on:
      - mongo
    ports:
      - "1026:1026"
    command: -dbURI mongodb://mongo:27017
    networks:
      - fiware

networks:
  fiware:
```
If you are unsure what this file specifies, please look up docker info, there are plenty of information online.

To run the file above, go to the directory and run the bash command

```bash
docker compose up -d
```
This should start the orion context broker on port 1026. Verify that the service is running by running.


```bash
curl -X GET 'http://localhost:1026/version'
```
The expected response is

```json
{
"orion" : {
  "version" : "4.1.0",
  "uptime" : "0 d, 0 h, 2 m, 3 s",
  "git_hash" : "95d82a97d3d1128b42c79e5c2f659a0d19a0687f",
  "compile_time" : "Thu Sep 12 12:02:08 UTC 2024",
  "compiled_by" : "root",
  "compiled_in" : "buildkitsandbox",
  "release_date" : "Thu Sep 12 12:02:08 UTC 2024",
  "machine" : "x86_64",
  "doc" : "https://fiware-orion.rtfd.io/en/4.1.0/"
}
}
```


# Register a wagon entity
We register a wagon with atributes WOstatus and Location. The wagons id is 9911 and it is of the type wagon.
```bash
curl -X POST 'http://localhost:1026/v2/entities' \
  -H 'Content-Type: application/json' \
  -d '{
    "id": "9911",
    "type": "Wagon",
    "WOstatus": {
      "type": "String"
    },
    "Location": {
      "type": "String"
    }
  }'
```

We also register a locomotive with the same attributes.
```bash
curl -X POST 'http://localhost:1026/v2/entities' \
  -H 'Content-Type: application/json' \
  -d '{
    "id": "113",
    "type": "Loco",
    "WOstatus": {
      "type": "String"
    },
    "Location": {
      "type": "String"
    }
  }'
```

Check that the assets are registrated. 

```bash
curl -X GET 'http://localhost:1026/v2/entities' | jq
```
(jq is a command-line JSON processor that makes the output more human-readable, however, it is not necessary) 

Expected outcome

```json
[
  {
    "id": "9911",
    "type": "Wagon",
    "Location": {
      "type": "String",
      "value": null,
      "metadata": {}
    },
    "WOstatus": {
      "type": "String",
      "value": null,
      "metadata": {}
    }
  },
  {
    "id": "113",
    "type": "Loco",
    "Location": {
      "type": "String",
      "value": null,
      "metadata": {}
    },
    "WOstatus": {
      "type": "String",
      "value": null,
      "metadata": {}
    }
  }
]

```


## Register a subscription for only wagons
The FIWARE ecosystem 
```bash
curl -X POST "http://localhost:1026/v2/subscriptions/" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d '{
       "description": "Notify when any attribute in any Wagon entity changes",
       "subject": {
         "entities": [
           {
             "idPattern": ".*",
             "type": "Wagon"
           }
         ],
         "condition": {
           "attrs": []
         }
       },
       "notification": {
         "http": {
           "url": "http://172.23.49.254:5000/notify"
         },
         "attrsFormat": "normalized"
       },
       "throttling": 5
     }'  
```

Check that it is registered

```bash
curl -X GET "http://localhost:1026/v2/subscriptions" -H "Accept: application/json"|jq
```

```json
[
  {
    "id": "67ca94599c391ffcea00324b",
    "description": "Notify when any attribute in any Wagon entity changes",
    "status": "active",
    "subject": {
      "entities": [
        {
          "idPattern": ".*",
          "type": "Wagon"
        }
      ],
      "condition": {
        "attrs": [],
        "notifyOnMetadataChange": true
      }
    },
    "notification": {
      "attrs": [],
      "onlyChangedAttrs": false,
      "attrsFormat": "normalized",
      "http": {
        "url": "http://172.23.49.254:5000/notify"
      },
      "covered": false
    },
    "throttling": 5
  }
]
```
# Start the subscription service
This simple app just prints the recived data.
```bash
python simple_app.py
```

```bash
 * Serving Flask app 'simple_app'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://172.23.49.254:5000
Press CTRL+C to quit
  
```


# Feed data to Orion
We are now ready to start feeding data to Orion using the NGSI-v2 standard.
In this short example we simulate work order generated for wagon 9911.

First the workorder is created when the wagon is in operation. XXYY is the current location along the line.

```bash
curl -X PATCH 'http://localhost:1026/v2/entities/9911/attrs'   -H 'Content-Type: application/json'   -d '{
    "Location": {
      "value": "In operations: XXYY"
    }, "WOstatus":{"value": "Created"}
  }'
```


```bash
Received notification: {'subscriptionId': '67ca94599c391ffcea00324b', 'data': [{'id': '9911', 'type': 'Wagon', 'Location': {'type': 'Text', 'value': 'In operations: XXYY', 'metadata': {}}, 'WOstatus': {'type': 'Text', 'value': 'Created', 'metadata': {}}}]}
172.19.0.3 - - [07/Mar/2025 08:02:11] "POST /notify HTTP/1.1" 200 -
```
### Wagon arrives to WS

```bash
curl -X PATCH 'http://localhost:1026/v2/entities/9911/attrs'   -H 'Content-Type: application/json'   -d '{
    "Location": {
      "value": "WS1"
    }
  }'
```

### Wagon is repaired
```bash
curl -X PATCH 'http://localhost:1026/v2/entities/9911/attrs'   -H 'Content-Type: application/json'   -d '{
    "WOstatus":{"value": "Done"}
  }'
```
# Remarks
- By defult, the context broker only holds the latest state for each entity 

 

