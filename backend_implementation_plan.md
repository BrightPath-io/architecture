# BrightPath Backend Implementation Plan

## Overview

This document outlines the detailed implementation plan for the BrightPath backend. The backend will be built using Django and Django REST Framework, providing a robust API for the frontend application and integrating with machine learning services for schedule generation and optimization.

## Technology Stack

- **Framework**: Django 4.2+ with Python 3.10+
- **API Framework**: Django REST Framework
- **Database**: PostgreSQL 14+
- **Authentication**: JWT-based authentication
- **ML Integration**: scikit-learn and TensorFlow
- **Task Queue**: Celery with Redis
- **Testing**: pytest
- **Documentation**: drf-spectacular for OpenAPI/Swagger
- **Containerization**: Docker
- **CI/CD**: GitHub Actions

## Project Structure

```
api/
├── brightpath/
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   ├── production.py
│   │   └── test.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── apps/
│   ├── users/
│   │   ├── migrations/
│   │   ├── api/
│   │   │   ├── serializers.py
│   │   │   ├── views.py
│   │   │   └── urls.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── services.py
│   │   └── tests.py
│   ├── children/
│   │   ├── migrations/
│   │   ├── api/
│   │   │   ├── serializers.py
│   │   │   ├── views.py
│   │   │   └── urls.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── services.py
│   │   └── tests.py
│   ├── schedules/
│   │   ├── migrations/
│   │   ├── api/
│   │   │   ├── serializers.py
│   │   │   ├── views.py
│   │   │   └── urls.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── services.py
│   │   └── tests.py
│   ├── subjects/
│   │   ├── migrations/
│   │   ├── api/
│   │   │   ├── serializers.py
│   │   │   ├── views.py
│   │   │   └── urls.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── services.py
│   │   └── tests.py
│   ├── feedback/
│   │   ├── migrations/
│   │   ├── api/
│   │   │   ├── serializers.py
│   │   │   ├── views.py
│   │   │   └── urls.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── services.py
│   │   └── tests.py
│   └── surveys/
│       ├── migrations/
│       ├── api/
│       │   ├── serializers.py
│       │   ├── views.py
│       │   └── urls.py
│       ├── models.py
│       ├── admin.py
│       ├── services.py
│       └── tests.py
├── ml/
│   ├── generator/
│   │   ├── model.py
│   │   ├── training.py
│   │   └── inference.py
│   ├── reward/
│   │   ├── model.py
│   │   ├── training.py
│   │   └── inference.py
│   ├── data/
│   │   ├── preprocessing.py
│   │   └── synthetic.py
│   └── utils/
│       ├── evaluation.py
│       └── visualization.py
├── utils/
│   ├── permissions.py
│   ├── validators.py
│   ├── pagination.py
│   └── exceptions.py
├── tasks/
│   ├── schedule_generation.py
│   ├── model_training.py
│   └── notifications.py
├── templates/
│   ├── emails/
│   └── admin/
├── manage.py
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
├── Dockerfile
├── docker-compose.yml
├── pytest.ini
└── .env.example
```

## Database Models

### User App

```python
# users/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    """Extended user model for BrightPath."""
    email = models.EmailField(unique=True)
    is_onboarded = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Required for using email as the username field
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    class Meta:
        verbose_name = 'User'
        verbose_name_plural = 'Users'

    def __str__(self):
        return self.email

class UserPreference(models.Model):
    """User preferences for the application."""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='preferences')
    key = models.CharField(max_length=100)
    value = models.JSONField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = ('user', 'key')
        verbose_name = 'User Preference'
        verbose_name_plural = 'User Preferences'

    def __str__(self):
        return f"{self.user.email} - {self.key}"
```

### Children App

```python
# children/models.py
from django.db import models
from apps.users.models import User

class Child(models.Model):
    """Child model for homeschooling students."""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='children')
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100, blank=True)
    birth_date = models.DateField()
    grade_level = models.CharField(max_length=50, blank=True)
    notes = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Child'
        verbose_name_plural = 'Children'

    def __str__(self):
        return f"{self.first_name} {self.last_name or ''}"

    @property
    def age(self):
        """Calculate child's age based on birth date."""
        from datetime import date
        today = date.today()
        return today.year - self.birth_date.year - (
            (today.month, today.day) < (self.birth_date.month, self.birth_date.day)
        )
```

### Subjects App

