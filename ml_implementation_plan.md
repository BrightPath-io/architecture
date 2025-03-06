# BrightPath ML Implementation Plan

## Overview

This document outlines the detailed implementation plan for the machine learning components of the BrightPath homeschooling scheduling platform. The ML system will consist of two primary models: a Schedule Generator Model that creates personalized schedules based on user preferences, and a Reward Model that evaluates schedule quality based on user feedback. These models will work together in a continuous improvement loop to enhance schedule quality over time.

## Technology Stack

- **Primary ML Framework**: scikit-learn for initial models, TensorFlow for advanced models
- **Data Processing**: pandas, NumPy
- **Model Persistence**: joblib
- **Evaluation**: scikit-learn metrics
- **Visualization**: matplotlib, seaborn
- **Experiment Tracking**: MLflow
- **Feature Store**: Optional - Redis or PostgreSQL with JSONField
- **Model Serving**: FastAPI for model serving (if needed)
- **Containerization**: Docker

## Project Structure

```
ml/
├── data/
│   ├── raw/                      # Raw data dumps
│   ├── processed/                # Processed datasets
│   ├── synthetic/                # Synthetic data for initial training
│   └── features/                 # Extracted features
├── models/
│   ├── generator/
│   │   ├── model.py              # Generator model implementation
│   │   ├── features.py           # Feature engineering for generator
│   │   ├── training.py           # Training pipeline
│   │   └── inference.py          # Inference pipeline
│   └── reward/
│       ├── model.py              # Reward model implementation
│       ├── features.py           # Feature engineering for reward model
│       ├── training.py           # Training pipeline
│       └── inference.py          # Inference pipeline
├── training/
│   ├── generator_training.py     # Training script for generator
│   ├── reward_training.py        # Training script for reward model
│   └── reinforcement.py          # Reinforcement learning integration
├── inference/
│   ├── generator_inference.py    # Inference script for generator
│   └── reward_inference.py       # Inference script for reward model
├── evaluation/
│   ├── metrics.py                # Evaluation metrics
│   ├── validation.py             # Validation utilities
│   └── visualization.py          # Result visualization
├── utils/
│   ├── data_utils.py             # Data processing utilities
│   ├── model_utils.py            # Model utilities
│   └── config.py                 # Configuration management
├── experiments/
│   ├── experiment_tracking.py    # MLflow integration
│   └── notebooks/                # Jupyter notebooks for experiments
├── pipelines/
│   ├── data_pipeline.py          # Data processing pipeline
│   ├── training_pipeline.py      # End-to-end training pipeline
│   └── inference_pipeline.py     # End-to-end inference pipeline
├── tests/
│   ├── test_generator.py         # Tests for generator model
│   ├── test_reward.py            # Tests for reward model
│   └── test_pipelines.py         # Tests for pipelines
└── requirements.txt              # ML-specific dependencies
```

## Machine Learning Models

### 1. Schedule Generator Model

The Schedule Generator Model will create personalized schedules based on user preferences collected during onboarding.

#### Model Architecture

**Initial Implementation (Phase 1):**
- **Model Type**: Random Forest Regressor or Gradient Boosting Regressor
- **Features**:
  - Flexibility vs. Structure score (derived from questionnaire)
  - Subject Rotation preference score
  - Time Management preference score
  - Long-Term Scheduling preference score
  - Prioritization preference score
  - Child age and grade level
- **Output**: Schedule parameters (structure level, block duration, breaks frequency, etc.)

**Advanced Implementation (Phase 2):**
- **Model Type**: Neural Network (MLP or Transformer-based)
- **Additional Features**:
  - Historical schedule adjustments
  - Seasonal patterns
  - Subject-specific preferences
  - Learning style indicators
- **Output**: Complete schedule configuration in JSON format

#### Feature Engineering

1. **Questionnaire Response Processing**:
   - Normalize Likert scale responses (1-5) to 0-1 range
   - Calculate composite scores for each preference category
   - Handle missing values with appropriate defaults

2. **Child Profile Features**:
   - Age-appropriate learning duration calculation
   - Grade-level subject requirements
   - Attention span estimation based on age

