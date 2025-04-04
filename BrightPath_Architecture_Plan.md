# BrightPath: Homeschooling Scheduling Platform
## Architecture and Implementation Plan

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

BrightPath will follow a modern, scalable architecture with clear separation of concerns, using a hybrid database approach with MongoDB and Redis/Valkey:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  React Frontend │────▶│  Django API     │────▶│  MongoDB        │
│  (Vite)         │◀────│  (REST)         │◀────│  (Document DB)  │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │   ▲                   │   ▲
                               │   │                   │   │
                               ▼   │                   ▼   │
                        ┌─────────────────┐     ┌─────────────────┐
                        │                 │     │                 │
                        │  ML Services    │─────│  Redis/Valkey   │
                        │                 │     │  (Cache/RT)     │
                        └─────────────────┘     └─────────────────┘
```

### Key Components:

1. **Client Application**
   - React-based SPA using Vite for fast development
   - Responsive design for all devices
   - Code splitting for performance optimization

2. **API Server**
   - Django + Django REST Framework
   - RESTful API endpoints
   - Authentication and authorization with Google OAuth

3. **Database (Hybrid Approach)**
   - **MongoDB**: Primary database for persistent storage
     - Document-oriented data model for flexible schema
     - Natural fit for hierarchical data structures
     - Efficient storage for questionnaire responses and schedule data
   - **Redis/Valkey**: Caching and real-time functionality
     - Session management
     - Real-time schedule updates
     - Caching for performance optimization

4. **ML Services**
   - Schedule Generator Model
   - Reward Model for feedback processing
   - Continuous improvement pipeline

5. **Infrastructure**
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

### Architecture
The frontend will follow a modular architecture with:

1. **Layouts**: Page-level components that define the overall structure
2. **Views**: Route-specific components that compose the UI for each page
3. **Components**: Reusable UI elements
4. **Hooks**: Custom React hooks for shared logic
5. **Services**: API client and other service abstractions
6. **Utils**: Helper functions and utilities

### Key Features
1. **Onboarding Flow**
   - TurboTax-like multi-step questionnaire
   - Progressive disclosure pattern
   - Smart defaults based on child age
   - Multi-child support
   - Save and continue functionality

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
│   ├── layouts/
│   ├── routes/
│   ├── services/
│   │   ├── api/
│   │   └── auth/
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

## Backend Implementation

### Technology Stack
- **Framework**: Django + Django REST Framework
- **API**: RESTful endpoints
- **Authentication**: Google OAuth
- **Database**: MongoDB (primary) + Redis/Valkey (caching/real-time)
- **ML Integration**: Python-based ML models with scikit-learn/TensorFlow

### Architecture
The backend will follow Django's architecture with adaptations for MongoDB:

1. **Models**: MongoDB document schemas (using Djongo or PyMongo)
2. **Views**: API endpoints and business logic
3. **Serializers**: Data transformation and validation
4. **Services**: Business logic and ML model integration
5. **Utils**: Helper functions and utilities
6. **Cache**: Redis/Valkey interface for caching and real-time updates

### Key Features
1. **User Management**
   - Google OAuth integration
   - Profile management
   - Child profile management
   - Session handling with Redis/Valkey

2. **Questionnaire System**
   - Progressive multi-step questionnaire
   - Draft response storage in Redis/Valkey
   - Final response persistence in MongoDB
   - Multi-child flow management

3. **ML Integration**
   - Schedule generator model integration
   - Reward model for feedback processing
   - ML parameter storage in MongoDB
   - Feature caching in Redis/Valkey
   - Continuous learning pipeline

4. **Schedule Management**
   - ML-powered schedule generation
   - Schedule storage in MongoDB
   - Real-time updates with Redis/Valkey
   - Activity completion tracking

5. **Feedback System**
   - Explicit and implicit feedback collection
   - Feedback analysis for ML improvement
   - Real-time schedule adjustments

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
├── cache/
│   ├── client.py
│   ├── session.py
│   └── schedules.py
├── db/
│   ├── connection.py
│   ├── schemas.py
│   └── validators.py
├── manage.py
└── requirements.txt
```

## Database Design

The database design will utilize a hybrid approach with MongoDB for persistent storage and Redis/Valkey for caching and real-time functionality.

### MongoDB Collections

#### 1. families Collection
```javascript
{
  "_id": ObjectId("..."),
  "googleId": "google_provided_id", // For OAuth authentication
  "email": "sarah@example.com", // From Google OAuth
  "name": "Sarah Kim", // From questionnaire
  "createdAt": ISODate("2025-04-01"),
  "lastLogin": ISODate("2025-04-03"),
  "preferences": {
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
  },
  "commitments": [
    // Family-wide commitments
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

#### 2. children Collection
```javascript
{
  "_id": ObjectId("..."),
  "familyId": ObjectId("..."), // Reference to parent family
  "name": "Noah",
  "age": 6,
  "createdAt": ISODate("2025-04-01"),
  "commitments": [
    // Child-specific commitments
    {
      "name": "Soccer Practice",
      "frequency": "weekly",
      "daysOfWeek": ["Monday", "Wednesday"],
      "startTime": "16:00",
      "endTime": "17:00"
    }
  ],
  "learningPreferences": {
    "bestLearningTimes": ["mid-morning", "early_afternoon"],
    "homeschoolingHours": {
      "startTime": "09:00",
      "endTime": "14:00"
    }
  },
  "subjects": [
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
    // Additional subjects...
  ]
}
```

#### 3. schedules Collection
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
        // More scheduled items...
      ]
    }
    // More days...
  ]
}
```

