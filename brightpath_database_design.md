# BrightPath Database Design: PostgreSQL with JSONB

This document outlines the revised database design for the BrightPath homeschooling scheduling platform, using PostgreSQL with JSONB fields to provide both relational structure and document flexibility in a single database, alongside browser local storage for questionnaire progress in the initial prototype.

## Database Strategy Overview

The BrightPath platform will use PostgreSQL with JSONB fields to leverage both relational and document database features:

1. **Core Relational Tables**: For structured data with clear relationships
   - User authentication and profiles
   - Family and child core information
   - Subject and commitment definitions
   - Schedule references and relationships

2. **JSONB Fields**: For flexible, hierarchical, or evolving data
   - Complete questionnaire responses
   - Preferences and learning styles
   - Schedule details and structure
   - ML model parameters and configurations

3. **Browser Local Storage**: For questionnaire progress in the initial prototype
   - Draft questionnaire responses
   - Section completion status
   - Multi-child progress tracking

This approach avoids database synchronization challenges while preserving the flexibility needed for complex, hierarchical data.

## PostgreSQL Schema Design

### Users Table
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

### Families Table
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

### Family Commitments Table
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

### Children Table
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

### Child Commitments Table
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

### Subjects Table
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

### Schedules Table
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

### Schedule Items Table
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

### Activity Logs Table
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

### User Feedback Table
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

### ML Models Table
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

## JSONB Structure Examples

This section provides examples of the structure and content of JSONB fields in key tables.

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

## Local Storage Schema for Questionnaire Progress

For the initial prototype, the browser's localStorage will be used to store questionnaire progress and draft responses as in the original design.

### Key Patterns

```
// Overall questionnaire progress
questionnaire_progress_{familyId} -> {
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
  "lastUpdated": "2025-04-01T09:35:00Z"
}

// Draft responses for each section
questionnaire_draft_{familyId}_section1 -> {
  "name": "Sarah Kim"
}

// Child-specific draft responses
questionnaire_draft_{familyId}_child_temp_child_1_section4 -> {
  "subjects": [
    {
      "name": "Mathematics",
      "isCoreSubject": true,
      "sessionDuration": 30, 
      // ...other subject details
    }
    // ...more subjects
  ]
}
```

## Implementation Considerations

### 1. Working with JSONB in Django

Django provides good support for PostgreSQL's JSONB fields. Here's how they can be implemented:

```python
# Example Django models
from django.db import models
from django.contrib.postgres.fields import JSONField

class Family(models.Model):
    user = models.ForeignKey('User', on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    scheduling_flexibility = models.CharField(max_length=50, null=True, blank=True)
    planning_approach = models.CharField(max_length=50, null=True, blank=True)
    preferences = JSONField(default=dict)
    questionnaire_responses = JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.name

class Schedule(models.Model):
    family = models.ForeignKey('Family', on_delete=models.CASCADE)
    child = models.ForeignKey('Child', on_delete=models.CASCADE)
    week_start_date = models.DateField()
    is_active = models.BooleanField(default=True)
    version = models.IntegerField(default=1)
    generation_method = models.CharField(max_length=50)
    schedule_data = JSONField()
    patterns = JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        unique_together = ('child', 'week_start_date', 'is_active')
    
    def __str__(self):
        return f"{self.child.name}'s Schedule - Week of {self.week_start_date}"
```

### 2. Querying JSONB Fields

PostgreSQL provides powerful operators for querying JSONB data:

```python
# Examples of JSONB queries in Django

# Find families that prefer Charlotte Mason philosophy (score >= 4)
families = Family.objects.filter(
    preferences__contains={'homeschoolingPhilosophy': {'charlotte_mason': 4}}
)

# Find schedules for children who learn best in the morning
morning_learners = Child.objects.filter(
    learning_preferences__contains={'bestLearningTimes': ['early_morning']}
)

# Find all schedule items for mathematics
math_items = ScheduleItem.objects.filter(
    name='Mathematics'
)

# Find all completed activities for a child in a date range
completed_activities = ActivityLog.objects.filter(
    child_id=child_id,
    date__range=[start_date, end_date],
    actual__contains={'completed': True}
)
```

