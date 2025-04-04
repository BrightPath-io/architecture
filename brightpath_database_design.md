# BrightPath Database Design: MongoDB and Redis/Valkey Hybrid Approach

This document outlines the database design for the BrightPath homeschooling scheduling platform, using a hybrid approach with MongoDB for persistent storage and Redis/Valkey for caching and real-time functionality.

## MongoDB Schema Design

MongoDB serves as the primary data store for all persistent application data.

### 1. families Collection

```javascript
{
  "_id": ObjectId("..."),
  "googleId": "google_provided_id", // For OAuth authentication
  "email": "sarah@example.com", // From Google OAuth
  "name": "Sarah Kim", // From Section 1
  "createdAt": ISODate("2025-04-01"),
  "lastLogin": ISODate("2025-04-03"),
  "preferences": {
    "schedulingFlexibility": "moderate_structure", // From Section 5
    "homeschoolingPhilosophy": {
      // Ratings from Section 7.1 (Homeschooling Philosophy)
      "traditional": 2,
      "charlotte_mason": 5,
      "montessori": 3,
      "classical": 1,
      "unschooling": 4
    },
    "schedulingStyle": {
      // Ratings from Section 7.1 (Scheduling Style)
      "block": 4,
      "loop": 3,
      "time_blocked": 2,
      "interest_led": 5,
      "grouped_activities": 3
    },
    "planningApproach": "mix_of_planned_and_spontaneous", // From Section 7.2
    "activityPreferences": {
      // Ratings from Section 7.3
      "textbooks_worksheets": 2,
      "nature_exploration": 5,
      "arts_crafts": 4,
      "memorization": 2,
      "research_projects": 3,
      "experiments": 5,
      "discussions": 4
    }
  },
  "commitments": [
    // Family-wide commitments from Section 3
    {
      "name": "Work Meeting",
      "frequency": "weekly",
      "daysOfWeek": ["Tuesday", "Thursday"],
      "startTime": "10:00",
      "endTime": "11:30"
    }
  ]
}
```

### 2. children Collection

```javascript
{
  "_id": ObjectId("..."),
  "familyId": ObjectId("..."), // Reference to parent family
  "name": "Noah", // From Section 2
  "age": 6, // From Section 2
  "createdAt": ISODate("2025-04-01"),
  "commitments": [
    // Child-specific commitments from Section 3
    {
      "name": "Soccer Practice",
      "frequency": "weekly",
      "daysOfWeek": ["Monday", "Wednesday"],
      "startTime": "16:00",
      "endTime": "17:00"
    }
  ],
  "learningPreferences": {
    "bestLearningTimes": ["mid-morning", "early_afternoon"], // From Section 6.1
    "homeschoolingHours": {
      "startTime": "09:00",
      "endTime": "14:00" // From Section 6.2
    }
  },
  "subjects": [
    // From Section 4
    {
      "name": "Mathematics",
      "isCoreSubject": true,
      "sessionDuration": 30, // minutes
      "frequency": "daily",
      "parentInvolvement": "some_assistance",
      "fixedTime": {
        "required": false,
        "preferredTime": null
      },
      "interestLevel": 4 // 1-5 scale
    },
    {
      "name": "Reading",
      "isCoreSubject": true,
      "sessionDuration": 45,
      "frequency": "daily",
      "parentInvolvement": "full_involvement",
      "fixedTime": {
        "required": true,
        "preferredTime": "09:00"
      },
      "interestLevel": 5
    }
    // Additional subjects...
  ]
}
```

### 3. schedules Collection

