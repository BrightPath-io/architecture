# BrightPath: Homeschooling Scheduling Platform
## Updated Architecture and Implementation Plan

## Table of Contents
1. [System Overview](#system-overview)
2. [System Architecture](#system-architecture)
3. [Frontend Implementation](#frontend-implementation)
4. [Backend Implementation](#backend-implementation)
5. [Database Design](#database-design)
6. [Machine Learning Implementation](#machine-learning-implementation)
7. [Development Roadmap](#development-roadmap)
8. [Testing Strategy](#testing-strategy)
9. [Deployment Strategy](#deployment-strategy)
10. [COPPA Compliance](#coppa-compliance)
11. [Future Enhancements](#future-enhancements)

## System Overview

BrightPath is a homeschooling scheduling platform that leverages AI/ML to help parents create and maintain personalized educational schedules for their children. The system takes into account individual constraints, needs, and teaching styles to generate optimal schedules and continuously improves through user feedback.

**Mission Statement:** BrightPath is your partner that supports and empowers you to intuitively guide your child through the homeschool journey you create.

**Primary User Persona:** First-time homeschoolers like Sarah Kim who need guidance, structure, and confidence in their homeschooling journey.

## System Architecture

BrightPath will follow a modern, scalable architecture with clear separation of concerns, using PostgreSQL with JSONB fields for persistent storage and browser local storage for questionnaire progress in the initial prototype:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  React Frontend │────▶│  Django API     │────▶│  PostgreSQL     │
│  (Vite)         │◀────│  (REST)         │◀────│  (with JSONB)   │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                        │                       
        │                        │                       
        ▼                        ▼                       
┌─────────────────┐     ┌─────────────────┐     
│                 │     │                 │     
│  Local Storage  │     │  ML Services    │     
│  (Questionnaire)│     │                 │     
└─────────────────┘     └─────────────────┘     
```

### Key Components:

1. **Client Application**
   - React-based SPA using Vite for fast development
   - Responsive design for all devices
   - Code splitting for performance optimization
   - Local storage integration for questionnaire progress

2. **API Server**
   - Django + Django REST Framework
   - RESTful API endpoints
   - Authentication and authorization with Google OAuth

3. **Database Strategy**
   - **PostgreSQL with JSONB fields**:
     - Relational tables for structured data (users, families, children, subjects)
     - JSONB fields for flexible, hierarchical data (preferences, schedules, ML model parameters)
     - Best of both worlds: Schema enforcement where needed, flexibility where appropriate

4. **Browser Local Storage**:
   - For questionnaire progress in initial prototype
   - Save and continue functionality across sessions
   - Draft questionnaire responses storage
   - Progress tracking without additional infrastructure

5. **ML Services**
   - Schedule Generator Model
   - Reward Model for feedback processing
   - Continuous improvement pipeline

6. **Infrastructure**
   - Containerized deployment with Docker
   - CI/CD pipeline with GitHub Actions
   - Monitoring and logging

## Frontend Implementation

### Technology Stack
- **Framework**: React
- **Build Tool**: Vite
- **Routing**: React Router
- **State Management**: React Context API + hooks (consider Redux for complex state)
- **Styling**: CSS Modules
- **Component Library**: Radix
- **Local Storage**: Browser's built-in localStorage API

### Architecture
The frontend will follow a modular architecture with:

1. **Layouts**: Page-level components that define the overall structure
2. **Views**: Route-specific components that compose the UI for each page
3. **Components**: Reusable UI elements
4. **Hooks**: Custom React hooks for shared logic
5. **Services**: API client and other service abstractions
6. **Utils**: Helper functions and utilities
7. **Storage**: Local storage service for managing questionnaire progress

### Key Features
1. **Onboarding Flow**
   - TurboTax-like multi-step questionnaire
   - Progressive disclosure pattern
   - Smart defaults based on child age
   - Multi-child support
   - Save and continue functionality using local storage
   - Automatic syncing of completed questionnaire data to backend

2. **Schedule Dashboard**
   - Interactive schedule view
   - Drag-and-drop adjustments
   - Daily, weekly, and monthly views
   - Activity completion tracking

3. **Schedule Generation**
   - AI-powered schedule creation
   - Preference-based customization
   - Real-time adjustments

4. **Feedback System**
   - Activity completion feedback
   - Schedule satisfaction ratings
   - Implicit feedback through user interactions

5. **Resource Library**
   - Subject-specific resources
   - Curriculum recommendations
   - Community-shared materials

### Code Organization
```
client/
├── public/
├── src/
│   ├── assets/
│   ├── components/
│   │   ├── common/
│   │   ├── feedback/
│   │   ├── onboarding/
│   │   └── schedule/
│   ├── hooks/
│   │   ├── useLocalStorage.js
│   │   └── ...
│   ├── layouts/
│   ├── routes/
│   ├── services/
│   │   ├── api/
│   │   ├── auth/
│   │   └── storage/
│   │       ├── questionnaireStorage.js
│   │       └── ...
│   ├── utils/
│   ├── views/
│   │   ├── dashboard/
│   │   ├── onboarding/
│   │   ├── profile/
│   │   └── schedule/
│   ├── App.jsx
│   └── main.jsx
├── .eslintrc.js
├── package.json
├── tsconfig.json
└── vite.config.js
```

### Local Storage Implementation
```javascript
// src/services/storage/questionnaireStorage.js

// Save questionnaire progress to local storage
export const saveQuestionnaireProgress = (familyId, progressData) => {
  const key = `questionnaire_progress_${familyId}`;
  const data = JSON.stringify({
    ...progressData,
    lastUpdated: new Date().toISOString()
  });
  localStorage.setItem(key, data);
};

// Retrieve questionnaire progress from local storage
export const getQuestionnaireProgress = (familyId) => {
  const key = `questionnaire_progress_${familyId}`;
  const data = localStorage.getItem(key);
  return data ? JSON.parse(data) : null;
};

// Save draft responses for a specific section
export const saveDraftResponses = (familyId, sectionId, responses) => {
  const key = `questionnaire_draft_${familyId}_${sectionId}`;
  localStorage.setItem(key, JSON.stringify(responses));
};

// Get draft responses for a section
export const getDraftResponses = (familyId, sectionId) => {
  const key = `questionnaire_draft_${familyId}_${sectionId}`;
  const data = localStorage.getItem(key);
  return data ? JSON.parse(data) : null;
};

// Clear questionnaire data when complete
export const clearQuestionnaireData = (familyId) => {
  // Get all keys in localStorage
  const keys = Object.keys(localStorage);
  
  // Filter keys related to this family's questionnaire
  const familyKeys = keys.filter(key => 
    key.startsWith(`questionnaire_progress_${familyId}`) || 
    key.startsWith(`questionnaire_draft_${familyId}_`)
  );
  
  // Remove each key
  familyKeys.forEach(key => localStorage.removeItem(key));
};
```

## Backend Implementation

### Technology Stack
- **Framework**: Django + Django REST Framework
- **API**: RESTful endpoints
- **Authentication**: Google OAuth
- **Database**: PostgreSQL with JSONB fields
- **ML Integration**: Python-based ML models with scikit-learn/TensorFlow

### Architecture
The backend will follow Django's architecture with adaptations for PostgreSQL+JSONB:

1. **Models**: Django models with PostgreSQL-specific fields (JSONField)
2. **Views**: API endpoints and business logic
3. **Serializers**: Data transformation and validation
4. **Services**: Business logic and ML model integration
5. **Utils**: Helper functions and utilities

### Key Features
1. **User Management**
   - Google OAuth integration
   - Profile management
   - Child profile management
   - Session handling

2. **Questionnaire System**
   - Endpoints to receive final questionnaire responses
   - PostgreSQL JSONB fields for completed questionnaires
   - Multi-child flow management
   - No server-side progress tracking in the initial prototype

3. **ML Integration**
   - Schedule generator model integration
   - Reward model for feedback processing
   - ML parameter storage in JSONB fields
   - Continuous learning pipeline

4. **Schedule Management**
   - ML-powered schedule generation
   - Schedule storage using hybrid relational/JSONB approach
   - Activity completion tracking

5. **Feedback System**
   - Explicit and implicit feedback collection
   - Feedback analysis for ML improvement
   - Schedule adjustments

### Code Organization
```
api/
├── brightpath/
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── users/
│   ├── children/
│   ├── schedules/
│   ├── subjects/
│   ├── feedback/
│   └── questionnaire/
├── ml/
│   ├── generator/
│   ├── reward/
│   └── training/
├── utils/
├── manage.py
└── requirements.txt
```

## Database Design

The database design will use PostgreSQL with JSONB fields to leverage both relational structure and document flexibility in a single database, alongside browser local storage for questionnaire progress in the initial prototype.

### Key Tables and Fields

1. **users**: Core user authentication and profile data
   - Standard fields: id, email, name, google_id, etc.
   - Questionnaire completion tracking

2. **families**: Family data and preferences
   - Core fields: id, user_id, name
   - JSONB fields: 
     - preferences (detailed preferences from questionnaire)
     - questionnaire_responses (complete family-wide responses)

3. **children**: Child profiles and learning preferences
   - Core fields: id, family_id, name, age
   - JSONB fields:
     - learning_preferences (learning times, styles)
     - questionnaire_responses (child-specific responses)

4. **subjects**: Academic subjects for each child
   - Core fields: name, session_duration, frequency, etc.
   - JSONB field: metadata (flexible additional data)

5. **schedules**: Weekly schedules
   - Core fields: child_id, week_start_date, version
   - JSONB fields:
     - schedule_data (complete schedule structure)
     - patterns (ML-detected patterns and insights)

6. **schedule_items**: Individual activities in schedules
   - Core fields: schedule_id, day_date, item_type, name, times
   - JSONB field: item_data (flexible item-specific details)

7. **activity_logs**: Record of activity completion
   - Core fields: schedule_item_id, date, activity_name
   - JSONB fields:
     - scheduled (original scheduled data)
     - actual (what actually happened, including completion)

### PostgreSQL-Specific Features

1. **JSONB Indexing**:
   - GIN indexes for containment operations
   - Path-specific indexes for frequently accessed JSON paths
   - B-tree indexes for equality comparisons on JSON values

2. **JSONB Queries**:
   - Containment operators (@>, ?, ?&, ?|)
   - Path expressions for direct value access
   - JSON functions for manipulation and transformation

3. **Schema Evolution**:
   - Core fields enforced by relational schema
   - Flexible evolution of JSONB fields without migrations

4. **Transactional Integrity**:
   - ACID guarantees for all operations
   - Atomicity across related tables and JSONB fields

## Machine Learning Implementation

BrightPath will implement two key ML models:

### 1. Schedule Generator Model

**Purpose**: Generate personalized schedules based on user preferences and constraints.

**Implementation**:
- **Model Type**: Initially a supervised learning model (Random Forest or Gradient Boosting)
- **Features**:
  - User questionnaire responses (Likert scale values)
  - Child age and characteristics
  - Subject preferences
  - Time constraints
- **Output**: Schedule configuration (PostgreSQL JSONB structure)
- **Training**: Initially trained on synthetic data, then refined with real user data

**Development Steps**:
1. Create synthetic training data based on expected user profiles
2. Train initial model with basic features
3. Implement A/B testing to compare different schedule generation approaches
4. Refine model based on user feedback

### 2. Reward Model

**Purpose**: Evaluate and improve schedule quality based on user feedback.

**Implementation**:
- **Model Type**: Regression model for satisfaction prediction
- **Features**:
  - Schedule characteristics
  - User feedback (explicit and implicit)
  - Schedule adjustments made by users
- **Output**: Satisfaction score (0-1)
- **Training**: Trained on user feedback data

**Development Steps**:
1. Implement feedback collection mechanisms
2. Build initial reward model based on explicit feedback
3. Integrate implicit feedback (e.g., schedule adjustments)
4. Use reinforcement learning to improve the generator model

### Feedback Loop Implementation

```
┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │
│  User Feedback  │────▶│  Reward Model   │
│                 │     │                 │
└─────────────────┘     └─────────────────┘
                               │
                               ▼
┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │
│  New Schedules  │◀────│ Generator Model │
│                 │     │                 │
└─────────────────┘     └─────────────────┘
```

### ML Model Storage

ML models and their parameters will be stored in PostgreSQL using JSONB fields:

```python
class MLModel(models.Model):
    name = models.CharField(max_length=255)
    version = models.CharField(max_length=50)
    description = models.TextField(blank=True)
    
    # JSONB fields for flexible storage
    parameters = JSONField()  # Model parameters
    feature_importance = JSONField(default=dict)  # Feature importance data
    performance_metrics = JSONField(default=dict)  # Performance metrics
    
    is_active = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

## Development Roadmap

### Phase 1: Foundation (Months 1-2)
- Set up project infrastructure
- Implement PostgreSQL database with JSONB fields
- Create database schemas and indexes
- Implement Google OAuth authentication
- Build local storage service for questionnaire progress
- Create initial questionnaire UI components

### Phase 2: Core Functionality (Months 3-4)
- Implement questionnaire with local storage save/continue functionality
- Create schedule generation (rule-based initially)
- Build schedule visualization and management
- Develop activity tracking and completion system
- Build final questionnaire submission process

### Phase 3: ML Integration (Months 5-6)
- Develop initial ML models
- Integrate ML models with schedule generation
- Implement feedback collection and processing
- Create A/B testing framework
- Refine user experience based on initial feedback

### Phase 4: Enhancement and Optimization (Months 7-8)
- Implement advanced schedule features
- Enhance ML models with more data
- Add resource recommendations
- Optimize database queries
- Conduct user testing and refinement

### Phase 5: Launch Preparation (Months 9-10)
- Comprehensive testing
- Security audits
- Performance optimization
- Documentation
- Beta testing with real users

## Testing Strategy

### Frontend Testing
- **Unit Tests**: Jest for component and utility testing
- **Integration Tests**: React Testing Library for component integration
- **E2E Tests**: Cypress for end-to-end testing
- **Visual Regression**: Storybook + Chromatic
- **Local Storage Tests**: Specific tests for localStorage interactions

### Backend Testing
- **Unit Tests**: Django test framework for models and services
- **API Tests**: DRF test framework for API endpoints
- **Database Tests**: Test PostgreSQL JSONB interactions
- **Performance Tests**: Load testing with Locust

### Database Testing
- **Schema Validation**: Test PostgreSQL validation rules
- **JSONB Query Tests**: Test complex JSONB queries and operations
- **Indexing Performance**: Validate query performance with realistic data volumes

### ML Testing
- **Model Validation**: Cross-validation and holdout testing
- **A/B Testing**: Compare different model versions
- **Fairness Testing**: Ensure models don't introduce bias
- **Performance Metrics**: Track accuracy, precision, recall, and F1 score

## Deployment Strategy

### Infrastructure
- **Containerization**: Docker for consistent environments
- **Orchestration**: Docker Compose for development, Kubernetes for production
- **CI/CD**: GitHub Actions for automated pipelines
- **Monitoring**: Prometheus + Grafana for metrics and alerting
- **Logging**: Structured logging with ELK stack for centralized logging

### Environments
- **Development**: Local development environment
- **Testing**: Automated testing environment
- **Staging**: Pre-production environment for final testing
- **Production**: Live environment for end users

### Database Deployment
- **PostgreSQL**: Managed PostgreSQL service in Digital Ocean
- **Backup Strategy**: Automated daily backups
- **Scaling Strategy**: Connection pooling and read replicas as needed

### Digital Ocean Infrastructure
- **Web Servers**: Digital Ocean Droplets for hosting the Django application
- **Load Balancer**: Digital Ocean Load Balancer for distributing traffic
- **Database**: Managed PostgreSQL Database Cluster
- **Object Storage**: Digital Ocean Spaces for static files and user uploads
- **Monitoring**: Digital Ocean Monitoring integrated with Prometheus/Grafana

## COPPA Compliance

To ensure compliance with the Children's Online Privacy Protection Act (COPPA):

### Data Collection
- Implement explicit parental consent mechanisms
- Limit collection of children's personal information
- Store only necessary information for platform functionality

### Privacy Policy
- Create a clear, comprehensive privacy policy
- Explain data collection, use, and sharing practices
- Provide mechanisms for parents to review and delete children's data

### Security Measures
- Implement robust data encryption
- Establish access controls and authentication
- Regularly audit security practices

### Implementation Steps
1. Consult with legal experts on COPPA requirements
2. Design consent flows for parent registration
3. Implement data minimization practices
4. Create age-appropriate features and interfaces
5. Establish regular compliance reviews

## Future Enhancements

### Enhanced Session Management
- **Server-Side Storage**: Migrate from localStorage to server-side storage
- **Multi-device Support**: Enable questionnaire continuation across devices
- **Real-time Synchronization**: Real-time progress updates
- **Analytics**: Advanced questionnaire completion analytics
- **Enhanced Security**: Better protection for in-progress data

### Generative AI Integration
- **Personalized Learning Paths**: AI-generated curriculum recommendations
- **Interactive Learning Assistants**: Virtual tutors for children
- **Creative Content Generation**: Custom educational materials
- **AI-Powered Lesson Planning**: Automated lesson plan creation
- **Virtual Co-Op Activities**: AI-facilitated group learning

### Community Features
- **Parent Forums**: Community discussion and support
- **Resource Sharing**: User-contributed educational resources
- **Co-Op Coordination**: Tools for organizing group activities
- **Mentor Matching**: Connect experienced homeschoolers with beginners

### Advanced Analytics
- **Learning Progress Tracking**: Visualize child's educational journey
- **Effectiveness Analysis**: Measure schedule effectiveness
- **Comparative Insights**: Anonymized benchmarking
- **Predictive Planning**: Anticipate future educational needs

### Mobile Applications
- **Native Mobile Apps**: iOS and Android applications
- **Offline Functionality**: Access schedules without internet
- **Push Notifications**: Reminders and updates
- **Mobile-Specific Features**: Location-based educational opportunities