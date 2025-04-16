# BrightPath Comprehensive Database Schema

## Overview

This document provides a comprehensive database schema for the BrightPath homeschooling scheduling platform. The schema is designed to leverage PostgreSQL with JSONB fields to provide both structured relational data and flexible document storage in a single database system.

## Core Design Principles

1. **Hybrid Relational-Document Approach**:
   - Structured tables for core entities and relationships
   - JSONB fields for flexible, hierarchical, or evolving data
   - Best of both worlds: schema enforcement where needed, flexibility where appropriate

2. **Data Storage Strategy**:
   - **Relational Tables**: For structured data with clear relationships
     - User authentication and profiles
     - Family and child core information
     - Subject and commitment definitions
     - Schedule references and relationships
   
   - **JSONB Fields**: For flexible, hierarchical, or evolving data
     - Complete questionnaire responses
     - Preferences and learning styles
     - Schedule details and structure
     - ML model parameters and configurations

3. **Storage Optimization**:
   - Indexed fields for frequently queried data
   - Specialized indexes for JSONB fields (GIN indexes)
   - Denormalization where appropriate for performance

## Database Tables

### 1. Users Table

Stores user authentication and account information.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    google_id VARCHAR(255) UNIQUE,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_login TIMESTAMP,
    questionnaire_completed BOOLEAN DEFAULT FALSE,
    questionnaire_completed_at TIMESTAMP
);