```python
# subjects/models.py
from django.db import models
from apps.users.models import User

class Subject(models.Model):
    """Subject model for educational subjects."""
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE, null=True, blank=True, related_name='custom_subjects')
    is_standard = models.BooleanField(default=True)
    color = models.CharField(max_length=7, default="#FFFFFF")  # Hex color code
    icon = models.CharField(max_length=50, blank=True)  # Icon identifier
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Subject'
        verbose_name_plural = 'Subjects'
        unique_together = ('name', 'user')

    def __str__(self):
        return self.name

class Resource(models.Model):
    """Educational resources linked to subjects."""
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    url = models.URLField(blank=True)
    subject = models.ForeignKey(Subject, on_delete=models.CASCADE, related_name='resources')
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='resources')
    resource_type = models.CharField(max_length=50)  # e.g., video, article, worksheet
    is_public = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Resource'
        verbose_name_plural = 'Resources'

    def __str__(self):
        return self.title
```

### Schedules App

```python
# schedules/models.py
from django.db import models
from apps.users.models import User
from apps.children.models import Child
from apps.subjects.models import Subject

class Schedule(models.Model):
    """Schedule model for storing generated schedules."""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='schedules')
    child = models.ForeignKey(Child, on_delete=models.CASCADE, related_name='schedules')
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    schedule_data = models.JSONField()  # Stores the full schedule configuration
    is_active = models.BooleanField(default=True)
    start_date = models.DateField()
    end_date = models.DateField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Schedule'
        verbose_name_plural = 'Schedules'

    def __str__(self):
        return f"{self.title} - {self.child.first_name}"

class ScheduleItem(models.Model):
    """Individual items within a schedule."""
    schedule = models.ForeignKey(Schedule, on_delete=models.CASCADE, related_name='items')
    subject = models.ForeignKey(Subject, on_delete=models.CASCADE, related_name='schedule_items')
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    start_time = models.TimeField()
    end_time = models.TimeField()
    day_of_week = models.IntegerField()  # 0=Monday, 6=Sunday
    is_recurring = models.BooleanField(default=True)
    specific_date = models.DateField(null=True, blank=True)  # For non-recurring items
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Schedule Item'
        verbose_name_plural = 'Schedule Items'

    def __str__(self):
        return f"{self.title} - {self.subject.name}"

class Activity(models.Model):
    """Completed activities tracked by the user."""
    schedule_item = models.ForeignKey(ScheduleItem, on_delete=models.CASCADE, related_name='activities')
    child = models.ForeignKey(Child, on_delete=models.CASCADE, related_name='activities')
    completed_at = models.DateTimeField()
    duration_minutes = models.IntegerField()
    notes = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Activity'
        verbose_name_plural = 'Activities'

    def __str__(self):
        return f"{self.schedule_item.title} - {self.completed_at.date()}"
```

### Surveys App

```python
# surveys/models.py
from django.db import models
from apps.users.models import User

class Category(models.Model):
    """Categories for grouping questionnaire questions."""
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    order = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Category'
        verbose_name_plural = 'Categories'
        ordering = ['order', 'name']

    def __str__(self):
        return self.name

class Question(models.Model):
    """Questions used in onboarding and feedback forms."""
    category = models.ForeignKey(Category, on_delete=models.CASCADE, related_name='questions')
    text = models.TextField()
    description = models.TextField(blank=True)
    weight = models.IntegerField(default=1)
    is_active = models.BooleanField(default=True)
    order = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Question'
        verbose_name_plural = 'Questions'
        ordering = ['category', 'order', 'id']

    def __str__(self):
        return self.text[:50]

class SurveySession(models.Model):
    """Tracks user onboarding questionnaire progress."""
    STATUS_CHOICES = (
        ('in_progress', 'In Progress'),
        ('completed', 'Completed'),
        ('abandoned', 'Abandoned'),
    )

    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='survey_sessions')
    started_at = models.DateTimeField(auto_now_add=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='in_progress')

    class Meta:
        verbose_name = 'Survey Session'
        verbose_name_plural = 'Survey Sessions'

    def __str__(self):
        return f"{self.user.email} - {self.started_at.date()}"

class SurveyResponse(models.Model):
    """Stores user answers to onboarding questions."""
    session = models.ForeignKey(SurveySession, on_delete=models.CASCADE, related_name='responses')
    question = models.ForeignKey(Question, on_delete=models.CASCADE, related_name='responses')
    rating = models.IntegerField()  # For Likert scale answers (1-5)
    text_response = models.TextField(blank=True)  # For open-ended questions
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name = 'Survey Response'
        verbose_name_plural = 'Survey Responses'
        unique_together = ('session', 'question')

    def __str__(self):
        return f"{self.session.user.email} - Q{self.question.id}"
```

### Feedback App

