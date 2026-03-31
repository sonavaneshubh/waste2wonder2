# Smart Notification System - Quick Start Guide

## 5-Minute Setup

### Step 1: Install Dependencies (1 min)

**Backend:**
```bash
cd waste-2-wonder-backend
pip install -r requirements.txt
```

**Frontend:**
```bash
cd waste-2-wonder-frontend
npm install
```

### Step 2: Configure Email (1 min)

Create/update `.env` file in `waste-2-wonder-backend/`:

```env
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USE_TLS=true
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_DEFAULT_SENDER=noreply@waste2wonder.com
```

To get Gmail App Password:
1. Go to https://myaccount.google.com/apppasswords
2. Select Mail → Windows Computer (or your device)
3. Copy the 16-character password
4. Use as MAIL_PASSWORD

### Step 3: Apply Database Migration (2 min)

1. Open Supabase Dashboard → SQL Editor
2. Copy content from: `waste-2-wonder-backend/supabase/migrations/20250330000003_add_notification_system.sql`
3. Paste and execute
4. Verify: Check tables: notifications, notification_preferences, etc.

### Step 4: Start Services (1 min)

**Terminal 1 - Backend:**
```bash
cd waste-2-wonder-backend
python main.py
```

**Terminal 2 - Frontend:**
```bash
cd waste-2-wonder-frontend
npm run dev
```

✅ **Setup Complete!** Go to http://localhost:5173

---

## Quick Test (5 minutes)

### Test 1: In-App Notification (Real-Time)

**Prerequisites:** Two browsers/windows open, logged in as different users

1. **Browser A (User1):** Click notification bell → Should show "No notifications yet"
2. **Browser B (User2):** Go to Community → Create a post
3. **Browser A (User1):** Go to Community → Like the post created by User2
4. **Browser B (User2):** Notification bell shows red badge "1"
5. **Browser B (User2):** Click notification bell → See "User1 liked your post"
6. **Browser B (User2):** Click notification → Changes to read (no longer bold)

**Expected Result:** ✅ Real-time notification appeared instantly

### Test 2: Unread Count Badge

1. **Browser A (User1):** Like 3 posts of User2
2. **Browser B (User2):** Notification bell shows "3" badge
3. **Browser B (User2):** Click "Mark all as read"
4. **Browser B (User2):** Badge disappears

**Expected Result:** ✅ Badge updates correctly

### Test 3: Notification Preferences

1. **Browser B (User2):** Click notification bell → Click gear icon (Preferences)
2. **Turn off:** "Send email when someone likes my post"
3. **Click Save**
4. **Browser A (User1):** Like another post
5. **Check email:** Should NOT receive email (but will get in-app notification)

**Expected Result:** ✅ Email not sent, but in-app notification still shown

### Test 4: Delete Notification

1. **Browser B (User2):** Notification bell → Click trash icon on a notification
2. **Notification is deleted**
3. **Unread count decreases**

**Expected Result:** ✅ Notification removed immediately

### Test 5: View All Notifications

1. **Browser B (User2):** Notification bell → Click "View all notifications →"
2. **See full notifications page** at `/notifications`
3. **Filter by type:** Click "❤️ Likes" to see only like notifications
4. **Pagination:** If >20 notifications, see page navigation

**Expected Result:** ✅ Full page shows all notifications with filters

### Test 6: Comment Notification

1. **Browser A (User1):** Go to User2's post → Write a comment
2. **Browser B (User2):** See notification "User1 commented on your post"
3. **Type:** 💬 (comment icon)

**Expected Result:** ✅ Comment notification received with correct icon

### Test 7: Email Notification

*(Requires gmail configured)*

1. **Browser A (User1):** Comment on User2's post
2. **Check User2's gmail:**
   - Subject: "User1 commented on your post"
   - Email body shows post title and link
   - Click link → Redirects to post

**Expected Result:** ✅ Email received with correct content

### Test 8: Push Notification (Optional)

1. **Browser B (User2):** Go to /notifications → Open Preferences
2. **Toggle "Enable browser push notifications"**
3. **Browser asks for permission** → Click "Allow"
4. **Browser B (User2):** Close the app/website (but leave page open)
5. **Browser A (User1):** Like a post by User2
6. **Browser B:** See native browser notification pop-up

**Expected Result:** ✅ Browser notification appeared

---

## Troubleshooting Quick Fixes

### Issue: No Notifications Appearing

**Fix 1:** Check browser console (F12)
- Look for: "Notification socket connected" message
- If not present: Socket.io not connected

**Fix 2:** Verify you're logged in
- Check header shows your name
- Logged-in status required for notifications

**Fix 3:** Both users logged in?
- Make sure you have 2 different accounts
- Same account won't notify himself

**Fix 4:** Refresh the page
```bash
Ctrl + Shift + R  # Hard refresh to clear cache
```

### Issue: Emails Not Sending

**Fix 1:** Check MAIL_USERNAME in Flask logs
```bash
# Look for message like:
# WARNING: Email notifications disabled: MAIL_USERNAME not configured
```

**Fix 2:** Verify .env file exists in backend folder
```bash
# File should be at: waste-2-wonder-backend/.env
cat waste-2-wonder-backend/.env
# Should show MAIL_USERNAME and MAIL_PASSWORD
```

**Fix 3:** Gmail App Password correct?
- Recheck at https://myaccount.google.com/apppasswords
- Make sure it's 16 characters with spaces removed
- Regular password won't work

