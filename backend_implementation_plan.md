# BrightPath Backend Implementation Plan

## Overview

This document outlines the detailed implementation plan for the BrightPath homeschooling scheduling platform backend. The backend will be built using Django and Django REST Framework, providing a robust API for the frontend application while leveraging PostgreSQL with JSONB fields for flexible data storage and integrating with machine learning services for intelligent schedule generation and optimization.

## Technology Stack

- **Framework**: Django 4.2+ with Python 3.12+
- **API Framework**: Django REST Framework
- **Database**: PostgreSQL 14+ with JSONB field support
- **Authentication**: Google OAuth
- **ML Integration**: scikit-learn and TensorFlow
- **Task Queue**: Celery with Redis (for background processing)
- **Testing**: pytest with coverage
- **Documentation**: drf-spectacular for OpenAPI/Swagger
- **Containerization**: Docker
- **CI/CD**: GitHub Actions
- **Monitoring**: Prometheus and Grafana

## Project Structure

```
brightpath_api/
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
│   ├── families/
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
│   ├── questionnaire/
│   │   ├── migrations/
│   │   ├── api/
│   │   │   ├── serializers.py
│   │   │   ├── views.py
│   │   │   └── urls.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── services.py
│   │   └── tests.py
│   └── feedback/
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
├── pytest.ini
└── .env.example
```

## Database Models Implementation

The following sections outline the implementation of key Django models based on the database schema.

### Users App

```python
# users/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    """Extended user model for BrightPath."""
    google_id = models.CharField(max_length=255, unique=True, null=True, blank=True)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    last_login = models.DateTimeField(null=True, blank=True)
    questionnaire_completed = models.BooleanField(default=False)
    questionnaire_completed_at = models.DateTimeField(null=True, blank=True)

    # Required for using email as the username field
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    class Meta:
        verbose_name = 'User'
        verbose_name_plural = 'Users'

    def __str__(self):
        return self.email
```

### Families App

```python
# families/models.py
from django.db import models
from django.contrib.postgres.fields import JSONField
from apps.users.models import User

class Family(models.Model):
    """Family model for storing family information and preferences."""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='families')
    name = models.CharField(max_length=255)
    
    # Structured fields for efficient querying
    scheduling_flexibility = models.CharField(max_length=50, null=True, blank=True)
    planning_approach = models.CharField(max_length=50, null=True, blank=True)
    
    # JSONB fields for flexible and hierarchical data
    preferences = JSONField(default=dict)
    questionnaire_responses = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Family'
        verbose_name_plural = 'Families'

    def __str__(self):
        return self.name

class FamilyCommitment(models.Model):
    """Model for family-wide commitments."""
    family = models.ForeignKey(Family, on_delete=models.CASCADE, related_name='commitments')
    name = models.CharField(max_length=255)
    frequency = models.CharField(max_length=50)  # 'daily', 'weekly', 'monthly', 'one-time'
    days_of_week = models.CharField(max_length=255, null=True, blank=True)  # JSON array as string
    start_time = models.TimeField()
    end_time = models.TimeField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Family Commitment'
        verbose_name_plural = 'Family Commitments'

    def __str__(self):
        return f"{self.family.name} - {self.name}"
```

### Children App

```python
# children/models.py
from django.db import models
from django.contrib.postgres.fields import JSONField
from apps.families.models import Family

class Child(models.Model):
    """Child model for storing information about children in a family."""
    family = models.ForeignKey(Family, on_delete=models.CASCADE, related_name='children')
    name = models.CharField(max_length=255)
    age = models.IntegerField()
    
    # JSONB fields for flexible data
    learning_preferences = JSONField(default=dict)
    questionnaire_responses = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Child'
        verbose_name_plural = 'Children'

    def __str__(self):
        return f"{self.name} ({self.family.name})"

class ChildCommitment(models.Model):
    """Model for child-specific commitments."""
    child = models.ForeignKey(Child, on_delete=models.CASCADE, related_name='commitments')
    name = models.CharField(max_length=255)
    frequency = models.CharField(max_length=50)  # 'daily', 'weekly', 'monthly', 'one-time'
    days_of_week = models.CharField(max_length=255, null=True, blank=True)  # JSON array as string
    start_time = models.TimeField()
    end_time = models.TimeField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Child Commitment'
        verbose_name_plural = 'Child Commitments'

    def __str__(self):
        return f"{self.child.name} - {self.name}"
```

### Subjects App

```python
# subjects/models.py
from django.db import models
from django.contrib.postgres.fields import JSONField
from apps.children.models import Child

class Subject(models.Model):
    """Subject model for educational subjects."""
    child = models.ForeignKey(Child, on_delete=models.CASCADE, related_name='subjects')
    name = models.CharField(max_length=255)
    is_core_subject = models.BooleanField(default=False)
    session_duration = models.IntegerField()  # in minutes
    frequency = models.CharField(max_length=50)  # 'daily', '2-3_times_per_week', 'once_a_week', 'occasionally'
    parent_involvement = models.CharField(max_length=50)  # 'full_involvement', 'some_assistance', etc.
    fixed_time_required = models.BooleanField(default=False)
    preferred_time = models.TimeField(null=True, blank=True)
    interest_level = models.IntegerField()
    
    # Additional JSONB field for flexible metadata
    metadata = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Subject'
        verbose_name_plural = 'Subjects'

    def __str__(self):
        return f"{self.name} ({self.child.name})"
```

### Schedules App

```python
# schedules/models.py
from django.db import models
from django.contrib.postgres.fields import JSONField
from apps.families.models import Family
from apps.children.models import Child

class Schedule(models.Model):
    """Schedule model for storing generated schedules."""
    family = models.ForeignKey(Family, on_delete=models.CASCADE, related_name='schedules')
    child = models.ForeignKey(Child, on_delete=models.CASCADE, related_name='schedules')
    week_start_date = models.DateField()
    is_active = models.BooleanField(default=True)
    version = models.IntegerField(default=1)
    generation_method = models.CharField(max_length=50)
    
    # JSONB fields for complete schedule structure
    schedule_data = JSONField()
    patterns = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Schedule'
        verbose_name_plural = 'Schedules'
        unique_together = ('child', 'week_start_date', 'is_active')

    def __str__(self):
        return f"{self.child.name}'s Schedule - Week of {self.week_start_date}"

class ScheduleItem(models.Model):
    """Model for individual items within a schedule."""
    schedule = models.ForeignKey(Schedule, on_delete=models.CASCADE, related_name='items')
    day_date = models.DateField()
    item_type = models.CharField(max_length=50)  # 'subject', 'commitment', 'break', etc.
    name = models.CharField(max_length=255)
    start_time = models.TimeField()
    end_time = models.TimeField()
    is_external = models.BooleanField(default=False)
    completed = models.BooleanField(default=False)
    completed_at = models.DateTimeField(null=True, blank=True)
    
    # Reference fields to link to original entities
    subject = models.ForeignKey('subjects.Subject', on_delete=models.SET_NULL, 
                               null=True, blank=True, related_name='schedule_items')
    commitment = models.ForeignKey('children.ChildCommitment', on_delete=models.SET_NULL,
                                  null=True, blank=True, related_name='schedule_items')
    
    # JSONB for item-specific details
    item_data = JSONField()
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = 'Schedule Item'
        verbose_name_plural = 'Schedule Items'

    def __str__(self):
        return f"{self.name} - {self.day_date} {self.start_time}"

class ActivityLog(models.Model):
    """Model for tracking activity completion and modifications."""
    schedule = models.ForeignKey(Schedule, on_delete=models.CASCADE, related_name='activity_logs')
    schedule_item = models.ForeignKey(ScheduleItem, on_delete=models.CASCADE, related_name='activity_logs')
    family = models.ForeignKey(Family, on_delete=models.CASCADE, related_name='activity_logs')
    child = models.ForeignKey(Child, on_delete=models.CASCADE, related_name='activity_logs')
    date = models.DateField()
    activity_name = models.CharField(max_length=255)
    activity_type = models.CharField(max_length=50)
    
    # JSONB fields for flexible data
    scheduled = JSONField()
    actual = JSONField(null=True, blank=True)
    
    notes = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name = 'Activity Log'
        verbose_name_plural = 'Activity Logs'

    def __str__(self):
        return f"{self.activity_name} - {self.date}"
```