```python
# feedback/models.py
from django.db import models
from apps.users.models import User
from apps.schedules.models import Schedule

class Feedback(models.Model):
    """Stores user ratings & comments on generated schedules."""
    schedule = models.ForeignKey(Schedule, on_delete=models.CASCADE, related_name='feedback')
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='feedback')
    star_rating = models.IntegerField()  # 1-5 rating
    feedback_data = models.JSONField()  # Stores Likert scale & text feedback
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name = 'Feedback'
        verbose_name_plural = 'Feedback'

    def __str__(self):
        return f"{self.user.email} - {self.schedule.title} - {self.star_rating}★"
```

## API Endpoints

### Authentication Endpoints

```
POST /api/auth/register/ - Register a new user
POST /api/auth/login/ - Login and get JWT token
POST /api/auth/token/refresh/ - Refresh JWT token
POST /api/auth/password/reset/ - Request password reset
POST /api/auth/password/reset/confirm/ - Confirm password reset
```

### User Endpoints

```
GET /api/users/me/ - Get current user profile
PATCH /api/users/me/ - Update current user profile
GET /api/users/preferences/ - Get user preferences
PUT /api/users/preferences/ - Update user preferences
```

### Children Endpoints

```
GET /api/children/ - List all children for current user
POST /api/children/ - Create a new child
GET /api/children/{id}/ - Get child details
PUT /api/children/{id}/ - Update child details
DELETE /api/children/{id}/ - Delete a child
```

### Subjects Endpoints

```
GET /api/subjects/ - List all subjects (standard + user's custom)
POST /api/subjects/ - Create a custom subject
GET /api/subjects/{id}/ - Get subject details
PUT /api/subjects/{id}/ - Update subject (only if user's custom)
DELETE /api/subjects/{id}/ - Delete subject (only if user's custom)

GET /api/subjects/{id}/resources/ - List resources for a subject
POST /api/subjects/{id}/resources/ - Add a resource to a subject
```

### Schedule Endpoints

```
GET /api/schedules/ - List all schedules for current user
POST /api/schedules/ - Create a new schedule
GET /api/schedules/{id}/ - Get schedule details
PUT /api/schedules/{id}/ - Update schedule
DELETE /api/schedules/{id}/ - Delete a schedule

GET /api/schedules/{id}/items/ - List items in a schedule
POST /api/schedules/{id}/items/ - Add an item to a schedule
PUT /api/schedules/{id}/items/{item_id}/ - Update a schedule item
DELETE /api/schedules/{id}/items/{item_id}/ - Delete a schedule item

POST /api/schedules/generate/ - Generate a new schedule based on preferences
```

### Survey Endpoints

```
GET /api/surveys/categories/ - List all question categories
GET /api/surveys/questions/ - List all questions (filterable by category)
POST /api/surveys/sessions/ - Start a new survey session
GET /api/surveys/sessions/{id}/ - Get session details
PATCH /api/surveys/sessions/{id}/ - Update session status

POST /api/surveys/sessions/{id}/responses/ - Submit responses for a session
GET /api/surveys/sessions/{id}/responses/ - Get all responses for a session
```

### Feedback Endpoints

```
POST /api/feedback/ - Submit feedback for a schedule
GET /api/feedback/ - List all feedback submitted by current user
GET /api/feedback/{id}/ - Get feedback details
```

### ML Endpoints

```
POST /api/ml/generate-schedule/ - Generate a schedule using ML model
POST /api/ml/evaluate-schedule/ - Evaluate a schedule using reward model
GET /api/ml/model-status/ - Get training status of ML models
```

## Authentication and Permissions

### JWT Authentication

The system will use JWT (JSON Web Tokens) for authentication with the following configuration:

```python
# settings/base.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=14),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
}
```

### Custom Permissions

```python
# utils/permissions.py
from rest_framework import permissions

class IsOwner(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to view or edit it.
    """
    def has_object_permission(self, request, view, obj):
        # Check if the object has a user attribute
        if hasattr(obj, 'user'):
            return obj.user == request.user

        # For objects with different owner field
        if hasattr(obj, 'owner'):
            return obj.owner == request.user

        return False

class IsParentOfChild(permissions.BasePermission):
    """
    Custom permission to only allow parents to access their children's data.
    """
    def has_object_permission(self, request, view, obj):
        # Check if the object is a Child
        if hasattr(obj, 'user'):
            return obj.user == request.user

        # Check if the object has a child attribute
        if hasattr(obj, 'child'):
            return obj.child.user == request.user

        return False
```

## Machine Learning Integration

### Schedule Generator Model