#### 4. activity_logs Collection
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
    "startTime": "09:35", // Optional
    "endTime": "10:05" // Optional
  },
  "notes": "Noah enjoyed the counting activities today",
  "createdAt": ISODate("2025-04-07T10:05:00Z")
}
```

#### 5. questionnaire_progress Collection
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
    // Temporary draft responses
  }
}
```

#### 6. ml_models Collection
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

### Redis/Valkey Key Patterns

#### 1. Session Management
```
// Session data with 24-hour expiry
SESSION:{user_id} -> { session data as JSON }
TTL: 86400 (24 hours)

// Authentication refresh tokens
AUTH:REFRESH:{user_id} -> "refresh_token_value"
TTL: 2592000 (30 days)
```

#### 2. Questionnaire Progress Caching
```
// Detailed questionnaire progress
QUESTIONNAIRE:PROGRESS:{family_id} -> { progress data as JSON }
TTL: 3600 (1 hour)

// Temporary form data
QUESTIONNAIRE:DRAFT:{family_id}:{section_id} -> { draft responses as JSON }
TTL: 86400 (24 hours)
```

#### 3. Schedule Caching
```
// Today's schedule for quick access
SCHEDULE:TODAY:{child_id} -> { today's schedule as JSON }
TTL: 86400 (24 hours)

// Current week's schedule
SCHEDULE:WEEK:{child_id} -> { week's schedule as JSON }
TTL: 604800 (7 days)
```

#### 4. Real-time Activity Status
```
// Current status of schedule items
SCHEDULE:ITEM:STATUS:{schedule_item_id} -> { status data as JSON }
TTL: 86400 (24 hours)

// Schedule modifications in progress
SCHEDULE:EDIT:{schedule_id} -> { modification data as JSON }
TTL: 1800 (30 minutes)
```

### Indexing Strategy
For MongoDB collections:
- `families`: `googleId` (unique), `email` (unique)
- `children`: `familyId`
- `schedules`: `familyId`, `childId`, `weekStartDate`
- `activity_logs`: `familyId`, `childId`, `date`

### Data Flow Between MongoDB and Redis/Valkey
1. Write-through cache pattern for critical data
2. Cache invalidation when MongoDB data is updated
3. Fallback to MongoDB when Redis/Valkey data is unavailable
4. Appropriate TTL values based on data volatility

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
- **Output**: Schedule configuration (MongoDB document structure)
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

### ML Service Architecture

```
ml/
├── data/
│   ├── preprocessing/
│   ├── synthetic/
│   └── validation/
├── models/
│   ├── generator/
│   └── reward/
├── training/
│   ├── generator_training.py
│   └── reward_training.py
├── inference/
│   ├── generator_inference.py
│   └── reward_inference.py
└── evaluation/
    ├── metrics.py
    └── validation.py
```

### ML Model Integration with Database
- Store model parameters in MongoDB for persistence
- Cache frequently used parameters in Redis/Valkey for performance
- Use MongoDB for storing training data and evaluation metrics
- Use Redis/Valkey for real-time feature extraction and inference

## Development Roadmap

### Phase 1: Foundation (Months 1-2)
- Set up project infrastructure
- Implement MongoDB and Redis/Valkey setup
- Create database schemas and indexes
- Implement Google OAuth authentication
- Build initial questionnaire flow

### Phase 2: Core Functionality (Months 3-4)
- Implement questionnaire with save/continue functionality
- Create schedule generation (rule-based initially)
- Build schedule visualization and management
- Develop activity tracking and completion system
- Implement cache synchronization between MongoDB and Redis/Valkey

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
- Optimize database queries and caching strategy
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

### Backend Testing
- **Unit Tests**: Django test framework for models and services
- **API Tests**: DRF test framework for API endpoints
- **Database Tests**: Test MongoDB and Redis/Valkey interactions
- **Performance Tests**: Load testing with Locust

### Database Testing
- **Schema Validation**: Test MongoDB validation rules
- **Indexing Performance**: Validate query performance
- **Cache Strategy**: Test cache hit/miss rates
- **Consistency Tests**: Ensure data consistency between MongoDB and Redis/Valkey

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
- **MongoDB**: MongoDB Atlas for managed service
- **Redis/Valkey**: Digital Ocean Managed Redis service
- **Backup Strategy**: Automated daily backups for MongoDB
- **Scaling Strategy**: Horizontal scaling for both database systems

### Digital Ocean Infrastructure
- **Web Servers**: Digital Ocean Droplets for hosting the Django application
- **Load Balancer**: Digital Ocean Load Balancer for distributing traffic
- **Databases**: MongoDB Atlas and DO Managed Redis
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