**Fix 4:** Check email preferences
- Go to /notifications → Preferences
- Toggle "Send email when someone comments on my post" ON
- Then test again

### Issue: Notification Badge Shows Wrong Number

**Fix 1:** Refresh the page
```bash
Ctrl + R
```

**Fix 2:** Check notification list
- Go to /notifications
- See if actual notifications match badge number
- If not: Click "Mark all as read" to reset

### Issue: Socket.io Timeout

**Error:** "Connect timeout"

**Fix:** Make sure Flask backend is running
```bash
# Terminal 1
cd waste-2-wonder-backend
python main.py
# Should see: WARNING in app.run() [...]
#            * Running on http://127.0.0.1:5000
```

---

## Integration Points

### Where to Add Notification Triggers

When implementing like/comment/follow features, add these calls:

#### In Post Like Handler:
```typescript
import { notificationService } from '../services/notificationService';

const handleLike = async (post) => {
  // Your existing like code
  await likePost(post.id);
  
  // Add this line:
  notificationService.notifyPostLiked(
    post.id,
    post.user_id,
    currentUser.id,
    post.title
  );
};
```

#### In Post Comment Handler:
```typescript
const handleComment = async (post, commentText) => {
  const comment = await createComment(post.id, commentText);
  
  // Add this line:
  notificationService.notifyPostCommented(
    post.id,
    post.user_id,
    currentUser.id,
    comment.id,
    post.title
  );
};
```

#### In Comment Reply Handler:
```typescript
const handleReply = async (comment, replyText) => {
  const reply = await createReply(comment.id, replyText);
  
  // Add this line:
  notificationService.notifyCommentReplied(
    comment.id,
    comment.user_id,
    currentUser.id,
    comment.post_id,
    comment.post.title
  );
};
```

#### In Follow Handler:
```typescript
const handleFollow = async (userId) => {
  await followUser(userId);
  
  // Add this line:
  notificationService.notifyUserFollowed(
    currentUser.id,
    userId
  );
};
```

---

## Files Modified/Created

### Backend
- ✅ `main.py` - Added 5 API endpoints + 4 Socket.io handlers
- ✅ `requirements.txt` - Added Flask-Mail, supabase
- ✅ `supabase/migrations/20250330000003_add_notification_system.sql` - Database schema
- ✅ `.env` - Email configuration

### Frontend
- ✅ `src/contexts/NotificationContext.tsx` - State management
- ✅ `src/components/NotificationBell.tsx` - Bell icon + badge
- ✅ `src/components/NotificationDropdown.tsx` - Notification list
- ✅ `src/components/NotificationPreferences.tsx` - User settings
- ✅ `src/components/NotificationsPage.tsx` - Full notifications page
- ✅ `src/services/notificationService.ts` - Trigger notifications
- ✅ `src/App.tsx` - Added routes + providers
- ✅ `src/components/Navbar.tsx` - Added notification bell

---

## API Summary

### User-Facing Routes
- **GET** `/api/notifications` - Get your notifications
- **PATCH** `/api/notifications/:id/read` - Mark as read
- **DELETE** `/api/notifications/:id` - Delete notification
- **PATCH** `/api/notifications/read-all` - Mark all as read
- **GET** `/api/notifications/count` - Get unread count
- **GET** `/api/notification-preferences` - Get your preferences
- **POST** `/api/notification-preferences` - Update preferences

### Socket.io Events (emit from frontend)
- `post_liked` - Post was liked
- `post_commented` - Post was commented
- `comment_replied` - Comment was replied
- `user_followed` - User was followed

### Socket.io Events (listen on frontend)
- `notification_received` - New notification arrived

---

## Next Steps

### Immediate (Do First)
1. ✅ Apply database migration
2. ✅ Configure email in .env
3. ✅ Start backend and frontend
4. ✅ Run quick tests above
5. ✅ Fix any issues with troubleshooting guide

### Short-Term (Next Week)
1. Integrate notification triggers in Community.tsx
2. Test with real post likes/comments
3. Monitor email delivery
4. Collect user feedback

### Long-Term (Next Month)
1. Add notification grouping (5 people liked this)
2. Implement digest notifications
3. Add more email templates
4. Set up monitoring/alerts

---

## Support & Questions

### Common Questions

**Q: Do I need to pay for email?**
A: No, Gmail is free. Just enable 2FA and create an App Password.

**Q: Will notifications work on mobile?**
A: Yes! In-app works everywhere. Push notifications need HTTPS (production only).

**Q: Do deleted notifications count against unread?**
A: No, only unread notifications are counted.

**Q: Can users opt-out completely?**
A: Yes, set digest_frequency to 'off' and toggle all notification types.

**Q: Does spamming likes spam notifications?**
A: No, duplicate notifications blocked for 5 minutes (smart deduplication).

---

## Deployment Checklist

- [ ] Email configured in production .env
- [ ] Database migration applied
- [ ] Backend deployed with Socket.io support
- [ ] Frontend deployed with NotificationContextProvider
- [ ] CORS origins updated for production domain
- [ ] Email SMTP tested with real account
- [ ] Unsubscribe link in emails
- [ ] Error monitoring configured (Sentry)
- [ ] Database backups enabled
- [ ] SSL/HTTPS enabled (required for push notifications)

---

That's it! 🎉

Your Smart Notification System is ready to go!
