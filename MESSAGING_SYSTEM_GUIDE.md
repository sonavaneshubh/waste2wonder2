# Real-Time Messaging & Communication Platform - Setup Guide

## Overview
This guide covers the full-stack real-time messaging system with online status tracking and typing indicators for the Waste2Wonder community platform.

## Features Implemented

### 1. **Real-Time Direct Messaging (1-to-1)**
- Send and receive messages in real-time using Supabase Realtime
- Message history persistence in the database
- View unread message count
- Mark messages as read

### 2. **Online/Offline Status**
- Track user online/offline status in real-time
- User presence automatically updated when they log in/out
- Visual indicators (green dot) for online users
- Last seen timestamp for offline users

### 3. **Typing Indicators**
- See when someone is typing ("user is typing...")
- Auto-dismiss after 3 seconds of inactivity
- Real-time updates via Supabase

### 4. **Conversation List**
- View all users you've messaged with
- Search conversations by name or email
- Display last message snippet
- Online status for each user

## Database Schema

### Messages Table
```sql
CREATE TABLE messages (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  sender_id uuid REFERENCES profiles(id) ON DELETE CASCADE,
  receiver_id uuid REFERENCES profiles(id) ON DELETE CASCADE,
  content text NOT NULL,
  is_read boolean DEFAULT false,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);
```

### User Presence Table
```sql
CREATE TABLE user_presence (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES profiles(id) ON DELETE CASCADE UNIQUE,
  is_online boolean DEFAULT false,
  last_seen timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);
```

### Typing Indicators Table
```sql
CREATE TABLE typing_indicators (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES profiles(id) ON DELETE CASCADE,
  receiver_id uuid REFERENCES profiles(id) ON DELETE CASCADE,
  expires_at timestamptz NOT NULL,
  created_at timestamptz DEFAULT now()
);
```

## Architecture

### Components

#### 1. **MessagingContext** (`src/contexts/MessagingContext.tsx`)
- Manages all messaging state and logic
- Handles Supabase subscriptions for real-time updates
- Provides hooks for components:
  - `useMessaging()` - Access messaging functionality

**State Management:**
- `messages[]` - Current conversation messages
- `userStatuses` - Map of user online/offline status
- `typingUsers` - Set of users currently typing
- `currentConversation` - ID of active conversation partner
- `conversationUsers[]` - List of all users in conversations

**Key Functions:**
```typescript
sendMessage(receiverId: string, content: string) - Send a message
fetchMessages(conversationPartner: string) - Load message history
markAsRead(messageId: string) - Mark message as read
setUserOnline(isOnline: boolean) - Update user presence
sendTypingIndicator(receiverId: string) - Send typing indicator
fetchConversationUsers() - Get all conversation partners
```

#### 2. **DirectMessages Component** (`src/components/DirectMessages.tsx`)
- Modal-based chat interface
- Two-panel layout:
  - **Left panel**: Conversation list with search
  - **Right panel**: Active chat/message view
- Features:
  - Real-time message display
  - Auto-scroll to latest message
  - Typing indicator animation
  - Message timestamps
  - Online status indicators

#### 3. **Navbar Integration** (`src/components/Navbar.tsx`)
- Added Mail/MessageCircle icon button
- Opens DirectMessages modal on click
- Accessible to all authenticated users

#### 4. **App Provider Wrapper** (`src/App.tsx`)
- MessagingProvider wraps the entire app
- Ensures messaging context available to all routes
- Initialized after AuthProvider for user context access

## Real-Time Subscriptions

### Message Channel
```typescript
channel(`messages:${currentUserId}:${partnerId}`)
  .on('postgres_changes', 
    event: INSERT/DELETE on messages table,
    filter: conversation participants
  )
```

### Presence Channel
```typescript
channel('user-presence')
  .on('postgres_changes',
    event: UPDATE on user_presence table,
    listens for all users' online/offline changes
  )
```

### Typing Channel
```typescript
channel(`typing:${currentUserId}:${partnerId}`)
  .on('postgres_changes',
    event: INSERT/DELETE on typing_indicators,
    filter: conversation-specific
  )
```

## Security

### Row-Level Security (RLS) Policies

**Messages:**
- Users can view messages they sent or received
- Users can insert messages they're sending
- Users can update (mark as read) messages they received
- Users can delete messages they sent

**User Presence:**
- All authenticated users can view all presence data
- Users can only update their own presence record

**Typing Indicators:**
- Users can view typing indicators for their conversations
- Users can insert and delete their own typing indicators

## Setup Instructions

### 1. **Run Database Migration**
```bash
# The migration file creates all necessary tables and security policies
# File: supabase/migrations/20250331000000_add_messaging_system.sql
# This should be applied automatically when you push to Supabase
supabase push
```

### 2. **Environment Variables**
Make sure your `.env.local` includes:
```
VITE_SUPABASE_URL=your_supabase_url
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
```