### Questionnaire App

```python
# questionnaire/models.py
from django.db import models
from django.contrib.postgres.fields import JSONField
from apps.users.models import User
from apps.families.models import Family

class QuestionnaireSession(models.Model):
    """Model for server-side storage of questionnaire progress."""
    STATUS_CHOICES = (
        ('in_progress', 'In Progress'),
        ('completed', 'Completed'),
        ('abandoned', 'Abandoned'),
    )
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='questionnaire_sessions')
    family = models.ForeignKey(Family, on_delete=models.CASCADE, null=True, blank=True, 
                              related_name='questionnaire_sessions')
    session_data = JSONField()
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='in_progress')
    last_updated = models.DateTimeField(auto_now=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name = 'Questionnaire Session'
        verbose_name_plural = 'Questionnaire Sessions'
        indexes = [
            models.Index(fields=['user']),
            models.Index(fields=['last_updated']),
        ]

    def __str__(self):
        return f"{self.user.email} - {self.created_at.date()}"
```

### Feedback App

```python
# feedback/models.py
from django.db import models
from django.contrib.postgres.fields import JSONField
from apps.families.models import Family
from apps.children.models import Child
from apps.schedules.models import Schedule

class UserFeedback(models.Model):
    """Model for storing user feedback on schedules."""
    family = models.ForeignKey(Family, on_delete=models.CASCADE, related_name='feedback')
    child = models.ForeignKey(Child, on_delete=models.SET_NULL, null=True, blank=True, 
                             related_name='feedback')
    schedule = models.ForeignKey(Schedule, on_delete=models.SET_NULL, null=True, blank=True, 
                                related_name='feedback')
    feedback_type = models.CharField(max_length=50)  # 'schedule_satisfaction', 'feature_suggestion', etc.
    rating = models.IntegerField(null=True, blank=True)
    comments = models.TextField(blank=True)
    
    # JSONB for detailed feedback data
    feedback_data = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name = 'User Feedback'
        verbose_name_plural = 'User Feedback'

    def __str__(self):
        return f"{self.family.name} - {self.feedback_type} - {self.created_at.date()}"
```

### ML Models App

```python
# ml/models.py
from django.db import models
from django.contrib.postgres.fields import JSONField

class MLModel(models.Model):
    """Model for storing ML model information and parameters."""
    name = models.CharField(max_length=255)
    version = models.CharField(max_length=50)
    description = models.TextField(blank=True)
    
    # JSONB for model parameters and metrics
    parameters = JSONField()
    feature_importance = JSONField(default=dict)
    performance_metrics = JSONField(default=dict)
    
    is_active = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name = 'ML Model'
        verbose_name_plural = 'ML Models'
        unique_together = ('name', 'version')

    def __str__(self):
        return f"{self.name} v{self.version}"
```

## API Endpoints Implementation

### Authentication

```python
# brightpath/settings/base.py
INSTALLED_APPS = [
    # ...
    'oauth2_provider',
    'social_django',
    'rest_framework_social_oauth2',
    # ...
]

# Google OAuth Configuration
AUTHENTICATION_BACKENDS = (
    'social_core.backends.google.GoogleOAuth2',
    'django.contrib.auth.backends.ModelBackend',
)

SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = os.environ.get('GOOGLE_OAUTH2_KEY')
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = os.environ.get('GOOGLE_OAUTH2_SECRET')
SOCIAL_AUTH_GOOGLE_OAUTH2_SCOPE = ['email', 'profile']

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        'rest_framework_social_oauth2.authentication.SocialAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
}
```

### User Authentication Endpoints

```python
# users/api/views.py
from rest_framework import viewsets, generics, permissions, status
from rest_framework.response import Response
from rest_framework.decorators import action
from apps.users.models import User
from apps.users.api.serializers import UserSerializer

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """Viewset for user-related endpoints."""
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]
    
    @action(detail=False, methods=['get'])
    def me(self, request):
        """Get current user's profile."""
        serializer = self.get_serializer(request.user)
        return Response(serializer.data)
    
    @action(detail=False, methods=['patch'])
    def update_me(self, request):
        """Update current user's profile."""
        serializer = self.get_serializer(request.user, data=request.data, partial=True)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)
```

### Family Endpoints

```python
# families/api/views.py
from rest_framework import viewsets, permissions
from apps.families.models import Family, FamilyCommitment
from apps.families.api.serializers import FamilySerializer, FamilyCommitmentSerializer
from apps.utils.permissions import IsOwner

class FamilyViewSet(viewsets.ModelViewSet):
    """Viewset for family management."""
    serializer_class = FamilySerializer
    permission_classes = [permissions.IsAuthenticated, IsOwner]
    
    def get_queryset(self):
        """Return families that belong to the current user."""
        return Family.objects.filter(user=self.request.user)
    
    def perform_create(self, serializer):
        """Set the user automatically when creating a family."""
        serializer.save(user=self.request.user)

class FamilyCommitmentViewSet(viewsets.ModelViewSet):
    """Viewset for managing family commitments."""
    serializer_class = FamilyCommitmentSerializer
    permission_classes = [permissions.IsAuthenticated]
    
    def get_queryset(self):
        """Return commitments for families owned by the current user."""
        return FamilyCommitment.objects.filter(family__user=self.request.user)
```

### Child Endpoints

```python
# children/api/views.py
from rest_framework import viewsets, permissions
from apps.children.models import Child, ChildCommitment
from apps.children.api.serializers import ChildSerializer, ChildCommitmentSerializer
from apps.utils.permissions import IsOwnerOfFamily

class ChildViewSet(viewsets.ModelViewSet):
    """Viewset for child management."""
    serializer_class = ChildSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwnerOfFamily]
    
    def get_queryset(self):
        """Return children that belong to the current user's families."""
        return Child.objects.filter(family__user=self.request.user)

class ChildCommitmentViewSet(viewsets.ModelViewSet):
    """Viewset for managing child commitments."""
    serializer_class = ChildCommitmentSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwnerOfFamily]
    
    def get_queryset(self):
        """Return commitments for children in families owned by the current user."""
        return ChildCommitment.objects.filter(child__family__user=self.request.user)
```

### Subject Endpoints

```python
# subjects/api/views.py
from rest_framework import viewsets, permissions
from apps.subjects.models import Subject
from apps.subjects.api.serializers import SubjectSerializer
from apps.utils.permissions import IsOwnerOfChild

class SubjectViewSet(viewsets.ModelViewSet):
    """Viewset for subject management."""
    serializer_class = SubjectSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwnerOfChild]
    
    def get_queryset(self):
        """Return subjects for children in families owned by the current user."""
        return Subject.objects.filter(child__family__user=self.request.user)
```

### Schedule Endpoints