3. **Temporal Features**:
   - Day of week encoding
   - Time of day preferences
   - Seasonal adjustments

#### Training Approach

**Initial Training (with Synthetic Data):**
1. Generate synthetic user profiles with varying preferences
2. Create "expert-designed" schedules for each profile
3. Train the model to map from preferences to schedule parameters
4. Validate using cross-validation

**Continuous Training (with Real Data):**
1. Collect user preferences and feedback
2. Use the Reward Model to score schedules
3. Retrain the Generator Model to maximize reward scores
4. Implement A/B testing to compare model versions

### 2. Reward Model

The Reward Model will evaluate schedule quality based on user feedback and implicit signals.

#### Model Architecture

**Initial Implementation (Phase 1):**
- **Model Type**: Gradient Boosting Regressor
- **Features**:
  - Schedule parameters (structure level, block duration, etc.)
  - User feedback (star ratings, Likert responses)
  - Schedule adjustments made by users
- **Output**: Satisfaction score (0-1)

**Advanced Implementation (Phase 2):**
- **Model Type**: Neural Network with attention mechanism
- **Additional Features**:
  - Detailed schedule structure
  - User interaction patterns
  - Completion rates of scheduled activities
  - Text feedback sentiment analysis
- **Output**: Multi-dimensional quality scores (engagement, satisfaction, effectiveness)

#### Feature Engineering

1. **Schedule Feature Extraction**:
   - Structure metrics (regularity, consistency)
   - Subject distribution metrics
   - Break frequency and duration
   - Daily learning load

2. **Feedback Processing**:
   - Normalize star ratings to 0-1
   - Process Likert scale responses
   - Extract sentiment from text feedback
   - Quantify schedule adjustments

3. **Implicit Feedback Features**:
   - Activity completion rates
   - Schedule adherence metrics
   - Frequency of adjustments

#### Training Approach

1. Collect explicit feedback (ratings, Likert responses)
2. Extract implicit feedback (adjustments, completion)
3. Train the model to predict satisfaction based on schedule features and feedback
4. Validate using cross-validation and holdout testing

### 3. Reinforcement Learning Integration

In the advanced phases, we'll implement a reinforcement learning approach to continuously improve the Generator Model based on the Reward Model's evaluations.

#### RL Framework

1. **Environment**: Schedule generation context
2. **State**: User preferences and constraints
3. **Actions**: Schedule parameter adjustments
4. **Reward**: Satisfaction score from Reward Model
5. **Agent**: Generator Model

#### Implementation Approach

1. Use the Reward Model as the reward function
2. Implement a policy gradient method (e.g., PPO) for the Generator Model
3. Train the model to maximize expected reward
4. Implement exploration strategies to discover new schedule patterns

## Data Management

### Synthetic Data Generation

For initial training before sufficient real user data is available:

```python
# Example synthetic data generation
def generate_synthetic_user_profiles(n_samples=1000):
    """Generate synthetic user profiles for initial training."""
    profiles = []

    for i in range(n_samples):
        # Generate random preference scores
        flexibility_score = np.random.uniform(0, 1)
        rotation_score = np.random.uniform(0, 1)
        time_score = np.random.uniform(0, 1)
        longterm_score = np.random.uniform(0, 1)
        priority_score = np.random.uniform(0, 1)

        # Generate child age (5-18)
        child_age = np.random.randint(5, 19)

        # Create profile
        profile = {
            'flexibility_score': flexibility_score,
            'rotation_score': rotation_score,
            'time_score': time_score,
            'longterm_score': longterm_score,
            'priority_score': priority_score,
            'child_age': child_age
        }

        profiles.append(profile)

    return pd.DataFrame(profiles)

def generate_synthetic_schedules(profiles):
    """Generate synthetic 'expert-designed' schedules for profiles."""
    schedules = []

    for _, profile in profiles.iterrows():
        # Calculate schedule parameters based on profile
        structure_level = 1 - profile['flexibility_score']  # Inverse of flexibility

        # Block duration increases with age but decreases with rotation preference
        block_duration = 20 + (profile['child_age'] - 5) * 2 - profile['rotation_score'] * 15
        block_duration = max(15, min(60, block_duration))  # Constrain to 15-60 minutes

        # Breaks frequency increases with rotation preference
        breaks_frequency = 2 + int(profile['rotation_score'] * 3)

        # Start time depends on time management preference
        start_hour = 8 + int((1 - profile['time_score']) * 2)  # 8-10 AM

        # Create schedule
        schedule = {
            'structure_level': structure_level,
            'block_duration': block_duration,
            'breaks_frequency': breaks_frequency,
            'start_hour': start_hour,
            # Add more parameters as needed
        }

        schedules.append(schedule)

    return pd.DataFrame(schedules)
```