CREATE INDEX idx_users_google_id ON users(google_id);
CREATE INDEX idx_users_email ON users(email);
```

### 2. Families Table

Stores family information and preferences.

```sql
CREATE TABLE families (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    
    -- Structured fields for efficient querying
    scheduling_flexibility VARCHAR(50), -- e.g., 'moderate_structure'
    planning_approach VARCHAR(50), -- e.g., 'mix_of_planned_and_spontaneous'
    
    -- JSONB fields for flexible and hierarchical data
    preferences JSONB, -- Detailed preferences from questionnaire
    questionnaire_responses JSONB, -- Complete family-wide responses
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_families_user_id ON families(user_id);
CREATE INDEX idx_families_scheduling_flexibility ON families(scheduling_flexibility);
CREATE INDEX idx_families_preferences_gin ON families USING GIN (preferences);
```

### 3. Family Commitments Table

Stores recurring commitments for the entire family.

```sql
CREATE TABLE family_commitments (
    id SERIAL PRIMARY KEY,
    family_id INTEGER REFERENCES families(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    frequency VARCHAR(50) NOT NULL, -- 'daily', 'weekly', 'monthly', 'one-time'
    days_of_week VARCHAR(255), -- JSON array as string: '["Monday", "Wednesday"]'
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_family_commitments_family_id ON family_commitments(family_id);
```

### 4. Children Table

Stores information about each child in a family.

```sql
CREATE TABLE children (
    id SERIAL PRIMARY KEY,
    family_id INTEGER REFERENCES families(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    age INTEGER NOT NULL CHECK (age >= 2 AND age <= 18),
    
    -- JSONB fields for flexible data
    learning_preferences JSONB, -- Preferred learning times, styles, etc.
    questionnaire_responses JSONB, -- Complete child-specific responses
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_children_family_id ON children(family_id);
CREATE INDEX idx_children_age ON children(age);
CREATE INDEX idx_children_learning_preferences_gin ON children USING GIN (learning_preferences);
```

### 5. Child Commitments Table

Stores recurring commitments for individual children.

```sql
CREATE TABLE child_commitments (
    id SERIAL PRIMARY KEY,
    child_id INTEGER REFERENCES children(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    frequency VARCHAR(50) NOT NULL, -- 'daily', 'weekly', 'monthly', 'one-time'
    days_of_week VARCHAR(255), -- JSON array as string: '["Monday", "Wednesday"]'
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_child_commitments_child_id ON child_commitments(child_id);
```

### 6. Subjects Table

Stores educational subjects for each child.

```sql
CREATE TABLE subjects (
    id SERIAL PRIMARY KEY,
    child_id INTEGER REFERENCES children(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    is_core_subject BOOLEAN NOT NULL DEFAULT FALSE,
    session_duration INTEGER NOT NULL, -- in minutes
    frequency VARCHAR(50) NOT NULL, -- 'daily', '2-3_times_per_week', 'once_a_week', 'occasionally'
    parent_involvement VARCHAR(50) NOT NULL, -- 'full_involvement', 'some_assistance', 'minimal_supervision', 'independent'
    fixed_time_required BOOLEAN NOT NULL DEFAULT FALSE,
    preferred_time TIME,
    interest_level INTEGER NOT NULL CHECK (interest_level >= 1 AND interest_level <= 5),
    
    -- Additional JSONB field for flexible metadata
    metadata JSONB,
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subjects_child_id ON subjects(child_id);
CREATE INDEX idx_subjects_is_core ON subjects(is_core_subject);
CREATE INDEX idx_subjects_name ON subjects(name);
```

### 7. Schedules Table

Stores generated weekly schedules for each child.

```sql
CREATE TABLE schedules (
    id SERIAL PRIMARY KEY,
    family_id INTEGER REFERENCES families(id) ON DELETE CASCADE,
    child_id INTEGER REFERENCES children(id) ON DELETE CASCADE,
    week_start_date DATE NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    version INTEGER NOT NULL DEFAULT 1,
    generation_method VARCHAR(50) NOT NULL,
    
    -- JSONB for complete schedule structure
    schedule_data JSONB NOT NULL,
    
    -- JSONB for ML insights about this schedule
    patterns JSONB,
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_schedules_child_id_week ON schedules(child_id, week_start_date);
CREATE INDEX idx_schedules_family_id ON schedules(family_id);
CREATE INDEX idx_schedules_is_active ON schedules(is_active);
CREATE INDEX idx_schedules_data_gin ON schedules USING GIN (schedule_data);
```

### 8. Schedule Items Table

Stores individual activities within a schedule.

```sql
CREATE TABLE schedule_items (
    id SERIAL PRIMARY KEY,
    schedule_id INTEGER REFERENCES schedules(id) ON DELETE CASCADE,
    day_date DATE NOT NULL,
    item_type VARCHAR(50) NOT NULL, -- 'subject', 'commitment', 'break', etc.
    name VARCHAR(255) NOT NULL,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    is_external BOOLEAN NOT NULL DEFAULT FALSE,
    completed BOOLEAN DEFAULT FALSE,
    completed_at TIMESTAMP,
    
    -- Reference fields to link to original entities
    subject_id INTEGER REFERENCES subjects(id) ON DELETE SET NULL,
    commitment_id INTEGER REFERENCES child_commitments(id) ON DELETE SET NULL,
    
    -- JSONB for item-specific details
    item_data JSONB NOT NULL,
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_schedule_items_schedule_id ON schedule_items(schedule_id);
CREATE INDEX idx_schedule_items_day_date ON schedule_items(day_date);
CREATE INDEX idx_schedule_items_completion ON schedule_items(completed, completed_at);
CREATE INDEX idx_schedule_items_subject_id ON schedule_items(subject_id);
CREATE INDEX idx_schedule_items_item_data_gin ON schedule_items USING GIN (item_data);
```

### 9. Activity Logs Table

Tracks completion and adjustments to scheduled activities.

```sql
CREATE TABLE activity_logs (
    id SERIAL PRIMARY KEY,
    schedule_id INTEGER REFERENCES schedules(id) ON DELETE CASCADE,
    schedule_item_id INTEGER REFERENCES schedule_items(id) ON DELETE CASCADE,
    family_id INTEGER REFERENCES families(id) ON DELETE CASCADE,
    child_id INTEGER REFERENCES children(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    activity_name VARCHAR(255) NOT NULL,
    activity_type VARCHAR(50) NOT NULL,
    
    -- JSONB fields for flexible data
    scheduled JSONB NOT NULL,
    actual JSONB,
    
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activity_logs_child_id_date ON activity_logs(child_id, date);
CREATE INDEX idx_activity_logs_schedule_item_id ON activity_logs(schedule_item_id);
CREATE INDEX idx_activity_logs_family_id ON activity_logs(family_id);
CREATE INDEX idx_activity_logs_actual_gin ON activity_logs USING GIN (actual);
```

### 10. User Feedback Table

Stores user feedback on schedules and features.

```sql
CREATE TABLE user_feedback (
    id SERIAL PRIMARY KEY,
    family_id INTEGER REFERENCES families(id) ON DELETE CASCADE,
    child_id INTEGER REFERENCES children(id) ON DELETE SET NULL,
    schedule_id INTEGER REFERENCES schedules(id) ON DELETE SET NULL,
    feedback_type VARCHAR(50) NOT NULL, -- 'schedule_satisfaction', 'feature_suggestion', etc.
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    comments TEXT,
    
    -- JSONB for detailed feedback data
    feedback_data JSONB,
    
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_feedback_family_id ON user_feedback(family_id);
CREATE INDEX idx_user_feedback_schedule_id ON user_feedback(schedule_id);
CREATE INDEX idx_user_feedback_feedback_data_gin ON user_feedback USING GIN (feedback_data);
```

### 11. ML Models Table

Stores machine learning models and their parameters.

```sql
CREATE TABLE ml_models (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    version VARCHAR(50) NOT NULL,
    description TEXT,
    
    -- JSONB for model parameters and metrics
    parameters JSONB NOT NULL,
    feature_importance JSONB,
    performance_metrics JSONB,
    
    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_ml_models_name_version ON ml_models(name, version);
CREATE INDEX idx_ml_models_is_active ON ml_models(is_active);
CREATE INDEX idx_ml_models_parameters_gin ON ml_models USING GIN (parameters);
```

### 12. Questionnaire Sessions Table (Future Enhancement)

For server-side storage of questionnaire progress in future iterations.

```sql
CREATE TABLE questionnaire_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    family_id INTEGER REFERENCES families(id) ON DELETE CASCADE NULL,
    session_data JSONB NOT NULL,
    last_updated TIMESTAMP NOT NULL DEFAULT NOW(),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_questionnaire_sessions_user_id ON questionnaire_sessions(user_id);
CREATE INDEX idx_questionnaire_sessions_last_updated ON questionnaire_sessions(last_updated);
```

## JSONB Structure Examples

### Family Preferences JSONB Structure

```json
{
  "schedulingFlexibility": "moderate_structure",
  "homeschoolingPhilosophy": {
    "traditional": 2,
    "charlotte_mason": 5,
    "montessori": 3,
    "classical": 1,
    "unschooling": 4
  },
  "schedulingStyle": {
    "block": 4,
    "loop": 3,
    "time_blocked": 2,
    "interest_led": 5,
    "grouped_activities": 3
  },
  "planningApproach": "mix_of_planned_and_spontaneous",
  "activityPreferences": {
    "textbooks_worksheets": 2,
    "nature_exploration": 5,
    "arts_crafts": 4,
    "memorization": 2,
    "research_projects": 3,
    "experiments": 5,
    "discussions": 4
  }
}
```

### Child Learning Preferences JSONB Structure

```json
{
  "bestLearningTimes": ["mid-morning", "early_afternoon"],
  "homeschoolingHours": {
    "startTime": "09:00",
    "endTime": "14:00"
  },
  "derivedInsights": {
    "attentionSpan": "30-45_minutes",
    "preferredLearningModality": "hands_on",
    "interestClusters": [
      {
        "name": "STEM",
        "subjects": ["Mathematics", "Science"],
        "averageInterest": 4.5
      }
    ],
    "recommendedBreaks": {
      "frequency": "every_45_minutes",
      "duration": "10_minutes",
      "activities": ["physical_movement", "snack"]
    }
  }
}
```

### Schedule Data JSONB Structure

```json
{
  "weekStartDate": "2025-04-07",
  "version": 1,
  "generationMethod": "ml_model_v1",
  "days": [
    {
      "date": "2025-04-07",
      "dayOfWeek": "Monday",
      "items": [
        {
          "id": 123, // References schedule_items.id
          "type": "subject",
          "name": "Mathematics",
          "startTime": "09:30",
          "endTime": "10:00",
          "subjectId": 5,
          "completed": false
        },
        {
          "id": 124,
          "type": "commitment",
          "name": "Soccer Practice",
          "commitmentId": 3,
          "startTime": "16:00",
          "endTime": "17:00",
          "isExternal": true
        }
      ]
    },
    {
      "date": "2025-04-08",
      "dayOfWeek": "Tuesday",
      "items": []
    }
    // More days...
  ],
  "patterns": {
    "learningBlocks": [
      {
        "type": "core_academic",
        "preferredTime": "morning",
        "subjects": ["Mathematics", "Reading", "Science"],
        "averageDuration": 120
      }
    ],
    "breakPatterns": {
      "frequency": "hourly",
      "duration": 15,
      "activitySuggestions": ["Physical Movement", "Snack Time"]
    }
  }
}
```

### Schedule Item Data JSONB Structure

```json
{
  "originalSubject": {
    "name": "Mathematics",
    "isCoreSubject": true,
    "sessionDuration": 30,
    "parentInvolvement": "some_assistance"
  },
  "notes": "",
  "resources": [
    {
      "type": "worksheet",
      "name": "Addition Practice",
      "url": "https://example.com/worksheets/addition"
    }
  ],
  "adjustmentHistory": [
    {
      "date": "2025-04-05T10:30:00Z",
      "previousStartTime": "10:00",
      "previousEndTime": "10:30",
      "reason": "Conflict with doctor appointment"
    }
  ]
}
```

### Activity Log JSONB Fields

```json
// scheduled JSONB example
{
  "startTime": "09:30",
  "endTime": "10:00",
  "original": {
    "subjectId": 5,
    "name": "Mathematics",
    "duration": 30
  }
}

// actual JSONB example
{
  "completed": true,
  "completedAt": "2025-04-07T10:05:00Z",
  "startTime": "09:35",
  "endTime": "10:05",
  "adjustmentReason": "Started late due to breakfast running long",
  "parentInvolvement": "full_assistance",
  "childEngagement": 4
}
```

### ML Model Parameters JSONB Structure

```json
{
  "weights": {
    "childInterest": 0.4,
    "learningTimes": 0.3,
    "parentInvolvement": 0.2,
    "subjectCoherence": 0.1
  },
  "hyperparameters": {
    "learning_rate": 0.01,
    "iterations": 500,
    "regularization": 0.001
  },
  "featureEngineering": {
    "childAgeTransformation": "log_scale",
    "interestLevelNormalization": "min_max",
    "timePreferenceEncoding": "one_hot"
  },
  "featureImportance": {
    "childAge": 0.18,
    "subjectInterestLevel": 0.15,
    "parentInvolvementLevel": 0.12,
    "preferredLearningTime": 0.11
  },
  "performance": {
    "averageSatisfactionScore": 4.2,
    "usageCount": 1245,
    "adjustmentRate": 0.18,
    "completionRate": 0.85
  },
  "trainingData": {
    "sampleCount": 5842,
    "lastUpdated": "2025-03-10",
    "datasetVersion": "v2.3"
  }
}
```

### Questionnaire Session JSONB Structure (Future Enhancement)

```json
{
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
      "childId": "temp_child_1", // Temporary ID until saved to backend
      "name": "Noah",
      "section3Child": "complete",
      "section4": "in_progress", 
      "section6": "not_started"
    }
  ],
  "draftResponses": {
    "section1": {
      "name": "Sarah Kim"
    },
    "section2": {
      "name": "Noah",
      "age": 6
    }
    // More draft responses...
  },
  "lastUpdated": "2025-04-01T09:35:00Z"
}
```

## Entity Relationship Diagram

```
┌─────────┐
│  users  │
└────┬────┘
     │
     │ 1:1
     ▼
┌─────────┐     ┌────────────────────┐
│ families ├────►family_commitments   │
└────┬────┘     └────────────────────┘
     │
     │ 1:N
     ▼
┌─────────┐     ┌───────────────────┐     ┌──────────┐
│ children ├────►child_commitments   │     │ subjects │
└────┬────┘     └───────────────────┘     └─────┬────┘
     │                                          │
     │ 1:N                                      │
     ▼                                          │
┌─────────┐                                     │
│schedules│◄────────────────────────────────────┘
└────┬────┘
     │
     │ 1:N
     ▼
┌─────────────┐     ┌─────────────┐
│schedule_items├────►activity_logs│
└─────────────┘     └─────────────┘
```

## Key Design Considerations

1. **Optimized for Common Queries**:
   - Schedules for a specific child in a date range
   - Completed vs. incomplete activities
   - Commitments and obligations for schedule generation
   - Feedback analysis for ML improvement

2. **Balance of Structure and Flexibility**:
   - Core entity relationships maintained with foreign keys
   - Complex data structures stored in JSONB
   - Index strategy optimized for both relational and JSONB queries

3. **Data Integrity and Consistency**:
   - Referential integrity enforced with foreign keys
   - Transactions ensure atomic operations
   - Check constraints for data validation

4. **Performance Considerations**:
   - Specialized GIN indexes for JSONB fields
   - Denormalization where appropriate for read performance
   - Balanced design for write and read operations

5. **Future Scalability**:
   - Schema designed to accommodate future growth
   - JSONB fields allow for evolving data requirements
   - Separation of concerns in table design

## Implementation Notes

1. **For Django Implementation**:
   - Use Django's `JSONField` for JSONB fields
   - Implement model managers for complex queries
   - Use Django's transaction management for atomic operations

2. **For PostgreSQL Configuration**:
   - Enable appropriate extensions (e.g., `btree_gin`)
   - Configure appropriate connection pooling
   - Set up regular VACUUM operations
   - Configure appropriate WAL settings for performance

3. **For Local Storage Integration**:
   - Implement browser localStorage service in frontend
   - Track questionnaire progress client-side
   - Submit completed questionnaire to backend for PostgreSQL storage
   - Clear localStorage after successful submission

## Conclusion

This database schema provides a comprehensive foundation for the BrightPath homeschooling scheduling platform. By leveraging PostgreSQL with JSONB fields, the schema combines the benefits of relational and document databases in a single system, providing both structure and flexibility as needed.

The design accommodates the initial prototype's use of browser localStorage for questionnaire progress while providing a robust backend storage solution for the platform's core functionality. As the platform evolves, the schema can be enhanced with server-side questionnaire progress tracking and other advanced features without significant architectural changes.