```python
# schedules/api/views.py
from rest_framework import viewsets, permissions, status
from rest_framework.decorators import action
from rest_framework.response import Response
from apps.schedules.models import Schedule, ScheduleItem, ActivityLog
from apps.schedules.api.serializers import (
    ScheduleSerializer, ScheduleItemSerializer, 
    ActivityLogSerializer, GenerateScheduleSerializer
)
from apps.utils.permissions import IsOwnerOfChildInSchedule
from apps.ml.services import ScheduleGeneratorService

class ScheduleViewSet(viewsets.ModelViewSet):
    """Viewset for schedule management."""
    serializer_class = ScheduleSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwnerOfChildInSchedule]
    
    def get_queryset(self):
        """Return schedules for children in families owned by the current user."""
        return Schedule.objects.filter(child__family__user=self.request.user)
    
    @action(detail=False, methods=['post'])
    def generate(self, request):
        """Generate a new schedule using ML model."""
        serializer = GenerateScheduleSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        # Get validated data
        child_id = serializer.validated_data['child_id']
        week_start_date = serializer.validated_data['week_start_date']
        
        # Call ML service to generate schedule
        generator = ScheduleGeneratorService()
        schedule = generator.generate_schedule(
            user=self.user,
            name='Test Family'
        )
        self.child = Child.objects.create(
            family=self.family,
            name='Test Child',
            age=7
        )
    
    def test_child_creation(self):
        """Test creating a child."""
        self.assertEqual(self.child.name, 'Test Child')
        self.assertEqual(self.child.age, 7)
        self.assertEqual(self.child.family, self.family)
    
    def test_child_learning_preferences(self):
        """Test adding learning preferences to a child."""
        self.child.learning_preferences = {
            'bestLearningTimes': ['mid-morning', 'early_afternoon'],
            'homeschoolingHours': {
                'startTime': '09:00',
                'endTime': '14:00'
            }
        }
        self.child.save()
        
        # Retrieve from database again
        child = Child.objects.get(id=self.child.id)
        self.assertEqual(child.learning_preferences['bestLearningTimes'], 
                         ['mid-morning', 'early_afternoon'])
        self.assertEqual(child.learning_preferences['homeschoolingHours']['startTime'], '09:00')

class ChildAPITests(TestCase):
    """Test the Child API endpoints."""
    
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='password123'
        )
        self.client.force_authenticate(user=self.user)
        
        self.family = Family.objects.create(
            user=self.user,
            name='Test Family'
        )
        self.child = Child.objects.create(
            family=self.family,
            name='Test Child',
            age=7
        )
        
        self.url = reverse('child-list')
    
    def test_get_children(self):
        """Test retrieving a list of children."""
        response = self.client.get(self.url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
        self.assertEqual(response.data[0]['name'], 'Test Child')
    
    def test_create_child(self):
        """Test creating a new child via API."""
        data = {
            'family': self.family.id,
            'name': 'New Child',
            'age': 5
        }
        response = self.client.post(self.url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Child.objects.count(), 2)
        self.assertEqual(Child.objects.get(name='New Child').age, 5)
    
    def test_update_child(self):
        """Test updating a child via API."""
        url = reverse('child-detail', args=[self.child.id])
        data = {
            'name': 'Updated Child',
            'age': 8
        }
        response = self.client.patch(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.child.refresh_from_db()
        self.assertEqual(self.child.name, 'Updated Child')
        self.assertEqual(self.child.age, 8)
```

```python
# schedules/tests.py
from django.test import TestCase
from django.urls import reverse
from rest_framework.test import APIClient
from rest_framework import status
from django.utils import timezone
from datetime import timedelta, datetime
from apps.users.models import User
from apps.families.models import Family
from apps.children.models import Child
from apps.subjects.models import Subject
from apps.schedules.models import Schedule, ScheduleItem

class ScheduleGenerationTests(TestCase):
    """Test the schedule generation functionality."""
    
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='password123'
        )
        self.client.force_authenticate(user=self.user)
        
        self.family = Family.objects.create(
            user=self.user,
            name='Test Family',
            scheduling_flexibility='moderate_structure',
            planning_approach='mix_of_planned_and_spontaneous',
            preferences={
                'schedulingFlexibility': 'moderate_structure',
                'homeschoolingPhilosophy': {
                    'traditional': 2,
                    'charlotte_mason': 5,
                    'montessori': 3
                }
            }
        )
        
        self.child = Child.objects.create(
            family=self.family,
            name='Test Child',
            age=7,
            learning_preferences={
                'bestLearningTimes': ['mid-morning', 'early_afternoon'],
                'homeschoolingHours': {
                    'startTime': '09:00',
                    'endTime': '14:00'
                }
            }
        )
        
        # Create subjects for the child
        self.subject1 = Subject.objects.create(
            child=self.child,
            name='Mathematics',
            is_core_subject=True,
            session_duration=30,
            frequency='daily',
            parent_involvement='some_assistance',
            interest_level=4
        )
        
        self.subject2 = Subject.objects.create(
            child=self.child,
            name='Reading',
            is_core_subject=True,
            session_duration=45,
            frequency='daily',
            parent_involvement='full_involvement',
            interest_level=5
        )
        
        # URL for schedule generation
        self.url = reverse('schedule-generate')
    
    def test_generate_schedule(self):
        """Test generating a schedule via API."""
        # Current Monday
        today = timezone.now().date()
        monday = today - timedelta(days=today.weekday())
        
        data = {
            'child_id': self.child.id,
            'week_start_date': monday.isoformat()
        }
        
        response = self.client.post(self.url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        
        # Check that a schedule was created
        self.assertTrue(Schedule.objects.filter(child=self.child).exists())
        
        # Get the created schedule
        schedule = Schedule.objects.get(child=self.child)
        
        # Check that it has the correct structure
        self.assertEqual(schedule.family, self.family)
        self.assertEqual(schedule.week_start_date, monday)
        self.assertTrue(schedule.is_active)
        
        # Check that schedule items were created
        items = ScheduleItem.objects.filter(schedule=schedule)
        self.assertTrue(len(items) > 0)
        
        # Check schedule data JSON structure
        self.assertIn('days', schedule.schedule_data)
        self.assertEqual(len(schedule.schedule_data['days']), 5)  # 5 weekdays
        
        # Check first day items
        first_day = schedule.schedule_data['days'][0]
        self.assertIn('items', first_day)
        self.assertTrue(len(first_day['items']) > 0)
        
        # Check that at least one subject is in the schedule
        subject_items = ScheduleItem.objects.filter(
            schedule=schedule,
            item_type='subject'
        )
        self.assertTrue(len(subject_items) > 0)
```