```javascript
{
  "_id": ObjectId("..."),
  "familyId": ObjectId("..."),
  "childId": ObjectId("..."),
  "weekStartDate": ISODate("2025-04-07"),
  "createdAt": ISODate("2025-04-01"),
  "updatedAt": ISODate("2025-04-01"),
  "version": 1, // Increments with each regeneration
  "isActive": true,
  "generationMethod": "ml_model_v1",
  "days": [
    {
      "date": ISODate("2025-04-07"),
      "dayOfWeek": "Monday",
      "items": [
        {
          "_id": ObjectId("..."),
          "type": "subject",
          "name": "Mathematics",
          "originalSubject": {
            "name": "Mathematics",
            "isCoreSubject": true,
            "sessionDuration": 30,
            "parentInvolvement": "some_assistance"
          },
          "startTime": "09:30",
          "endTime": "10:00",
          "completed": false,
          "notes": ""
        },
        {
          "_id": ObjectId("..."),
          "type": "commitment",
          "name": "Soccer Practice",
          "startTime": "16:00",
          "endTime": "17:00",
          "isExternal": true
        }
        // More scheduled items...
      ]
    }
    // More days...
  ]
}
```

### 4. activity_logs Collection

```javascript
{
  "_id": ObjectId("..."),
  "scheduleId": ObjectId("..."),
  "scheduleItemId": ObjectId("..."),
  "familyId": ObjectId("..."),
  "childId": ObjectId("..."),
  "date": ISODate("2025-04-07"),
  "activityName": "Mathematics",
  "activityType": "subject",
  "scheduled": {
    "startTime": "09:30",
    "endTime": "10:00"
  },
  "actual": {
    "completed": true,
    "completedAt": ISODate("2025-04-07T10:05:00Z"), 
    "startTime": "09:35", // Optional - if user tracks actual times
    "endTime": "10:05" // Optional - if user tracks actual times
  },
  "notes": "Noah enjoyed the counting activities today",
  "createdAt": ISODate("2025-04-07T10:05:00Z")
}
```

### 5. questionnaire_progress Collection

```javascript
{
  "_id": ObjectId("..."),
  "familyId": ObjectId("..."),
  "startedAt": ISODate("2025-04-01T09:15:00Z"),
  "lastUpdatedAt": ISODate("2025-04-01T09:35:00Z"),
  "currentSection": 4,
  "progress": {
    "section1": "complete",
    "section2": "complete",
    "section3": "complete",
    "section4": "in_progress",
    "section5": "not_started",
    "section6": "not_started",
    "section7": "not_started"
  },
  "childrenProgress": [
    {
      "childId": ObjectId("..."),
      "section3Child": "complete",
      "section4": "in_progress", 
      "section6": "not_started"
    }
  ],
  "draftResponses": {
    // Store in-progress responses that haven't been committed
    // This allows for save & continue functionality
  }
}
```

### 6. ml_models Collection

```javascript
{
  "_id": ObjectId("..."),
  "name": "schedule_generator_v1",
  "version": "1.0.0",
  "createdAt": ISODate("2025-03-15"),
  "parameters": {
    // Model-specific parameters
  },
  "featureImportance": {
    // Information about which features most impact schedule generation
  },
  "performance": {
    "averageSatisfactionScore": 4.2,
    "usageCount": 1245
  }
}
```

## Redis/Valkey Schema Design

Redis/Valkey is used for caching, session management, and real-time functionality.

### 1. Session Management

```
// Session data with 24-hour expiry
SESSION:{user_id} -> {
  "userId": "123",
  "name": "Sarah Kim",
  "email": "sarah@example.com",
  "lastActivity": "2025-04-03T14:30:00Z"
}
TTL: 86400 (24 hours)

// Authentication refresh tokens
AUTH:REFRESH:{user_id} -> "refresh_token_value"
TTL: 2592000 (30 days)
```

### 2. Questionnaire Progress Caching

```
// Detailed questionnaire progress with short TTL
QUESTIONNAIRE:PROGRESS:{family_id} -> {
  "currentSection": 4,
  "currentChild": "child_id_1",
  "lastUpdated": "2025-04-03T15:30:00Z",
  "sectionsCompleted": ["section1", "section2", "section3"],
  "childSpecificProgress": {
    "child_id_1": {
      "section3Child": "complete",
      "section4": "in_progress"
    }
  }
}
TTL: 3600 (1 hour, then fall back to MongoDB)

// Temporary form data with longer TTL
QUESTIONNAIRE:DRAFT:{family_id}:{section_id} -> {
  // Draft responses for current section
}
TTL: 86400 (24 hours)
```

