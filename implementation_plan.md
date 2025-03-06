# BrightPath Implementation Plan Changes

## Overview

This document outlines the changes needed to implement the following adjustments to the BrightPath project:

1. Remove Kubernetes references
2. Use simpler Digital Ocean resources (Droplets instead of Kubernetes)
3. Add Python type annotations in code examples
4. Add structlog for logging

## 1. Architecture Changes

### 1.1 Remove Kubernetes References

In `BrightPath_Architecture_Plan.md`:
- Replace "Orchestration: Kubernetes for container management" with "Orchestration: Docker Compose for container management"
- Update deployment strategy to focus on Droplets instead of Kubernetes

### 1.2 Infrastructure Changes

In `terraform_infrastructure_plan.md`:
- Remove the Kubernetes module
- Create a new Droplets module
- Update database firewall rules to reference Droplets instead of Kubernetes
- Update environment configurations to use Droplets

## 2. Python Type Annotations

Add type annotations to all Python code examples in `backend_implementation_plan.md`:
- Add return type annotations to functions
- Add parameter type annotations
- Use appropriate typing imports (List, Dict, Optional, etc.)
- Ensure compatibility with Python 3.12

## 3. Structlog Integration

Add structlog configuration and usage examples:
- Add structlog configuration to settings
- Replace print statements with structured logging
- Add context to log messages
- Implement proper log formatting

## 4. Implementation Details

### 4.1 Droplet Configuration

Create a new Droplets module in Terraform:
```hcl
# modules/droplets/main.tf
resource "digitalocean_droplet" "web" {
  name     = "${var.environment}-web-${count.index + 1}"
  size     = var.droplet_size
  image    = var.droplet_image
  region   = var.region
  vpc_uuid = var.vpc_id
  ssh_keys = var.ssh_key_ids
  count    = var.web_droplet_count
  tags     = ["${var.environment}", "web", "brightpath"]
}

resource "digitalocean_droplet" "worker" {
  name     = "${var.environment}-worker-${count.index + 1}"
  size     = var.droplet_size
  image    = var.droplet_image
  region   = var.region
  vpc_uuid = var.vpc_id
  ssh_keys = var.ssh_key_ids
  count    = var.worker_droplet_count
  tags     = ["${var.environment}", "worker", "brightpath"]
}

resource "digitalocean_loadbalancer" "web" {
  name   = "${var.environment}-loadbalancer"
  region = var.region
  vpc_uuid = var.vpc_id

  forwarding_rule {
    entry_port     = 80
    entry_protocol = "http"
    target_port     = 8000
    target_protocol = "http"
  }

  forwarding_rule {
    entry_port     = 443
    entry_protocol = "https"
    target_port     = 8000
    target_protocol = "http"
    certificate_id = var.certificate_id
  }

  healthcheck {
    port     = 8000
    protocol = "http"
    path     = "/health/"
  }

  droplet_ids = digitalocean_droplet.web[*].id
}
```

### 4.2 Python Type Annotations Example

Update Python code with type annotations:

```python
from typing import Dict, List, Optional, Any, Union, Tuple
import datetime

class ScheduleGeneratorModel:
    """Model for generating personalized schedules based on user preferences."""

    def __init__(self, model_path: Optional[str] = None) -> None:
        """Initialize the model, optionally loading from a file."""
        if model_path:
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestRegressor(
                n_estimators=100,
                max_depth=10,
                random_state=42
            )

    def preprocess_features(self, user_data: Dict[str, Any]) -> np.ndarray:
        """Convert user data to model features."""
        # Extract features from user questionnaire responses
        features: List[float] = []

        # Flexibility vs. Structure (questions 1-5)
        flexibility_score = np.mean([
            user_data.get('q1', 3),
            6 - user_data.get('q2', 3),  # Reverse scale
            user_data.get('q3', 3),
            6 - user_data.get('q4', 3),  # Reverse scale
            user_data.get('q5', 3)
        ])
        features.append(flexibility_score)

        # Additional features...

        return np.array(features).reshape(1, -1)
```

### 4.3 Structlog Configuration

Add structlog configuration:

```python
# settings/base.py
import structlog

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json_formatter": {
            "()": structlog.stdlib.ProcessorFormatter,
            "processor": structlog.processors.JSONRenderer(),
        },
        "plain_console": {
            "()": structlog.stdlib.ProcessorFormatter,
            "processor": structlog.dev.ConsoleRenderer(),
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "plain_console",
        },
        "json_file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "logs/brightpath.json",
            "formatter": "json_formatter",
            "maxBytes": 10485760,  # 10MB
            "backupCount": 5,
        },
    },
    "loggers": {
        "django": {
            "handlers": ["console", "json_file"],
            "level": "INFO",
        },
        "brightpath": {
            "handlers": ["console", "json_file"],
            "level": "INFO",
            "propagate": False,
        },
    }
}

structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
    ],
    context_class=structlog.threadlocal.wrap_dict(dict),
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)
```

Example usage in code:

```python
import structlog

logger = structlog.get_logger("brightpath.schedules")

def generate_schedule(user_id: int, child_id: int, title: str, description: str,
                     start_date: datetime.date, end_date: Optional[datetime.date] = None,
                     subjects: Optional[List[str]] = None) -> int:
    """
    Generate a schedule using the ML model.
    This task is designed to run asynchronously for potentially long-running operations.
    """
    logger = logger.bind(user_id=user_id, child_id=child_id)
    logger.info("Starting schedule generation")

    try:
        # Get user and child
        user = User.objects.get(id=user_id)
        child = Child.objects.get(id=child_id, user=user)

        logger.debug("Retrieved user and child data",
                    child_age=child.age,
                    is_onboarded=user.is_onboarded)

        # Additional implementation...

        logger.info("Schedule generated successfully", schedule_id=schedule.id)
        return schedule.id

    except Exception as e:
        logger.exception("Error generating schedule", error=str(e))
        raise
```

## 5. Next Steps

1. Update `BrightPath_Architecture_Plan.md`
2. Update `terraform_infrastructure_plan.md`
3. Update `backend_implementation_plan.md`
4. Create a summary of changes for stakeholder review