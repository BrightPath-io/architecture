# BrightPath Essential Questionnaire

This document outlines the essential questions for the BrightPath homeschooling scheduling platform's initial prototype, including the updated implementation approach using browser local storage for progress tracking with PostgreSQL and JSONB fields for final data storage.

## Questionnaire Flow Overview

The questionnaire follows these key design principles:
1. **Progressive disclosure** - Questions are organized in logical sections that build upon each other
2. **Smart defaults** - Age-appropriate suggestions are provided where possible
3. **Visual assistance** - Examples and visual aids enhance understanding
4. **Save progress** - Progress is automatically saved to browser local storage for later continuation

### Multi-Child Flow
For families homeschooling multiple children:
1. Parent completes family-wide information (Sections 1, 5, part of 3)
2. Parent adds each child (Section 2)
3. For each child, parent completes child-specific sections (Parts of Section 3, Section 4, Section 6)
4. Parent completes family teaching preferences (Section 7)

## Section 1: Parent/Guardian Information (Family-wide)

1. **Name**
   - Text field
   - Required
   - Note: May be pre-filled from Google profile but editable

## Section 2: Child Information (Repeated for each child)

*Note: After completing information for one child, parent will have the option to "Add Another Child" or continue with the questionnaire for the current child.*

1. **Child's Name**
   - Text field
   - Required

2. **Child's Age**
   - Number field (2-18)
   - Required
   - *Smart default: Affects suggested subjects and time durations in later sections*

## Section 3: Commitments & Obligations

### Family-wide Commitments
1. **Do you have any recurring family or parent commitments that need to be considered in your schedule?**
   - Yes/No selection
   - Required
   - If Yes, show sub-questions:

2. **Please add your family/parent commitments:**
   - Repeatable form section with:
     - Commitment Name (Text field)
     - Frequency (Dropdown: Daily, Weekly, Monthly, One-time)
     - Days of Week (Multi-select checkbox: Monday-Sunday) [Show if Weekly selected]
     - Start Time (Time picker)
     - End Time (Time picker)
     - Add Another Commitment (Button)