### 3. Schedule Caching

```
// Today's schedule for quick access
SCHEDULE:TODAY:{child_id} -> {
  "date": "2025-04-03",
  "items": [
    {
      "id": "item_id_1",
      "name": "Mathematics",
      "startTime": "09:30",
      "endTime": "10:00",
      "completed": false
    },
    // More items...
  ]
}
TTL: 86400 (24 hours)

// Current week's schedule
SCHEDULE:WEEK:{child_id} -> {
  "weekOf": "2025-04-03",
  "days": [
    // Daily schedules
  ]
}
TTL: 604800 (7 days)
```

### 4. Real-time Activity Status

```
// Current status of schedule items (for real-time updates)
SCHEDULE:ITEM:STATUS:{schedule_item_id} -> {
  "completed": true,
  "completedAt": "2025-04-03T10:05:00Z",
  "notes": "Great progress today"
}
TTL: 86400 (24 hours)

// Schedule modifications in progress (for drag & drop)
SCHEDULE:EDIT:{schedule_id} -> {
  "itemsMoved": [
    {
      "itemId": "item_id_1",
      "oldStartTime": "09:30",
      "newStartTime": "10:30"
    }
  ],
  "lastModified": "2025-04-03T14:22:00Z"
}
TTL: 1800 (30 minutes)
```

### 5. ML Model Caching

```
// Cached ML model parameters for quick schedule generation
ML:PARAMS:{model_id} -> {
  // Model parameters
}
TTL: 86400 (24 hours)

// Family-specific ML features
ML:FEATURES:{family_id} -> {
  // Extracted features from family preferences
}
TTL: 86400 (24 hours)
```

### 6. Real-time Metrics

```
// User activity metrics for monitoring
METRICS:USER_ACTIVITY:{date} -> {
  "activeUsers": 187,
  "completedActivities": 2145,
  "generatedSchedules": 42
}
TTL: 604800 (7 days)

// Per-user activity counters
METRICS:USER:{user_id}:{date} -> {
  "logins": 3,
  "completedActivities": 12,
  "scheduleEdits": 4
}
TTL: 604800 (7 days)
```

## Data Flow Between MongoDB and Redis/Valkey

This section explains how data flows between the two database systems for different application processes.

### User Authentication Flow

1. User authenticates via Google OAuth
2. Backend validates OAuth token and retrieves user information
3. System checks if user exists in MongoDB:
   - If user exists: Update last login timestamp
   - If user is new: Create new family document in MongoDB
4. Create session in Redis/Valkey with user information
5. Return session token to client
6. For subsequent requests:
   - Validate session token from Redis/Valkey
   - If session is valid, proceed with request
   - If session is invalid or expired, require re-authentication

### Questionnaire Flow

1. User begins questionnaire:
   - Create questionnaire_progress document in MongoDB
   - Cache progress in Redis/Valkey for fast access
   
2. User completes sections:
   - Store draft responses in Redis/Valkey during data entry
   - On section completion, write finalized section data to MongoDB
   - Update progress in both MongoDB and Redis/Valkey
   
3. Child-specific sections:
   - For each child, store related data in children collection
   - Update child-specific progress in questionnaire_progress
   
4. Handling interruptions:
   - If user leaves mid-questionnaire, data is preserved in Redis/Valkey
   - On return, restore progress from Redis/Valkey if available
   - If Redis/Valkey data expired, fall back to MongoDB progress data
   
5. Questionnaire completion:
   - All data is committed to MongoDB (families, children, etc.)
   - Progress document is marked complete
   - Trigger schedule generation process

### Schedule Generation Flow

1. Initial data gathering:
   - Load family preferences from MongoDB
   - Load child information and subjects from MongoDB
   - Extract features for ML model
   