```python
# questionnaire/tests.py
from django.test import TestCase
from django.urls import reverse
from rest_framework.test import APIClient
from rest_framework import status
from django.utils import timezone
from apps.users.models import User
from apps.families.models import Family, FamilyCommitment
from apps.children.models import Child, ChildCommitment
from apps.subjects.models import Subject
from apps.questionnaire.models import QuestionnaireSession

class QuestionnaireSubmissionTests(TestCase):
    """Test the questionnaire submission process."""
    
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='password123'
        )
        self.client.force_authenticate(user=self.user)
        
        # Create a questionnaire session
        self.session = QuestionnaireSession.objects.create(
            user=self.user,
            session_data={'progress': 'in_progress'},
            status='in_progress'
        )
        
        # URL for questionnaire submission
        self.url = reverse('questionnaire-submit')
        
        # Sample questionnaire data
        self.questionnaire_data = {
            'session_id': self.session.id,
            'data': {
                'section1': {
                    'name': 'Smith Family'
                },
                'section2': {
                    'name': 'John',
                    'age': 8
                },
                'section3': {
                    'commitments': [
                        {
                            'name': 'Soccer Practice',
                            'frequency': 'weekly',
                            'daysOfWeek': ['Tuesday', 'Thursday'],
                            'startTime': '16:00',
                            'endTime': '17:30'
                        }
                    ]
                },
                'section4': {
                    'subjects': [
                        {
                            'name': 'Mathematics',
                            'isCoreSubject': True,
                            'sessionDuration': 30,
                            'frequency': 'daily',
                            'parentInvolvement': 'some_assistance',
                            'interestLevel': 4
                        },
                        {
                            'name': 'Reading',
                            'isCoreSubject': True,
                            'sessionDuration': 45,
                            'frequency': 'daily',
                            'parentInvolvement': 'full_involvement',
                            'interestLevel': 5
                        }
                    ]
                },
                'section5': {
                    'flexibility': 'moderate_structure'
                },
                'section6': {
                    'bestLearningTimes': ['mid-morning', 'early_afternoon'],
                    'startTime': '09:00',
                    'endTime': '14:00'
                },
                'section7': {
                    'homeschoolingPhilosophy': {
                        'traditional': 2,
                        'charlotte_mason': 5,
                        'montessori': 3,
                        'classical': 1,
                        'unschooling': 4
                    },
                    'schedulingStyle': {
                        'block': 4,
                        'loop': 3,
                        'time_blocked': 2,
                        'interest_led': 5,
                        'grouped_activities': 3
                    },
                    'planningApproach': 'mix_of_planned_and_spontaneous',
                    'activityPreferences': {
                        'textbooks_worksheets': 2,
                        'nature_exploration': 5,
                        'arts_crafts': 4,
                        'memorization': 2,
                        'research_projects': 3
                    }
                },
                'children': [
                    {
                        'section2': {
                            'name': 'John',
                            'age': 8
                        },
                        'section3Child': {
                            'commitments': [
                                {
                                    'name': 'Piano Lessons',
                                    'frequency': 'weekly',
                                    'daysOfWeek': ['Monday'],
                                    'startTime': '15:00',
                                    'endTime': '16:00'
                                }
                            ]
                        },
                        'section4': {
                            'subjects': [
                                {
                                    'name': 'Mathematics',
                                    'isCoreSubject': True,
                                    'sessionDuration': 30,
                                    'frequency': 'daily',
                                    'parentInvolvement': 'some_assistance',
                                    'interestLevel': 4
                                },
                                {
                                    'name': 'Reading',
                                    'isCoreSubject': True,
                                    'sessionDuration': 45,
                                    'frequency': 'daily',
                                    'parentInvolvement': 'full_involvement',
                                    'interestLevel': 5
                                }
                            ]
                        },
                        'section6': {
                            'bestLearningTimes': ['mid-morning', 'early_afternoon'],
                            'startTime': '09:00',
                            'endTime': '14:00'
                        }
                    }
                ]
            }
        }
    
    def test_questionnaire_submission(self):
        """Test submitting a complete questionnaire."""
        response = self.client.post(
            self.url, 
            self.questionnaire_data,
            format='json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['status'], 'success')
        
        # Check that user is marked as having completed questionnaire
        self.user.refresh_from_db()
        self.assertTrue(self.user.questionnaire_completed)
        self.assertIsNotNone(self.user.questionnaire_completed_at)
        
        # Check that session is marked as completed
        self.session.refresh_from_db()
        self.assertEqual(self.session.status, 'completed')
        
        # Check that family was created
        family = Family.objects.filter(user=self.user).first()
        self.assertIsNotNone(family)
        self.assertEqual(family.name, 'Smith Family')
        self.assertEqual(family.scheduling_flexibility, 'moderate_structure')
        self.assertEqual(family.planning_approach, 'mix_of_planned_and_spontaneous')
        
        # Check that family commitment was created
        commitment = FamilyCommitment.objects.filter(family=family).first()
        self.assertIsNotNone(commitment)
        self.assertEqual(commitment.name, 'Soccer Practice')
        
        # Check that child was created
        child = Child.objects.filter(family=family).first()
        self.assertIsNotNone(child)
        self.assertEqual(child.name, 'John')
        self.assertEqual(child.age, 8)
        
        # Check that child commitment was created
        child_commitment = ChildCommitment.objects.filter(child=child).first()
        self.assertIsNotNone(child_commitment)
        self.assertEqual(child_commitment.name, 'Piano Lessons')
        
        # Check that subjects were created
        subjects = Subject.objects.filter(child=child)
        self.assertEqual(subjects.count(), 2)
        self.assertTrue(subjects.filter(name='Mathematics').exists())
        self.assertTrue(subjects.filter(name='Reading').exists())
```

## Deployment Configuration

### Docker Configuration

```dockerfile
# Dockerfile
FROM python:3.12-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory
WORKDIR /app

# Install dependencies
RUN apt-get update && apt-get install -y build-essential libpq-dev

# Install Python dependencies
COPY requirements/base.txt /app/requirements/
COPY requirements/production.txt /app/requirements/
RUN pip install --no-cache-dir -r requirements/production.txt

# Copy project
COPY . /app/

# Expose port
EXPOSE 8000

# Start gunicorn
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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - brightpath-network

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - brightpath-network

  web:
    build: .
    restart: always
    command: gunicorn brightpath.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - .:/app
      - static_volume:/app/staticfiles
      - media_volume:/app/mediafiles
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis
    env_file:
      - ./.env
    networks:
      - brightpath-network

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
    networks:
      - brightpath-network

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
    networks:
      - brightpath-network

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    ports:
      - "9090:9090"
    networks:
      - brightpath-network

  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - brightpath-network

networks:
  brightpath-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
  prometheus_data:
  grafana_data:
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
name: BrightPath CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.12'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements/development.txt
    
    - name: Lint with flake8
      run: |
        flake8 .
    
    - name: Test with pytest
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/test_db
        SECRET_KEY: test_secret_key
      run: |
        pytest

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1
    
    - name: Login to Digital Ocean Container Registry
      uses: docker/login-action@v1
      with:
        registry: registry.digitalocean.com
        username: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
        password: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
    
    - name: Build and push Docker images
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: registry.digitalocean.com/brightpath/backend:latest
    
    - name: Deploy to Digital Ocean Droplet
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.DROPLET_HOST }}
        username: ${{ secrets.DROPLET_USERNAME }}
        key: ${{ secrets.DROPLET_SSH_KEY }}
        script: |
          cd /app/brightpath
          docker-compose pull
          docker-compose down
          docker-compose up -d
```

## Implementation Timeline

### Phase 1: Core Setup (Weeks 1-2)
- Set up project structure and environment
- Configure database with PostgreSQL and JSONB support
- Implement authentication with Google OAuth
- Create and migrate core models (User, Family, Child)
- Set up testing framework with pytest
- Configure Docker for development

### Phase 2: Data Models and API (Weeks 3-4)
- Implement remaining data models (Subject, Schedule, etc.)
- Develop REST API endpoints for core entities
- Configure serializers and permissions
- Implement API tests
- Set up Swagger documentation

### Phase 3: Questionnaire Implementation (Weeks 5-6)
- Build questionnaire API endpoints
- Implement the questionnaire service
- Create local storage integration API
- Set up data validation
- Develop functionality to save final questionnaire to PostgreSQL/JSONB

### Phase 4: Schedule Generation (Weeks 7-9)
- Set up ML model infrastructure
- Implement schedule generator service
- Create scheduler API endpoints
- Develop schedule visualization data structures
- Set up Celery for background processing
- Implement schedule recommendation algorithms

### Phase 5: Feedback System (Weeks 10-11)
- Build feedback collection API
- Implement reward model service
- Set up model training infrastructure
- Create feedback analysis dashboards
- Develop continuous improvement pipeline

### Phase 6: Testing and Optimization (Weeks 12-13)
- Comprehensive system testing
- Performance optimization
- Security hardening
- Load testing
- Documentation updates

### Phase 7: Deployment Setup (Weeks 14-15)
- Configure production environment
- Set up CI/CD with GitHub Actions
- Configure monitoring with Prometheus/Grafana
- Implement logging and alerting
- Create deployment documentation

## COPPA Compliance Implementation

### Privacy and Data Protection

