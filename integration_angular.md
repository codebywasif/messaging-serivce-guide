# Angular Operator Web App - Integration Guide

This guide details how to integrate the Angular web application (used by Operators) with the Three-Entity Messaging Service Backend.

## 1. Prerequisites & Dependencies

You will need the official Socket.IO client wrapper for Angular:
```bash
npm install socket.io-client ngx-socket-io
```

## 2. Setting up the Socket.IO Service

Configure the `SocketIoModule` in your `app.module.ts`:

```typescript
import { SocketIoModule, SocketIoConfig } from 'ngx-socket-io';

const config: SocketIoConfig = { 
  url: 'http://localhost:3001', // Replace with production URL
  options: {
    autoConnect: false // We connect manually after obtaining the JWT
  } 
};

@NgModule({
  imports: [
    SocketIoModule.forRoot(config),
    // ...
  ]
})
export class AppModule { }
```

## 3. Communication Service Implementation

Create a dedicated `ChatService` to manage REST and WebSocket events.

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Socket } from 'ngx-socket-io';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ChatService {
  private apiUrl = 'http://localhost:3001/api';

  constructor(private socket: Socket, private http: HttpClient) {}

  // --- 1. Connection & Authentication ---
  connectUser(jwtToken: string) {
    this.socket.ioSocket.io.opts.auth = { token: jwtToken };
    this.socket.connect();
  }

  disconnectUser() {
    this.socket.disconnect();
  }

  // --- 2. Real-Time Events (WebSockets) ---
  
  // Listen for incoming messages from Drivers/Customers or Sent-Sync from other Operators
  onMessageReceived(): Observable<any> {
    return this.socket.fromEvent('message:receive');
  }

  // Listen for cross-operator read synchronizations (to clear unread badges)
  onReadAckSync(): Observable<any> {
    return this.socket.fromEvent('read_ack_sync');
  }

  // Listen for typing indicators
  onTypingStatus(): Observable<any> {
    return this.socket.fromEvent('typing:status');
  }

  // Join a specific job room to listen to its scoped messages
  joinJobRoom(jobId: string) {
    this.socket.emit('room:join', jobId);
  }

  // Emit typing indicators
  emitTypingStart(room: string) {
    this.socket.emit('typing:start', { room });
  }

  emitTypingStop(room: string) {
    this.socket.emit('typing:stop', { room });
  }


  // --- 3. REST API Actions ---

  // Send a message
  sendMessage(operatorId: string, content: string, jobId?: string, recipientId?: string, isHighPriority = false) {
    const payload = {
      content,
      messageType: 'text',
      jobId, // Provide if replying inside a specific job room
      recipientType: recipientId ? 'driver' : 'all',
      recipientId,
      isHighPriority
    };
    return this.http.post(`${this.apiUrl}/chat/operator/${operatorId}/send`, payload);
  }

  // Get specific conversation history
  getConversationMessages(conversationId: string) {
    return this.http.get(`${this.apiUrl}/history/conversation/${conversationId}/messages`);
  }

  // Get chat history for a specific job
  getJobHistory(jobId: string) {
    return this.http.get(`${this.apiUrl}/history/job/${jobId}`);
  }

  // Fetch all recent offline messages when the operator logs in
  getOfflineMessages() {
    return this.http.get(`${this.apiUrl}/chat/offline`);
  }

  // --- 4. Operator Specific Administration ---

  // Get all currently online drivers for the operator's company
  getOnlineDrivers() {
    return this.http.get(`${this.apiUrl}/users/online-drivers`);
  }

  // Broadcast a message (Target: 'all_drivers', 'online_drivers', 'all_operators')
  sendBroadcast(content: string, target: string) {
    return this.http.post(`${this.apiUrl}/chat/broadcast`, {
      content,
      target,
      messageType: 'text'
    });
  }

  // Delete a specific message across all clients
  deleteMessage(messageId: string) {
    return this.http.delete(`${this.apiUrl}/chat/message/${messageId}`);
  }

  // Delete an entire conversation across all clients
  deleteConversation(conversationId: string) {
    return this.http.delete(`${this.apiUrl}/chat/conversation/${conversationId}`);
  }

  // Block a maliciously behaving user
  blockUser(userId: string) {
    return this.http.post(`${this.apiUrl}/users/block/${userId}`, {});
  }
}
```

## 4. Operator Entity Sync

Because Operators act as a *single entity*, your Angular UI must be reactive. 
When `onMessageReceived()` fires, **check if the `sender_id` belongs to a different operator**. If it does, render it on the right-side of the chat bubble (as an outgoing message) rather than the left side, so that the Operator knows their colleague replied to the driver. When `onReadAckSync()` fires, find the message in your local state and mark `isRead: true`.

* New features added recently also emit via socket when utilized by an operator:
* Listen for `message:deleted` and `conversation:deleted` socket events. When they emit, instantly remove them from your Angular frontend view.
