# BrightPath: Revised Technology Stack

Based on stakeholder feedback, this document outlines the revised technology stack for the BrightPath homeschooling scheduling platform. These revisions supersede the technology choices outlined in previous planning documents.

## Core Technology Changes

| Component | Plan  | Impact |
|-----------|--------------|--------|
| Frontend Language |  JavaScript | Simplified development without static typing |
| UI Framework |  Radix UI | More accessible, pre-built components with consistent design |
| Cloud Provider |  Digital Ocean | Simplified deployment and potentially lower costs |
| Python Version |  3.12 | Access to newer Python features and performance improvements |
| Authentication |  Google OAuth | Simplified authentication with secure third-party provider |
| Message Bus |  Valkey | Modern message bus with improved performance |

## Detailed Technology Stack

### Frontend

#### Language and Framework
- **JavaScript** with React 18+
- **Vite** for build tooling
- **React Router** for navigation
- **React Query** for data fetching and state management

#### UI Components
- **Radix UI** for accessible, unstyled components
- **Custom styling** built on top of Radix primitives
- **CSS Modules** for component-specific styling

#### Development Tools
- **ESLint** for code quality
- **Prettier** for code formatting
- **Jest** and **React Testing Library** for testing
- **Storybook** for component development and documentation

### Backend

#### Framework and Language
- **Django 4.2+** with **Python 3.12**
- **Django REST Framework** for API development
- **PostgreSQL** for database
- **Google OAuth** for authentication
- **Type annotations** for all Python code

#### Task Processing
- **Valkey** as the message bus
- **Background workers** for asynchronous processing
- **Scheduled tasks** for recurring operations

#### Development Tools
- **pipenv** for Python virtual environment management
- **pytest** for testing
- **Black** for code formatting
- **Flake8** for linting
- **mypy** for type checking
- **structlog** for structured logging

### Machine Learning

- **scikit-learn** for initial models
- **TensorFlow** for advanced models
- **pandas** and **NumPy** for data processing
- **MLflow** for experiment tracking

### Infrastructure

#### Hosting
- **Digital Ocean** Droplets and App Platform
- **Docker** for containerization
- **GitHub Actions** for CI/CD
- **Digital Ocean Spaces** for object storage
- **Terraform** for infrastructure as code

#### Monitoring and Logging
- **Prometheus** for metrics
- **Grafana** for dashboards
- **structlog** for structured logging
- **Sentry** for error tracking

## Implementation Impact

### Frontend Implementation Changes

The shift from TypeScript to JavaScript and from Tailwind to Radix UI will require the following adjustments:

1. **Component Structure**
   - Use Radix UI primitives as the foundation for UI components
   - Build custom component library on top of Radix primitives
   - Implement consistent styling across components

2. **Type Safety**
   - Use JSDoc comments for documentation and editor hints
   - Implement runtime validation with libraries like Zod or Yup
   - Add PropTypes for component props validation

3. **Styling Approach**
   - Create a design system with CSS variables for tokens
   - Use CSS Modules for component-specific styling
   - Implement responsive design with custom media queries

### Backend Implementation Changes

The adoption of Python 3.12, Google OAuth, and Valkey will require these adjustments:

1. **Authentication System**
   - Implement Google OAuth integration using `django-allauth`
   - Create user profiles linked to Google accounts
   - Handle authentication flow and token management

2. **Asynchronous Processing**
   - Set up Valkey as the message broker
   - Implement worker processes for background tasks
   - Create task queues for different types of operations

3. **Python 3.12 Features**
   - Utilize new language features for improved code quality
   - Leverage performance improvements in Python 3.12
   - Update dependencies to versions compatible with Python 3.12

### Infrastructure Changes

Moving from AWS to Digital Ocean will require these adjustments:

1. **Deployment Strategy**
   - Create Docker Compose configurations for local development
   - Set up Digital Ocean App Platform or Kubernetes for production
   - Configure CI/CD with GitHub Actions for Digital Ocean deployment

2. **Storage Strategy**
   - Use Digital Ocean Spaces for object storage
   - Implement backup strategy using Digital Ocean tools
   - Configure CDN for static assets

3. **Database Management**
   - Use Digital Ocean Managed Databases for PostgreSQL
   - Configure connection pooling and optimization
   - Implement backup and recovery procedures

## Google OAuth Implementation

### Authentication Flow

1. **User Registration/Login**
   - User clicks "Sign in with Google" button
   - User is redirected to Google consent screen
   - User authorizes the application
   - User is redirected back to BrightPath with authorization code
   - Backend exchanges code for tokens
   - User session is created