### Real Data Collection

1. **Onboarding Questionnaire**:
   - Store raw responses
   - Process into feature vectors
   - Link to user profiles

2. **Feedback Collection**:
   - Star ratings (1-5)
   - Likert scale questions
   - Free-text feedback
   - Schedule adjustments tracking

3. **Activity Tracking**:
   - Completion status
   - Duration
   - User notes

### Data Pipeline

1. **Data Extraction**:
   - Query database for new user data and feedback
   - Extract relevant features

2. **Preprocessing**:
   - Normalize numerical features
   - Encode categorical features
   - Handle missing values

3. **Feature Store**:
   - Store processed features for reuse
   - Track feature versions

4. **Training Data Preparation**:
   - Create training/validation/test splits
   - Balance datasets if needed
   - Apply data augmentation techniques

## Model Training Pipeline

### Generator Model Training

```python
# Example training pipeline for Generator Model
def train_generator_model(X_train, y_train, X_val, y_val, config):
    """Train the schedule generator model."""
    # Initialize model
    if config['model_type'] == 'random_forest':
        model = RandomForestRegressor(
            n_estimators=config['n_estimators'],
            max_depth=config['max_depth'],
            random_state=42
        )
    elif config['model_type'] == 'gradient_boosting':
        model = GradientBoostingRegressor(
            n_estimators=config['n_estimators'],
            max_depth=config['max_depth'],
            learning_rate=config['learning_rate'],
            random_state=42
        )
    else:
        raise ValueError(f"Unsupported model type: {config['model_type']}")

    # Train model
    model.fit(X_train, y_train)

    # Evaluate on validation set
    val_predictions = model.predict(X_val)
    val_mse = mean_squared_error(y_val, val_predictions)
    val_r2 = r2_score(y_val, val_predictions)

    print(f"Validation MSE: {val_mse:.4f}")
    print(f"Validation R²: {val_r2:.4f}")

    # Log metrics with MLflow
    if config['log_mlflow']:
        mlflow.log_param("model_type", config['model_type'])
        mlflow.log_param("n_estimators", config['n_estimators'])
        mlflow.log_param("max_depth", config['max_depth'])
        mlflow.log_metric("val_mse", val_mse)
        mlflow.log_metric("val_r2", val_r2)

    return model
```

### Reward Model Training

```python
# Example training pipeline for Reward Model
def train_reward_model(X_train, y_train, X_val, y_val, config):
    """Train the schedule reward model."""
    # Initialize model
    if config['model_type'] == 'gradient_boosting':
        model = GradientBoostingRegressor(
            n_estimators=config['n_estimators'],
            max_depth=config['max_depth'],
            learning_rate=config['learning_rate'],
            random_state=42
        )
    elif config['model_type'] == 'neural_network':
        model = MLPRegressor(
            hidden_layer_sizes=config['hidden_layers'],
            activation=config['activation'],
            solver=config['solver'],
            max_iter=config['max_iter'],
            random_state=42
        )
    else:
        raise ValueError(f"Unsupported model type: {config['model_type']}")

    # Train model
    model.fit(X_train, y_train)

    # Evaluate on validation set
    val_predictions = model.predict(X_val)
    val_mse = mean_squared_error(y_val, val_predictions)
    val_r2 = r2_score(y_val, val_predictions)

    print(f"Validation MSE: {val_mse:.4f}")
    print(f"Validation R²: {val_r2:.4f}")

    # Log metrics with MLflow
    if config['log_mlflow']:
        mlflow.log_param("model_type", config['model_type'])
        mlflow.log_param("n_estimators", config.get('n_estimators'))
        mlflow.log_param("max_depth", config.get('max_depth'))
        mlflow.log_metric("val_mse", val_mse)
        mlflow.log_metric("val_r2", val_r2)

    return model
```