```python
# ml/generator/model.py
import numpy as np
import pandas as pd
from sklearn.ensemble import RandomForestRegressor
import joblib

class ScheduleGeneratorModel:
    """Model for generating personalized schedules based on user preferences."""

    def __init__(self, model_path=None):
        """Initialize the model, optionally loading from a file."""
        if model_path:
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestRegressor(
                n_estimators=100,
                max_depth=10,
                random_state=42
            )

    def preprocess_features(self, user_data):
        """Convert user data to model features."""
        # Extract features from user questionnaire responses
        features = []

        # Flexibility vs. Structure (questions 1-5)
        flexibility_score = np.mean([
            user_data.get('q1', 3),
            6 - user_data.get('q2', 3),  # Reverse scale
            user_data.get('q3', 3),
            6 - user_data.get('q4', 3),  # Reverse scale
            user_data.get('q5', 3)
        ])
        features.append(flexibility_score)

        # Subject Rotation (questions 6-10)
        rotation_score = np.mean([
            user_data.get('q6', 3),
            6 - user_data.get('q7', 3),  # Reverse scale
            user_data.get('q8', 3),
            user_data.get('q9', 3),
            user_data.get('q10', 3)
        ])
        features.append(rotation_score)

        # Time Management (questions 11-15)
        time_score = np.mean([
            user_data.get('q11', 3),
            user_data.get('q12', 3),
            6 - user_data.get('q13', 3),  # Reverse scale
            user_data.get('q14', 3),
            user_data.get('q15', 3)
        ])
        features.append(time_score)

        # Long-Term Scheduling (questions 16-20)
        longterm_score = np.mean([
            user_data.get('q16', 3),
            user_data.get('q17', 3),
            user_data.get('q18', 3),
            user_data.get('q19', 3),
            user_data.get('q20', 3)
        ])
        features.append(longterm_score)

        # Prioritization (questions 21-25)
        priority_score = np.mean([
            user_data.get('q21', 3),
            6 - user_data.get('q22', 3),  # Reverse scale
            user_data.get('q23', 3),
            user_data.get('q24', 3),
            user_data.get('q25', 3)
        ])
        features.append(priority_score)

        # Child age (if available)
        if 'child_age' in user_data:
            features.append(user_data['child_age'])
        else:
            features.append(8)  # Default age

        return np.array(features).reshape(1, -1)

    def train(self, X, y):
        """Train the model on the provided data."""
        self.model.fit(X, y)

    def predict(self, user_data):
        """Generate a schedule based on user data."""
        features = self.preprocess_features(user_data)

        # Predict schedule parameters
        params = self.model.predict(features)[0]

        # Convert parameters to schedule configuration
        schedule = self._create_schedule_from_params(params, user_data)

        return schedule

    def _create_schedule_from_params(self, params, user_data):
        """Convert model parameters to a concrete schedule."""
        # This is a simplified example - actual implementation would be more complex

        # Extract parameters
        structure_level = params[0]  # 0-1 scale, higher means more structured
        block_duration = int(20 + params[1] * 40)  # 20-60 minute blocks
        breaks_frequency = int(2 + params[2] * 3)  # After every 2-5 blocks

        # Create schedule structure
        schedule = {
            'meta': {
                'structure_level': float(structure_level),
                'block_duration_minutes': block_duration,
                'breaks_frequency': breaks_frequency,
            },
            'days': []
        }

        # Generate daily schedules
        for day in range(5):  # Monday to Friday
            daily_schedule = self._generate_daily_schedule(
                day,
                structure_level,
                block_duration,
                breaks_frequency,
                user_data
            )
            schedule['days'].append(daily_schedule)

        return schedule

    def _generate_daily_schedule(self, day, structure_level, block_duration, breaks_frequency, user_data):
        """Generate a daily schedule based on parameters."""
        # This is a simplified example

        # Start time depends on structure level
        if structure_level > 0.7:
            start_hour = 8
        elif structure_level > 0.4:
            start_hour = 9
        else:
            start_hour = 10

        # Get subjects (would come from user data in real implementation)
        subjects = user_data.get('subjects', ['Math', 'Reading', 'Science', 'History', 'Art'])

        # Create blocks
        blocks = []
        current_hour = start_hour
        current_minute = 0

        for i in range(len(subjects)):
            # Add a break after specified number of blocks
            if i > 0 and i % breaks_frequency == 0:
                # Add a break
                blocks.append({
                    'type': 'break',
                    'start_time': f"{current_hour:02d}:{current_minute:02d}",
                    'duration_minutes': 15
                })

                # Update time
                current_minute += 15
                if current_minute >= 60:
                    current_hour += 1
                    current_minute -= 60

            # Add a subject block
            blocks.append({
                'type': 'subject',
                'subject': subjects[i % len(subjects)],
                'start_time': f"{current_hour:02d}:{current_minute:02d}",
                'duration_minutes': block_duration
            })

            # Update time
            current_minute += block_duration
            while current_minute >= 60:
                current_hour += 1
                current_minute -= 60

        # Create the daily schedule
        daily_schedule = {
            'day': day,
            'day_name': ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'][day],
            'start_time': f"{start_hour:02d}:00",
            'blocks': blocks
        }

        return daily_schedule

    def save(self, model_path):
        """Save the model to a file."""
        joblib.dump(self.model, model_path)
```