### 3. Indexing Strategy for JSONB Fields

Proper indexing is crucial for JSONB field performance:

```sql
-- GIN index for JSON containment operators (@>, ?, ?& and ?| operators)
CREATE INDEX idx_family_preferences_gin ON families USING GIN (preferences);

-- For specific JSON paths that are frequently queried
CREATE INDEX idx_child_best_learning_times ON children USING GIN ((learning_preferences->'bestLearningTimes'));

-- B-tree index for equality comparisons on JSON fields
CREATE INDEX idx_schedule_item_completed ON schedule_items ((item_data->>'completed'));
```

### 4. Transaction Management

PostgreSQL's ACID compliance simplifies transaction management:

```python
# Example of atomic transaction with JSONB fields
from django.db import transaction

@transaction.atomic
def mark_schedule_item_completed(item_id, completion_notes=None):
    # Get the schedule item
    item = ScheduleItem.objects.get(id=item_id)
    
    # Update the schedule item
    item.completed = True
    item.completed_at = timezone.now()
    item.save()
    
    # Update the item in the schedule_data JSONB field
    schedule = Schedule.objects.get(id=item.schedule_id)
    schedule_data = schedule.schedule_data
    
    # Find the item in the schedule data
    for day in schedule_data['days']:
        for day_item in day['items']:
            if day_item['id'] == item.id:
                day_item['completed'] = True
                break
    
    # Save the updated schedule data
    schedule.schedule_data = schedule_data
    schedule.save()
    
    # Create activity log
    ActivityLog.objects.create(
        schedule_id=item.schedule_id,
        schedule_item_id=item.id,
        family_id=item.schedule.family_id,
        child_id=item.schedule.child_id,
        date=item.day_date,
        activity_name=item.name,
        activity_type=item.item_type,
        scheduled={
            "startTime": item.start_time.isoformat(),
            "endTime": item.end_time.isoformat(),
            "original": {
                "subjectId": item.subject_id,
                "name": item.name,
                "duration": (
                    datetime.combine(date.min, item.end_time) - 
                    datetime.combine(date.min, item.start_time)
                ).seconds // 60
            }
        },
        actual={
            "completed": True,
            "completedAt": timezone.now().isoformat()
        },
        notes=completion_notes
    )
    
    return True
```

### 5. Handling Schedule Manipulations

When users interact with schedules (drag and drop, mark complete, etc.):

```python
# Example of handling schedule item dragging/rescheduling
@transaction.atomic
def reschedule_item(item_id, new_day_date, new_start_time, new_end_time):
    # Update the schedule item in the relational table
    item = ScheduleItem.objects.get(id=item_id)
    
    # Store original values for activity log
    original_date = item.day_date
    original_start = item.start_time
    original_end = item.end_time
    
    # Update item
    item.day_date = new_day_date
    item.start_time = new_start_time
    item.end_time = new_end_time
    
    # Update adjustment history in item_data JSONB
    item_data = item.item_data
    
    if 'adjustmentHistory' not in item_data:
        item_data['adjustmentHistory'] = []
        
    item_data['adjustmentHistory'].append({
        "date": timezone.now().isoformat(),
        "previousStartTime": original_start.isoformat(),
        "previousEndTime": original_end.isoformat(),
        "reason": "User rescheduled activity"
    })
    
    item.item_data = item_data
    item.save()
    
    # Update the schedule_data JSONB field
    schedule = Schedule.objects.get(id=item.schedule_id)
    schedule_data = schedule.schedule_data
    
    # Remove item from original day
    for day in schedule_data['days']:
        if day['date'] == original_date.isoformat():
            day['items'] = [i for i in day['items'] if i['id'] != item.id]
    
    # Add item to new day
    new_day_exists = False
    for day in schedule_data['days']:
        if day['date'] == new_day_date.isoformat():
            new_day_exists = True
            day['items'].append({
                "id": item.id,
                "type": item.item_type,
                "name": item.name,
                "startTime": new_start_time.isoformat(),
                "endTime": new_end_time.isoformat(),
                "completed": item.completed,
                "subjectId": item.subject_id
            })
            # Sort items by start time
            day['items'] = sorted(day['items'], key=lambda x: x['startTime'])
    
    # If new day doesn't exist, create it
    if not new_day_exists:
        new_day = {
            "date": new_day_date.isoformat(),
            "dayOfWeek": new_day_date.strftime("%A"),
            "items": [{
                "id": item.id,
                "type": item.item_type,
                "name": item.name,
                "startTime": new_start_time.isoformat(),
                "endTime": new_end_time.isoformat(),
                "completed": item.completed,
                "subjectId": item.subject_id
            }]
        }
        schedule_data['days'].append(new_day)
        # Sort days by date
        schedule_data['days'] = sorted(schedule_data['days'], key=lambda x: x['date'])
    
    # Save updated schedule
    schedule.schedule_data = schedule_data
    schedule.save()
    
    # Create activity log
    ActivityLog.objects.create(
        schedule_id=item.schedule_id,
        schedule_item_id=item.id,
        family_id=schedule.family_id,
        child_id=schedule.child_id,
        date=original_date,
        activity_name=item.name,
        activity_type=item.item_type,
        scheduled={
            "startTime": original_start.isoformat(),
            "endTime": original_end.isoformat()
        },
        actual={
            "rescheduled": True,
            "newDate": new_day_date.isoformat(),
            "newStartTime": new_start_time.isoformat(),
            "newEndTime": new_end_time.isoformat()
        },
        notes="User rescheduled activity"
    )
    
    return True
```

