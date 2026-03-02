# iOS Mobile App - Integration Guide

This guide details how to integrate the iOS mobile application (used by Drivers/Customers) with the Three-Entity Messaging Service Backend.

## 1. Prerequisites

Add the official Socket.IO Swift Client using Swift Package Manager or CocoaPods.

```ruby
# Podfile
use_frameworks!

target 'YourApp' do
    pod 'Socket.IO-Client-Swift', '~> 16.0.0'
    # Optional: Alamofire for REST calls
    pod 'Alamofire', '~> 5.6'
end
```

## 2. Managing the Socket.IO Connection

Create an `ObservableObject` singleton to manage your WebSockets across your iOS SwiftUI/UIKit application.

```swift
import Foundation
import SocketIO

class ChatSocketManager: ObservableObject {
    static let shared = ChatSocketManager()
    
    let manager: SocketManager
    var socket: SocketIOClient
    
    @Published var incomingMessages: [Message] = []

    init() {
        let url = URL(string: "https://your-api-url.com")!
        
        manager = SocketManager(socketURL: url, config: [.log(true), .compress])
        socket = manager.defaultSocket
    }
    
    // Connect user and pass JWT context
    func connect(jwtToken: String) {
        socket.manager?.config = .insert([
            .extraHeaders(["Authorization": "Bearer \(jwtToken)"]),
            .auth(["token": jwtToken]) // Used for SocketIO explicitly
        ])
        
        setupListeners()
        socket.connect()
    }

    func disconnect() {
        socket.disconnect()
    }
    
    // Listeners parsing raw backend structs
    private func setupListeners() {
        socket.on("message:receive") { data, ack in
            guard let payload = data[0] as? [String: Any] else { return }
            
            // Map JSON to your native Swift struct
            // let content = payload["content"] as? String ?? ""
            // let recipientType = payload["recipient_type"] as? String
            // if recipientType == "all_drivers" { // It's a broadcast message! }
            
            DispatchQueue.main.async {
                // Update UI State
                // self.incomingMessages.append(Message(id: ..., text: content))
            }
        }
        
        socket.on("message:deleted") { data, ack in
            guard let payload = data[0] as? [String: Any],
                  let messageId = payload["message_id"] as? String else { return }
            
            DispatchQueue.main.async {
                // Remove messageId from self.incomingMessages instantly
            }
        }
        
        socket.on("typing:status") { data, ack in
            guard let payload = data[0] as? [String: Any],
                  let isTyping = payload["isTyping"] as? Bool else { return }
            
            DispatchQueue.main.async {
                // Toggle "Operator is typing..." banner
            }
        }
    }
    
    // Emit your own typing status
    func sendTypingStatus(room: String, isTyping: Bool) {
        let event = isTyping ? "typing:start" : "typing:stop"
        socket.emit(event, ["room": room])
    }
}
```

## 3. Sending Messages via REST

Unlike pure WebSocket apps, messages strictly funnel through Express REST routes to correctly handle Redis queue insertions and database durability when users toggle online/offline on mobile networks.

```swift
import Foundation

// Example using URLSession
func sendMessage(token: String, driverId: String, content: String, jobId: String) {
    let url = URL(string: "https://your-api-url.com/api/chat/driver/\(driverId)/send")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")

    let body: [String: Any] = [
        "content": content,
        "messageType": "text",
        "recipientType": "operators",
        "jobId": jobId,
        "isHighPriority": false
    ]
    
    request.httpBody = try? JSONSerialization.data(withJSONObject: body)

    URLSession.shared.dataTask(with: request) { data, response, error in
        // Message sent! It will echo natively through WebSockets.
    }.resume()
}

// Example fetching Chat History
func fetchRecentChats(token: String) {
    let url = URL(string: "https://your-api-url.com/api/history/recent")!
    var request = URLRequest(url: url)
    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")

    URLSession.shared.dataTask(with: request) { data, response, error in
        // Decode Array of Conversation structs
    }.resume()
}
```

## 4. Background Handling During Active Jobs

When a driver is on an active job, they are frequently using Apple Maps or Google Maps, placing your app in the background. iOS aggressively kills WebSockets for backgrounded apps to preserve battery. 

**Strategy for Active Jobs:**
If your app tracks the driver's location during an active job, you can use the **Location Background Mode** (`UIBackgroundModes: location`). 
While receiving continuous location updates (`allowsBackgroundLocationUpdates = true`), iOS will give your app background execution time. During this time, your Socket.IO connection *can* remain active, allowing instant message delivery.

However, do not rely on this exclusively. Always implement the APNs fallback below.

## 5. APNs Push Notifications & Background Flow (Fallback)

For times when there is no active job, or if iOS forcefully suspends network activity, the OS will automatically terminate your WebSockets. Our backend circumvents this automatically:

1. When an Operator hits "send", the API realizes your mobile WebSocket is dead.
2. It buffers the message to the Redis `queue:offline` stack.
3. It natively fires an **Apple Push Notification (APNs)** trigger to your device. By sending a payload with `content-available: 1` (a silent push) or a standard alert, you can wake the app.
4. When your iOS app receives `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)`, perform a silent background REST fetch against `/api/chat/offline` to proactively cache missing chats before the user taps the notification banner, or simply show the local notification so they tap to open the app and reconnect.