### Reinforcement Learning Training

```python
# Example reinforcement learning integration
def train_generator_with_rl(generator_model, reward_model, user_profiles, config):
    """Train the generator model using reinforcement learning."""
    # Initialize RL environment
    env = ScheduleEnvironment(reward_model)

    # Initialize policy
    policy = SchedulePolicy(generator_model)

    # Training loop
    for episode in range(config['n_episodes']):
        # Sample a random user profile
        profile_idx = np.random.randint(0, len(user_profiles))
        profile = user_profiles[profile_idx]

        # Reset environment with this profile
        state = env.reset(profile)

        # Generate a schedule
        action = policy.select_action(state)

        # Get reward from environment
        next_state, reward, done, info = env.step(action)

        # Update policy
        policy.update(state, action, reward, next_state, done)

        # Log metrics
        if episode % config['log_interval'] == 0:
            print(f"Episode {episode}, Reward: {reward:.4f}")
            if config['log_mlflow']:
                mlflow.log_metric("episode_reward", reward, step=episode)

    # Return updated generator model
    return policy.get_model()
```

## Model Inference Pipeline

### Schedule Generation

```python
# Example inference pipeline for schedule generation
def generate_schedule(user_data, generator_model, config):
    """Generate a schedule for a user."""
    # Preprocess user data
    features = preprocess_user_data(user_data)

    # Generate schedule parameters
    params = generator_model.predict(features.reshape(1, -1))[0]

    # Convert parameters to schedule configuration
    schedule = create_schedule_from_params(params, user_data, config)

    return schedule

def create_schedule_from_params(params, user_data, config):
    """Convert model parameters to a concrete schedule."""
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
        daily_schedule = generate_daily_schedule(
            day,
            structure_level,
            block_duration,
            breaks_frequency,
            user_data
        )
        schedule['days'].append(daily_schedule)

    return schedule
```

### Schedule Evaluation

```python
# Example inference pipeline for schedule evaluation
def evaluate_schedule(schedule_data, feedback_data, reward_model):
    """Evaluate a schedule using the reward model."""
    # Preprocess schedule and feedback data
    features = preprocess_schedule_feedback(schedule_data, feedback_data)

    # Predict reward score
    reward = reward_model.predict(features.reshape(1, -1))[0]

    # Ensure reward is between 0 and 1
    reward = max(0, min(1, reward))

    return reward
```

## Model Evaluation and Monitoring

### Evaluation Metrics

1. **Generator Model**:
   - Mean Squared Error (MSE)
   - R² Score
   - Custom schedule quality metrics

2. **Reward Model**:
   - Mean Squared Error (MSE)
   - R² Score
   - Correlation with user ratings

3. **Overall System**:
   - User satisfaction ratings
   - Schedule adherence rate
   - Adjustment frequency

### A/B Testing Framework

```python
# Example A/B testing implementation
def ab_test_models(model_a, model_b, test_profiles, reward_model, config):
    """Compare two generator models using A/B testing."""
    results_a = []
    results_b = []

    for profile in test_profiles:
        # Generate schedules with both models
        schedule_a = generate_schedule(profile, model_a, config)
        schedule_b = generate_schedule(profile, model_b, config)

        # Evaluate schedules with reward model
        reward_a = evaluate_schedule(schedule_a, {}, reward_model)
        reward_b = evaluate_schedule(schedule_b, {}, reward_model)

        results_a.append(reward_a)
        results_b.append(reward_b)

    # Calculate statistics
    mean_a = np.mean(results_a)
    mean_b = np.mean(results_b)
    std_a = np.std(results_a)
    std_b = np.std(results_b)

    # Perform t-test
    t_stat, p_value = ttest_ind(results_a, results_b)

    print(f"Model A: {mean_a:.4f} ± {std_a:.4f}")
    print(f"Model B: {mean_b:.4f} ± {std_b:.4f}")
    print(f"T-statistic: {t_stat:.4f}, p-value: {p_value:.4f}")

    # Determine winner
    if p_value < 0.05:
        if mean_a > mean_b:
            winner = "Model A"
        else:
            winner = "Model B"
        print(f"Winner: {winner} (statistically significant)")
    else:
        print("No statistically significant difference")

    return {
        'model_a': {
            'mean': mean_a,
            'std': std_a,
            'results': results_a
        },
        'model_b': {
            'mean': mean_b,
            'std': std_b,
            'results': results_b
        },
        't_stat': t_stat,
        'p_value': p_value
    }
```