### 6. Question Response Processing and Storage

When the questionnaire is completed, the responses are stored in both relational tables and JSONB fields:

```python
@transaction.atomic
def save_questionnaire_data(user_id, data):
    """
    Process and save questionnaire responses to PostgreSQL
    """
    try:
        # Get the user
        user = User.objects.get(id=user_id)
        
        # Create or update family record
        family, created = Family.objects.get_or_create(user=user)
        family.name = data['section1']['name']
        family.scheduling_flexibility = data['section5']['flexibility']
        family.planning_approach = data['section7']['planningApproach']
        
        # Save complete questionnaire data to JSONB field
        family.questionnaire_responses = {
            'section1': data['section1'],
            'section5': data['section5'],
            'section7': data['section7']
        }
        
        # Process family preferences into structured JSONB
        family.preferences = {
            'schedulingFlexibility': data['section5']['flexibility'],
            'homeschoolingPhilosophy': data['section7']['homeschoolingPhilosophy'],
            'schedulingStyle': data['section7']['schedulingStyle'],
            'planningApproach': data['section7']['planningApproach'],
            'activityPreferences': data['section7']['activityPreferences']
        }
        
        family.questionnaire_completed = True
        family.questionnaire_completed_at = timezone.now()
        family.save()
        
        # Save family commitments if any
        if 'commitments' in data['section3']:
            # First delete existing commitments
            FamilyCommitment.objects.filter(family=family).delete()
            
            # Then create new commitments
            for commitment in data['section3']['commitments']:
                FamilyCommitment.objects.create(
                    family=family,
                    name=commitment['name'],
                    frequency=commitment['frequency'],
                    days_of_week=json.dumps(commitment.get('daysOfWeek', [])),
                    start_time=commitment['startTime'],
                    end_time=commitment['endTime']
                )
        
        # Process children data
        for child_data in data['children']:
            # Create or update child record
            if child_data.get('id') and not child_data['id'].startswith('temp_'):
                child = Child.objects.get(id=child_data['id'], family=family)
            else:
                child = Child(family=family)
            
            child.name = child_data['section2']['name']
            child.age = child_data['section2']['age']
            
            # Save child questionnaire data
            child.questionnaire_responses = {
                'section2': child_data['section2'],
                'section4': child_data['section4'],
                'section6': child_data['section6']
            }
            
            # Process child learning preferences
            child.learning_preferences = {
                'bestLearningTimes': child_data['section6']['bestLearningTimes'],
                'homeschoolingHours': {
                    'startTime': child_data['section6']['startTime'],
                    'endTime': child_data['section6']['endTime']
                }
            }
            
            child.save()
            
            # Save child commitments if any
            if 'commitments' in child_data.get('section3Child', {}):
                # First delete existing commitments
                ChildCommitment.objects.filter(child=child).delete()
                
                # Then create new commitments
                for commitment in child_data['section3Child']['commitments']:
                    ChildCommitment.objects.create(
                        child=child,
                        name=commitment['name'],
                        frequency=commitment['frequency'],
                        days_of_week=json.dumps(commitment.get('daysOfWeek', [])),
                        start_time=commitment['startTime'],
                        end_time=commitment['endTime']
                    )
            
            # Save subjects
            if 'subjects' in child_data.get('section4', {}):
                # First delete existing subjects
                Subject.objects.filter(child=child).delete()
                
                # Then create new subjects
                for subject_data in child_data['section4']['subjects']:
                    # Basic subject fields
                    subject = Subject(
                        child=child,
                        name=subject_data['name'],
                        is_core_subject=subject_data['isCoreSubject'],
                        session_duration=subject_data['sessionDuration'],
                        frequency=subject_data['frequency'],
                        parent_involvement=subject_data['parentInvolvement'],
                        fixed_time_required=subject_data.get('fixedTime', {}).get('required', False),
                        interest_level=subject_data['interestLevel']
                    )
                    
                    # Set preferred time if applicable
                    if subject.fixed_time_required and 'preferredTime' in subject_data.get('fixedTime', {}):
                        subject.preferred_time = subject_data['fixedTime']['preferredTime']
                    
                    # Any additional metadata in JSONB field
                    metadata = {k: v for k, v in subject_data.items() 
                                if k not in ['name', 'isCoreSubject', 'sessionDuration', 
                                             'frequency', 'parentInvolvement', 'fixedTime', 
                                             'interest_level']}
                    if metadata:
                        subject.metadata = metadata
                    
                    subject.save()
        
        # Clear localStorage after successful submission
        # (This would happen in the frontend, just noted here for completeness)
        
        return True, family.id
        
    except Exception as e:
        # Handle errors appropriately
        print(f"Error saving questionnaire data: {str(e)}")
        # In a real application, you'd want proper error logging
        return False, str(e)
```

