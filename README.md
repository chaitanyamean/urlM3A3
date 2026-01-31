# Long Polling for Image Uploads

We have implemented a long-polling mechanism to handle asynchronous image processing (e.g., thumbnail generation). This allows clients to receive immediate feedback when processing is complete without spamming the server with requests.

## Workflow

1.  **Upload Image**
    -   **Endpoint**: `POST /upload`
    -   **Input**: `multipart/form-data` with key `image`.
    -   **Response**: Returns success message and the saved **Image ID**.
    -   *Behind the scenes*: The server saves the image, sets status to `PENDING`, and publishes an event to the background worker.

2.  **Poll for Status**
    -   **Endpoint**: `GET /image/status/{id}`
    -   **Behavior**:
        -   The request **blocks** (waits) until the background worker finishes processing.
        -   If processing finishes quickly, it returns immediately.
        -   If it takes time, the server holds the connection open (Long Polling).
    -   **Response**: Returns the status string (e.g., `COMPLETED` or `FAILED`).

## Architecture Components

-   **ImageController**: Handles the HTTP requests and manages `DeferredResult` for holding connections.
-   **ImageStatusService**: A centralized service that maps `imageId` to waiting `DeferredResult` objects.
-   **ThumbnailQueueService**: The background worker that processes the image and notifies `ImageStatusService` upon completion.


### [Q1] 1. Database Changes
- Modified `ImageEntity` to include a `status` field (PENDING, PROCESSING, COMPLETED, FAILED).

### 2. Service Layer
- **Created `ImageStatusService`**: Uses `DeferredResult<String>` to hold client connections. It maintains a concurrent map of `imageId` -> `DeferredResult`.
- **Updated `ThumbnailQueueService`**:
    - Listen for `IMAGE_UPLOADED` event.
    - Updates DB status to `PROCESSING`.
    - Simulates processing (sleep).
    - Updates DB status to `COMPLETED`.
    - Calls `imageStatusService.notify(imageId, status)` to release the waiting client.

### 3. Controller Layer
- **`ImageController`**:
    - Injected `ImageStatusService`.
    - Added `GET /image/status/{id}` endpoint.
    - Returns `DeferredResult<String>` from the service.

## How to Verify

You can use the provided script `verify_polling.sh` to test the flow automatically.

```bash
./verify_polling.sh
```

Or manually with cURL:

```bash
# 1. Upload
curl -u admin:admin -F "image=@test.jpg" http://localhost:8080/upload

# 2. Check Status (Substitute ID from above)
curl http://localhost:8080/image/status/1
```

## [Q2] Instead of returning the response immediately when the status is pending, keep the API call open and return when the task is complete.
 - ** DeferredResult: In ImageStatusService.java, we create a DeferredResult object. When the controller returns this object, Spring holds the HTTP connection open instead of responding immediately.
 - ** Wait State: The client's request hangs (waits) until one of two things happens:
    - We call result.setResult(status) (Success/Failure).
    - The timeout is reached (set to 30 seconds).

## [Q3] What happens when the request times out in long polling? Implement a retry mechanism.
- **Timeout Behavior**: If the processing takes longer than the timeout (30s) or the notification is missed, the server returns "TIMEOUT".
- **Retry Mechanism**:
    - **Client-Side**: The client receives "TIMEOUT" and must retry the request.
    - **Server-Side**: The `ImageStatusService` now checks the database state immediately upon subscription. If the task is already `COMPLETED` (e.g., during the retry), it returns the result immediately, handling the "Lost Notification" race condition.

## [Q4] What are the pros/cons of short polling vs long polling?

| Feature | Short Polling | Long Polling |
| :--- | :--- | :--- |
| **Efficiency** | **Low** (Wastes resources on empty checks) | **High** (Server waits, no empty checks) |
| **Latency** | **High** (Depends on polling interval) | **Low** (Instant response) |
| **Complexity** | **Low** (Simple HTTP) | **Medium** (Requires Async support) |
| **Server Load** | **High** (Many requests) | **Low** (Open connections, but low CPU) |

## Game Score Posting → Leaderboard Update

In this use-case, the leaderboard can update async after our game score has been posted.

But there’s another requirement here. Leaderboard should update based on scores posted by others as well!

You can poll, say every 5 seconds, for the latest leaderboard data but it is stale by 5 seconds. (Think live sport scores)

Polling would work, yes, but to get real-time updates, you’d have to keep reducing the interval. This will increase the server load.

Enter Websockets, which allow you to send bidirectional messages without repeated HTTP calls.

### [Q5] Implement a websocket to fetch leaderboard updates from the server.

#### Implementation Steps
1.  **Dependencies**: Added `spring-boot-starter-websocket` to `pom.xml`.
2.  **Handler**: Created `LeaderboardHandler` (extends `TextWebSocketHandler`) to manage sessions and broadcast messages.
3.  **Configuration**: Created `WebSocketConfig` to register the handler at `/ws/leaderboard`.
4.  **Service**: Created `LeaderboardService` to simulate score updates every 2 seconds and push them to clients.
5.  **Security**: Updated `SecurityConfig` to permit access to `/ws/**` (whitelisted).

### [Q6] You’d see that the websocket URLs start with ws. Find out the difference between ws:// and wss://
- **ws://**: Unencrypted WebSocket connection (like HTTP).
- **wss://**: Encrypted WebSocket connection (like HTTPS). Secure and recommended for production.

### [Q7] How would you authenticate and authorize websocket connections?
- **Token Query Param**: Pass JWT in URL `ws://host/chat?token=xyz`.
- **Ticket System**: Client requests a short-lived ticket via HTTPS, then uses it to connect WS.
- **Headers**: (Note: Standard JS WebSocket API doesn't support custom headers, so Query Param or Ticket is common).

## Order Placement Flow

Previously, we split synchronous vs async parts (e.g. payment = sync, email/warehouse = async via queue/pubsub).

For notifying external systems, we use Webhooks. They are **HTTP callbacks** you make to someone else’s URL.

We use it when *our system* needs to tell *another* system about an event, like:

- Notify an external warehouse management system we don’t own.
- Tell a shipping provider to create a label.
- Inform a third-party CRM or ERP system about a new order.

### [Q8] Implement a webhook to send analytics data to a 3rd party service.

#### Implementation Steps
1.  **Configuration**: Added `webhook.analytics.url` to `application.properties`.
2.  **Service**: Created `WebhookService` to send HTTP POST requests using `RestTemplate`.
3.  **Integration**: Updated `ThumbnailQueueService` to trigger the webhook on `IMAGE_UPLOADED` events.



