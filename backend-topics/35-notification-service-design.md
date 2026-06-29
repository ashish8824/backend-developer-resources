# Notification Service Design

## What is a Notification Service?

A Notification Service is a dedicated system that handles sending notifications to users across multiple channels (Email, SMS, Push Notification, WebSocket, In-App) in a reliable, scalable, and centralized way.

```
Without centralized Notification Service:
Order Service    -> directly calls SendGrid API
Payment Service  -> directly calls Twilio API
Auth Service     -> directly calls Firebase FCM
-> Each service manages its own provider credentials
-> No unified retry logic, no delivery tracking, no template management

With centralized Notification Service:
Order Service    -> publish "ORDER_PLACED" event
Payment Service  -> publish "PAYMENT_COMPLETED" event
Auth Service     -> publish "OTP_REQUESTED" event
                        ↓
              Notification Service
              - reads events
              - decides channels (Email + SMS + Push)
              - applies templates
              - handles retries
              - tracks delivery
              - sends via providers
```

## Requirements Clarification (Interview Step 1)

### Functional Requirements

- Send notifications via: Email, SMS, Push Notification, In-App, WebSocket
- Support notification templates (with variable substitution)
- Schedule notifications (send at a specific time)
- Track delivery status (SENT, DELIVERED, FAILED, READ)
- User preferences (opted-out channels, quiet hours, language)
- Batch/bulk notifications (promotional campaigns)
- Retry on failure with exponential backoff
- Dead letter queue for permanently failed notifications

### Non-Functional Requirements

- High throughput: millions of notifications per day
- Low latency: transactional notifications (OTP, order confirmation) < 5 seconds
- High availability: 99.9% uptime
- Idempotency: no duplicate notifications
- Observability: delivery metrics, failure rates per channel

### Scale Estimation

```
Daily active users:     10 million
Avg notifications/user: 5 / day
Total:                  50 million notifications / day
Peak:                   50M / (24h * 3600s) * 10 (peak factor) = ~6,000/sec

Email:     30% = 15M/day
SMS:       20% = 10M/day
Push:      40% = 20M/day
In-App:    10% = 5M/day
```

## System Architecture

```
Event Sources                    Notification Service                    Channels
─────────────                    ────────────────────                    ────────
Order Service ──► Kafka ─────►  ┌──────────────────────────────────┐  ► Email (SendGrid)
Payment Svc   ──► Kafka ─────►  │  Event Consumer                  │  ► SMS (Twilio)
Auth Service  ──► Kafka ─────►  │       │                          │  ► Push (FCM/APNs)
REST API      ──────────────►  │  Preference Filter                │  ► WebSocket
                                │       │                          │  ► In-App DB
                                │  Template Renderer               │
                                │       │                          │
                                │  Channel Router                  │
                                │       │                          │
                                │  Job Queue (BullMQ/SQS)          │
                                │       │                          │
                                │  Channel Workers (×N per channel)│
                                │       │                          │
                                │  Delivery Tracker                │
                                └──────────────────────────────────┘
                                         │
                                    Redis + PostgreSQL
```

## Database Schema