### Model Monitoring

1. **Performance Monitoring**:
   - Track prediction accuracy over time
   - Monitor feature distribution shifts
   - Alert on performance degradation

2. **User Feedback Monitoring**:
   - Track satisfaction trends
   - Identify problematic schedule patterns
   - Detect user segments with lower satisfaction

3. **Model Health Checks**:
   - Regular evaluation on holdout datasets
   - Periodic retraining assessment
   - Version comparison

## Integration with Backend

### API Integration

The ML models will be integrated with the Django backend through the following interfaces:

1. **Schedule Generation API**:
   - Endpoint: `/api/ml/generate-schedule/`
   - Input: User preferences, child information, constraints
   - Output: Complete schedule configuration

2. **Schedule Evaluation API**:
   - Endpoint: `/api/ml/evaluate-schedule/`
   - Input: Schedule configuration, user feedback
   - Output: Quality score and improvement suggestions

3. **Model Training API**:
   - Endpoint: `/api/ml/train-models/`
   - Input: Training parameters
   - Output: Training status and metrics

### Asynchronous Processing

For potentially long-running operations:

```python
# Example Celery task for schedule generation
@shared_task
def generate_schedule_task(user_id, child_id, preferences):
    """Generate a schedule asynchronously using the ML model."""
    try:
        # Get user and child data
        user = User.objects.get(id=user_id)
        child = Child.objects.get(id=child_id, user=user)

        # Prepare user data
        user_data = {
            'preferences': preferences,
            'child_age': child.age,
            # Add more data as needed
        }

        # Load the generator model
        generator = ScheduleGeneratorModel('ml/models/generator_model.joblib')

        # Generate schedule
        schedule_data = generator.predict(user_data)

        # Create schedule in database
        schedule = Schedule.objects.create(
            user=user,
            child=child,
            title=f"{child.first_name}'s Schedule",
            schedule_data=schedule_data,
            start_date=date.today()
        )

        # Create schedule items
        create_schedule_items(schedule, schedule_data)

        return schedule.id

    except Exception as e:
        # Log the error
        print(f"Error generating schedule: {e}")
        raise
```

## Implementation Timeline

### Phase 1: Foundation (Weeks 1-4)
- Set up ML project structure
- Implement data processing utilities
- Create synthetic data generation
- Develop initial Generator Model (rule-based)
- Implement basic Reward Model

### Phase 2: Basic ML Models (Weeks 5-8)
- Train Generator Model on synthetic data
- Implement feature engineering pipeline
- Create model persistence mechanism
- Develop evaluation metrics
- Integrate with Django API

### Phase 3: Feedback Loop (Weeks 9-12)
- Implement feedback collection
- Enhance Reward Model with real feedback
- Create continuous training pipeline
- Develop A/B testing framework
- Implement model monitoring

### Phase 4: Advanced Models (Weeks 13-16)
- Implement neural network models
- Develop reinforcement learning integration
- Enhance feature engineering
- Improve model accuracy and performance
- Implement advanced evaluation metrics

### Phase 5: Optimization and Scaling (Weeks 17-20)
- Optimize inference performance
- Implement model versioning
- Develop automated retraining
- Create comprehensive monitoring dashboard
- Conduct user testing and refinement

## Testing Strategy

### Unit Tests