```python
# utils/coppa.py
from django.conf import settings
from rest_framework.permissions import BasePermission
from apps.users.models import User

class RequiresParentalConsent(BasePermission):
    """
    Permission that ensures parental consent has been obtained.
    """
    message = "Parental consent is required for this operation."

    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.has_parental_consent

def log_data_access(user_id, data_type, action):
    """
    Log all data access for COPPA compliance auditing.
    """
    from apps.users.models import DataAccessLog
    
    DataAccessLog.objects.create(
        user_id=user_id,
        data_type=data_type,
        action=action,
        ip_address=get_client_ip(request)
    )

def get_client_ip(request):
    """Get the client IP address from request."""
    x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
    if x_forwarded_for:
        ip = x_forwarded_for.split(',')[0]
    else:
        ip = request.META.get('REMOTE_ADDR')
    return ip

def minimize_child_data(child_data):
    """
    Apply data minimization to child data before storing or transmitting.
    This helps ensure COPPA compliance by limiting what is collected.
    """
    allowed_fields = ['name', 'age', 'learning_preferences']
    return {k: v for k, v in child_data.items() if k in allowed_fields}
```

### Terms of Service and Consent

```python
# users/models.py
# Add to User model:
has_parental_consent = models.BooleanField(default=False)
consent_date = models.DateTimeField(null=True, blank=True)
consent_ip = models.GenericIPAddressField(null=True, blank=True)
consent_version = models.CharField(max_length=20, null=True, blank=True)
```

```python
# users/api/views.py
@action(detail=False, methods=['post'])
def parental_consent(self, request):
    """Record parental consent for COPPA compliance."""
    user = request.user
    
    # Validate consent form data
    serializer = ParentalConsentSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    
    # Record consent
    user.has_parental_consent = True
    user.consent_date = timezone.now()
    user.consent_ip = get_client_ip(request)
    user.consent_version = settings.CURRENT_TOS_VERSION
    user.save()
    
    return Response({'status': 'consent recorded'})
```

## Security Implementation

### Security Middleware

```python
# brightpath/settings/base.py
MIDDLEWARE = [
    # Django security middleware
    'django.middleware.security.SecurityMiddleware',
    # ...
    # Custom security middleware
    'apps.utils.middleware.SecurityHeadersMiddleware',
]

# Security settings
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# In production
if not DEBUG:
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
```

```python
# utils/middleware.py
class SecurityHeadersMiddleware:
    """Add security headers to all responses."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        response = self.get_response(request)
        
        # Add security headers
        response['Content-Security-Policy'] = self._get_csp_header()
        response['X-Content-Type-Options'] = 'nosniff'
        response['X-Frame-Options'] = 'DENY'
        response['X-XSS-Protection'] = '1; mode=block'
        response['Referrer-Policy'] = 'strict-origin-when-cross-origin'
        response['Permissions-Policy'] = 'geolocation=(), microphone=(), camera=()'
        
        return response
    
    def _get_csp_header(self):
        """Generate Content Security Policy header value."""
        directives = {
            'default-src': ["'self'"],
            'img-src': ["'self'", 'data:'],
            'style-src': ["'self'", "'unsafe-inline'"],
            'script-src': ["'self'"],
            'connect-src': ["'self'"],
            'font-src': ["'self'", 'https://fonts.gstatic.com'],
            'object-src': ["'none'"]
        }
        
        return '; '.join([
            f"{key} {' '.join(value)}" 
            for key, value in directives.items()
        ])
```

## Monitoring and Logging

### Prometheus Setup

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'brightpath-api'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['web:8000']
  
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

### Django Monitoring

```python
# brightpath/settings/base.py
INSTALLED_APPS += [
    'django_prometheus',
]

MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    # ... other middleware
    'django_prometheus.middleware.PrometheusAfterMiddleware',
]
```

```python
# brightpath/urls.py
urlpatterns = [
    # ... other URL patterns
    path('metrics/', include('django_prometheus.urls')),
]
```

## API Documentation

```python
# brightpath/settings/base.py
INSTALLED_APPS += [
    'drf_spectacular',
]

REST_FRAMEWORK = {
    # ... other settings
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'BrightPath API',
    'DESCRIPTION': 'API for BrightPath homeschooling scheduling platform',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'SCHEMA_PATH_PREFIX': '/api',
}
```

```python
# brightpath/urls.py
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularRedocView,
    SpectacularSwaggerView,
)

urlpatterns = [
    # ... other URL patterns
    
    # OpenAPI schema
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    
    # Swagger UI
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    
    # ReDoc UI
    path('api/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

## Conclusion

This backend implementation plan provides a comprehensive roadmap for developing the BrightPath homeschooling scheduling platform's backend services. By leveraging Django with PostgreSQL and JSONB fields, we can build a flexible, scalable, and robust system that meets the specific requirements of the project.

Key features of this implementation include:
- Hybrid relational-document database approach using PostgreSQL with JSONB
- RESTful API endpoints for all core functionalities
- Machine learning integration for intelligent schedule generation
- Seamless questionnaire implementation with localStorage integration
- Comprehensive testing and monitoring
- Containerized deployment with Docker
- CI/CD pipeline for automated testing and deployment

With this plan, the development team can proceed with confidence, knowing that all aspects of the backend implementation have been carefully considered and aligned with the project's requirements and architecture.
=request.user,
            child_id=child_id,
            week_start_date=week_start_date
        )
        
        return Response(
            ScheduleSerializer(schedule).data,
            status=status.HTTP_201_CREATED
        )

class ScheduleItemViewSet(viewsets.ModelViewSet):
    """Viewset for schedule item management."""
    serializer_class = ScheduleItemSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwnerOfChildInSchedule]
    
    def get_queryset(self):
        """Return schedule items for schedules owned by the current user."""
        return ScheduleItem.objects.filter(schedule__child__family__user=self.request.user)
    
    @action(detail=True, methods=['post'])
    def mark_complete(self, request, pk=None):
        """Mark a schedule item as completed."""
        item = self.get_object()
        item.completed = True
        item.completed_at = timezone.now()
        item.save()
        
        # Create activity log entry
        ActivityLog.objects.create(
            schedule=item.schedule,
            schedule_item=item,
            family=item.schedule.family,
            child=item.schedule.child,
            date=item.day_date,
            activity_name=item.name,
            activity_type=item.item_type,
            scheduled={
                "startTime": item.start_time.isoformat(),
                "endTime": item.end_time.isoformat(),
                "original": {"name": item.name}
            },
            actual={
                "completed": True,
                "completedAt": timezone.now().isoformat()
            }
        )
        
        return Response({'status': 'completed'})
```

### Questionnaire Endpoints

```python
# questionnaire/api/views.py
from rest_framework import viewsets, permissions, status
from rest_framework.decorators import action
from rest_framework.response import Response
from apps.questionnaire.models import QuestionnaireSession
from apps.questionnaire.api.serializers import (
    QuestionnaireSessionSerializer, 
    QuestionnaireSubmissionSerializer
)
from apps.families.models import Family
from apps.utils.permissions import IsOwner
from apps.questionnaire.services import QuestionnaireService

class QuestionnaireSessionViewSet(viewsets.ModelViewSet):
    """Viewset for managing questionnaire sessions."""
    serializer_class = QuestionnaireSessionSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwner]
    
    def get_queryset(self):
        """Return questionnaire sessions for the current user."""
        return QuestionnaireSession.objects.filter(user=self.request.user)
    
    def perform_create(self, serializer):
        """Set the user automatically when creating a session."""
        serializer.save(user=self.request.user)
    
    @action(detail=False, methods=['post'])
    def submit_questionnaire(self, request):
        """Submit completed questionnaire and save to database."""
        serializer = QuestionnaireSubmissionSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        # Process questionnaire data
        service = QuestionnaireService()
        success, family_id = service.save_questionnaire_data(
            user_id=request.user.id,
            data=serializer.validated_data['data']
        )
        
        if success:
            # Mark user as having completed questionnaire
            request.user.questionnaire_completed = True
            request.user.questionnaire_completed_at = timezone.now()
            request.user.save()
            
            # Mark the session as completed
            session_id = serializer.validated_data.get('session_id')
            if session_id:
                session = QuestionnaireSession.objects.get(id=session_id)
                session.status = 'completed'
                session.family_id = family_id
                session.save()
            
            return Response({'status': 'success', 'family_id': family_id})
        else:
            return Response(
                {'status': 'error', 'message': 'Failed to save questionnaire data'},
                status=status.HTTP_500_INTERNAL_SERVER_ERROR
            )
```