### Reward Model

```python
# ml/reward/model.py
import numpy as np
import pandas as pd
from sklearn.ensemble import GradientBoostingRegressor
import joblib

class ScheduleRewardModel:
    """Model for evaluating schedule quality based on user feedback."""

    def __init__(self, model_path=None):
        """Initialize the model, optionally loading from a file."""
        if model_path:
            self.model = joblib.load(model_path)
        else:
            self.model = GradientBoostingRegressor(
                n_estimators=100,
                max_depth=5,
                learning_rate=0.1,
                random_state=42
            )

    def preprocess_features(self, schedule_data, feedback_data):
        """Convert schedule and feedback data to model features."""
        features = []

        # Extract schedule features
        meta = schedule_data.get('meta', {})
        features.append(meta.get('structure_level', 0.5))
        features.append(meta.get('block_duration_minutes', 30) / 60.0)  # Normalize to 0-1
        features.append(meta.get('breaks_frequency', 3) / 5.0)  # Normalize to 0-1

        # Extract feedback features
        likert_responses = feedback_data.get('likert_responses', {})
        for q_id in range(1, 6):  # Assuming 5 Likert questions
            features.append(likert_responses.get(f'q{q_id}', 3) / 5.0)  # Normalize to 0-1

        # Add any adjustments made to the schedule
        adjustments = feedback_data.get('adjustments', {})
        features.append(adjustments.get('time_shifted', False) * 1.0)
        features.append(adjustments.get('blocks_resized', False) * 1.0)
        features.append(adjustments.get('subjects_reordered', False) * 1.0)

        return np.array(features).reshape(1, -1)

    def train(self, X, y):
        """Train the model on the provided data."""
        self.model.fit(X, y)

    def predict(self, schedule_data, feedback_data=None):
        """
        Predict the reward score for a schedule.
        If feedback_data is provided, it's used for evaluation.
        Otherwise, a default feedback is assumed.
        """
        if feedback_data is None:
            # Create default feedback data for prediction
            feedback_data = {
                'likert_responses': {f'q{i}': 3 for i in range(1, 6)},
                'adjustments': {
                    'time_shifted': False,
                    'blocks_resized': False,
                    'subjects_reordered': False
                }
            }

        features = self.preprocess_features(schedule_data, feedback_data)
        reward = self.model.predict(features)[0]

        # Ensure reward is between 0 and 1
        reward = max(0, min(1, reward))

        return reward

    def save(self, model_path):
        """Save the model to a file."""
        joblib.dump(self.model, model_path)
```

## API Views Implementation

### Schedule Generation API