```sql
-- Notification templates
CREATE TABLE notification_templates (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) UNIQUE NOT NULL,  -- 'order_placed', 'otp_verification'
    channel     VARCHAR(20) NOT NULL,           -- 'EMAIL', 'SMS', 'PUSH'
    subject     TEXT,                           -- for email
    body        TEXT NOT NULL,                  -- with {{variable}} placeholders
    language    VARCHAR(5) DEFAULT 'en',
    active      BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- User notification preferences
CREATE TABLE notification_preferences (
    user_id        BIGINT PRIMARY KEY REFERENCES users(id),
    email_enabled  BOOLEAN DEFAULT TRUE,
    sms_enabled    BOOLEAN DEFAULT TRUE,
    push_enabled   BOOLEAN DEFAULT TRUE,
    inapp_enabled  BOOLEAN DEFAULT TRUE,
    quiet_hours_start TIME,    -- don't send during quiet hours
    quiet_hours_end   TIME,
    timezone       VARCHAR(50) DEFAULT 'UTC',
    language       VARCHAR(5)  DEFAULT 'en',
    unsubscribed   BOOLEAN DEFAULT FALSE   -- global unsubscribe
);

-- Notification log (every notification ever sent)
CREATE TABLE notifications (
    id              BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE,  -- prevent duplicates
    user_id         BIGINT REFERENCES users(id),
    event_type      VARCHAR(100),         -- 'ORDER_PLACED', 'OTP_REQUESTED'
    channel         VARCHAR(20),          -- 'EMAIL', 'SMS', 'PUSH'
    template_name   VARCHAR(100),
    recipient       VARCHAR(255),         -- email address, phone, device token
    subject         TEXT,
    body            TEXT,
    status          VARCHAR(20) DEFAULT 'PENDING',
    attempt_count   INT DEFAULT 0,
    last_attempted_at TIMESTAMP,
    delivered_at    TIMESTAMP,
    read_at         TIMESTAMP,
    provider        VARCHAR(50),          -- 'SENDGRID', 'TWILIO', 'FCM'
    provider_message_id VARCHAR(255),     -- provider's message ID for tracking
    error_message   TEXT,
    metadata        JSONB,               -- extra data (order ID, etc.)
    scheduled_at    TIMESTAMP,           -- for scheduled notifications
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_notif_user_id   ON notifications(user_id);
CREATE INDEX idx_notif_status    ON notifications(status);
CREATE INDEX idx_notif_idem_key  ON notifications(idempotency_key);
CREATE INDEX idx_notif_scheduled ON notifications(scheduled_at) WHERE status = 'PENDING';

-- User device tokens (for push notifications)
CREATE TABLE device_tokens (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    token       TEXT NOT NULL,
    platform    VARCHAR(10),   -- 'IOS', 'ANDROID', 'WEB'
    active      BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT NOW(),
    UNIQUE (user_id, token)
);

-- In-app notification inbox
CREATE TABLE inapp_notifications (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    title       VARCHAR(200),
    body        TEXT,
    action_url  TEXT,
    read        BOOLEAN DEFAULT FALSE,
    read_at     TIMESTAMP,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

## Core Implementation

### Notification Request DTO

```java
@Data
@Builder
public class NotificationRequest {
    private String idempotencyKey;        // client-generated UUID
    private Long userId;
    private String eventType;             // 'ORDER_PLACED', 'OTP_REQUESTED'
    private String templateName;
    private Map<String, String> variables; // template variables
    private List<String> channels;        // override channels (optional)
    private LocalDateTime scheduledAt;    // null = send immediately
    private Map<String, Object> metadata; // extra data
}
```

### Notification Service (Core Orchestrator)

```java
@Service
public class NotificationService {

    @Autowired private NotificationRepository notifRepo;
    @Autowired private TemplateService templateService;
    @Autowired private PreferenceService preferenceService;
    @Autowired private ChannelRouter channelRouter;
    @Autowired private NotificationQueue queue;

    @Transactional
    public NotificationResult send(NotificationRequest request) {

        // 1. Idempotency check
        if (notifRepo.existsByIdempotencyKey(request.getIdempotencyKey())) {
            return NotificationResult.alreadySent(request.getIdempotencyKey());
        }

        // 2. Load user preferences
        NotificationPreferences prefs =
                preferenceService.getPreferences(request.getUserId());

        if (prefs.isUnsubscribed()) {
            return NotificationResult.skipped("User globally unsubscribed");
        }

        // 3. Determine channels (override or default per event type)
        List<Channel> channels = request.getChannels() != null
                ? toChannels(request.getChannels())
                : channelRouter.resolveChannels(request.getEventType(), prefs);

        // 4. Render templates and create notification records
        List<Notification> notifications = new ArrayList<>();
        for (Channel channel : channels) {

            if (!prefs.isChannelEnabled(channel)) continue;

            RenderedTemplate rendered = templateService.render(
                    request.getTemplateName(), channel, request.getVariables(),
                    prefs.getLanguage());

            Notification notif = Notification.builder()
                    .idempotencyKey(request.getIdempotencyKey() + ":" + channel)
                    .userId(request.getUserId())
                    .eventType(request.getEventType())
                    .channel(channel)
                    .templateName(request.getTemplateName())
                    .recipient(resolveRecipient(request.getUserId(), channel))
                    .subject(rendered.getSubject())
                    .body(rendered.getBody())
                    .status(NotificationStatus.PENDING)
                    .scheduledAt(request.getScheduledAt())
                    .metadata(toJson(request.getMetadata()))
                    .build();

            notifications.add(notifRepo.save(notif));
        }

        // 5. Enqueue for sending (async)
        for (Notification notif : notifications) {
            if (notif.getScheduledAt() == null) {
                queue.enqueue(notif);  // immediate
            } else {
                queue.scheduleAt(notif, notif.getScheduledAt());  // delayed
            }
        }

        return NotificationResult.queued(notifications.size());
    }
}
```

### Template Service

```java
@Service
public class TemplateService {