### Child-specific Commitments (Repeated for each child)
1. **Does [Child's Name] have any recurring activities or commitments?**
   - Yes/No selection
   - Required
   - If Yes, show sub-questions:

2. **Please add [Child's Name]'s commitments:**
   - Repeatable form section with:
     - Commitment Name (Text field)
     - Frequency (Dropdown: Daily, Weekly, Monthly, One-time)
     - Days of Week (Multi-select checkbox: Monday-Sunday) [Show if Weekly selected]
     - Start Time (Time picker)
     - End Time (Time picker)
     - Add Another Commitment (Button)

## Section 4: Subjects & Activities (Repeated for each child)

*Note: This entire section repeats for each child added in Section 2.*

1. **What subjects will [Child's Name] be learning?**
   - Multi-select with common options plus "Other":
     - Reading/Language Arts
     - Mathematics
     - Science
     - Social Studies/History
     - Art
     - Music
     - Physical Education
     - Foreign Language
     - Other (with text field)
   - Required
   - *Smart default: Age-appropriate subject suggestions are pre-selected based on child's age*

2. **For each selected subject, please provide details:**
   - Dynamic form that repeats for each selected subject with:
     - Is this a core focus subject? (Yes/No)
     - How much time should be spent on this subject per session? (Dropdown: 15 min, 30 min, 45 min, 60 min, 90+ min)
       - *Smart default: Age-appropriate time suggestions*
     - How often should this subject be taught? (Dropdown: Daily, 2-3 times per week, Once a week, Occasionally)
     - Does this subject require parent involvement? (Dropdown: Full involvement, Some assistance, Minimal supervision, Independent)
       - *Smart default: Based on subject and age*
     - Does this subject need to be done at a specific time? (Yes/No)
       - If Yes: Preferred time (Time picker)
     - [Child's Name]'s interest level in this subject (5-point scale: Very Low to Very High)

## Section 5: Schedule Flexibility (Family-wide)

1. **How flexible do you want your homeschool schedule to be?**
   - Single selection (Required):
     - Very flexible, use schedule as rough guide
     - Generally follow schedule but easily adjusted
     - Moderate structure with some flexibility
     - Mostly structured with occasional adjustments
     - Strictly structured, prefer consistent routine
   - *Visual assistance: Brief examples of each flexibility level shown*

## Section 6: Preferred Learning Times (Repeated for each child)

*Note: This section repeats for each child added in Section 2.*

1. **When does [Child's Name] learn best?** (Multi-select)
   - Early morning (6am-9am)
   - Mid-morning (9am-12pm)
   - Early afternoon (12pm-3pm)
   - Late afternoon (3pm-6pm)
   - Evening (After 6pm)
   - Required

2. **Please specify your preferred homeschooling hours for [Child's Name]:**
   - Start Time (Time picker)
   - End Time (Time picker)
   - Required

## Section 7: Learning & Teaching Preferences (Family-wide)

1. **How much do you agree with the following statements?** (Rate each 1-5, from Strongly Disagree to Strongly Agree)
   
   **For Homeschooling Philosophy determination:**
   - "I prefer using structured curriculum materials similar to traditional schools"
   - "I believe children learn best through real books rather than textbooks"
   - "Hands-on learning and self-discovery are more important than structured lessons"
   - "I value teaching classical subjects like logic and rhetoric"
   - "I believe children should follow their interests with minimal formal instruction"
   
   **For Scheduling Style determination:**
   - "I prefer focusing deeply on fewer subjects each day rather than covering all subjects daily"
   - "I'd rather rotate through subjects over a period of days rather than having a fixed daily schedule"
   - "Having specific time slots for each subject helps keep our day organized"
   - "I prefer to let my child's interests determine what we study each day"
   - "I find it helpful to group several shorter activities together at the start of our day"

2. **When planning your children's learning, which approach do you prefer?** (Select one)
   - Following a set curriculum closely
   - Using curriculum as a guide but adapting as needed
   - Creating your own lesson plans from various resources
   - Following your child's interests and providing resources accordingly
   - A mix of planned lessons and spontaneous learning opportunities

3. **How do you feel about the following learning activities?** (Rate each 1-5, from Strongly Dislike to Strongly Like)
   - Textbook reading and worksheets
   - Nature exploration and observation
   - Arts and crafts projects
   - Memorization and recitation
   - Independent research projects
   - Hands-on experiments
   - Group discussions about ideas

## Local Storage Implementation for Questionnaire Progress

### Progress Saving & Navigation

- Progress is automatically saved to browser local storage after completing each section
- Users can navigate back to previous sections to edit responses
- A progress bar shows completion status across all sections
- Email reminders can be sent with a direct link to resume the questionnaire
- After completing all sections, users are directed to schedule generation
- Clear notifications inform users that progress is saved locally to their device

### Local Storage Schema

The following key patterns will be used for storing data in localStorage:

```javascript
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

### Key Local Storage Implementation Features

1. **Automatic Saving**
   - Save after each field change or at regular intervals
   - Throttle saves to prevent excessive writes
   - Silent background saving with visual indicator

2. **Resume Functionality**
   - Check for saved data on page load
   - Offer to resume from last position
   - Clearly indicate unsaved changes

3. **Multi-Child Support**
   - Store child-specific sections with child identifiers
   - Track progress separately for each child
   - Allow navigation between children

4. **Completion & Submission**
   - Final submission sends all data to PostgreSQL backend
   - Store completion status and full questionnaire responses in PostgreSQL JSONB fields
   - Clear localStorage after successful submission

### Implementation Code (React)

Here's a simplified implementation of the local storage service for the questionnaire:

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

### React Hook for Local Storage Integration

To simplify working with localStorage in React components:

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

### PostgreSQL Models for Questionnaire Data

When the questionnaire is completed, data is transferred from localStorage to PostgreSQL:

```python
# Django models for questionnaire data storage

from django.db import models
from django.contrib.postgres.fields import JSONField

class Family(models.Model):
    user = models.ForeignKey('User', on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    scheduling_flexibility = models.CharField(max_length=50, null=True, blank=True)
    planning_approach = models.CharField(max_length=50, null=True, blank=True)
    
    # JSONB fields for flexible data
    preferences = JSONField(default=dict)
    questionnaire_responses = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.name

class Child(models.Model):
    family = models.ForeignKey(Family, on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    age = models.IntegerField()
    
    # JSONB fields for flexible data
    learning_preferences = JSONField(default=dict)
    questionnaire_responses = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return f"{self.name} ({self.family.name})"

class Subject(models.Model):
    child = models.ForeignKey(Child, on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    is_core_subject = models.BooleanField(default=False)
    session_duration = models.IntegerField()  # in minutes
    frequency = models.CharField(max_length=50)
    parent_involvement = models.CharField(max_length=50)
    fixed_time_required = models.BooleanField(default=False)
    preferred_time = models.TimeField(null=True, blank=True)
    interest_level = models.IntegerField()
    
    # JSONB field for additional metadata
    metadata = JSONField(default=dict)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return f"{self.name} ({self.child.name})"
```

### Handling Storage Limitations

1. **Space Constraints**
   - localStorage is limited to 5-10MB per domain
   - Implement data pruning for unused draft responses
   - Monitor storage usage with QuestionnaireStorageService.checkStorageSpace()
   - Alert users if storage limit is approaching

2. **Data Loss Prevention**
   - Offer to email a resumption link
   - Warn users not to clear browser data during questionnaire completion
   - Consider adding reminder messages about device-specific storage

3. **Final Submission**
   - Send data to backend for permanent storage in PostgreSQL with JSONB fields
   - Provide clear confirmation of successful submission
   - Clean up localStorage after successful submission

## Questionnaire Technical Implementation

### Progressive Disclosure Implementation
- Each section is loaded only when previous section is complete
- Section components are dynamically mounted/unmounted based on progress
- Child-specific sections are dynamically generated based on number of children
- Complex questions use a step-by-step approach with additional context provided as needed
- Implementation includes:
  ```javascript
  // Example progressive disclosure component structure
  function QuestionnaireFlow({ familyId }) {
    const { data: progress } = useQuestionnaireStorage(
      familyId,
      'progress'
    );
    
    return (
      <div className="questionnaire-flow">
        {/* Section 1 is always shown first */}
        {(!progress || progress.currentSection === 1) && (
          <Section1 
            onComplete={() => handleSectionComplete(1, 2)} 
            familyId={familyId} 
          />
        )}
        
        {/* Section 2 is shown after Section 1 is complete */}
        {progress && progress.currentSection === 2 && (
          <Section2 
            onComplete={() => handleSectionComplete(2, 3)} 
            onBack={() => handleSectionBack(2, 1)}
            familyId={familyId} 
          />
        )}
        
        {/* Dynamic rendering of child-specific sections */}
        {progress && progress.children && progress.children.map(child => (
          <ChildSpecificSections 
            key={child.id}
            childId={child.id}
            currentSection={progress.currentSection}
            familyId={familyId}
          />
        ))}
        
        {/* More sections... */}
      </div>
    );
  }
  ```

### Smart Defaults Implementation
- Age-appropriate defaults are calculated based on child's age and educational research
- Common patterns based on similar user profiles are suggested
- Defaults are always presented as suggestions that can be easily changed
- Implementation includes:
  ```javascript
  // Example smart defaults utility for subject recommendations
  const getSubjectRecommendations = (childAge) => {
    // Base recommendations for all ages
    const recommendations = [
      { name: "Reading/Language Arts", isCoreSubject: true },
      { name: "Mathematics", isCoreSubject: true },
    ];
    
    // Age-specific additions
    if (childAge >= 6) {
      recommendations.push({ 
        name: "Science", 
        isCoreSubject: true,
        sessionDuration: childAge < 10 ? 30 : 45, // Younger kids get shorter sessions
      });
      
      recommendations.push({ 
        name: "Social Studies/History", 
        isCoreSubject: true,
        sessionDuration: childAge < 10 ? 30 : 45,
      });
    }
    
    if (childAge >= 8) {
      recommendations.push({ 
        name: "Foreign Language", 
        isCoreSubject: false,
        sessionDuration: 30,
        frequency: "2-3_times_per_week"
      });
    }
    
    // Add enrichment subjects
    recommendations.push(
      { name: "Art", isCoreSubject: false, sessionDuration: 30, frequency: "once_a_week" },
      { name: "Music", isCoreSubject: false, sessionDuration: 30, frequency: "once_a_week" },
      { name: "Physical Education", isCoreSubject: false, sessionDuration: 45, frequency: "2-3_times_per_week" }
    );
    
    return recommendations;
  };
  
  // Usage in a component
  function SubjectsSelection({ childAge, familyId, childId }) {
    const [subjects, setSubjects] = useState([]);
    
    useEffect(() => {
      // First check if we have saved data
      const savedData = QuestionnaireStorageService.getChildSectionDraft(
        familyId, 
        childId, 
        'section4'
      );
      
      if (savedData && savedData.subjects) {
        setSubjects(savedData.subjects);
      } else {
        // Apply smart defaults based on child's age
        setSubjects(getSubjectRecommendations(childAge));
      }
    }, [childAge, familyId, childId]);
    
    // Component rendering...
  }
  ```

### Visual Assistance Implementation
- Tooltips provide additional context for educational terms and concepts
- Sample schedule visualizations help illustrate different flexibility levels
- Icons help distinguish between different types of activities and subjects
- Visual indicators show progress through the questionnaire flow
- Implementation includes:
  ```javascript
  // Example tooltip component
  function EducationalTooltip({ term, children }) {
    const tooltipContent = {
      'charlotte_mason': 'An educational philosophy focusing on living books, nature study, and narration.',
      'montessori': 'A child-centered approach emphasizing independence, freedom within limits, and respect for natural development.',
      'classical': 'A language-focused curriculum divided into three stages: grammar, logic, and rhetoric.',
      'unschooling': 'An approach where children learn through their natural life experiences, including play, household responsibilities, and social interaction.',
      // More definitions...
    };
    
    return (
      <Tooltip content={tooltipContent[term] || 'No definition available'}>
        {children}
        <InfoIcon className="ml-1 text-gray-400" size={16} />
      </Tooltip>
    );
  }
  
  // Example schedule visualization component
  function ScheduleFlexibilityVisualizer({ flexibilityLevel }) {
    // Visual representations of different schedule types
    const scheduleVisuals = {
      'very_flexible': <FlexibleScheduleVisualization />,
      'somewhat_flexible': <SomewhatFlexibleVisualization />,
      'moderate_structure': <ModerateStructureVisualization />,
      'mostly_structured': <MostlyStructuredVisualization />,
      'strictly_structured': <StrictlyStructuredVisualization />
    };
    
    return (
      <div className="schedule-visualizer">
        <h4>Example Schedule</h4>
        {scheduleVisuals[flexibilityLevel] || scheduleVisuals['moderate_structure']}
      </div>
    );
  }
  ```

## Questionnaire Data Submission and Storage

When the user completes the questionnaire, the data is submitted to the backend and stored in PostgreSQL using a combination of relational tables and JSONB fields:

```python
@transaction.atomic
def save_questionnaire_data(user_id, data):
    """Process and save questionnaire data to PostgreSQL"""
    try:
        # Get user
        user = User.objects.get(id=user_id)
        
        # Create or update family
        family, created = Family.objects.get_or_create(user=user)
        family.name = data['section1']['name']
        family.scheduling_flexibility = data['section5']['flexibility']
        family.planning_approach = data['section7']['planningApproach']
        
        # Store complete questionnaire responses in JSONB
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
        
        # Process children data
        for child_data in data['children']:
            # Create or update child
            if child_data.get('id') and not child_data['id'].startswith('temp_'):
                child = Child.objects.get(id=child_data['id'], family=family)
            else:
                child = Child(family=family)
            
            child.name = child_data['section2']['name']
            child.age = child_data['section2']['age']
            
            # Store child questionnaire data in JSONB
            child.questionnaire_responses = {
                'section2': child_data['section2'],
                'section4': child_data['section4'],
                'section6': child_data['section6']
            }
            
            # Process learning preferences into structured JSONB
            child.learning_preferences = {
                'bestLearningTimes': child_data['section6']['bestLearningTimes'],
                'homeschoolingHours': {
                    'startTime': child_data['section6']['startTime'],
                    'endTime': child_data['section6']['endTime']
                }
            }
            
            child.save()
            
            # Save subjects to structured tables
            if 'subjects' in child_data.get('section4', {}):
                # First delete existing subjects
                Subject.objects.filter(child=child).delete()
                
                # Create new subjects
                for subject_data in child_data['section4']['subjects']:
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
                    
                    # Store additional data in JSONB field
                    metadata = {k: v for k, v in subject_data.items() 
                                if k not in ['name', 'isCoreSubject', 'sessionDuration', 
                                           'frequency', 'parentInvolvement', 'fixedTime', 
                                           'interestLevel']}
                    if metadata:
                        subject.metadata = metadata
                    
                    subject.save()
            
            # Save commitments to structured tables
            if 'commitments' in child_data.get('section3Child', {}):
                # First delete existing commitments
                ChildCommitment.objects.filter(child=child).delete()
                
                # Create new commitments
                for commitment in child_data['section3Child']['commitments']:
                    ChildCommitment.objects.create(
                        child=child,
                        name=commitment['name'],
                        frequency=commitment['frequency'],
                        days_of_week=json.dumps(commitment.get('daysOfWeek', [])),
                        start_time=commitment['startTime'],
                        end_time=commitment['endTime']
                    )
        
        # Save family commitments to structured tables
        if 'commitments' in data.get('section3', {}):
            # First delete existing commitments
            FamilyCommitment.objects.filter(family=family).delete()
            
            # Create new commitments
            for commitment in data['section3']['commitments']:
                FamilyCommitment.objects.create(
                    family=family,
                    name=commitment['name'],
                    frequency=commitment['frequency'],
                    days_of_week=json.dumps(commitment.get('daysOfWeek', [])),
                    start_time=commitment['startTime'],
                    end_time=commitment['endTime']
                )
        
        # Mark questionnaire as completed
        family.questionnaire_completed = True
        family.questionnaire_completed_at = timezone.now()
        family.save()
        
        return True, family.id
    
    except Exception as e:
        # Log the error and return
        logger.error(f"Error saving questionnaire data: {str(e)}")
        return False, str(e)
```

## Future Enhancements for Questionnaire Storage

In future iterations, the questionnaire progress tracking can be enhanced by:

1. **Server-Side Storage**
   - Create a `questionnaire_sessions` table in PostgreSQL
   - Enable cross-device continuation of the questionnaire
   - Implement real-time progress tracking and analytics

2. **Multi-Device Support**
   - QR code to transfer progress to mobile device
   - Email links for resuming on different devices
   - Account synchronization across multiple browsers/devices

3. **Enhanced Security**
   - Encrypted storage for sensitive information
   - Automatic session timeouts with data preservation
   - Protection against data tampering

4. **Advanced Analytics**
   - Track completion rates and abandonment points
   - Identify problematic questionnaire sections
   - Perform A/B testing on question wording and flow

5. **API Endpoints for Session Management**

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

6. **PostgreSQL Session Storage Model**

```python
class QuestionnaireSession(models.Model):
    user = models.ForeignKey('User', on_delete=models.CASCADE)
    family = models.ForeignKey('Family', on_delete=models.CASCADE, null=True, blank=True)
    session_data = JSONField()
    last_updated = models.DateTimeField(auto_now=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['user']),
            models.Index(fields=['last_updated'])
        ]
```

## Conclusion

The BrightPath questionnaire implementation leverages browser localStorage for the initial prototype to provide a seamless user experience with save-and-continue functionality. When the questionnaire is completed, the data is stored in PostgreSQL using a combination of structured tables for core data and JSONB fields for flexible, hierarchical data.

This hybrid approach provides:
1. Simple, client-side progress tracking during completion
2. Strong data integrity for core entities in the database
3. Flexible storage for complex, hierarchical questionnaire responses
4. A clear migration path to server-side session management in future iterations

By focusing on progressive disclosure, smart defaults, and visual assistance, the questionnaire guides users through the complex process of setting up homeschooling schedules while collecting the necessary data to generate personalized, effective schedules tailored to each family's unique situation.