### Feedback Endpoints

```python
# feedback/api/views.py
from rest_framework import viewsets, permissions
from apps.feedback.models import UserFeedback
from apps.feedback.api.serializers import UserFeedbackSerializer
from apps.utils.permissions import IsOwnerOfFamily
from apps.ml.services import RewardModelService

class UserFeedbackViewSet(viewsets.ModelViewSet):
    """Viewset for user feedback management."""
    serializer_class = UserFeedbackSerializer
    permission_classes = [permissions.IsAuthenticated, IsOwnerOfFamily]
    
    def get_queryset(self):
        """Return feedback for families owned by the current user."""
        return UserFeedback.objects.filter(family__user=self.request.user)
    
    def perform_create(self, serializer):
        """Process feedback with reward model when creating."""
        feedback = serializer.save()
        
        # If feedback is for a schedule, update the reward model
        if feedback.schedule:
            reward_service = RewardModelService()
            reward_service.process_feedback(feedback)
        
        return feedback
```

## Machine Learning Implementation

### Schedule Generator Service

```python
# ml/services.py
import numpy as np
import pandas as pd
from sklearn.ensemble import RandomForestRegressor
import joblib
from django.utils import timezone
from apps.children.models import Child
from apps.subjects.models import Subject
from apps.schedules.models import Schedule, ScheduleItem
from apps.ml.models import MLModel

class ScheduleGeneratorService:
    """Service for generating personalized schedules using ML."""
    
    def __init__(self, model_path=None):
        """Initialize the service, optionally loading a model from file."""
        if model_path:
            self.model = joblib.load(model_path)
        else:
            # Try to load the latest active model from database
            try:
                latest_model = MLModel.objects.filter(
                    name='schedule_generator',
                    is_active=True
                ).latest('created_at')
                
                self.model = RandomForestRegressor()
                self.model.set_params(**latest_model.parameters.get('model_params', {}))
                # In a real implementation, we would also load weights
                
            except MLModel.DoesNotExist:
                # Create a default model if none exists
                self.model = RandomForestRegressor(
                    n_estimators=100,
                    max_depth=10,
                    random_state=42
                )
    
    def generate_schedule(self, user, child_id, week_start_date):
        """Generate a complete schedule for a child."""
        # Get child data
        child = Child.objects.get(id=child_id, family__user=user)
        
        # Get family preferences
        family = child.family
        preferences = family.preferences
        
        # Get child's subjects
        subjects = Subject.objects.filter(child=child)
        
        # Get child's learning preferences
        learning_prefs = child.learning_preferences
        
        # Extract features for ML model
        features = self._extract_features(child, family, subjects, learning_prefs)
        
        # Generate schedule parameters using ML model
        params = self._predict_schedule_params(features)
        
        # Create schedule structure
        schedule_data = self._create_schedule_structure(
            child, 
            subjects, 
            week_start_date, 
            params
        )
        
        # Save to database
        # First mark any existing active schedule as inactive
        Schedule.objects.filter(
            child=child,
            week_start_date=week_start_date,
            is_active=True
        ).update(is_active=False)
        
        # Create new schedule
        schedule = Schedule.objects.create(
            family=family,
            child=child,
            week_start_date=week_start_date,
            is_active=True,
            version=1,
            generation_method='ml_v1',
            schedule_data=schedule_data,
            patterns=schedule_data.get('patterns', {})
        )
        
        # Create schedule items
        self._create_schedule_items(schedule, schedule_data)
        
        return schedule
    
    def _extract_features(self, child, family, subjects, learning_prefs):
        """Extract features for the ML model from user data."""
        features = []
        
        # Child age
        features.append(child.age)
        
        # Schedule flexibility preference
        flexibility_map = {
            'very_flexible': 0.1,
            'somewhat_flexible': 0.3,
            'moderate_structure': 0.5,
            'mostly_structured': 0.7,
            'strictly_structured': 0.9
        }
        flexibility_score = flexibility_map.get(family.scheduling_flexibility, 0.5)
        features.append(flexibility_score)
        
        # Planning approach
        planning_map = {
            'follow_curriculum': 0.1,
            'curriculum_as_guide': 0.3,
            'create_own_plans': 0.5,
            'follow_interests': 0.7,
            'mix_of_planned_and_spontaneous': 0.5
        }
        planning_score = planning_map.get(family.planning_approach, 0.5)
        features.append(planning_score)
        
        # Learning time preferences
        times = learning_prefs.get('bestLearningTimes', [])
        time_scores = {
            'early_morning': 0 in times,
            'mid_morning': 1 in times,
            'early_afternoon': 2 in times,
            'late_afternoon': 3 in times
        }
        features.extend(time_scores.values())
        
        # Subject interest levels and complexity
        avg_interest = np.mean([s.interest_level for s in subjects]) / 5.0
        features.append(avg_interest)
        
        # Number of subjects
        features.append(len(subjects) / 10.0)  # Normalize
        
        # Convert to numpy array
        return np.array(features).reshape(1, -1)
    
    def _predict_schedule_params(self, features):
        """Use ML model to predict schedule parameters."""
        # In a real implementation, this would use the trained model
        # For now, we'll use sensible defaults with some variance based on features
        
        # Extract key features
        child_age = features[0, 0]
        flexibility = features[0, 1]
        planning_approach = features[0, 2]
        
        # Calculate parameters
        params = {
            'block_duration': max(15, min(60, int(20 + child_age * 2))),  # 15-60 minutes
            'breaks_frequency': max(2, min(5, int(6 - child_age / 3))),  # Every 2-5 blocks
            'start_time_hour': 8 if flexibility > 0.6 else (9 if flexibility > 0.3 else 10),
            'structure_level': flexibility,
            'daily_learning_hours': max(2, min(6, int(3 + child_age / 3))),  # 2-6 hours
            'variety_level': planning_approach * 10  # 0-10 scale
        }
        
        return params
    
    def _create_schedule_structure(self, child, subjects, week_start_date, params):
        """Create the full schedule structure based on parameters."""
        from datetime import datetime, timedelta
        
        # Basic schedule metadata
        schedule = {
            'weekStartDate': week_start_date.isoformat(),
            'version': 1,
            'generationMethod': 'ml_v1',
            'days': [],
            'patterns': {
                'learningBlocks': [],
                'breakPatterns': {
                    'frequency': f"every_{params['breaks_frequency']}_blocks",
                    'duration': 15,
                    'activitySuggestions': ['Physical Movement', 'Snack Time']
                }
            }
        }
        
        # Convert subjects to a list for easier handling
        subject_list = list(subjects)
        
        # Generate 5 days (Monday-Friday)
        for day_offset in range(5):
            current_date = week_start_date + timedelta(days=day_offset)
            day_name = current_date.strftime('%A')
            
            # Create day structure
            day = {
                'date': current_date.isoformat(),
                'dayOfWeek': day_name,
                'items': []
            }
            
            # Current time tracker
            current_hour = params['start_time_hour']
            current_minute = 0
            
            # Add blocks for this day
            block_count = 0
            for i in range(min(len(subject_list), params['daily_learning_hours'] * 2)):
                # Add a break after specified number of blocks
                if block_count > 0 and block_count % params['breaks_frequency'] == 0:
                    # Add a break
                    day['items'].append({
                        'id': None,  # Will be set after database save
                        'type': 'break',
                        'name': 'Break',
                        'startTime': f"{current_hour:02d}:{current_minute:02d}",
                        'endTime': self._advance_time(current_hour, current_minute, 15)
                    })
                    
                    # Update time
                    current_hour, current_minute = self._parse_time(
                        self._advance_time(current_hour, current_minute, 15)
                    )
                
                # Select subject (with rotation for variety)
                subject = subject_list[(i + day_offset) % len(subject_list)]
                
                # Add subject block
                day['items'].append({
                    'id': None,  # Will be set after database save
                    'type': 'subject',
                    'name': subject.name,
                    'subjectId': subject.id,
                    'startTime': f"{current_hour:02d}:{current_minute:02d}",
                    'endTime': self._advance_time(current_hour, current_minute, subject.session_duration),
                    'completed': False
                })
                
                # Update time
                current_hour, current_minute = self._parse_time(
                    self._advance_time(current_hour, current_minute, subject.session_duration)
                )
                
                block_count += 1
            
            # Add day to schedule
            schedule['days'].append(day)
        
        # Add learning block patterns
        core_subjects = [s for s in subject_list if s.is_core_subject]
        if core_subjects:
            schedule['patterns']['learningBlocks'].append({
                'type': 'core_academic',
                'preferredTime': 'morning',
                'subjects': [s.name for s in core_subjects],
                'averageDuration': np.mean([s.session_duration for s in core_subjects])
            })
        
        return schedule
    
    def _advance_time(self, hour, minute, duration):
        """Advance time by a specified duration in minutes."""
        total_minutes = hour * 60 + minute + duration
        new_hour = total_minutes // 60
        new_minute = total_minutes % 60
        return f"{new_hour:02d}:{new_minute:02d}"
    
    def _parse_time(self, time_str):
        """Parse a time string into hour and minute components."""
        hour, minute = map(int, time_str.split(':'))
        return hour, minute
    
    def _create_schedule_items(self, schedule, schedule_data):
        """Create ScheduleItem records from schedule_data."""
        from datetime import datetime
        
        for day_data in schedule_data['days']:
            day_date = datetime.fromisoformat(day_data['date']).date()
            
            for item_data in day_data['items']:
                # Convert time strings to time objects
                start_time = datetime.strptime(item_data['startTime'], '%H:%M').time()
                end_time = datetime.strptime(item_data['endTime'], '%H:%M').time()
                
                # Create item in database
                item = ScheduleItem.objects.create(
                    schedule=schedule,
                    day_date=day_date,
                    item_type=item_data['type'],
                    name=item_data['name'],
                    start_time=start_time,
                    end_time=end_time,
                    is_external=False,
                    subject_id=item_data.get('subjectId'),
                    item_data=item_data
                )
                
                # Update the ID in schedule_data
                item_data['id'] = item.id
        
        # Save updated schedule_data with IDs
        schedule.schedule_data = schedule_data
        schedule.save(update_fields=['schedule_data'])
```