    @Autowired
    private TemplateRepository templateRepo;

    // Templates stored in DB with {{variable}} placeholders
    // "Hello {{userName}}, your order {{orderId}} has been placed!"

    public RenderedTemplate render(String templateName, Channel channel,
                                   Map<String, String> variables, String language) {

        NotificationTemplate template = templateRepo
                .findByNameAndChannelAndLanguage(templateName, channel, language)
                .orElseGet(() -> templateRepo
                        .findByNameAndChannelAndLanguage(templateName, channel, "en")
                        .orElseThrow(() -> new TemplateNotFoundException(templateName)));

        String renderedSubject = substituteVariables(template.getSubject(), variables);
        String renderedBody = substituteVariables(template.getBody(), variables);

        return new RenderedTemplate(renderedSubject, renderedBody);
    }

    private String substituteVariables(String text, Map<String, String> variables) {
        if (text == null || variables == null) return text;
        for (Map.Entry<String, String> entry : variables.entrySet()) {
            text = text.replace("{{" + entry.getKey() + "}}", entry.getValue());
        }
        return text;
    }
}
```

### Channel Workers

```java
// Email Worker
@Service
public class EmailNotificationWorker {

    @Autowired
    private SendGridEmailProvider sendGrid;

    @Autowired
    private NotificationRepository notifRepo;

    public void process(Notification notification) {
        notifRepo.updateStatus(notification.getId(),
                NotificationStatus.SENDING, LocalDateTime.now());

        try {
            String messageId = sendGrid.send(
                    notification.getRecipient(),
                    notification.getSubject(),
                    notification.getBody()
            );

            notifRepo.markDelivered(notification.getId(), messageId, LocalDateTime.now());

        } catch (ProviderException e) {
            handleFailure(notification, e);
        }
    }

    private void handleFailure(Notification notification, Exception e) {
        int attempts = notification.getAttemptCount() + 1;

        if (attempts >= 3) {
            notifRepo.markFailed(notification.getId(), e.getMessage());
            // Move to DLQ for investigation
            dlqService.send(notification, e.getMessage());
        } else {
            // Exponential backoff retry
            long delayMs = (long) Math.pow(2, attempts) * 1000;  // 2s, 4s, 8s
            notifRepo.updateForRetry(notification.getId(), attempts);
            queue.scheduleAt(notification, LocalDateTime.now().plusNanos(delayMs * 1_000_000));
        }
    }
}

// Push Notification Worker
@Service
public class PushNotificationWorker {

    @Autowired
    private FirebaseMessagingService fcm;

    @Autowired
    private DeviceTokenRepository tokenRepo;

    public void process(Notification notification) {
        List<String> tokens = tokenRepo.findActiveTokensByUserId(notification.getUserId());

        if (tokens.isEmpty()) {
            notifRepo.markFailed(notification.getId(), "No device tokens registered");
            return;
        }

        for (String token : tokens) {
            try {
                String messageId = fcm.sendToDevice(
                        token,
                        notification.getSubject(),
                        notification.getBody(),
                        notification.getMetadata()
                );
                log.info("Push sent to token {}: messageId={}", token, messageId);

            } catch (InvalidTokenException e) {
                // Token is no longer valid (app uninstalled) — deactivate it
                tokenRepo.deactivate(token);
            } catch (Exception e) {
                handleFailure(notification, e);
            }
        }
    }
}
```

## Node.js Implementation

```javascript
const { Queue, Worker } = require('bullmq');
const sgMail = require('@sendgrid/mail');
const twilio = require('twilio');
const { Expo } = require('expo-server-sdk');  // push notifications

const connection = { host: 'localhost', port: 6379 };

