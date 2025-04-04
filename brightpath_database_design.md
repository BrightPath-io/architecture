# BrightPath Database Design: MongoDB with Local Storage for Questionnaire Progress

This document outlines the revised database design for the BrightPath homeschooling scheduling platform, using MongoDB for persistent storage and browser local storage for questionnaire progress in the initial prototype.

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
  ],
  "questionnaireCompleted": true,
  "questionnaireCompletedAt": ISODate("2025-04-01T10:30:00Z")
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

### 5. ml_models Collection

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

## Local Storage Schema for Questionnaire Progress

For the initial prototype, the browser's localStorage will be used to store questionnaire progress and draft responses. This approach simplifies the initial implementation while providing a seamless user experience.

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

### 1. MongoDB Indexing Strategy

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

### 2. Local Storage Considerations

#### Benefits of Local Storage for Initial Prototype
- **Simple Implementation**: No need for additional backend infrastructure
- **Performance**: Fast access to data during the questionnaire process
- **Offline Capability**: Users can continue even with temporary connectivity issues
- **Reduced Backend Load**: Minimizes database operations during questionnaire completion

#### Limitations and Mitigation Strategies
- **Storage Limit** (5-10MB per domain):
  - Implement efficient data storage with minimal overhead
  - Periodically clean up unnecessary data
  - Monitor storage usage and warn users if approaching limits

- **Data Loss Risks**:
  - Provide clear instructions to not clear browser data during questionnaire
  - Implement "save to account" functionality at key points
  - Offer email links to resume questionnaire

- **Device-Specific Storage**:
  - Clearly communicate that progress is tied to current device
  - Provide clear UI indicators for saved progress
  - Suggest completing questionnaire in one session when possible

- **Security Considerations**:
  - Store only non-sensitive information locally
  - Use HTTPS to protect data in transit
  - Implement final submission through secure API

### 3. Data Flow Between Frontend and Backend

1. **Initial Authentication**:
   - User authenticates via Google OAuth
   - Backend creates/retrieves user in MongoDB
   - Frontend initializes local storage with family ID

2. **Questionnaire Progress**:
   - All draft responses and progress saved to local storage
   - No backend calls for saving progress in the initial prototype
   - Frontend manages questionnaire state and navigation

3. **Final Submission**:
   - When questionnaire is complete, all data is sent to backend
   - Backend validates and stores data in MongoDB
   - Frontend clears localStorage after successful submission

4. **Schedule Generation**:
   - Backend retrieves questionnaire data from MongoDB
   - ML model generates schedule based on user preferences
   - Schedule is stored in MongoDB and returned to frontend

### 4. Error Handling

- **localStorage Unavailability**:
  - Detect localStorage support on page load
  - Provide fallback option to complete questionnaire in one session
  - Warn users about potential data loss if localStorage is unavailable

- **Storage Quota Exceeded**:
  - Implement storage monitoring
  - Offer data cleanup options if quota is reached
  - Prioritize critical data if storage is limited

- **Data Corruption**:
  - Validate data structure when loading from localStorage
  - Implement recovery mechanisms for partially corrupted data
  - Log errors for debugging purposes

### 5. Migration Path to Server-Side Storage

For future iterations, a clear migration path to server-side storage with Redis/Valkey should be planned:

1. **Implementation Phases**:
   - Phase 1 (Current): localStorage for questionnaire progress
   - Phase 2: Dual storage (localStorage with optional server sync)
   - Phase 3: Primary server-side storage with Redis/Valkey
   - Phase 4: Full Redis/Valkey implementation with enhanced features

2. **Data Migration**:
   - Develop API endpoints for progress synchronization
   - Implement data conversion utilities
   - Ensure backward compatibility during transition

3. **Feature Enhancements with Server Storage**:
   - Cross-device continuation
   - Real-time updates and collaboration
   - Advanced analytics on questionnaire completion
   - Better session management and security

## Code Examples for Implementation

### 1. Frontend Local Storage Service