```python
# apps/schedules/api/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django.shortcuts import get_object_or_404

from apps.schedules.models import Schedule, ScheduleItem
from apps.children.models import Child
from apps.surveys.models import SurveySession, SurveyResponse
from apps.schedules.api.serializers import (
    ScheduleSerializer,
    ScheduleItemSerializer,
    ScheduleGenerationSerializer
)
from ml.generator.model import ScheduleGeneratorModel

class ScheduleViewSet(viewsets.ModelViewSet):
    """ViewSet for managing schedules."""
    serializer_class = ScheduleSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        """Return schedules for the current user."""
        return Schedule.objects.filter(user=self.request.user)

    def perform_create(self, serializer):
        """Set the user when creating a schedule."""
        serializer.save(user=self.request.user)

    @action(detail=False, methods=['post'])
    def generate(self, request):
        """Generate a new schedule using the ML model."""
        serializer = ScheduleGenerationSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        # Get the child
        child_id = serializer.validated_data['child_id']
        child = get_object_or_404(Child, id=child_id, user=request.user)

        # Get user preferences from survey responses
        try:
            session = SurveySession.objects.filter(
                user=request.user,
                status='completed'
            ).latest('completed_at')

            responses = SurveyResponse.objects.filter(session=session)
            user_data = {
                f'q{response.question.id}': response.rating
                for response in responses
            }
        except SurveySession.DoesNotExist:
            # Use default values if no survey completed
            user_data = {f'q{i}': 3 for i in range(1, 26)}

        # Add child age
        user_data['child_age'] = child.age

        # Add subjects
        user_data['subjects'] = list(
            serializer.validated_data.get('subjects', [])
        )

        # Generate schedule
        generator = ScheduleGeneratorModel()
        schedule_data = generator.predict(user_data)

        # Create schedule in database
        schedule = Schedule.objects.create(
            user=request.user,
            child=child,
            title=serializer.validated_data.get('title', f"{child.first_name}'s Schedule"),
            description=serializer.validated_data.get('description', ''),
            schedule_data=schedule_data,
            start_date=serializer.validated_data.get('start_date'),
            end_date=serializer.validated_data.get('end_date')
        )

        # Create schedule items
        for day_schedule in schedule_data['days']:
            day = day_schedule['day']
            for block in day_schedule['blocks']:
                if block['type'] == 'subject':
                    # Find or create the subject
                    subject_name = block['subject']
                    subject = Subject.objects.filter(
                        name__iexact=subject_name
                    ).first()

                    if not subject:
                        # Use a default subject if not found
                        subject = Subject.objects.get(name='General')

                    # Parse start time
                    hours, minutes = map(int, block['start_time'].split(':'))

                    # Create schedule item
                    ScheduleItem.objects.create(
                        schedule=schedule,
                        subject=subject,
                        title=subject_name,
                        start_time=f"{hours:02d}:{minutes:02d}:00",
                        end_time=f"{hours:02d}:{(minutes + block['duration_minutes']) % 60:02d}:00",
                        day_of_week=day,
                        is_recurring=True
                    )

        return Response(
            ScheduleSerializer(schedule).data,
            status=status.HTTP_201_CREATED
        )
```

### Feedback API

```python
# apps/feedback/api/views.py
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

from apps.feedback.models import Feedback
from apps.schedules.models import Schedule
from apps.feedback.api.serializers import FeedbackSerializer
from ml.reward.model import ScheduleRewardModel

class FeedbackViewSet(viewsets.ModelViewSet):
    """ViewSet for managing feedback."""
    serializer_class = FeedbackSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        """Return feedback for the current user."""
        return Feedback.objects.filter(user=self.request.user)

    def perform_create(self, serializer):
        """Set the user when creating feedback and update reward model."""
        feedback = serializer.save(user=self.request.user)

        # Get the schedule
        schedule = feedback.schedule

        # Update the reward model with this feedback
        try:
            reward_model = ScheduleRewardModel()

            # Calculate reward score
            reward_score = reward_model.predict(
                schedule.schedule_data,
                feedback.feedback_data
            )

            # In a real implementation, we would:
            # 1. Store this reward score
            # 2. Periodically retrain the reward model
            # 3. Use the reward model to improve the generator model

            # For now, just log the score
            print(f"Reward score for schedule {schedule.id}: {reward_score}")

        except Exception as e:
            # Log the error but don't fail the request
            print(f"Error updating reward model: {e}")
```

## Celery Tasks

### Schedule Generation Task

