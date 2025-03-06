# BrightPath Database Schema

This document provides a comprehensive database schema for the BrightPath homeschooling scheduling platform using Database Markup Language (DBML).

## Schema Diagram

[View the DBML](./brightpath.dbml)

## Schema Overview

This database schema represents the complete database structure for the BrightPath platform, incorporating all the details from both the architecture plan and the backend implementation plan. Here's a breakdown of the main components:

### 1. Users and Authentication
- **User table** with extended fields from Django's AbstractUser
- **UserPreference table** for storing user-specific settings

### 2. Children
- **Child table** for storing information about homeschooled children

### 3. Subjects and Resources
- **Subject table** for educational subjects (both predefined and custom)
- **Resource table** for educational materials linked to subjects

### 4. Schedules and Activities
- **Schedule table** for storing generated schedules
- **ScheduleItem table** for individual items within a schedule
- **Activity table** for tracking completed activities

### 5. Feedback System
- **Feedback table** for storing user ratings and comments on schedules

### 6. Survey System
- **Category table** for grouping questions
- **Question table** for storing survey questions
- **SurveySession table** for tracking survey progress
- **SurveyResponse table** for storing user answers

## Implementation Notes

- The database uses PostgreSQL with JSONB fields for flexible data storage (particularly for schedule configurations and feedback data)
- Timestamps are used for tracking creation and modification times
- Foreign key relationships maintain data integrity
- Indexes are defined for performance optimization
- Unique constraints prevent duplicate entries