```javascript
// src/services/storage/questionnaireStorage.js

const STORAGE_PREFIX = 'brightpath_';

export const QuestionnaireStorageService = {
  // Save overall progress
  saveProgress: (familyId, progressData) => {
    const key = `${STORAGE_PREFIX}questionnaire_progress_${familyId}`;
    const data = {
      ...progressData,
      lastUpdated: new Date().toISOString()
    };
    try {
      localStorage.setItem(key, JSON.stringify(data));
      return true;
    } catch (error) {
      console.error('Error saving questionnaire progress:', error);
      return false;
    }
  },
  
  // Get saved progress
  getProgress: (familyId) => {
    const key = `${STORAGE_PREFIX}questionnaire_progress_${familyId}`;
    try {
      const data = localStorage.getItem(key);
      return data ? JSON.parse(data) : null;
    } catch (error) {
      console.error('Error retrieving questionnaire progress:', error);
      return null;
    }
  },
  
  // Save section draft data
  saveSectionDraft: (familyId, sectionId, data) => {
    const key = `${STORAGE_PREFIX}questionnaire_draft_${familyId}_${sectionId}`;
    try {
      localStorage.setItem(key, JSON.stringify(data));
      return true;
    } catch (error) {
      console.error(`Error saving section ${sectionId} draft:`, error);
      return false;
    }
  },
  
  // Get section draft data
  getSectionDraft: (familyId, sectionId) => {
    const key = `${STORAGE_PREFIX}questionnaire_draft_${familyId}_${sectionId}`;
    try {
      const data = localStorage.getItem(key);
      return data ? JSON.parse(data) : null;
    } catch (error) {
      console.error(`Error retrieving section ${sectionId} draft:`, error);
      return null;
    }
  },
  
  // Save child-specific section draft
  saveChildSectionDraft: (familyId, childId, sectionId, data) => {
    const key = `${STORAGE_PREFIX}questionnaire_draft_${familyId}_child_${childId}_${sectionId}`;
    try {
      localStorage.setItem(key, JSON.stringify(data));
      return true;
    } catch (error) {
      console.error(`Error saving child ${childId} section ${sectionId} draft:`, error);
      return false;
    }
  },
  
  // Get child-specific section draft
  getChildSectionDraft: (familyId, childId, sectionId) => {
    const key = `${STORAGE_PREFIX}questionnaire_draft_${familyId}_child_${childId}_${sectionId}`;
    try {
      const data = localStorage.getItem(key);
      return data ? JSON.parse(data) : null;
    } catch (error) {
      console.error(`Error retrieving child ${childId} section ${sectionId} draft:`, error);
      return null;
    }
  },
  
  // Clear all questionnaire data after successful submission
  clearQuestionnaireData: (familyId) => {
    try {
      // Get all keys in localStorage
      const keys = Object.keys(localStorage);
      
      // Filter keys related to this family's questionnaire
      const familyKeys = keys.filter(key => 
        key.startsWith(`${STORAGE_PREFIX}questionnaire_`) && 
        key.includes(`_${familyId}_`)
      );
      
      // Remove each key
      familyKeys.forEach(key => localStorage.removeItem(key));
      
      // Also remove the main progress key
      localStorage.removeItem(`${STORAGE_PREFIX}questionnaire_progress_${familyId}`);
      
      return true;
    } catch (error) {
      console.error('Error clearing questionnaire data:', error);
      return false;
    }
  },
  
  // Check available storage space
  checkStorageSpace: () => {
    try {
      const testKey = `${STORAGE_PREFIX}storage_test`;
      let data = 'a';
      
      // Try to fill storage to test limits
      while (true) {
        localStorage.setItem(testKey, data);
        data = data + data; // Double the size each time
        
        // Safety check - stop if data gets too large
        if (data.length > 1000000) {
          break;
        }
      }
    } catch (e) {
      // We reached the storage limit
      localStorage.removeItem(`${STORAGE_PREFIX}storage_test`);
      return false;
    }
    
    // If we got here without exception, storage is available
    localStorage.removeItem(`${STORAGE_PREFIX}storage_test`);
    return true;
  }
};
```

### 2. React Hook for Local Storage Integration

```javascript
// src/hooks/useQuestionnaireStorage.js

import { useState, useEffect } from 'react';
import { QuestionnaireStorageService } from '../services/storage/questionnaireStorage';

export function useQuestionnaireStorage(familyId, sectionId, initialData = null) {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  // Load data on initial render
  useEffect(() => {
    const loadData = async () => {
      try {
        setIsLoading(true);
        setError(null);
        
        // Get data from localStorage
        const savedData = QuestionnaireStorageService.getSectionDraft(familyId, sectionId);
        
        // Use saved data if available, otherwise use initial data
        setData(savedData || initialData);
      } catch (err) {
        setError('Failed to load saved data');
        console.error(err);
        setData(initialData); // Fallback to initial data
      } finally {
        setIsLoading(false);
      }
    };
    
    loadData();
  }, [familyId, sectionId, initialData]);

  // Save data function
  const saveData = async (newData) => {
    try {
      setData(newData);
      QuestionnaireStorageService.saveSectionDraft(familyId, sectionId, newData);
      return true;
    } catch (err) {
      setError('Failed to save data');
      console.error(err);
      return false;
    }
  };

  return { data, isLoading, error, saveData };
}
```

### 3. Backend Questionnaire Submission Endpoint