2. **Backend Configuration**
   ```python
   # settings.py
   INSTALLED_APPS = [
       # ...
       'django.contrib.sites',
       'allauth',
       'allauth.account',
       'allauth.socialaccount',
       'allauth.socialaccount.providers.google',
       # ...
   ]

   AUTHENTICATION_BACKENDS = [
       'django.contrib.auth.backends.ModelBackend',
       'allauth.account.auth_backends.AuthenticationBackend',
   ]

   SITE_ID = 1

   SOCIALACCOUNT_PROVIDERS = {
       'google': {
           'APP': {
               'client_id': 'your-client-id',
               'secret': 'your-client-secret',
               'key': ''
           },
           'SCOPE': [
               'profile',
               'email',
           ],
           'AUTH_PARAMS': {
               'access_type': 'online',
           }
       }
   }
   ```

3. **Frontend Integration**
   ```javascript
   // LoginPage.js
   import React from 'react';
   import { Button } from '@radix-ui/react-button';

   function LoginPage() {
     const handleGoogleLogin = () => {
       window.location.href = '/accounts/google/login/';
     };

     return (
       <div className="login-container">
         <h1>Welcome to BrightPath</h1>
         <Button onClick={handleGoogleLogin}>
           Sign in with Google
         </Button>
       </div>
     );
   }

   export default LoginPage;
   ```

## Radix UI Implementation

### Component Library Structure

```
client/
├── src/
│   ├── components/
│   │   ├── common/
│   │   │   ├── Button/
│   │   │   │   ├── Button.jsx
│   │   │   │   ├── Button.module.css
│   │   │   │   └── index.js
│   │   │   ├── Dialog/
│   │   │   ├── Dropdown/
│   │   │   └── ...
│   │   ├── feedback/
│   │   ├── onboarding/
│   │   └── schedule/
│   ├── styles/
│   │   ├── tokens.css
│   │   ├── global.css
│   │   └── utils.css
```

### Example Component Implementation

```javascript
// Button.jsx
import React from 'react';
import * as RadixButton from '@radix-ui/react-button';
import styles from './Button.module.css';

const Button = React.forwardRef(({
  children,
  variant = 'primary',
  size = 'medium',
  ...props
}, ref) => {
  return (
    <RadixButton.Root
      className={`${styles.button} ${styles[variant]} ${styles[size]}`}
      ref={ref}
      {...props}
    >
      {children}
    </RadixButton.Root>
  );
});

Button.displayName = 'Button';

export default Button;
```

```css
/* Button.module.css */
.button {
  font-family: var(--font-family);
  font-weight: 500;
  border-radius: var(--radius-md);
  transition: background-color 0.2s, color 0.2s, border-color 0.2s;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.primary {
  background-color: var(--color-primary);
  color: white;
  border: none;
}

.secondary {
  background-color: transparent;
  color: var(--color-primary);
  border: 1px solid var(--color-primary);
}

/* Size variants */
.small {
  padding: 6px 12px;
  font-size: 14px;
}

.medium {
  padding: 8px 16px;
  font-size: 16px;
}

.large {
  padding: 12px 24px;
  font-size: 18px;
}
```

## Valkey Message Bus Implementation

### Configuration

```python
# settings.py
VALKEY = {
    'HOST': 'localhost',
    'PORT': 6379,
    'DB': 0,
    'PASSWORD': None,
    'SSL': False,
}

# For task queues
TASK_QUEUES = {
    'default': 'brightpath:tasks:default',
    'schedules': 'brightpath:tasks:schedules',
    'ml': 'brightpath:tasks:ml',
    'notifications': 'brightpath:tasks:notifications',
}
```

### Task Processing

```python
# tasks/worker.py
import json
import time
import valkey
from django.conf import settings

# Connect to Valkey
client = valkey.Redis(
    host=settings.VALKEY['HOST'],
    port=settings.VALKEY['PORT'],
    db=settings.VALKEY['DB'],
    password=settings.VALKEY['PASSWORD'],
    ssl=settings.VALKEY['SSL'],
)

def process_queue(queue_name):
    """Process tasks from a specific queue."""
    queue_key = settings.TASK_QUEUES[queue_name]

    while True:
        # Block until a task is available
        task_data = client.blpop([queue_key], timeout=1)

        if task_data:
            _, task_json = task_data
            task = json.loads(task_json)

            try:
                # Process the task
                task_type = task.get('type')
                task_payload = task.get('payload', {})

                if task_type == 'generate_schedule':
                    process_schedule_generation(task_payload)
                elif task_type == 'train_model':
                    process_model_training(task_payload)
                # Add more task types as needed

                # Log successful processing
                print(f"Processed task: {task_type}")

            except Exception as e:
                # Handle errors
                print(f"Error processing task: {e}")

                # Optionally retry or move to dead letter queue
                client.rpush(f"{queue_key}:failed", task_json)

        # Small delay to prevent CPU spinning
        time.sleep(0.01)

def process_schedule_generation(payload):
    """Process a schedule generation task."""
    user_id = payload.get('user_id')
    child_id = payload.get('child_id')
    preferences = payload.get('preferences', {})

    # Actual implementation would go here
    print(f"Generating schedule for user {user_id}, child {child_id}")

    # Example: store the result
    result = {"status": "completed", "schedule_id": 123}
    client.set(f"task:result:{payload.get('task_id')}", json.dumps(result))

# Add more task processors as needed
```