```python
# tasks/schedule_generation.py
from celery import shared_task
from apps.schedules.models import Schedule
from apps.children.models import Child
from apps.subjects.models import Subject
from apps.surveys.models import SurveySession, SurveyResponse
from ml.generator.model import ScheduleGeneratorModel

@shared_task
def generate_schedule(user_id, child_id, title, description, start_date, end_date, subjects=None):
    """
    Generate a schedule using the ML model.
    This task is designed to run asynchronously for potentially long-running operations.
    """
    from django.contrib.auth import get_user_model
    User = get_user_model()

    try:
        # Get user and child
        user = User.objects.get(id=user_id)
        child = Child.objects.get(id=child_id, user=user)

        # Get user preferences from survey responses
        try:
            session = SurveySession.objects.filter(
                user=user,
                status='completed'
            ).latest('completed_at')

            responses = SurveyResponse.objects.filter(session=session)
            user_data = {
                f'q{response.question.id}': response.rating
                for response in responses
            }
        except SurveySession.DoesNotExist:
            # Use default values if no survey completed
            user_data = {f'q{i}': 3 for i in range(1, 26)}

        # Add child age
        user_data['child_age'] = child.age

        # Add subjects
        if subjects:
            user_data['subjects'] = subjects
        else:
            # Use default subjects
            user_data['subjects'] = ['Math', 'Reading', 'Science', 'History', 'Art']

        # Generate schedule
        generator = ScheduleGeneratorModel()
        schedule_data = generator.predict(user_data)

        # Create schedule in database
        schedule = Schedule.objects.create(
            user=user,
            child=child,
            title=title or f"{child.first_name}'s Schedule",
            description=description or '',
            schedule_data=schedule_data,
            start_date=start_date,
            end_date=end_date
        )

        # Create schedule items
        for day_schedule in schedule_data['days']:
            day = day_schedule['day']
            for block in day_schedule['blocks']:
                if block['type'] == 'subject':
                    # Find or create the subject
                    subject_name = block['subject']
                    subject = Subject.objects.filter(
                        name__iexact=subject_name
                    ).first()

                    if not subject:
                        # Use a default subject if not found
                        subject = Subject.objects.get(name='General')

                    # Parse start time
                    hours, minutes = map(int, block['start_time'].split(':'))

                    # Calculate end time
                    total_minutes = minutes + block['duration_minutes']
                    end_hours = hours + (total_minutes // 60)
                    end_minutes = total_minutes % 60

                    # Create schedule item
                    ScheduleItem.objects.create(
                        schedule=schedule,
                        subject=subject,
                        title=subject_name,
                        start_time=f"{hours:02d}:{minutes:02d}:00",
                        end_time=f"{end_hours:02d}:{end_minutes:02d}:00",
                        day_of_week=day,
                        is_recurring=True
                    )

        return schedule.id

    except Exception as e:
        # Log the error
        print(f"Error generating schedule: {e}")
        raise
```

### Model Training Task

```python
# tasks/model_training.py
from celery import shared_task
import numpy as np
import pandas as pd
from ml.generator.model import ScheduleGeneratorModel
from ml.reward.model import ScheduleRewardModel
from apps.feedback.models import Feedback

@shared_task
def train_reward_model():
    """
    Train the reward model using collected feedback.
    This task should be scheduled to run periodically.
    """
    try:
        # Get all feedback with associated schedules
        feedbacks = Feedback.objects.select_related('schedule').all()

        if not feedbacks:
            print("No feedback data available for training")
            return

        # Prepare training data
        X = []
        y = []

        reward_model = ScheduleRewardModel()

        for feedback in feedbacks:
            try:
                # Extract features
                features = reward_model.preprocess_features(
                    feedback.schedule.schedule_data,
                    feedback.feedback_data
                )

                # Use star rating as target
                target = feedback.star_rating / 5.0  # Normalize to 0-1

                X.append(features[0])  # Flatten the feature array
                y.append(target)

            except Exception as e:
                print(f"Error processing feedback {feedback.id}: {e}")
                continue

        if not X:
            print("No valid training examples extracted")
            return

        # Convert to numpy arrays
        X = np.array(X)
        y = np.array(y)

        # Train the model
        reward_model.train(X, y)

        # Save the model
        reward_model.save('ml/models/reward_model.joblib')

        print(f"Reward model trained successfully with {len(X)} examples")

        # Now update the generator model based on the reward model
        train_generator_model()

    except Exception as e:
        print(f"Error training reward model: {e}")
        raise

@shared_task
def train_generator_model():
    """
    Train the generator model using the reward model for guidance.
    This implements a simple form of reinforcement learning.
    """
    try:
        # Load the reward model
        reward_model = ScheduleRewardModel('ml/models/reward_model.joblib')

        # Create or load the generator model
        generator_model = ScheduleGeneratorModel()

        # In a real implementation, we would:
        # 1. Generate many candidate schedules
        # 2. Evaluate them with the reward model
        # 3. Use the evaluations to update the generator model

        # For this simplified example, we'll just save the current model
        generator_model.save('ml/models/generator_model.joblib')

        print("Generator model updated successfully")

    except Exception as e:
        print(f"Error training generator model: {e}")
        raise
```

## Testing Strategy

### Unit Tests