```python
# Example unit tests for Generator Model
def test_generator_model_prediction():
    """Test that the generator model produces valid predictions."""
    # Create a test user profile
    user_data = {
        'q1': 4, 'q2': 2, 'q3': 5, 'q4': 1, 'q5': 3,  # Flexibility vs. Structure
        'q6': 3, 'q7': 4, 'q8': 2, 'q9': 5, 'q10': 3,  # Subject Rotation
        'q11': 5, 'q12': 2, 'q13': 3, 'q14': 4, 'q15': 1,  # Time Management
        'q16': 2, 'q17': 5, 'q18': 3, 'q19': 4, 'q20': 2,  # Long-Term Scheduling
        'q21': 4, 'q22': 3, 'q23': 5, 'q24': 2, 'q25': 4,  # Prioritization
        'child_age': 8
    }

    # Initialize model
    model = ScheduleGeneratorModel()

    # Generate schedule
    schedule = model.predict(user_data)

    # Validate schedule structure
    assert 'meta' in schedule
    assert 'days' in schedule
    assert len(schedule['days']) == 5  # 5 days of the week

    # Validate schedule parameters
    assert 0 <= schedule['meta']['structure_level'] <= 1
    assert 15 <= schedule['meta']['block_duration_minutes'] <= 60
    assert 2 <= schedule['meta']['breaks_frequency'] <= 5

    # Validate daily schedule
    day = schedule['days'][0]
    assert 'day' in day
    assert 'day_name' in day
    assert 'blocks' in day
    assert len(day['blocks']) > 0

    # Validate blocks
    block = day['blocks'][0]
    assert 'type' in block
    assert 'start_time' in block
    if block['type'] == 'subject':
        assert 'subject' in block
        assert 'duration_minutes' in block
```

### Integration Tests

```python
# Example integration test for ML API
def test_schedule_generation_api():
    """Test the schedule generation API endpoint."""
    # Create test client
    client = APIClient()

    # Create test user and authenticate
    user = User.objects.create_user(
        username='testuser',
        email='test@example.com',
        password='testpass123'
    )
    client.force_authenticate(user=user)

    # Create test child
    child = Child.objects.create(
        user=user,
        first_name='Test',
        birth_date=date(2015, 1, 1)
    )

    # Create test data
    data = {
        'child_id': child.id,
        'preferences': {
            'q1': 4, 'q2': 2, 'q3': 5, 'q4': 1, 'q5': 3,
            'q6': 3, 'q7': 4, 'q8': 2, 'q9': 5, 'q10': 3,
            'q11': 5, 'q12': 2, 'q13': 3, 'q14': 4, 'q15': 1,
            'q16': 2, 'q17': 5, 'q18': 3, 'q19': 4, 'q20': 2,
            'q21': 4, 'q22': 3, 'q23': 5, 'q24': 2, 'q25': 4
        }
    }

    # Call API
    response = client.post('/api/ml/generate-schedule/', data, format='json')

    # Check response
    assert response.status_code == 201
    assert 'id' in response.data
    assert 'schedule_data' in response.data

    # Validate schedule data
    schedule_data = response.data['schedule_data']
    assert 'meta' in schedule_data
    assert 'days' in schedule_data
```

## Deployment Strategy

### Model Serving

1. **Embedded Models**:
   - Models are loaded directly in the Django application
   - Suitable for initial deployment with moderate load

2. **Dedicated Model Service** (for scaling):
   - FastAPI service for model inference
   - Redis queue for asynchronous processing
   - Horizontal scaling for high-load scenarios

### Containerization

```dockerfile
# Dockerfile for ML service
FROM python:3.10-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy ML code
COPY ml/ ml/

# Run FastAPI server
CMD ["uvicorn", "ml.api.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

### Monitoring and Logging

1. **Model Performance Monitoring**:
   - Track prediction accuracy
   - Monitor inference time
   - Log feature distributions

2. **Error Handling**:
   - Graceful fallback to rule-based schedules
   - Comprehensive error logging
   - Automated alerts for critical issues

3. **Audit Logging**:
   - Track all model predictions
   - Log training runs and parameters
   - Record model versions and deployments

## Conclusion

This ML implementation plan provides a comprehensive roadmap for developing the machine learning components of the BrightPath platform. By following this structured approach, we can create a robust, adaptive scheduling system that continuously improves based on user feedback and delivers personalized, effective schedules for homeschooling families.