2. ML model processing:
   - Cache extracted features in Redis/Valkey for performance
   - Load ML model parameters (from Redis/Valkey if cached, otherwise MongoDB)
   - Generate schedule based on features and parameters
   
3. Schedule storage:
   - Store generated schedule in MongoDB schedules collection
   - Cache current day/week schedule in Redis/Valkey for quick access
   
4. Notification:
   - Notify user of new schedule (through application channels)

### Schedule Interaction Flow

1. Schedule viewing:
   - Check Redis/Valkey first for cached schedule
   - If not in cache, load from MongoDB and cache results
   - Return schedule data to client
   
2. Completing activities:
   - Update status in Redis/Valkey immediately for real-time feedback
   - Create activity_log entry in MongoDB
   - Update schedule item completion status in MongoDB
   - Refresh Redis/Valkey cache with updated data
   
3. Schedule modifications (drag & drop):
   - Store temporary changes in Redis/Valkey during interaction
   - When changes are confirmed, update MongoDB schedule document
   - Update caches in Redis/Valkey with the new schedule
   - Log schedule modification for ML feedback

4. Adding notes:
   - Store notes temporarily in Redis/Valkey
   - Commit to MongoDB in activity_logs collection
   - Update cache with note information

### ML Feedback Loop

1. Data collection:
   - Gather completion data and modifications from MongoDB
   - Collect user satisfaction metrics
   
2. Feature extraction:
   - Process raw data into ML features
   - Store processed features temporarily in Redis/Valkey
   
3. Model improvement:
   - Update ML model parameters based on feedback
   - Store updated parameters in MongoDB
   - Cache new parameters in Redis/Valkey

## Implementation Considerations

### 1. Indexing Strategy

For MongoDB collections:

- `families`:
  - `googleId` (unique)
  - `email` (unique)

- `children`:
  - `familyId`
  - `name` (for search functionality)

- `schedules`:
  - `familyId`
  - `childId`
  - `weekStartDate`
  - Compound index: `{childId: 1, weekStartDate: 1}`

- `activity_logs`:
  - `scheduleId`
  - `scheduleItemId`
  - `date`
  - Compound index: `{childId: 1, date: 1}`

- `questionnaire_progress`:
  - `familyId` (unique)

### 2. Data Consistency Strategy

To maintain consistency between MongoDB and Redis/Valkey:

- Use a **write-through cache** pattern for critical data:
  1. Write data to MongoDB first
  2. Then update Redis/Valkey cache
  3. If Redis/Valkey update fails, implement retry logic or cache invalidation

- Implement **cache invalidation** when data is updated directly in MongoDB:
  1. After MongoDB update, delete or update corresponding Redis/Valkey keys
  2. Use event-driven architecture to trigger cache updates when possible

- Set appropriate **TTL values** for Redis/Valkey keys:
  - Short TTLs for frequently changing data
  - Longer TTLs for relatively static data
  - Match TTLs to usage patterns (e.g., longer TTLs during active sessions)

- Implement **fallback mechanism**:
  1. First try to read from Redis/Valkey
  2. If key missing or expired, read from MongoDB
  3. Re-populate Redis/Valkey cache with fresh data

### 3. Error Handling

- Implement **retry logic** for failed MongoDB operations
- Create **recovery procedures** for Redis/Valkey cache misses
- Log synchronization issues between databases
- Implement monitoring alerts for data inconsistencies

### 4. Scaling Considerations

- Use MongoDB replica sets for high availability
- Implement sharding strategy for MongoDB based on `familyId`
- Configure Redis/Valkey cluster mode for larger deployments
- Use connection pooling for both database systems
- Implement proper query timeout and circuit breaker patterns

## Next Steps for Database Implementation

1. Set up MongoDB Atlas cluster or self-hosted MongoDB
2. Configure Redis/Valkey instance with appropriate memory allocation
3. Create database schemas with validation
4. Implement data access layer for consistent database interactions
5. Set up monitoring and alerting for both database systems
6. Create backup and recovery procedures