### Reward Model Service

```python
# ml/services.py
import numpy as np
import pandas as pd
from sklearn.ensemble import GradientBoostingRegressor
import joblib
from apps.feedback.models import UserFeedback
from apps.ml.models import MLModel

class RewardModelService:
    """Service for processing schedule feedback and training the reward model."""
    
    def __init__(self, model_path=None):
        """Initialize the service, optionally loading a model from file."""
        if model_path:
            self.model = joblib.load(model_path)
        else:
            # Try to load the latest active model from database
            try:
                latest_model = MLModel.objects.filter(
                    name='reward_model',
                    is_active=True
                ).latest('created_at')
                
                self.model = GradientBoostingRegressor()
                self.model.set_params(**latest_model.parameters.get('model_params', {}))
                # In a real implementation, we would also load weights
                
            except MLModel.DoesNotExist:
                # Create a default model if none exists
                self.model = GradientBoostingRegressor(
                    n_estimators=100,
                    max_depth=5,
                    learning_rate=0.1,
                    random_state=42
                )
    
    def process_feedback(self, feedback):
        """Process user feedback and update reward model."""
        # Extract features from schedule and feedback
        features = self._extract_features(feedback)
        
        # Calculate reward score
        reward_score = self.model.predict([features])[0]
        
        # Log the reward score (in a real implementation, this would be stored)
        print(f"Reward score for schedule {feedback.schedule.id}: {reward_score}")
        
        # In a real implementation, this method would:
        # 1. Store the reward score with the feedback
        # 2. Periodically trigger retraining of the reward model
        # 3. Use the reward model to improve the generator model
    
    def _extract_features(self, feedback):
        """Extract features from feedback and schedule data."""
        features = []
        
        # Schedule structure features
        schedule_data = feedback.schedule.schedule_data
        
        # Calculate average block duration
        block_durations = []
        for day in schedule_data.get('days', []):
            for item in day.get('items', []):
                if item.get('type') == 'subject':
                    start = self._time_to_minutes(item.get('startTime', '00:00'))
                    end = self._time_to_minutes(item.get('endTime', '00:00'))
                    block_durations.append(end - start)
        
        avg_duration = np.mean(block_durations) if block_durations else 30
        features.append(avg_duration / 60.0)  # Normalize to hours
        
        # Number of subjects per day
        avg_subjects_per_day = np.mean([
            len([i for i in day.get('items', []) if i.get('type') == 'subject'])
            for day in schedule_data.get('days', [])
        ])
        features.append(avg_subjects_per_day / 10.0)  # Normalize
        
        # Start time
        start_times = []
        for day in schedule_data.get('days', []):
            items = day.get('items', [])
            if items:
                start_times.append(self._time_to_minutes(items[0].get('startTime', '09:00')))
        avg_start_time = np.mean(start_times) if start_times else 540  # Default to 9:00
        features.append(avg_start_time / 1440.0)  # Normalize to fraction of day
        
        # Feedback features
        feedback_data = feedback.feedback_data
        
        # Star rating
        features.append(feedback.rating / 5.0)  # Normalize to 0-1
        
        # Specific feedback metrics
        specific_ratings = feedback_data.get('ratings', {})
        for metric in ['pace', 'difficulty', 'variety', 'structure']:
            rating = specific_ratings.get(metric, 3)
            features.append(rating / 5.0)  # Normalize to 0-1
        
        # Completion rate
        completion_rate = feedback_data.get('completion_rate', 0.5)
        features.append(completion_rate)
        
        # Adjustment data
        adjustments = feedback_data.get('adjustments', {})
        features.append(adjustments.get('time_shifted', False) * 1.0)
        features.append(adjustments.get('items_reordered', False) * 1.0)
        features.append(adjustments.get('items_removed', False) * 1.0)
        
        return features
    
    def _time_to_minutes(self, time_str):
        """Convert a time string (HH:MM) to minutes since midnight."""
        try:
            hours, minutes = map(int, time_str.split(':'))
            return hours * 60 + minutes
        except:
            return 0  # Default to midnight if parsing fails
    
    def train_model(self):
        """Train the reward model using collected feedback."""
        # Get all feedback with schedule data
        feedback_entries = UserFeedback.objects.filter(
            schedule__isnull=False
        ).select_related('schedule')
        
        if not feedback_entries:
            print("No feedback data available for training")
            return
        
        # Prepare training data
        X = []
        y = []
        
        for feedback in feedback_entries:
            try:
                features = self._extract_features(feedback)
                target = feedback.rating / 5.0  # Normalize to 0-1
                
                X.append(features)
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
        self.model.fit(X, y)
        
        # Save model to database
        model_params = {
            'model_params': self.model.get_params(),
            'feature_count': X.shape[1],
            'training_samples': X.shape[0]
        }
        
        # Calculate feature importance
        if hasattr(self.model, 'feature_importances_'):
            feature_importance = {
                f"feature_{i}": float(imp) 
                for i, imp in enumerate(self.model.feature_importances_)
            }
        else:
            feature_importance = {}
        
        # Create a new model version
        MLModel.objects.create(
            name='reward_model',
            version=f"v{timezone.now().strftime('%Y%m%d%H%M')}",
            description='Trained reward model for schedule evaluation',
            parameters=model_params,
            feature_importance=feature_importance,
            performance_metrics={
                'training_score': float(self.model.score(X, y))
            },
            is_active=True
        )
        
        # Deactivate old models
        MLModel.objects.filter(
            name='reward_model',
            is_active=True
        ).exclude(
            version=f"v{timezone.now().strftime('%Y%m%d%H%M')}"
        ).update(is_active=False)
        
        print(f"Reward model trained successfully with {len(X)} examples")
```