// Queues per channel
const emailQueue = new Queue('notifications:email', { connection });
const smsQueue   = new Queue('notifications:sms', { connection });
const pushQueue  = new Queue('notifications:push', { connection });

// Main service
async function sendNotification(request) {
    const { userId, eventType, templateName, variables, idempotencyKey } = request;

    // Idempotency check
    const exists = await Notification.findOne({ idempotencyKey });
    if (exists) return { status: 'already_sent' };

    // Load preferences
    const prefs = await NotificationPreference.findOne({ userId });
    if (prefs?.unsubscribed) return { status: 'unsubscribed' };

    // Resolve channels
    const channels = resolveChannels(eventType, prefs);

    for (const channel of channels) {
        if (!prefs[`${channel.toLowerCase()}Enabled`]) continue;

        const template = await renderTemplate(templateName, channel, variables, prefs.language);
        const recipient = await resolveRecipient(userId, channel);

        const notification = await Notification.create({
            idempotencyKey: `${idempotencyKey}:${channel}`,
            userId, eventType, channel, templateName,
            recipient, subject: template.subject, body: template.body,
            status: 'PENDING'
        });

        // Enqueue based on channel
        const queue = channel === 'EMAIL' ? emailQueue :
                      channel === 'SMS'   ? smsQueue   : pushQueue;

        await queue.add(channel.toLowerCase(), { notificationId: notification.id }, {
            attempts: 3,
            backoff: { type: 'exponential', delay: 2000 }
        });
    }

    return { status: 'queued' };
}

// Email Worker
new Worker('notifications:email', async (job) => {
    const { notificationId } = job.data;
    const notification = await Notification.findById(notificationId);

    await sgMail.send({
        to: notification.recipient,
        from: 'noreply@myapp.com',
        subject: notification.subject,
        html: notification.body
    });

    await Notification.findByIdAndUpdate(notificationId, {
        status: 'DELIVERED',
        deliveredAt: new Date()
    });

    console.log(`Email sent to ${notification.recipient}`);
}, { connection, concurrency: 20 });

// SMS Worker
const twilioClient = twilio(process.env.TWILIO_SID, process.env.TWILIO_TOKEN);

new Worker('notifications:sms', async (job) => {
    const notification = await Notification.findById(job.data.notificationId);

    const message = await twilioClient.messages.create({
        body: notification.body,
        from: process.env.TWILIO_PHONE,
        to: notification.recipient
    });

    await Notification.findByIdAndUpdate(job.data.notificationId, {
        status: 'DELIVERED',
        providerMessageId: message.sid,
        deliveredAt: new Date()
    });
}, { connection, concurrency: 10 });
```

## Handling User Preferences

```java
@Service
public class PreferenceService {

    public List<Channel> resolveChannels(String eventType,
                                          NotificationPreferences prefs) {

        // Define default channels per event type
        Map<String, List<Channel>> defaults = Map.of(
            "ORDER_PLACED",       List.of(EMAIL, PUSH, IN_APP),
            "PAYMENT_FAILED",     List.of(EMAIL, SMS, PUSH),
            "OTP_REQUESTED",      List.of(SMS),  // SMS only — critical, ignore most prefs
            "PROMOTIONAL",        List.of(EMAIL),
            "ACCOUNT_ALERT",      List.of(EMAIL, SMS, PUSH),
            "DELIVERY_UPDATE",    List.of(PUSH, IN_APP)
        );

        List<Channel> channels = defaults.getOrDefault(eventType, List.of(EMAIL));

        // Filter by user preferences (except for critical/OTP)
        if (!isCritical(eventType)) {
            channels = channels.stream()
                    .filter(prefs::isChannelEnabled)
                    .filter(c -> !isQuietHours(prefs, c))
                    .toList();
        }

        return channels;
    }

    private boolean isQuietHours(NotificationPreferences prefs, Channel channel) {
        if (prefs.getQuietHoursStart() == null || channel == SMS) return false;

        LocalTime now = LocalTime.now(ZoneId.of(prefs.getTimezone()));
        LocalTime start = prefs.getQuietHoursStart();
        LocalTime end = prefs.getQuietHoursEnd();

        if (start.isBefore(end)) {
            return now.isAfter(start) && now.isBefore(end);
        } else {
            // Crosses midnight
            return now.isAfter(start) || now.isBefore(end);
        }
    }
}
```

## Notification Tracking & Webhooks

```java
// Provider webhooks update delivery status
@RestController
@RequestMapping("/webhooks")
public class WebhookController {