### 7. Schedule Generation and Storage

When generating a schedule, both relational tables and JSONB fields are used effectively:

```python
@transaction.atomic
def generate_schedule(child_id, week_start_date=None):
    """
    Generate a schedule for a child using ML model
    """
    try:
        # Get the child
        child = Child.objects.get(id=child_id)
        family = child.family
        
        # Determine week start date if not provided
        if not week_start_date:
            today = date.today()
            # Calculate Monday of current week
            week_start = today - timedelta(days=today.weekday())
            week_start_date = week_start
        
        # Get active ML model
        model = MLModel.objects.filter(is_active=True).first()
        if not model:
            raise ValueError("No active ML model found")
        
        # Get child's subjects
        subjects = Subject.objects.filter(child=child)
        
        # Get child's commitments
        child_commitments = ChildCommitment.objects.filter(child=child)
        
        # Get family commitments
        family_commitments = FamilyCommitment.objects.filter(family=family)
        
        # Call ML service to generate schedule
        # (In a real implementation, this would call your ML model)
        schedule_data = generate_schedule_with_ml(
            child=child,
            subjects=subjects,
            child_commitments=child_commitments,
            family_commitments=family_commitments,
            week_start_date=week_start_date,
            model=model
        )
        
        # Mark any existing active schedule for this week as inactive
        Schedule.objects.filter(
            child=child,
            week_start_date=week_start_date,
            is_active=True
        ).update(is_active=False)
        
        # Create new schedule record
        schedule = Schedule.objects.create(
            family=family,
            child=child,
            week_start_date=week_start_date,
            is_active=True,
            version=1,
            generation_method=model.name,
            schedule_data=schedule_data,
            patterns=schedule_data.get('patterns', {})
        )
        
        # Create schedule items in the relational table for efficient querying
        for day in schedule_data['days']:
            day_date = datetime.fromisoformat(day['date']).date()
            
            for item in day['items']:
                # Convert time strings to time objects
                start_time = datetime.strptime(item['startTime'], '%H:%M').time()
                end_time = datetime.strptime(item['endTime'], '%H:%M').time()
                
                # Determine referenced entities
                subject_id = item.get('subjectId')
                commitment_id = item.get('commitmentId')
                
                # Create schedule item
                schedule_item = ScheduleItem.objects.create(
                    schedule=schedule,
                    day_date=day_date,
                    item_type=item['type'],
                    name=item['name'],
                    start_time=start_time,
                    end_time=end_time,
                    is_external=item.get('isExternal', False),
                    subject_id=subject_id,
                    commitment_id=commitment_id,
                    item_data={
                        # Copy original item data
                        **item,
                        # Add ID reference back to this record
                        'scheduleItemId': None  # Will update after save
                    }
                )
                
                # Update the ID in both places
                schedule_item.item_data['scheduleItemId'] = schedule_item.id
                schedule_item.save(update_fields=['item_data'])
                
                # Update the ID in the schedule_data
                item['id'] = schedule_item.id
        
        # Save the updated schedule_data with all IDs
        schedule.schedule_data = schedule_data
        schedule.save(update_fields=['schedule_data'])
        
        return schedule
        
    except Exception as e:
        # Handle errors appropriately
        print(f"Error generating schedule: {str(e)}")
        # In a real application, you'd want proper error logging
        return None
```