## Background Tasks Implementation

### Celery Configuration

```python
# brightpath/settings/base.py
CELERY_BROKER_URL = os.environ.get('CELERY_BROKER_URL', 'redis://localhost:6379/0')
CELERY_RESULT_BACKEND = os.environ.get('CELERY_RESULT_BACKEND', 'redis://localhost:6379/0')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60  # 30 minutes
```

```python
# brightpath/celery.py
import os
from celery import Celery

# Set the default Django settings module
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'brightpath.settings.development')

app = Celery('brightpath')

# Use Django settings for Celery
app.config_from_object('django.conf:settings', namespace='CELERY')

# Auto-discover tasks from all registered Django apps
app.autodiscover_tasks()
```

### Schedule Generation Task

```python
# tasks/schedule_generation.py
from celery import shared_task
from apps.ml.services import ScheduleGeneratorService
from apps.children.models import Child
from django.contrib.auth import get_user_model

User = get_user_model()

@shared_task
def generate_schedule_task(user_id, child_id, week_start_date):
    """Background task to generate a schedule using ML model."""
    try:
        user = User.objects.get(id=user_id)
        child = Child.objects.get(id=child_id, family__user=user)
        
        service = ScheduleGeneratorService()
        schedule = service.generate_schedule(
            user=user,
            child_id=child_id,
            week_start_date=week_start_date
        )
        
        return {
            'schedule_id': schedule.id,
            'success': True
        }
    except Exception as e:
        return {
            'error': str(e),
            'success': False
        }
```

### Model Training Task

```python
# tasks/model_training.py
from celery import shared_task
from celery.schedules import crontab
from celery.task import periodic_task
from apps.ml.services import RewardModelService

@shared_task
def train_reward_model_task():
    """Background task to train the reward model."""
    try:
        service = RewardModelService()
        service.train_model()
        return {'success': True}
    except Exception as e:
        return {
            'error': str(e),
            'success': False
        }

# Periodic task to train reward model weekly
@periodic_task(
    run_every=(crontab(hour=0, minute=0, day_of_week=1)),  # Monday at midnight
    name="train_reward_model_weekly"
)
def train_reward_model_weekly():
    """Weekly task to retrain the reward model."""
    return train_reward_model_task.delay()
```

## Utilities and Permissions

### Custom Permissions

```python
# utils/permissions.py
from rest_framework import permissions

class IsOwner(permissions.BasePermission):
    """
    Allow access only to the owner of the object.
    Object must have a 'user' attribute that is a ForeignKey to User.
    """
    def has_object_permission(self, request, view, obj):
        return obj.user == request.user

class IsOwnerOfFamily(permissions.BasePermission):
    """
    Allow access only to the owner of the family.
    Object must have a 'family' attribute that is a ForeignKey to Family.
    """
    def has_object_permission(self, request, view, obj):
        return obj.family.user == request.user

class IsOwnerOfChild(permissions.BasePermission):
    """
    Allow access only to the owner of the child.
    Object must have a 'child' attribute that is a ForeignKey to Child.
    """
    def has_object_permission(self, request, view, obj):
        return obj.child.family.user == request.user

class IsOwnerOfChildInSchedule(permissions.BasePermission):
    """
    Allow access only to the owner of the schedule's child.
    """
    def has_object_permission(self, request, view, obj):
        return obj.child.family.user == request.user
```

### Questionnaire Service

```python
# questionnaire/services.py
from django.db import transaction
from django.utils import timezone
import json
from apps.users.models import User
from apps.families.models import Family, FamilyCommitment
from apps.children.models import Child, ChildCommitment
from apps.subjects.models import Subject

class QuestionnaireService:
    """Service for processing questionnaire data and saving to database."""
    
    @transaction.atomic
    def save_questionnaire_data(self, user_id, data):
        """
        Process and save questionnaire responses to PostgreSQL.
        
        Args:
            user_id: The ID of the user submitting the questionnaire
            data: The complete questionnaire data
            
        Returns:
            Tuple of (success, family_id)
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
            
            # Process preferences into structured JSONB
            family.preferences = {
                'schedulingFlexibility': data['section5']['flexibility'],
                'homeschoolingPhilosophy': data['section7']['homeschoolingPhilosophy'],
                'schedulingStyle': data['section7']['schedulingStyle'],
                'planningApproach': data['section7']['planningApproach'],
                'activityPreferences': data['section7']['activityPreferences']
            }
            
            family.save()
            
            # Save family commitments if any
            if 'commitments' in data.get('section3', {}):
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
            for child_data in data.get('children', []):
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
                    'section4': child_data.get('section4', {}),
                    'section6': child_data.get('section6', {})
                }
                
                # Process learning preferences
                if 'section6' in child_data:
                    child.learning_preferences = {
                        'bestLearningTimes': child_data['section6'].get('bestLearningTimes', []),
                        'homeschoolingHours': {
                            'startTime': child_data['section6'].get('startTime', '09:00'),
                            'endTime': child_data['section6'].get('endTime', '15:00')
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
                        subject = Subject(
                            child=child,
                            name=subject_data['name'],
                            is_core_subject=subject_data.get('isCoreSubject', False),
                            session_duration=subject_data.get('sessionDuration', 30),
                            frequency=subject_data.get('frequency', 'daily'),
                            parent_involvement=subject_data.get('parentInvolvement', 'some_assistance'),
                            fixed_time_required=subject_data.get('fixedTime', {}).get('required', False),
                            interest_level=subject_data.get('interestLevel', 3)
                        )
                        
                        # Set preferred time if applicable
                        if subject.fixed_time_required and 'preferredTime' in subject_data.get('fixedTime', {}):
                            subject.preferred_time = subject_data['fixedTime']['preferredTime']
                        
                        # Store additional metadata
                        metadata = {k: v for k, v in subject_data.items() 
                                   if k not in ['name', 'isCoreSubject', 'sessionDuration', 
                                              'frequency', 'parentInvolvement', 'fixedTime', 
                                              'interestLevel']}
                        if metadata:
                            subject.metadata = metadata
                        
                        subject.save()
            
            # Mark user as having completed questionnaire
            user.questionnaire_completed = True
            user.questionnaire_completed_at = timezone.now()
            user.save()
            
            return True, family.id
            
        except Exception as e:
            # Log the error
            print(f"Error saving questionnaire data: {str(e)}")
            # In a real implementation, use proper logging
            return False, str(e)
```

## Testing Implementation

### Unit Tests

```python
# children/tests.py
from django.test import TestCase
from django.urls import reverse
from rest_framework.test import APIClient
from rest_framework import status
from apps.users.models import User
from apps.families.models import Family
from apps.children.models import Child
import json

class ChildModelTests(TestCase):
    """Test the Child model."""
    
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='password123'
        )
        self.family = Family.objects.create(
            user