### 3. **Provider Wrapping**
The MessagingProvider is already wrapped in `App.tsx`:
```typescript
<AuthProvider>
  <NotificationProvider>
    <MessagingProvider>  {/* ← Wraps all other providers */}
      <ChatProvider>
        {/* ... other providers ... */}
      </ChatProvider>
    </MessagingProvider>
  </NotificationProvider>
</AuthProvider>
```

### 4. **Component Integration**
DirectMessages is already integrated into the Navbar:
```typescript
<button onClick={() => setIsDirectMessagesOpen(true)}>
  <Mail size={20} />
</button>
<DirectMessages 
  isOpen={isDirectMessagesOpen} 
  onClose={() => setIsDirectMessagesOpen(false)} 
/>
```

## Usage

### For Users
1. Click the **Mail** icon in the navbar
2. See all your conversations in the left sidebar
3. Click a user to open their chat
4. Type and send messages in real-time
5. See typing indicators when they're composing
6. View online status (green dot)
7. Search conversations by name/email

### For Developers

#### Access Messaging in Components
```typescript
import { useMessaging } from '../contexts/MessagingContext';

function MyComponent() {
  const {
    messages,
    currentConversation,
    sendMessage,
    userStatuses,
    typingUsers
  } = useMessaging();

  const handleSend = async () => {
    await sendMessage(userId, messageText);
  };

  // Use the data...
}
```

#### Send a Message
```typescript
await sendMessage('user-uuid', 'Hello there!');
```

#### Check User Online Status
```typescript
const user = userStatuses.get('user-uuid');
if (user?.is_online) {
  console.log('User is online');
}
```

#### Fetch Conversation History
```typescript
await fetchMessages('partner-user-uuid');
// Messages loaded into state
```

## Performance Optimizations

1. **Message Pagination** (can be added)
   - Currently loads all messages
   - Can implement pagination for performance

2. **Typing Indicator Cleanup**
   - Auto-deletes expired indicators (3-second timeout)
   - Prevents stale typing indicators

3. **Selective Subscriptions**
   - Subscribe to specific conversation channels
   - Unsubscribe when closing conversations
   - Prevents unnecessary data transfer

4. **Indexed Queries**
   - Messages table has indexes on sender_id, receiver_id
   - Conversation lookup indexes for fast retrieval
   - Presence indexes for online/offline status queries

## Troubleshooting

### Messages Not Appearing
1. Check Supabase RLS policies are correct
2. Verify both users are authenticated
3. Check database migration was applied
4. Check browser console for errors

### Online Status Not Updating
1. Verify user_presence table exists
2. Check user is properly authenticated
3. Ensure setUserOnline is called on mount/unmount
4. Check network connection

### Typing Indicator Not Working
1. Verify typing_indicators table exists
2. Check 3-second timeout is configured
3. Ensure sendTypingIndicator is called on input change
4. Check RLS policies allow insert/delete

### Realtime Not Working
1. Enable Realtime in Supabase Project Settings
2. Verify it's enabled for the specific tables
3. Check browser WebSocket connection
4. Review Supabase logs for errors

## Future Enhancements

1. **Message Threading**
   - Reply to specific messages
   - Thread views for conversations

2. **File Sharing**
   - Send images/documents
   - File preview thumbnails

3. **Group Chat**
   - Extend 1-to-1 to multi-participant
   - Group presence/typing

4. **Message Reactions**
   - Emoji reactions to messages
   - Message edit/delete history

5. **Media Support**
   - Video/voice messages
   - Real-time call integration

6. **Message Search**
   - Full-text search across messages
   - Search by date, user, keywords

## API Endpoints (for Reference)

If you want to add backend Express routes for messaging:

```javascript
// POST /api/messages - Send message
POST /api/messages
{
  "receiver_id": "uuid",
  "content": "message text"
}
Response: { "id": "uuid", "created_at": timestamp }

// GET /api/messages/:conversationId - Fetch conversation
GET /api/messages/partner-uuid
Response: [{ message objects }]

// PUT /api/messages/:messageId/read - Mark as read
PUT /api/messages/message-uuid/read
Response: { "id": "uuid", "is_read": true }

// DELETE /api/messages/:messageId - Delete message
DELETE /api/messages/message-uuid
Response: { "success": true }

// GET /api/presence/:userId - Check online status
GET /api/presence/user-uuid
Response: { "is_online": true, "last_seen": timestamp }
```

## File Structure
```
src/
├── contexts/
│   ├── MessagingContext.tsx       ← Messaging logic & state
│   ├── AuthContext.tsx            (unchanged)
│   └── ...
├── components/
│   ├── DirectMessages.tsx         ← Chat UI
│   ├── Navbar.tsx                 (updated with DM button)
│   └── ...
├── lib/
│   └── supabase.ts                (updated with types)
└── App.tsx                        (updated with provider)

supabase/
└── migrations/
    └── 20250331000000_add_messaging_system.sql
```

---

**Last Updated:** March 31, 2026
**Version:** 1.0.0
**Status:** Production Ready