## Django API Endpoints

Here are examples of key API endpoints for the BrightPath platform:

```python
# Example API endpoint for schedule retrieval
@api_view(['GET'])
@permission_classes([IsAuthenticated])
def get_child_schedule(request, child_id, week_start_date=None):
    # Determine week start date if not provided
    if not week_start_date:
        today = date.today()
        # Calculate Monday of current week
        week_start = today - timedelta(days=today.weekday())
        week_start_date = week_start.isoformat()
    else:
        week_start_date = datetime.strptime(week_start_date, '%Y-%m-%d').date()
    
    # Verify child belongs to user's family
    try:
        child = Child.objects.get(
            id=child_id,
            family__user=request.user
        )
    except Child.DoesNotExist:
        return Response({"error": "Child not found"}, status=404)
    
    # Get schedule
    try:
        schedule = Schedule.objects.get(
            child_id=child_id,
            week_start_date=week_start_date,
            is_active=True
        )
        
        # Return the complete schedule from JSONB field
        return Response(schedule.schedule_data)
        
    except Schedule.DoesNotExist:
        # Generate a new schedule if none exists
        schedule = generate_schedule(child_id, week_start_date)
        if schedule:
            return Response(schedule.schedule_data)
        else:
            return Response({"error": "Could not generate schedule"}, status=500)

# Example API endpoint for marking an activity complete
@api_view(['POST'])
@permission_classes([IsAuthenticated])
def mark_activity_complete(request, item_id):
    try:
        # Get the schedule item
        item = ScheduleItem.objects.select_related('schedule', 'schedule__family', 'schedule__child').get(id=item_id)
        
        # Verify user has permission
        if item.schedule.family.user_id != request.user.id:
            return Response({"error": "Not authorized"}, status=403)
        
        # Mark as complete
        completion_notes = request.data.get('notes', '')
        success = mark_schedule_item_completed(item_id, completion_notes)
        
        if success:
            return Response({"status": "success"})
        else:
            return Response({"error": "Failed to mark activity complete"}, status=500)
            
    except ScheduleItem.DoesNotExist:
        return Response({"error": "Activity not found"}, status=404)

# Example API endpoint for rescheduling an activity
@api_view(['POST'])
@permission_classes([IsAuthenticated])
def reschedule_activity(request, item_id):
    try:
        # Get the schedule item
        item = ScheduleItem.objects.select_related('schedule', 'schedule__family').get(id=item_id)
        
        # Verify user has permission
        if item.schedule.family.user_id != request.user.id:
            return Response({"error": "Not authorized"}, status=403)
        
        # Get rescheduling details
        new_day_date = request.data.get('newDate')
        new_start_time = request.data.get('newStartTime')
        new_end_time = request.data.get('newEndTime')
        
        if not all([new_day_date, new_start_time, new_end_time]):
            return Response({"error": "Missing required fields"}, status=400)
        
        # Convert string dates/times to proper types
        new_day_date = datetime.strptime(new_day_date, '%Y-%m-%d').date()
        new_start_time = datetime.strptime(new_start_time, '%H:%M').time()
        new_end_time = datetime.strptime(new_end_time, '%H:%M').time()
        
        # Reschedule the item
        success = reschedule_item(item_id, new_day_date, new_start_time, new_end_time)
        
        if success:
            return Response({"status": "success"})
        else:
            return Response({"error": "Failed to reschedule activity"}, status=500)
            
    except ScheduleItem.DoesNotExist:
        return Response({"error": "Activity not found"}, status=404)
```

