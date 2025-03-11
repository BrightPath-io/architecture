# BrightPath: Homeschooling Scheduling Platform

BrightPath is an AI-powered scheduling platform that helps homeschooling parents create and maintain personalized educational schedules for their children. The platform adapts to individual learning styles, family constraints, and educational goals to provide intelligent scheduling recommendations.

## Project Overview

BrightPath's mission is to support and empower parents to intuitively guide their children through the homeschool journey they create. The platform uses machine learning to understand each family's unique needs and preferences, generating schedules that work for their specific situation and continuously improving through feedback.

### Key Features

- **AI-Powered Schedule Generation**: Create personalized schedules based on learning styles and preferences
- **Interactive Schedule Management**: Easily view, adjust, and track progress on daily, weekly, and monthly schedules
- **Comprehensive Onboarding**: Questionnaire-based approach to understand family needs and preferences
- **Feedback-Driven Improvement**: Continuous refinement of schedules based on user feedback
- **Resource Library**: Access to subject-specific educational resources and recommendations

## Project Structure

The BrightPath project is organized into the following main directories:

- `/api`: Django backend API
- `/client`: React frontend application
- `/db`: Database migrations and scripts
- `/infrastructure`: Infrastructure as code and deployment configurations
- `/services`: Additional services including ML models

## Documentation

Comprehensive documentation for the BrightPath project is available in the following files:

- [Project Summary](BrightPath_Project_Summary.md): High-level overview of the project
- [Architecture Plan](BrightPath_Architecture_Plan.md): System architecture and component design
- [Frontend Implementation Plan](frontend_implementation_plan.md): Detailed frontend development plan
- [Backend Implementation Plan](backend_implementation_plan.md): Detailed backend development plan
- [Database Schema](database_schema.md): Comprehensive database diagram using DBML
- [ML Implementation Plan](ml_implementation_plan.md): Machine learning implementation details
- [Revised Technology Stack](revised_technology_stack.md): Updated technology choices
- [Docker Development Environment](docker_development_environment.md): Dockerized development setup
- [Terraform Infrastructure Plan](terraform_infrastructure_plan.md): Infrastructure as code for Digital Ocean
- [Testing Strategy](testing_strategy.md): Comprehensive testing approach
- [Deployment Strategy](deployment_strategy.md): Deployment and infrastructure details

## Technology Stack

### Frontend
- React with JavaScript
- Vite for build tooling
- React Router for navigation
- Radix UI for component primitives
- CSS Modules for styling

### Backend
- Django with Python 3.12
- Django REST Framework
- PostgreSQL for database
- Google OAuth for authentication
- Valkey for message bus

### Machine Learning
- scikit-learn for initial models
- TensorFlow for advanced models
- Feedback-driven continuous improvement

### Infrastructure
- Digital Ocean for cloud hosting
- Docker for containerization
- GitHub Actions for CI/CD
- Prometheus and Grafana for monitoring

## Getting Started

### Prerequisites

- Docker and Docker Compose
- Git
- Digital Ocean CLI (for deployment)

### Dockerized Development Environment

BrightPath uses a fully Dockerized development environment to ensure consistency across all developer machines and enable quick onboarding for new team members. The Docker setup includes all necessary services:

- Frontend (React)
- Backend API (Django)
- Database (PostgreSQL)
- Message Bus (Valkey)
- Background Workers
- ML Services

For detailed information about the Docker setup, see [Docker Development Environment](docker_development_environment.md).

### Quick Start

1. **Clone the repository**

```bash
git clone https://github.com/your-organization/brightpath.git
cd brightpath
```

2. **Create .env file**

Create a `.env` file in the project root with the necessary environment variables (see below).

3. **Start the development environment**

```bash
docker-compose up
```

4. **Access the services**

- Frontend: http://localhost:3000
- Backend API: http://localhost:8000
- API Documentation: http://localhost:8000/api/docs/
- PgAdmin: http://localhost:5050 (admin@brightpath.com / admin)

### Manual Setup (Alternative)

If you prefer to run services directly on your machine without Docker:

1. **Install dependencies**
   - Python 3.12+
   - Node.js 16+
   - PostgreSQL
   - Valkey

2. **Set up the backend**

```bash
cd api
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements/development.txt
python manage.py migrate
python manage.py runserver
```

3. **Set up the frontend**

```bash
cd client
npm install
npm run dev
```

### Environment Variables

Create a `.env` file in the project root with the following variables:

```
# Django settings
DEBUG=True
SECRET_KEY=your-secret-key
ALLOWED_HOSTS=localhost,127.0.0.1

# Database settings
DB_NAME=brightpath
DB_USER=postgres
DB_PASSWORD=postgres
DB_HOST=localhost
DB_PORT=5432

# Google OAuth settings
GOOGLE_OAUTH_CLIENT_ID=your-client-id
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret

# Valkey settings
VALKEY_HOST=localhost
VALKEY_PORT=6379
VALKEY_DB=0
```

## Development Workflow

1. **Create a feature branch**

```bash
git checkout -b feature/your-feature-name
```

2. **Make your changes and commit**

```bash
git add .
git commit -m "Add your feature description"
```

3. **Run tests**

```bash
# Backend tests
cd api
pytest

# Frontend tests
cd client
npm test
```

4. **Push your changes and create a pull request**

```bash
git push origin feature/your-feature-name
```

5. **After review and approval, merge to develop branch**

## Deployment

Refer to the [Deployment Strategy](deployment_strategy.md) document for detailed deployment instructions.

Quick deployment commands:

```bash
# Deploy to Digital Ocean App Platform
doctl apps create --spec .do/app.yaml

# Update existing deployment
doctl apps update your-app-id --spec .do/app.yaml
```

## Contributing

1. Follow the development workflow described above
2. Adhere to the coding standards defined in the linting configurations
3. Write tests for new features
4. Update documentation as needed
5. Participate in code reviews

## License

This project is proprietary and confidential. Unauthorized copying, distribution, or use is strictly prohibited.

## Contact

For questions or support, please contact the project maintainers:

- Project Manager: [pm@brightpath.com](mailto:pm@brightpath.com)
- Technical Lead: [tech@brightpath.com](mailto:tech@brightpath.com)