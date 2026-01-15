# Streaming data (v4)

The Stream V4 service allows clients to subscribe to real-time data events within the Sensara Care ecosystem. This API utilizes Server-Sent Events (SSE) to push data updates to the client efficiently, rather than requiring the client to constantly poll for changes.

It follows a two-step process: register a stream to get a token, then consume the data using that token.

## Workflow
- Discovery: Query the available data types to understand what can be streamed and which properties can be filtered.
- Registration: Submit a request defining the specific data types and filters you wish to monitor. This generates a unique registration token.
- Listen: Open a persistent connection using the token to receive the event stream.
- Termination: Explicitly delete the registration when the stream is no longer needed to free up server resources.

---

# Swagger documentation

| Environment | url                                                                    |
|-------------|------------------------------------------------------------------------|
| test        | https://api.test.sensara.network/v3/swagger-ui/index.html#/Stream%20V4 |
| acceptance  | https://api.acc.sensara.network/v3/swagger-ui/index.html#/Stream%20V4  |
| production  | https://api.sensara.eu/v3/swagger-ui/index.html#/Stream%20V4           |

---

# Endpoints

## Register Stream

This endpoint initializes a streaming session. You must provide a schema defining exactly what data you want to receive (e.g., specific events or alarms) and any filters to apply (e.g., specific organization IDs).
- Description: Register for streaming data using filters.
- Response: If successful, the API returns a text/plain string containing a token. This token is required for the subsequent GET call to consume the data.
- [List of supported data type](#data-types)


    Method: POST

    Endpoint: /v4/streams

    Response (201): Returns a token (String) to be used for the data stream.

<b>Example json body, filter on organizationId and sectorId for AlarmEvents & ResidentStateEvents</b>

```
{
    "dataTypeRequests": [
              {
            "dataType": "AlarmEvent",
            "filters": [
                {
                    "filterProperty": "organizationId",
                    "filterValues": [
                        "SENSARA"
                    ]
                },
                {
                    "filterProperty": "sectorId",
                    "filterValues": [
                        "NEW_REGISTRATIONS"
                    ]
                }
            ]
        },
        {
            "dataType": "ResidentStateEvent",
            "filters": [
                {
                    "filterProperty": "organizationId",
                    "filterValues": [
                        "SENSARA"
                    ]
                },
                {
                    "filterProperty": "sectorId",
                    "filterValues": [
                        "NEW_REGISTRATIONS"
                    ]
                }
            ]
        }
    ],
    "maxFlowRateMillis": 100
}
```

---

## Listen to Stream (SSE)

This endpoint opens the Server-Sent Events (SSE) channel. It is a long-lived connection that stays open to receive data pushes.
- Heartbeat: If no data events occur, the server sends a heartbeat event approximately every 30 seconds to keep the connection alive.
- Tooling: Standard Swagger/OpenAPI interfaces often do not display SSE streams well. It is recommended to use browser developer tools, Postman or a dedicated SSE client to view the data flow.


    Method: GET

    Endpoint: /v4/streams/{token}/data

    Path Parameter: token (UUID) - The token received from the registration endpoint.


### Resuming Streams with Last-Event-ID

The Stream V4 API supports the standard Server-Sent Events (SSE) mechanism for resuming interrupted connections. This ensures that clients do not miss critical data (such as alarms or sensor events) if the network connection drops momentarily or if the client needs to restart.

<b>How it Works</b>

Every event sent over the data stream includes a unique identifier (id). The client should track the id of the most recent event successfully processed. If the connection is severed, the client can request the server to replay any events that occurred after that specific point in time.

<b>Implementation Steps</b>

- Capture the ID: When listening to the /v4/streams/{token}/data endpoint, every incoming event message will contain an id field. Store this value locally.
- Detect Disconnection: If the SSE connection is closed unexpectedly or timed out.
- Reconnect with Header: Initiate a new GET request to the same endpoint. Include the Last-Event-ID header with the value of the last stored id.

---

## Discover Data Types

Before registering, use this endpoint to discover which data types are available for streaming and what properties can be used for filtering.
- Description: Retrieve supported data types and (complex) data type properties used in streaming data.
- Usage: The response provides a map where keys are data types (e.g., ALARM) and values are lists of properties (e.g., organizationId).


    Method: GET

    Endpoint: /v4/streams/data-types

    Response (200): A map of data types to their properties.

---

## Delete Registration

When a client is finished with a stream, they should call this endpoint to invalidate the token and stop the server from processing filters for that session.

    Method: DELETE

    Endpoint: /v4/streams/registrations/{token}

    Path Parameter: token (UUID).

    Response (204): Stream registration deleted.

---

# Supported data types and properties used in streaming data.
<a id="data-types"></a>

### AdlEventResponse
| Property Name | Property Type | Filterable |
| :--- | :--- | :--- |
| `organizationId` | String | true |
| `residentId` | String | true |
| `sectorId` | String | true |
| `correlationId` | String | true |
| `type` | String | false |
| `timestamp` | Instant | false |

### AlarmEvent
| Property Name | Property Type | Filterable |
| :--- | :--- | :--- |
| `organizationId` | String | true |
| `residentId` | String | true |
| `sectorId` | String | true |
| `correlationId` | String | true |
| `id` | UUID | false |
| `timestamp` | Instant | false |
| `createdDate` | Instant | false |
| `status` | String | false |
| `type` | String | false |
| `location` | String | false |
| `escalated` | Boolean | false |
| `creatorId` | String | false |
| `assigneeId` | String | false |
| `userIds` | List | true |

### LastLocationResponse
| Property Name | Property Type | Filterable |
| :--- | :--- | :--- |
| `organizationId` | String | true |
| `residentId` | String | true |
| `sectorId` | String | true |
| `correlationId` | String | true |
| `iD` | UUID | false |
| `deviceId` | String | false |
| `label` | String | false |
| `timestamp` | LocalDateTime | false |
| `location` | SensorLocation | false |

### NotificationResponse
| Property Name | Property Type | Filterable |
| :--- | :--- | :--- |
| `organizationId` | String | true |
| `residentId` | String | true |
| `sectorId` | String | true |
| `correlationId` | String | true |
| `id` | String | false |
| `notificationType` | String | false |
| `intervalStartTime` | Instant | false |
| `sensorLocation` | String | false |
| `parameters` | Map | false |

### ResidentStateEvent
| Property Name | Property Type | Filterable |
| :--- | :--- | :--- |
| `organizationId` | String | true |
| `residentId` | String | true |
| `sectorId` | String | true |
| `correlationId` | String | true |
| `timestamp` | Instant | false |
| `newState` | String | false |
| `location` | String | false |

### StateExtramuralResponse
| Property Name | Property Type | Filterable |
| :--- | :--- | :--- |
| `organizationId` | String | true |
| `residentId` | String | true |
| `sectorId` | String | true |
| `correlationId` | String | true |
| `type` | String | false |
| `state` | String | false |
| `timestamp` | Instant | false |

### TechnicalAlarmEvent
| Property Name | Property Type | Filterable |
| :--- | :--- | :--- |
| `organizationId` | String | true |
| `residentId` | String | true |
| `sectorId` | String | true |
| `correlationId` | String | true |
| `id` | UUID | false |
| `timestamp` | Instant | false |
| `createdDate` | Instant | false |
| `expirationDate` | Instant | false |

---