## Advantages of PostgreSQL with JSONB Approach

1. **Single Database to Maintain**: All data stored in PostgreSQL eliminates the need to synchronize across multiple databases.

2. **Flexible Schema Evolution**: JSONB fields can accommodate changing requirements without schema migrations.

3. **Strong Transactional Guarantees**: ACID properties ensure data consistency even with complex operations.

4. **Powerful Querying Capabilities**: Combine relational queries with JSON path expressions for complex data retrieval.

5. **Efficient Indexing Options**: GIN and B-tree indexes for JSONB enable high-performance queries.

6. **Schema Validation When Needed**: Core fields have schema enforcement while flexible data can evolve freely.

7. **Simplified Backup and Recovery**: Single database backup strategy with point-in-time recovery.

8. **Optimized for Different Data Access Patterns**:
   - Relational tables for data that needs to be queried frequently
   - JSONB for complex hierarchical data and infrequently queried details

## Migration Path from Local Storage to Server-Side

In future iterations, the questionnaire progress tracking can be enhanced by moving from localStorage to server-side storage:

1. **Session Table Addition**:
```sql
CREATE TABLE questionnaire_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    family_id INTEGER REFERENCES families(id) ON DELETE CASCADE,
    session_data JSONB NOT NULL,
    last_updated TIMESTAMP NOT NULL DEFAULT NOW(),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_questionnaire_sessions_user_id ON questionnaire_sessions(user_id);
CREATE INDEX idx_questionnaire_sessions_last_updated ON questionnaire_sessions(last_updated);
```

2. **API Endpoints for Session Management**:
```python
@api_view(['POST'])
@permission_classes([IsAuthenticated])
def save_questionnaire_progress(request):
    """Save questionnaire progress to server"""
    user_id = request.user.id
    session_data = request.data.get('sessionData')
    
    if not session_data:
        return Response({"error": "No session data provided"}, status=400)
    
    # Check if session already exists
    session = QuestionnaireSession.objects.filter(user_id=user_id).first()
    
    if session:
        # Update existing session
        session.session_data = session_data
        session.last_updated = timezone.now()
        session.save()
    else:
        # Create new session
        session = QuestionnaireSession.objects.create(
            user_id=user_id,
            session_data=session_data
        )
    
    return Response({"status": "success", "sessionId": session.id})

@api_view(['GET'])
@permission_classes([IsAuthenticated])
def get_questionnaire_progress(request):
    """Get saved questionnaire progress from server"""
    user_id = request.user.id
    
    try:
        session = QuestionnaireSession.objects.get(user_id=user_id)
        return Response(session.session_data)
    except QuestionnaireSession.DoesNotExist:
        return Response({"status": "not_found"}, status=404)
```

## Conclusion

The revised database design for BrightPath leverages PostgreSQL with JSONB fields to provide both the structured relationships of a relational database and the flexibility of a document database in a single system. This approach:

1. **Simplifies the Technology Stack**: Single database system to maintain and scale
2. **Provides Schema Flexibility**: Accommodates evolving requirements without sacrificing data integrity
3. **Optimizes for Different Access Patterns**: Structured tables for frequent queries, JSONB for complex hierarchical data
4. **Maintains ACID Compliance**: Strong transaction guarantees for all operations
5. **Offers Powerful Querying**: Combines SQL and JSON path expressions for complex data retrieval

The use of browser localStorage for questionnaire progress in the initial prototype remains an effective approach to simplify early development, with a clear migration path to server-side storage in future iterations.

This hybrid relational-document approach using PostgreSQL with JSONB provides an ideal balance between structure and flexibility for the BrightPath homeschooling scheduling platform, supporting all current requirements while allowing for future growth and feature enhancements.