```python
# apps/schedules/tests.py
from django.test import TestCase
from django.contrib.auth import get_user_model
from rest_framework.test import APIClient
from rest_framework import status
from apps.schedules.models import Schedule, ScheduleItem
from apps.children.models import Child
from apps.subjects.models import Subject
import json
from datetime import date

User = get_user_model()

class ScheduleModelTests(TestCase):
    """Test the Schedule model."""

    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.child = Child.objects.create(
            user=self.user,
            first_name='Test',
            birth_date=date(2015, 1, 1)
        )
        self.schedule = Schedule.objects.create(
            user=self.user,
            child=self.child,
            title='Test Schedule',
            schedule_data={'test': 'data'},
            start_date=date(2023, 1, 1)
        )

    def test_schedule_creation(self):
        """Test creating a schedule."""
        self.assertEqual(self.schedule.title, 'Test Schedule')
        self.assertEqual(self.schedule.user, self.user)
        self.assertEqual(self.schedule.child, self.child)
        self.assertEqual(self.schedule.schedule_data, {'test': 'data'})
        self.assertEqual(self.schedule.start_date, date(2023, 1, 1))
        self.assertTrue(self.schedule.is_active)

class ScheduleAPITests(TestCase):
    """Test the Schedule API."""

    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.client.force_authenticate(user=self.user)

        self.child = Child.objects.create(
            user=self.user,
            first_name='Test',
            birth_date=date(2015, 1, 1)
        )

        self.subject = Subject.objects.create(
            name='Math',
            is_standard=True
        )

        self.schedule_data = {
            'title': 'Test Schedule',
            'child': self.child.id,
            'schedule_data': {'test': 'data'},
            'start_date': '2023-01-01'
        }

    def test_create_schedule(self):
        """Test creating a schedule via the API."""
        response = self.client.post(
            '/api/schedules/',
            self.schedule_data,
            format='json'
        )
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Schedule.objects.count(), 1)
        self.assertEqual(Schedule.objects.get().title, 'Test Schedule')

    def test_get_schedule_list(self):
        """Test retrieving a list of schedules."""
        Schedule.objects.create(
            user=self.user,
            child=self.child,
            title='Test Schedule 1',
            schedule_data={'test': 'data1'},
            start_date=date(2023, 1, 1)
        )
        Schedule.objects.create(
            user=self.user,
            child=self.child,
            title='Test Schedule 2',
            schedule_data={'test': 'data2'},
            start_date=date(2023, 1, 2)
        )

        response = self.client.get('/api/schedules/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 2)

    def test_generate_schedule(self):
        """Test generating a schedule via the API."""
        data = {
            'child_id': self.child.id,
            'title': 'Generated Schedule',
            'start_date': '2023-01-01',
            'subjects': ['Math', 'Science', 'Reading']
        }

        response = self.client.post(
            '/api/schedules/generate/',
            data,
            format='json'
        )
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Schedule.objects.count(), 1)
        self.assertEqual(Schedule.objects.get().title, 'Generated Schedule')

        # Check that schedule items were created
        self.assertTrue(ScheduleItem.objects.exists())
```

## Deployment Configuration

### Docker Configuration

```dockerfile
# Dockerfile
FROM python:3.10-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory
WORKDIR /app

# Install dependencies
COPY requirements/base.txt requirements/base.txt
COPY requirements/production.txt requirements/production.txt
RUN pip install --no-cache-dir -r requirements/production.txt

# Copy project
COPY . .

# Run gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "brightpath.wsgi:application"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_DB=${DB_NAME}
    ports:
      - "5432:5432"

  web:
    build: .
    command: gunicorn brightpath.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - db
    env_file:
      - ./.env

  redis:
    image: redis:6
    ports:
      - "6379:6379"

  celery:
    build: .
    command: celery -A brightpath worker -l info
    volumes:
      - .:/app
    depends_on:
      - db
      - redis
    env_file:
      - ./.env

  celery-beat:
    build: .
    command: celery -A brightpath beat -l info
    volumes:
      - .:/app
    depends_on:
      - db
      - redis
    env_file:
      - ./.env

volumes:
  postgres_data:
```

## Implementation Timeline

### Phase 1: Foundation (Weeks 1-4)
- Set up project structure and configuration
- Implement user authentication
- Create basic models and migrations
- Set up API endpoints for core functionality
- Implement basic frontend integration

### Phase 2: Core Functionality (Weeks 5-8)
- Implement schedule management
- Create survey system
- Build child profile management
- Develop subject and resource management
- Implement basic ML models

### Phase 3: ML Integration (Weeks 9-12)
- Implement schedule generation
- Create feedback collection system
- Develop reward model
- Integrate ML models with API
- Set up Celery tasks for background processing

### Phase 4: Testing and Refinement (Weeks 13-16)
- Write comprehensive tests
- Optimize performance
- Refine API endpoints
- Improve error handling
- Enhance security

### Phase 5: Deployment and Documentation (Weeks 17-20)
- Set up Docker configuration
- Create CI/CD pipeline
- Write API documentation
- Prepare deployment scripts
- Conduct final testing

## Conclusion

This backend implementation plan provides a comprehensive roadmap for developing the BrightPath backend. By following this structured approach, we can ensure a robust, scalable, and maintainable backend that integrates seamlessly with the frontend and provides powerful ML-driven scheduling capabilities.