### Task Submission

```python
# tasks/client.py
import json
import uuid
import valkey
from django.conf import settings

# Connect to Valkey
client = valkey.Redis(
    host=settings.VALKEY['HOST'],
    port=settings.VALKEY['PORT'],
    db=settings.VALKEY['DB'],
    password=settings.VALKEY['PASSWORD'],
    ssl=settings.VALKEY['SSL'],
)

def submit_task(queue_name, task_type, payload, priority=0):
    """Submit a task to a queue."""
    queue_key = settings.TASK_QUEUES[queue_name]
    task_id = str(uuid.uuid4())

    task = {
        'id': task_id,
        'type': task_type,
        'payload': payload,
        'created_at': time.time(),
    }

    task_json = json.dumps(task)

    if priority > 0:
        # High priority tasks go to the front of the queue
        client.lpush(queue_key, task_json)
    else:
        # Normal priority tasks go to the back
        client.rpush(queue_key, task_json)

    return task_id

def get_task_result(task_id, timeout=None):
    """Get the result of a task."""
    result_key = f"task:result:{task_id}"

    if timeout is None:
        # Non-blocking check
        result_json = client.get(result_key)
        if result_json:
            return json.loads(result_json)
        return None

    # Blocking wait with timeout
    end_time = time.time() + timeout
    while time.time() < end_time:
        result_json = client.get(result_key)
        if result_json:
            return json.loads(result_json)
        time.sleep(0.1)

    return None
```

## Digital Ocean Deployment

### App Platform Configuration

```yaml
# .do/app.yaml
name: brightpath
region: nyc
services:
  - name: web
    github:
      repo: your-org/brightpath
      branch: main
      deploy_on_push: true
    build_command: cd api && pip install -r requirements.txt && python manage.py collectstatic --noinput
    run_command: cd api && gunicorn brightpath.wsgi:application --bind 0.0.0.0:$PORT
    envs:
      - key: DJANGO_SETTINGS_MODULE
        value: brightpath.settings.production
      - key: SECRET_KEY
        value: ${SECRET_KEY}
      - key: DATABASE_URL
        value: ${DATABASE_URL}
      - key: VALKEY_URL
        value: ${VALKEY_URL}
    http_port: 8080
    instance_count: 2
    instance_size_slug: basic-xs

  - name: frontend
    github:
      repo: your-org/brightpath
      branch: main
      deploy_on_push: true
    build_command: cd client && npm install && npm run build
    output_dir: client/dist
    envs:
      - key: API_URL
        value: ${API_URL}
    routes:
      - path: /

  - name: worker
    github:
      repo: your-org/brightpath
      branch: main
      deploy_on_push: true
    build_command: cd api && pip install -r requirements.txt
    run_command: cd api && python -m tasks.worker
    envs:
      - key: DJANGO_SETTINGS_MODULE
        value: brightpath.settings.production
      - key: DATABASE_URL
        value: ${DATABASE_URL}
      - key: VALKEY_URL
        value: ${VALKEY_URL}
    instance_count: 1
    instance_size_slug: basic-xs

databases:
  - name: db
    engine: PG
    version: "14"
    size: db-s-dev-database

  - name: valkey
    engine: VALKEY
    version: "7"
    size: db-s-dev-database
```

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Digital Ocean

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Deploy to Digital Ocean App Platform
        run: doctl apps create-deployment ${{ secrets.APP_ID }}
```

## Conclusion

These revised technology choices maintain the core functionality and architecture of BrightPath while aligning with stakeholder preferences. The changes will result in:

1. **Simplified Development**: Using JavaScript instead of TypeScript reduces complexity while maintaining flexibility.

2. **Enhanced UI Components**: Radix UI provides accessible, composable primitives that can be styled to match the design system.

3. **Streamlined Authentication**: Google OAuth simplifies the authentication flow and provides a secure, familiar experience for users.

4. **Modern Infrastructure**: Digital Ocean offers a more straightforward deployment experience with potentially lower costs.

5. **Improved Performance**: Python 3.12 and Valkey provide performance improvements for backend processing.

These technology choices will be reflected in the implementation of BrightPath, and the development team should refer to this document when making technical decisions.