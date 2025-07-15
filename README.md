# Volunteer Chat and Notification Pipeline Documentation
## Better Together Feature - Complete System Architecture

### Table of Contents
1. [System Overview](#system-overview)
2. [Database Collections](#database-collections)
3. [Core Components](#core-components)
4. [Chat Session Lifecycle](#chat-session-lifecycle)
5. [Volunteer Allocation Algorithm](#volunteer-allocation-algorithm)
6. [Notification System](#notification-system)
7. [Real-time Communication](#real-time-communication)
8. [Scheduled Tasks](#scheduled-tasks)
9. [Error Handling](#error-handling)
10. [API Endpoints](#api-endpoints)
11. [Socket Events](#socket-events)
12. [Deployment Considerations](#deployment-considerations)

---

## System Overview

The "Better Together" feature is a peer support system that connects users seeking mental health support with trained volunteers. The system operates through a real-time chat interface with comprehensive notification management and automated session management.

### Key Features
- **Unlimited Chat Access**: Users can start unlimited chat sessions (recently updated from limited access)
- **6-Hour Time Window**: Chat sessions have a 6-hour window for user inactivity
- **Real-time Messaging**: Socket.IO-based real-time communication
- **Push Notifications**: Firebase Cloud Messaging (FCM) for notifications
- **Automated Session Management**: Scheduled tasks for session cleanup and reminders
- **Volunteer Availability Management**: Dynamic volunteer allocation and availability tracking

---

## Database Collections

### Core Collections

#### 1. **sessions_pending**
Stores chat requests waiting for volunteer acceptance
```javascript
{
  "_id": ObjectId,
  "user": "user@email.com",
  "volunteer": "", // Empty until accepted
  "message": "I need help with...",
  "start_time": 0,
  "request_time": ISODate,
  "accept_time": 0,
  "last_opened_user": 0,
  "last_opened_volunteer": 0,
  "chat_tags": [],
  "isResolved": false,
  "requested_volunteers": [volunteer_objects],
  "ended_by_user": false,
  "ended_by_vol": false,
  "ended_by_system": false,
  "conversation_type": "better_together",
  "dozy_chatjob_id": "user@email.com2024-01-01T10:00:00",
  "end_chatjob_id": "",
  "flag_reason": ""
}
```

#### 2. **sessions_started**
Active chat sessions between users and volunteers
```javascript
{
  "_id": ObjectId,
  "user": "user@email.com",
  "volunteer": "volunteer@nabi.health",
  "message": "I need help with...",
  "start_time": ISODate, // Set when both parties open chat
  "request_time": ISODate,
  "accept_time": ISODate,
  "last_opened_user": ISODate,
  "last_opened_volunteer": ISODate,
  "chat_tags": [],
  "isResolved": false,
  "requested_volunteers": [volunteer_objects],
  "ended_by_user": false,
  "ended_by_vol": false,
  "ended_by_system": false,
  "conversation_type": "better_together",
  "dozy_chatjob_id": "user@email.com2024-01-01T10:00:00",
  "end_chatjob_id": "user@email.com2024-01-01T16:00:00",
  "inactive_chat_job_id": "inactive_user@email.com_volunteer@nabi.health_2024-01-01T10:00:00",
  "reminder_job_id": "reminder_user@email.com_volunteer@nabi.health_2024-01-01T10:00:00",
  "last_activity_time": ISODate,
  "flag_reason": ""
}
```

#### 3. **sessions_ended**
Completed chat sessions
```javascript
{
  "_id": ObjectId,
  "user": "user@email.com",
  "volunteer": "volunteer@nabi.health",
  "message": "I need help with...",
  "start_time": ISODate,
  "request_time": ISODate,
  "accept_time": ISODate,
  "end_time": ISODate,
  "chat_tags": [],
  "isResolved": true,
  "ended_by_user": false,
  "ended_by_vol": false,
  "ended_by_system": true,
  "conversation_type": "better_together",
  "flag_reason": ""
}
```

#### 4. **messages**
Individual chat messages
```javascript
{
  "_id": ObjectId,
  "chat_id": "session_object_id",
  "text_message": "Hello, how can I help you?",
  "sender_mail": "volunteer@nabi.health",
  "timestamp": "2024-01-01T10:00:00.000Z",
  "reply_message": null // For reply functionality
}
```

#### 5. **users**
User profiles and authentication
```javascript
{
  "_id": ObjectId,
  "email": "user@email.com",
  "name": "User Name",
  "fcmtoken": "firebase_fcm_token",
  "in_chat_session": false,
  "role": ["user"],
  "expires_on": ISODate
}
```

#### 6. **volunteers**
Volunteer profiles and availability
```javascript
{
  "_id": ObjectId,
  "email": "volunteer@nabi.health",
  "name": "Volunteer Name",
  "available": true,
  "connected_to_users_right_now": 0,
  "time_spent_volunteering": 0,
  "role": "volunteer",
  "isLead": false,
  "isActive": true,
  "rating": 0,
  "no_of_reviews": 0
}
```

#### 7. **notifications_with_fcm**
Notification delivery tracking
```javascript
{
  "_id": ObjectId,
  "tokens": ["fcm_token_1", "fcm_token_2"],
  "sender_mail": "sender@email.com",
  "chat_id": "chat_object_id",
  "title": "Notification Title",
  "body": "Notification message",
  "content.channelKey": "msg_ch",
  "screen": "volunteer_dashboard",
  "timeStamp": "2024-01-01T10:00:00.000Z",
  "status": "sent"
}
```

---

## Core Components

### 1. **Flask Application (app.py)**
- **Entry Point**: Main application initialization
- **Database Connection**: MongoDB setup with collections
- **Socket.IO**: Real-time communication setup
- **Firebase Admin**: Push notification initialization
- **Scheduler**: APScheduler for automated tasks
- **Notification Channels**: Channel definitions for different notification types

### 2. **Socket Controller (socket_controller.py)**
- **Real-time Events**: Handles all Socket.IO events
- **Chat Session Management**: Start, accept, and end chat sessions
- **Message Handling**: Real-time message delivery
- **Room Management**: User and volunteer room assignments

### 3. **Chat Session Controller (chat_session_controller.py)**
- **Session Lifecycle**: Manages chat session states
- **Scheduled Tasks**: Handles timeouts and reminders
- **Notification Sending**: Push notification delivery
- **Session Cleanup**: Automated session termination

### 4. **Volunteer Controller (volunteer_controller.py)**
- **Volunteer Allocation**: Dynamic volunteer assignment
- **Availability Management**: Tracks volunteer availability
- **Feedback System**: Volunteer feedback collection
- **Lead Notifications**: Volunteer lead notifications

### 5. **Push Notification System (utils/push_notification.py)**
- **FCM Integration**: Firebase Cloud Messaging
- **Token Management**: FCM token validation and refresh
- **Notification Delivery**: Multi-platform notification sending
- **Error Handling**: Failed notification tracking

---

## Chat Session Lifecycle

### Phase 1: Chat Request Initiation
```mermaid
sequenceDiagram
    participant U as User (Flutter App)
    participant B as Backend
    participant DB as MongoDB
    participant S as Scheduler
    participant V as Volunteers

    U->>B: POST /start_chat_session
    B->>DB: Find available volunteers
    B->>DB: Create session in sessions_pending
    B->>S: Schedule dozy job (24 hours)
    B->>V: Emit 'chat_request' to 15 volunteers
    B->>V: Send push notification
    B->>U: Return success response
```

**Code Flow:**
1. **User initiates chat** via `POST /start_chat_session`
2. **Backend finds available volunteers** (max 15, randomly shuffled)
3. **Creates pending session** in `sessions_pending` collection
4. **Schedules dozy job** for 24-hour timeout
5. **Emits socket events** to selected volunteers
6. **Sends push notifications** to volunteers

### Phase 2: Volunteer Acceptance
```mermaid
sequenceDiagram
    participant V as Volunteer (Flutter App)
    participant B as Backend
    participant DB as MongoDB
    participant S as Scheduler
    participant U as User

    V->>B: POST /accept_chat_request
    B->>DB: Move session to sessions_started
    B->>DB: Update volunteer availability
    B->>S: Schedule 6-hour inactivity job
    B->>S: Schedule 5-minute reminder
    B->>U: Send acceptance notification
    B->>V: Emit 'chat_started' event
    B->>U: Emit 'chat_started' event
```

**Code Flow:**
1. **Volunteer accepts chat** via `POST /accept_chat_request`
2. **Session moves to started** from pending
3. **Volunteer availability updated** (connected_to_users_right_now +1)
4. **Schedules inactivity job** (6 hours)
5. **Schedules reminder job** (5 minutes)
6. **Sends notifications** to both parties
7. **Emits socket events** for real-time updates

### Phase 3: Active Chat Session
```mermaid
sequenceDiagram
    participant U as User
    participant B as Backend
    participant DB as MongoDB
    participant V as Volunteer

    U->>B: Socket 'chat_message'
    B->>DB: Store message in messages collection
    B->>DB: Update last_activity_time
    B->>V: Emit 'chat_message' to volunteer
    B->>V: Send push notification
    V->>B: Socket 'chat_message'
    B->>DB: Store message
    B->>U: Emit 'chat_message' to user
    B->>U: Send push notification
```

**Code Flow:**
1. **Messages sent via Socket.IO** `chat_message` event
2. **Messages stored** in `messages` collection
3. **Activity time updated** to reset inactivity timer
4. **Real-time delivery** via Socket.IO
5. **Push notifications** sent to offline recipients

### Phase 4: Session Termination
```mermaid
sequenceDiagram
    participant S as Scheduler
    participant B as Backend
    participant DB as MongoDB
    participant U as User
    participant V as Volunteer

    S->>B: 6-hour inactivity timeout
    B->>DB: Move session to sessions_ended
    B->>DB: Update volunteer availability
    B->>V: Send thank you notification
    B->>U: Send session ended notification
    B->>V: Emit 'chat_ended' socket event
    B->>U: Emit 'chat_ended' socket event
```

**Code Flow:**
1. **Scheduler triggers** after 6 hours of inactivity
2. **Session moved to ended** collection
3. **Volunteer availability reset** (connected_to_users_right_now -1)
4. **Thank you notification** sent to volunteer
5. **Session ended notifications** sent to both parties
6. **Socket events emitted** for real-time updates

---

## Volunteer Allocation Algorithm

### Primary Allocation Strategy
```python
# Find available volunteers
available_vols = volunteers.find({
    'available': True, 
    'email': {'$ne': user_email}
})

# Random shuffle for fair distribution
random.shuffle(available_vols_list)

# Select up to 15 volunteers
list_length = min(15, len(available_vols_list))
selected_volunteers = available_vols_list[:list_length]
```

### Fallback Strategy
If no volunteers with zero connections available:
1. **Find volunteers with 1 connection**
2. **Randomly sample up to 10 volunteers**
3. **Apply same notification and socket emission logic**

### Re-allocation Strategy
```python
def allocate_new_vols():
    # Find sessions pending for more than 10 minutes
    sessions = sessions_pending.find({
        "request_time": {"$lte": (datetime.now() - timedelta(minutes=10))}
    })
    
    for session in sessions:
        # Find new volunteers not previously requested
        don_t_request_these = listify_requested_vols(session['requested_volunteers'])
        available_vols = volunteers.find({
            'available': True, 
            'email': {'$nin': don_t_request_these}
        })
        
        # Send new requests to up to 15 volunteers
        for vol in available_vols_list[:15]:
            socketio.emit('chat_request', {
                'user_email': session['user'], 
                'i_need_message': session['message']
            }, room=vol['email'])
```

---

## Notification System

### Notification Channels
```python
notification_channel = {
    "misc_channel": "misc_ch",
    "msg_channel": "msg_ch",
    "vol_channel": "volmsg_ch",
    "voldashboard_newrequest_channel": "vol_newreq_ch",
    "application_channel": "application_channel",
    "vollead_channel": "vol_lead_ch"
}
```

### Notification Types

#### 1. **Chat Request Notifications**
- **Recipients**: Available volunteers
- **Channel**: `voldashboard_newrequest_channel`
- **Message**: "Looks like someone wants to connect!"
- **Screen**: "volunteer_dashboard"
- **Priority**: High

#### 2. **Chat Acceptance Notifications**
- **Recipients**: User
- **Channel**: `msg_channel`
- **Message**: "Looks like someone's there to talk!"
- **Priority**: High

#### 3. **Message Notifications**
- **Recipients**: Chat participants
- **Channel**: `msg_channel` (users) / `vol_channel` (volunteers)
- **Message**: "You received a new message"
- **Priority**: High

#### 4. **Session End Notifications**
- **Recipients**: Both parties
- **Channel**: Respective channels
- **Message**: Session ended notifications
- **Priority**: Medium

#### 5. **Thank You Notifications**
- **Recipients**: Volunteers
- **Channel**: `vol_channel`
- **Message**: "Thank you for making a difference in someone's life"
- **Priority**: Medium

### FCM Token Management
```python
def send_fcmandname_for_email(email):
    target_user = users.find_one({"email": email})
    if not target_user:
        return {"email": "", "name": ""}
    
    fcm_token = target_user.get('fcmtoken', '')
    return {
        "email": fcm_token,
        "name": target_user.get('first_name', '')
    }
```

### Token Validation
```python
def validate_fcm_token(token, user_email):
    try:
        message = messaging.Message(
            data={"type": "token_validation"},
            token=token
        )
        response = messaging.send(message, dry_run=True)
        return True
    except messaging.UnregisteredError:
        users.update_one(
            {"email": user_email},
            {"$set": {"fcmtoken_valid": False}}
        )
        return False
```

---

## Real-time Communication

### Socket.IO Events

#### **Connection Events**
```javascript
// Client connects
socket.on('connect', () => {
    console.log('Connected to server');
});

// Join user room
socket.emit('join', user_email);
```

#### **Chat Request Events**
```javascript
// Volunteer receives chat request
socket.on('chat_request', (data) => {
    // data: {user_email, i_need_message}
    // Show notification in volunteer dashboard
});

// Cancel chat request (when another volunteer accepts)
socket.on('cancel_chat_request', (data) => {
    // data: {user_email}
    // Remove from volunteer dashboard
});
```

#### **Chat Session Events**
```javascript
// Chat started (both parties)
socket.on('chat_started', (data) => {
    // data: {user_email, volunteer_email, chat_id}
    // Navigate to chat screen
});

// Real-time messages
socket.on('chat_message', (data) => {
    // data: {sender_mail, chat_id, message, timestamp, reply_message}
    // Display message in chat
});

// Chat ended
socket.on('chat_ended', (data) => {
    // Navigate back to dashboard
});

// Thank you notification
socket.on('thank_you_volunteer', (data) => {
    // data: {message, title}
    // Show thank you message
});
```

#### **Activity Tracking Events**
```javascript
// Update user activity
socket.emit('update_last_opened_user', {chat_id});

// Update volunteer activity
socket.emit('update_last_opened_volunteer', {chat_id});

// Start timer (when both parties open chat)
socket.on('start_timer', () => {
    // Start 40-minute session timer
});
```

### Room Management
```python
# Users join rooms with their email
@socketio.on('join')
def handle_join_initialroom(uid):
    join_room(uid)

# Messages sent to specific rooms
socketio.emit('chat_message', data, room=recipient_email)
```

---

## Scheduled Tasks

### 1. **FCM Token Validation**
```python
@scheduler.task('cron', id='validate_fcm_tokens', hour=3, minute=0)
def scheduled_token_validation():
    invalid_count = refresh_volunteer_tokens()
    print(f"Token validation complete. Found {invalid_count} invalid tokens.")
```

### 2. **Volunteer Re-allocation**
```python
def allocate_new_vols():
    # Runs every 10 minutes
    sessions = sessions_pending.find({
        "request_time": {"$lte": (datetime.now() - timedelta(minutes=10))}
    })
    # Allocate new volunteers for pending sessions
```

### 3. **Session Timeouts**

#### **Dozy Session (24 hours)**
```python
def mark_chat_session_dozy(user_id, job_id, message):
    # Move pending sessions to dozy after 24 hours
    sessions_dozy.insert_one(session)
    sessions_pending.delete_one({'_id': session['_id']})
```

#### **Inactive Chat (6 hours)**
```python
def release_inactive_chat(chat_id, user_id, volunteer_id, job_id):
    # End chat after 6 hours of inactivity
    # Send notifications to both parties
    # Move session to ended
```

### 4. **User Reminders**
```python
def send_user_reminder(chat_id, user_id, volunteer_id, job_id):
    # Send reminder 5 minutes after volunteer accepts
    # Personalized message encouraging user to respond
```

### 5. **Follow-up Reminders**
```python
def send_followup_reminder_1(chat_id, user_id, volunteer_id, job_id):
    # First follow-up after 6 hours
    
def send_followup_reminder_2(chat_id, user_id, volunteer_id, job_id):
    # Second follow-up after 4 hours
    
def send_expiry_warning(chat_id, user_id, volunteer_id, job_id):
    # Warning 1 hour before expiry
```

---

## Error Handling

### 1. **FCM Token Errors**
```python
def send_push_notification_without_object(registration_token, message, title, ...):
    try:
        response = messaging.send_each_for_multicast(message)
        success_count = response.success_count
        failure_count = response.failure_count
        
        # Log failures
        if failure_count > 0:
            for idx, result in enumerate(response.responses):
                if not result.success:
                    error = result.exception
                    print(f"Failed to send to token {idx}: {error}")
                    
    except Exception as e:
        print(f"Error sending notification: {str(e)}")
        # Log error in notifications_with_fcm collection
```

### 2. **Socket Event Errors**
```python
try:
    socketio.emit('chat_started', data, room=volunteer_email)
except Exception as e:
    print(f"Error emitting chat started to volunteer: {str(e)}")
```

### 3. **Database Connection Errors**
```python
try:
    session = sessions_started.find_one({'_id': ObjectId(chat_id)})
    if session:
        # Process session
    else:
        return abort(404, description='Session not found')
except Exception as e:
    print(f"Database error: {str(e)}")
    return abort(500, description='Database error')
```

### 4. **Scheduler Job Errors**
```python
try:
    requests.post(schedulerURL + '/AddRemoveJob',
                  headers={'Content-type': "application/json"},
                  json=dataTosendToscheduler)
except Exception as e:
    print(f"Error scheduling job: {str(e)}")
```

---

## API Endpoints

### 1. **Start Chat Session**
```http
POST /start_chat_session
Content-Type: application/json

{
  "user_email": "user@email.com",
  "i_need_message": "I need help with anxiety",
  "conversation_type": "better_together"
}

Response: 201 Created
{
  "message": "Request Successful"
}
```

### 2. **Accept Chat Request**
```http
POST /accept_chat_request
Content-Type: application/json

{
  "volunteer_email": "volunteer@nabi.health",
  "user_email": "user@email.com"
}

Response: 201 Created
{
  "message": "Chat accepted!"
}
```

### 3. **Test Volunteer Notification**
```http
POST /test_volunteer_notification
Content-Type: application/json

{
  "email": "volunteer@nabi.health"
}

Response: 200 OK
{
  "success": true,
  "message": "Test notification sent",
  "token": "fcm_token"
}
```

---

## Socket Events

### **Client to Server Events**
```javascript
// Join user room
socket.emit('join', user_email);

// Send chat message
socket.emit('chat_message', {
    chat_id: "session_id",
    sender_mail: "sender@email.com",
    message: "Hello!",
    reciever_mail: "recipient@email.com",
    reply_message: null
});

// Update user activity
socket.emit('update_last_opened_user', {chat_id});

// Update volunteer activity
socket.emit('update_last_opened_volunteer', {chat_id});

// End chat with consent
socket.emit('chat_ended', {
    email: "user@email.com",
    by_user: true,
    chat_id: "session_id"
});
```

### **Server to Client Events**
```javascript
// Chat request received
socket.on('chat_request', (data) => {
    // data: {user_email, i_need_message}
});

// Chat started
socket.on('chat_started', (data) => {
    // data: {user_email, volunteer_email, chat_id}
});

// New message received
socket.on('chat_message', (data) => {
    // data: {sender_mail, chat_id, message, timestamp, reply_message}
});

// Chat ended
socket.on('chat_ended', (data) => {
    // Navigate to dashboard
});

// Thank you notification
socket.on('thank_you_volunteer', (data) => {
    // data: {message, title}
});

// Refresh volunteer dashboard
socket.on('refresh_vol_dash', () => {
    // Refresh dashboard data
});

// Start session timer
socket.on('start_timer', () => {
    // Start 40-minute timer
});

// Cancel chat request
socket.on('cancel_chat_request', (data) => {
    // data: {user_email}
});

// No availability
socket.on('no_availability', () => {
    // Show no volunteers available message
});
```

---

## Deployment Considerations

### 1. **Environment Variables**
```bash
DB_URL_TWO=mongodb+srv://username:password@cluster.mongodb.net/nabi_db
SECRET=your-secret-key
MAIL_USER=volunteer@nabi.health
MAIL_PASS=your-mail-password
```

### 2. **Firebase Configuration**
- **Service Account**: `nabiservice.json` file required
- **FCM Tokens**: Stored in user documents
- **Token Validation**: Daily validation at 3 AM

### 3. **MongoDB Collections**
- **Indexes**: Email fields should be indexed
- **TTL Indexes**: Consider for session cleanup
- **Backup**: Regular backups recommended

### 4. **Scheduler Service**
- **External Scheduler**: `http://sch.nabi.health`
- **Job Management**: Add/remove jobs via HTTP API
- **Error Handling**: Failed jobs logged and retried

### 5. **Socket.IO Configuration**
```python
socketio = SocketIO(app, cors_allowed_origins="*")
```

### 6. **Production Considerations**
- **SSL/TLS**: HTTPS required for production
- **Load Balancing**: Multiple server instances
- **Monitoring**: Log aggregation and monitoring
- **Rate Limiting**: Implement API rate limiting
- **Error Tracking**: Comprehensive error logging

---

## Monitoring and Maintenance

### 1. **Key Metrics to Monitor**
- **Active Sessions**: Number of sessions in `sessions_started`
- **Pending Requests**: Number of sessions in `sessions_pending`
- **Volunteer Availability**: Count of available volunteers
- **Notification Success Rate**: FCM delivery success rate
- **Socket Connections**: Active Socket.IO connections

### 2. **Log Analysis**
```python
# Notification delivery tracking
notifications_with_fcm.insert_one({
    "tokens": registration_tokens,
    "title": title,
    "body": message,
    "content.channelKey": channel_id,
    "timeStamp": datetime.now().isoformat(),
    "status": "sent"
})
```

### 3. **Regular Maintenance Tasks**
- **Token Cleanup**: Remove invalid FCM tokens
- **Session Cleanup**: Archive old sessions
- **Volunteer Availability**: Reset stuck availability flags
- **Database Optimization**: Regular index maintenance

---

## Troubleshooting Guide

### 1. **Volunteers Not Receiving Notifications**
- Check FCM token validity
- Verify volunteer availability status
- Check notification channel configuration
- Review Firebase service account permissions

### 2. **Chat Sessions Not Starting**
- Verify volunteer availability in database
- Check Socket.IO connection status
- Review session creation logic
- Validate user permissions

### 3. **Messages Not Delivering**
- Check Socket.IO room assignments
- Verify message storage in database
- Review real-time event emission
- Check recipient online status

### 4. **Scheduled Tasks Not Running**
- Verify scheduler service connectivity
- Check job creation/removal API
- Review task function implementations
- Monitor scheduler logs

---