    // SendGrid delivery events
    @PostMapping("/sendgrid")
    public ResponseEntity<?> handleSendGridEvent(@RequestBody List<SendGridEvent> events) {
        for (SendGridEvent event : events) {
            switch (event.getEvent()) {
                case "delivered" -> notifRepo.markDelivered(
                        event.getMessageId(), LocalDateTime.now());
                case "open"      -> notifRepo.markRead(
                        event.getMessageId(), LocalDateTime.now());
                case "bounce", "dropped" -> notifRepo.markFailed(
                        event.getMessageId(), event.getReason());
            }
        }
        return ResponseEntity.ok().build();
    }

    // Twilio SMS status
    @PostMapping("/twilio")
    public ResponseEntity<?> handleTwilioStatus(@RequestParam String MessageSid,
                                                 @RequestParam String MessageStatus) {
        switch (MessageStatus) {
            case "delivered" -> notifRepo.markDeliveredBySid(MessageSid, LocalDateTime.now());
            case "failed", "undelivered" -> notifRepo.markFailedBySid(MessageSid, "Delivery failed");
        }
        return ResponseEntity.ok().build();
    }
}
```

## Production Architecture

```
REST API / Kafka Events
         │
         ▼
   Notification Service
   (3 instances, load balanced)
         │
    ┌────┴────┐
    │         │
   Redis    PostgreSQL
   (queues) (notification log,
             templates, prefs)
         │
    ┌────┴──────────────────────────┐
    │           │                  │
Email Workers  SMS Workers     Push Workers
(×10)          (×5)            (×5)
    │           │                  │
SendGrid     Twilio            FCM / APNs
         │
    Delivery Status Webhooks
         │
    PostgreSQL (status update)
         │
    WebSocket / SSE
    (notify frontend of status)
```

## Interview Questions

**Q1: Why should a Notification Service be a separate microservice?**

Multiple services need to send notifications (Order, Payment, Auth, Marketing). A centralized service avoids duplicating provider credentials, retry logic, template management, and delivery tracking across every service. It provides consistent formatting, unified user preferences, delivery analytics, and a single place to add new channels or switch providers.

**Q2: How do you prevent duplicate notifications?**

Idempotency keys — each notification request carries a unique key (UUID). The service checks if a notification with that key already exists before processing. If found, return the existing result. This is critical because event queues deliver at-least-once, retries happen on failure, and users complain about receiving the same OTP three times.

**Q3: How do you handle notification failures and retries?**

Attempt up to N times (e.g. 3) with exponential backoff (2s, 4s, 8s). After all retries fail, move to a Dead Letter Queue. Alert the team, preserve the notification in the DLQ for investigation and manual replay. Track attempt count and error messages in the notification log. For critical notifications (OTP), retry aggressively and alert immediately on failure.

**Q4: How do you handle user preferences and quiet hours?**

Store per-user preferences: enabled channels, quiet hours (start/end time), timezone, language, global unsubscribe. Before routing, filter channels by user preferences. During quiet hours, delay non-critical notifications until morning. Critical notifications (OTP, account security alerts) bypass quiet hours and some channel preferences. Respect unsubscribe for marketing; mandatory transactional notifications override it.

**Q5: How do you handle push notifications when a user has multiple devices?**

Store all active device tokens per user (one row per device). When pushing, send to all active tokens for that user. Handle `InvalidTokenException` (app uninstalled) by marking that token as inactive in the DB — don't retry invalid tokens. Use FCM's multicast API to send to multiple tokens in a single API call for efficiency.

**Q6: How would you design the notification service for 50 million notifications per day?**

Separate workers per channel (email/SMS/push scale independently). Use a job queue (BullMQ/SQS) with concurrency tuned per channel and provider limits. Batch emails (SendGrid batch API). Cache user preferences in Redis (avoid DB hit per notification). Partition the notifications table by date for efficient queries and cleanup. Use provider webhooks for delivery status rather than polling. Deploy workers as auto-scaling groups triggered by queue depth metrics.
