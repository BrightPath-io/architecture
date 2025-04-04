# BrightPath Essential Questionnaire

This document outlines the essential questions for the BrightPath homeschooling scheduling platform's initial prototype. These questions gather the minimum required information to generate personalized homeschool schedules.

## Questionnaire Flow Overview

The questionnaire follows these key design principles:
1. **Progressive disclosure** - Questions are organized in logical sections that build upon each other
2. **Smart defaults** - Age-appropriate suggestions are provided where possible
3. **Visual assistance** - Examples and visual aids enhance understanding
4. **Save progress** - All progress is automatically saved for later continuation

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

## Progress Saving & Navigation

- Progress is automatically saved after completing each section
- Users can navigate back to previous sections to edit responses
- A progress bar shows completion status across all sections
- Email reminders can be sent for incomplete questionnaires
- After completing all sections, users are directed to schedule generation

## Questionnaire Technical Implementation

### Progressive Disclosure Implementation
- Each section is loaded only when previous section is complete
- Child-specific sections are dynamically generated based on number of children
- Complex questions use a step-by-step approach with additional context provided as needed

### Smart Defaults Implementation
- Age-appropriate defaults are calculated server-side based on child's age and educational research
- Common patterns based on similar user profiles can be suggested (in future versions)
- Defaults are always presented as suggestions that can be easily changed

### Visual Assistance Implementation
- Tooltips provide additional context for educational terms and concepts
- Sample schedule visualizations help illustrate different flexibility levels
- Icons help distinguish between different types of activities and subjects