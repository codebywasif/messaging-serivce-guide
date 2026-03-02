# Android Mobile App - Integration Guide

This guide details how to integrate the Android mobile application (used by Drivers/Customers) with the Three-Entity Messaging Service Backend.

## 1. Prerequisites

Add the Socket.IO Java Client and Retrofit (or OkHttp) to your `build.gradle`:

```gradle
dependencies {
    // Socket.IO for real-time
    implementation ('io.socket:socket.io-client:2.1.0') {
        exclude group: 'org.json', module: 'json'
    }
    // Network calls
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
}
```

## 2. Managing the Socket.IO Connection

You will use a Singleton or bounded Service in Android to keep the WebSocket alive while the app is in the foreground.

```java
import io.socket.client.IO;
import io.socket.client.Socket;
import io.socket.emitter.Emitter;

public class ChatSocketManager {
    private Socket mSocket;
    private static final String SERVER_URL = "https://your-api-url.com";

    public void connect(String jwtToken) {
        try {
            IO.Options opts = new IO.Options();
            // Pass JWT for multi-tenant auth
            opts.auth = java.util.Collections.singletonMap("token", jwtToken);
            
            mSocket = IO.socket(SERVER_URL, opts);
            
            // Register Listeners
            mSocket.on("message:receive", onNewMessage);
            mSocket.on("typing:status", onTyping);
            
            mSocket.connect();
        } catch (URISyntaxException e) {
            e.printStackTrace();
        }
    }

    public void disconnect() {
        if (mSocket != null) {
            mSocket.disconnect();
            mSocket.off("message:receive", onNewMessage);
        }
    }
    
    // Listeners
    private Emitter.Listener onNewMessage = new Emitter.Listener() {
        @Override
        public void call(final Object... args) {
            JSONObject data = (JSONObject) args[0];
            // Broadcast via EventBus/LiveData to your Chat UI Fragment
            // String content = data.getString("content");
            // Note: If data.getString("recipient_type") contains "drivers" or "operators", it's a broadcast!
        }
    };

    private Emitter.Listener onMessageDeleted = new Emitter.Listener() {
        @Override
        public void call(final Object... args) {
            JSONObject data = (JSONObject) args[0];
            // String messageId = data.getString("message_id");
            // Remove from your local UI list instantly
        }
    };
}
```

## 3. Emitting Typing Indicators

When the user focuses the text input or types:

```java
public void sendTypingStart(String jobRoomId) {
    if (mSocket != null && mSocket.connected()) {
        JSONObject data = new JSONObject();
        data.put("room", jobRoomId);
        mSocket.emit("typing:start", data);
    }
}
```

## 4. Sending Messages via REST

Messages should be dispatched via `POST` requests, not WebSockets, to ensure delivery tracking and database persistence.

Using Retrofit interface:

```java
public interface ChatApi {
    @POST("/api/chat/driver/{driverId}/send")
    Call<MessageResponse> sendMessage(
        @Path("driverId") String driverId,
        @Header("Authorization") String token,
        @Body MessageRequest body
    );

    @GET("/api/history/recent")
    Call<List<Conversation>> getRecentConversations(@Header("Authorization") String token);

    @GET("/api/history/conversation/{convId}/messages")
    Call<List<Message>> getConversationMessages(
        @Path("convId") String convId, 
        @Header("Authorization") String token
    );
}

// Request Body payload:
// {
//   "content": "Arrived at destination",
//   "messageType": "text",
//   "recipientType": "operators",
//   "jobId": "job123",
//   "isHighPriority": false
// }
```

## 5. Background Execution & Handling Active Jobs

When drivers are on an active job, they frequently use external navigation apps like Google Maps, placing your app in the background. By default, Android may suspend your app's network state, disconnecting the WebSocket.

**Keeping the WebSocket Alive (Active Jobs Only):**
To ensure real-time messages arrive instantly without relying on FCM fallback, you should run your `ChatSocketManager` inside an **Android Foreground Service** when a job is active.

1. Create a `Service` and start it with `startForeground()`.
2. Provide a persistent notification (e.g., "Active Job in Progress - Listening for messages").
3. Connect the Socket.IO client inside this service.
4. Android will prevent the OS from killing your WebSocket while this Foreground Service is running, ensuring instant message delivery even if Google Maps is taking up the screen.
5. Once the job concludes, stop the Foreground Service so the app gracefully sleeps and saves battery.

## 6. Offline Queuing & Push Notifications (FCM Fallback)

For times when the app is completely killed or there's no active job (so no Foreground Service is running), the WebSocket will disconnect. 

1. When the app `onResume()` triggers after a cold start or background sleep, always execute a GET request to `/api/chat/offline` to pull down any messages they missed while disconnected.
2. Ensure Firebase Cloud Messaging (FCM) is configured. When a driver is offline, the backend will trigger a data-payload push notification. Have your `FirebaseMessagingService` catch this and spawn a local Android Notification waking the user up.
