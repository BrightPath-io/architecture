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

BrightPath will follow a modern, scalable architecture with clear separation of concerns, using MongoDB for persistent storage and browser local storage for questionnaire progress in the initial prototype:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  React Frontend │────▶│  Django API     │────▶│  MongoDB        │
│  (Vite)         │◀────│  (REST)         │◀────│  (Document DB)  │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                        │                     │   
        │                        │                     │   
        ▼                        ▼                     ▼   
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
   - **MongoDB**: Primary database for persistent storage
     - Document-oriented data model for flexible schema
     - Natural fit for hierarchical data structures
     - Efficient storage for final questionnaire responses and schedule data
   - **Browser Local Storage**: For questionnaire progress in initial prototype
     - Save and continue functionality across sessions
     - Draft questionnaire responses storage
     - Progress tracking without additional infrastructure

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
- **Database**: MongoDB (primary)
- **ML Integration**: Python-based ML models with scikit-learn/TensorFlow

### Architecture
The backend will follow Django's architecture with adaptations for MongoDB:

1. **Models**: MongoDB document schemas (using Djongo or PyMongo)
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
   - MongoDB persistence for completed questionnaires
   - Multi-child flow management
   - No server-side progress tracking in the initial prototype

3. **ML Integration**
   - Schedule generator model integration
   - Reward model for feedback processing
   - ML parameter storage in MongoDB
   - Continuous learning pipeline

4. **Schedule Management**
   - ML-powered schedule generation
   - Schedule storage in MongoDB
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
├── db/
│   ├── connection.py
│   ├── schemas.py
│   └── validators.py
├── manage.py
└── requirements.txt
```

## Database Design

The database design will utilize MongoDB for persistent storage, with browser local storage for questionnaire progress in the initial prototype.

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
  ],
  "questionnaireCompleted": true,
  "questionnaireCompletedAt": ISODate("2025-04-01T10:30:00Z")
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

#### 5. ml_models Collection
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

### Local Storage Schema

For the initial prototype, the browser's localStorage will be used to store questionnaire progress and draft responses. The following key patterns will be used:

```
// Overall questionnaire progress
questionnaire_progress_{familyId} -> {
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

// Draft responses for each section
questionnaire_draft_{familyId}_{sectionId} -> {
  // Draft responses for current section
}

// Child-specific draft responses
questionnaire_draft_{familyId}_{childId}_{sectionId} -> {
  // Child-specific draft responses
}
```

### Implementation Considerations

#### 1. Indexing Strategy for MongoDB

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

#### 2. Local Storage Limitations and Solutions

- **Storage Limit**: Most browsers limit localStorage to 5-10MB per domain. Solutions:
  - Minimal data storage (only essential responses and state)
  - Data compression for larger responses
  - Periodic cleanup of unnecessary data

- **Data Loss Risks**: localStorage can be cleared by users. Solutions:
  - Regular reminders to complete the questionnaire
  - Automatic syncing of completed sections to MongoDB when possible
  - Clear instructions to not clear browser data during questionnaire

- **Security Considerations**: localStorage is not secure for sensitive data. Solutions:
  - Store only non-sensitive information locally
  - Final submission of data occurs through secure API connection to backend
  - Encrypt any potentially sensitive information with a session key

- **Cross-Device Limitations**: localStorage is device-specific. Solutions:
  - Clearly communicate that questionnaire progress is tied to current device
  - Offer to email a resumption link for later continuation
  - In future iterations, migrate to server-based progress tracking with Redis/Valkey

#### 3. Migration Path to Server-Side Storage

As part of the future enhancements, a clear migration path should be established for moving from local storage to Redis/Valkey for questionnaire progress:

1. **Data Migration**:
   - Add backend endpoints for saving and retrieving questionnaire progress
   - Implement dual-write approach during transition phase
   - Gradually migrate to server-side storage while maintaining backward compatibility

2. **Feature Enhancements**:
   - Multi-device continuation
   - Real-time progress tracking
   - Advanced analytics on questionnaire completion rates
   - Better handling of multi-user scenarios

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
- Use MongoDB for storing training data and evaluation metrics

## Development Roadmap

### Phase 1: Foundation (Months 1-2)
- Set up project infrastructure
- Implement MongoDB setup
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
- **Database Tests**: Test MongoDB interactions
- **Performance Tests**: Load testing with Locust

### Database Testing
- **Schema Validation**: Test MongoDB validation rules
- **Indexing Performance**: Validate query performance

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
- **Backup Strategy**: Automated daily backups for MongoDB
- **Scaling Strategy**: Horizontal scaling for MongoDB

### Digital Ocean Infrastructure
- **Web Servers**: Digital Ocean Droplets for hosting the Django application
- **Load Balancer**: Digital Ocean Load Balancer for distributing traffic
- **Databases**: MongoDB Atlas
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
- **Redis/Valkey Integration**: Migrate from local storage to Redis/Valkey
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