```python
# api/apps/questionnaire/views.py

from django.http import JsonResponse
from rest_framework.decorators import api_view, authentication_classes, permission_classes
from rest_framework.authentication import TokenAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework import status
from django.utils import timezone

from .serializers import QuestionnaireSerializer
from ..db.models import Family, Child

@api_view(['POST'])
@authentication_classes([TokenAuthentication])
@permission_classes([IsAuthenticated])
def submit_questionnaire(request):
    """
    Endpoint for submitting the completed questionnaire data
    """
    serializer = QuestionnaireSerializer(data=request.data)
    if serializer.is_valid():
        # Get the authenticated family
        family = Family.objects.get(user=request.user)
        
        # Update family preferences and commitments
        family.preferences = serializer.validated_data.get('preferences')
        family.commitments = serializer.validated_data.get('commitments')
        family.questionnaireCompleted = True
        family.questionnaireCompletedAt = timezone.now()
        family.save()
        
        # Process children data
        children_data = serializer.validated_data.get('children', [])
        for child_data in children_data:
            child_id = child_data.get('id')
            if child_id and child_id.startswith('temp_'):
                # This is a new child that was created during the questionnaire
                child = Child(
                    familyId=family.id,
                    name=child_data.get('name'),
                    age=child_data.get('age'),
                    commitments=child_data.get('commitments', []),
                    learningPreferences=child_data.get('learningPreferences', {}),
                    subjects=child_data.get('subjects', [])
                )
                child.save()
            else:
                # Update existing child
                try:
                    child = Child.objects.get(id=child_id, familyId=family.id)
                    child.name = child_data.get('name', child.name)
                    child.age = child_data.get('age', child.age)
                    child.commitments = child_data.get('commitments', child.commitments)
                    child.learningPreferences = child_data.get('learningPreferences', child.learningPreferences)
                    child.subjects = child_data.get('subjects', child.subjects)
                    child.save()
                except Child.DoesNotExist:
                    return Response(
                        {"error": f"Child with ID {child_id} not found"}, 
                        status=status.HTTP_404_NOT_FOUND
                    )
        
        # Trigger schedule generation (could be done asynchronously)
        # schedule_generation_service.generate_schedules_for_family(family.id)
        
        return Response({"status": "success", "message": "Questionnaire data saved successfully"})
    else:
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

## Progressive Disclosure Implementation for Questionnaire

The questionnaire will follow a progressive disclosure pattern to guide users through the process:

```javascript
// src/components/questionnaire/ProgressiveSection.jsx

import React, { useState, useEffect } from 'react';
import { QuestionnaireStorageService } from '../../services/storage/questionnaireStorage';

export default function ProgressiveSection({ 
  familyId, 
  sectionId, 
  previousSection,
  nextSection,
  children 
}) {
  const [isVisible, setIsVisible] = useState(false);
  const [completed, setCompleted] = useState(false);
  
  useEffect(() => {
    // Load progress from localStorage
    const progress = QuestionnaireStorageService.getProgress(familyId);
    
    if (progress) {
      // Determine if this section should be visible
      const sectionStatus = progress.progress[`section${sectionId}`];
      const previousSectionStatus = previousSection 
        ? progress.progress[`section${previousSection}`] 
        : 'complete';
      
      // Section is visible if previous is complete or if this section is in progress
      setIsVisible(
        previousSectionStatus === 'complete' || 
        sectionStatus === 'in_progress' ||
        sectionStatus === 'complete'
      );
      
      // Set completed state
      setCompleted(sectionStatus === 'complete');
    } else if (sectionId === 1) {
      // First section is always visible
      setIsVisible(true);
    }
  }, [familyId, sectionId, previousSection]);
  
  const markSectionCompleted = () => {
    // Get current progress
    const progress = QuestionnaireStorageService.getProgress(familyId) || {
      currentSection: 1,
      progress: {}
    };
    
    // Update progress for this section
    progress.progress[`section${sectionId}`] = 'complete';
    
    // Set next section as current and in-progress
    if (nextSection) {
      progress.currentSection = nextSection;
      progress.progress[`section${nextSection}`] = 'in_progress';
    }
    
    // Save updated progress
    QuestionnaireStorageService.saveProgress(familyId, progress);
    
    // Update local state
    setCompleted(true);
  };
  
  if (!isVisible) {
    return null;
  }
  
  return (
    <div className="questionnaire-section">
      {React.cloneElement(children, { 
        familyId, 
        sectionId,
        onComplete: markSectionCompleted,
        isCompleted: completed
      })}
    </div>
  );
}
```

## Conclusion

The revised database design for BrightPath uses MongoDB for persistent storage while leveraging the browser's localStorage for questionnaire progress in the initial prototype. This approach offers several advantages:

1. **Simplified Initial Implementation**: Using localStorage removes the need for additional backend infrastructure during the prototype phase.

2. **Enhanced User Experience**: Progress is automatically saved locally, allowing users to continue from where they left off.

3. **Scalable Database Design**: The MongoDB schema is designed to accommodate future growth and feature enhancements.

4. **Clear Migration Path**: A straightforward upgrade path to Redis/Valkey for server-side storage in future iterations.

This hybrid approach allows for rapid development of the initial prototype while maintaining a clear path to a more robust server-side solution as the application matures.