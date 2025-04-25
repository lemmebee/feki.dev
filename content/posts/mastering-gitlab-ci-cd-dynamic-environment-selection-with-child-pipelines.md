+++
date = '2025-04-25T20:30:56+02:00'
draft = false
title = 'Mastering GitLab CI/CD: Dynamic Environment Selection with Child Pipelines'
+++
## Table of Contents
- [The Problem](#the-problem)
- [The Key Challenge](#the-key-challenge)
- [Failed Approaches](#failed-approaches)
- [The Solution: Child Pipelines](#the-solution-child-pipelines)
- [How the Solution Works](#how-the-solution-works)
- [Key Advantages](#key-advantages)
- [Gotchas and Lessons Learned](#gotchas-and-lessons-learned)
- [Conclusion](#conclusion)

In modern software development, CI/CD pipelines are crucial for automating testing and deployment processes. However, when your testing strategy involves multiple environments and complex scheduling, pipeline configuration can become challenging. In this article, I'll share how I solved a particularly tricky GitLab CI/CD problem involving environment-specific variables and dynamic environment selection.

## The Problem

I needed to implement a scheduled testing strategy with the following requirements:

1. Tests should run on a 2-week cycle across different environments:
   - Week 1: Wednesday to Friday - Run on develop environment
   - Week 2: Monday to Wednesday - Run on develop environment
   - Week 2: Thursday - Run on staging environment
   - Other days: No tests

2. Each environment (develop, staging) had its own set of credentials and configuration values stored as GitLab CI/CD variables with environment scope.

3. The tests needed access to these environment-specific variables to run properly.

The initial implementation looked like this:

```yaml
tests-scheduled:
  extends: .tests
  environment:
    name: placeholder
  rules:
    - if: '($CI_PIPELINE_SOURCE == "schedule" && $SCHEDULER_JOB == "testing")'
  script:
    - | #shell
      # Get current day of week (1-7, Monday is 1)
      DAY_OF_WEEK=$(date +%u)
      
      # Determine which week in the 2-week cycle
      WEEK_NUMBER=$(date +%V)
      CYCLE_WEEK=$((WEEK_NUMBER % 2))
      
      # Apply 2-week cycle logic
      if [[ $CYCLE_WEEK == 0 && $DAY_OF_WEEK -ge 3 && $DAY_OF_WEEK -le 5 ]]; then
        # Week 1: Wednesday-Friday -> develop
        export TARGET_ENV="develop"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK == 4 ]]; then
        # Week 2: Thursday -> staging
        export TARGET_ENV="staging"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK -ge 1 && $DAY_OF_WEEK -le 3 ]]; then
        # Week 2: Monday-Wednesday -> develop
        export TARGET_ENV="develop"
      else
        echo "Today doesn't match any scheduled testing pattern. Skipping tests."
        exit 0
      fi
      
      echo "Running tests for environment: $TARGET_ENV"
      npm run test:remote
```

## The Key Challenge

When running this job, I encountered a fundamental problem: the environment-specific variables (like credentials) weren't accessible to the job. Why? Because GitLab CI/CD only makes environment-scoped variables available when the job's environment matches the variable's scope.

My job had `environment: name: placeholder`, but the variables were scoped to "develop" and "staging" environments. Even though I was calculating `TARGET_ENV` correctly, GitLab wasn't injecting the right variables into the execution context.

## Failed Approaches

I tried several approaches that didn't work:

1. **Dynamic environment setting**: Attempting to change the environment mid-job using `CI_ENVIRONMENT_NAME` variable didn't work properly.

2. **Creating a .env file**: Checking for a non-existent `.env` file that would magically contain our environment variables wasn't successful.

3. **Hardcoding values**: This would be a security risk and maintenance nightmare.

## The Solution: Child Pipelines

The solution I implemented leverages GitLab's child pipeline feature to dynamically create a pipeline with the correct environment context. Here's the two-job approach I developed:

```yaml
# First job: determines environment and creates child pipeline YAML
determine-environment:
  stage: prepare
  rules:
    - if: '($CI_PIPELINE_SOURCE == "schedule" && $SCHEDULER_JOB == "testing")'
  artifacts:
    paths:
      - child-pipeline.yml
  script:
    - | #shell
      # Get current day of week (1-7, Monday is 1)
      DAY_OF_WEEK=$(date +%u)
      
      # Determine which week in the 2-week cycle
      WEEK_NUMBER=$(date +%V)
      CYCLE_WEEK=$((WEEK_NUMBER % 2))
      
      # Apply 2-week cycle logic
      if [[ $CYCLE_WEEK == 0 && $DAY_OF_WEEK -ge 3 && $DAY_OF_WEEK -le 5 ]]; then
        # Week 1: Wednesday-Friday -> develop
        export TARGET_ENV="develop"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK == 4 ]]; then
        # Week 2: Thursday -> staging
        export TARGET_ENV="staging"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK -ge 1 && $DAY_OF_WEEK -le 3 ]]; then
        # Week 2: Monday-Wednesday -> develop
        export TARGET_ENV="develop"
      else
        echo "Today doesn't match any scheduled testing pattern. Skipping tests."
        # Create a placeholder file to satisfy the artifacts requirement
        touch child-pipeline.yml
        # Exit with error code to mark job as failed and stop the pipeline
        exit 1
      fi
      
      echo "Creating child pipeline for environment: $TARGET_ENV"
      
      # Create child pipeline with proper environment configuration
      cat > child-pipeline.yml << EOF
      image: node:latest
      
      stages:
        - test
        - reporting
      
      run-tests:
        stage: test
        environment:
          name: ${TARGET_ENV}
        before_script:
          - npm ci
        after_script:
          - echo "Completed tests"
        parallel: 3
        artifacts:
          when: always
          expire_in: 2 days
          paths:
            - test-results
        script:
          - echo "Running tests in ${TARGET_ENV} environment" 
          - npm run test:remote
      
      generate-report:
        stage: reporting
        before_script:
          - npm ci
        script:
          - npm run generate-report
          - npm run notify-team
        artifacts:
          when: always
          expire_in: 7 days
          paths:
            - test-report
      EOF

# Second job: triggers the child pipeline
tests-scheduled:
  stage: test
  needs:
    - determine-environment
  rules:
    - if: '($CI_PIPELINE_SOURCE == "schedule" && $SCHEDULER_JOB == "testing")'
  trigger:
    include:
      - artifact: child-pipeline.yml
        job: determine-environment
```

## How the Solution Works

1. **Job Separation**: I split the logic into two jobs:
   - `determine-environment`: Decides which environment to test based on the date
   - `tests-scheduled`: Triggers a child pipeline with the configuration from the first job

2. **Dynamic YAML Generation**: The first job creates a complete YAML configuration for the child pipeline, including:
   - The Docker image
   - Job configuration
   - The proper `environment: name: ${TARGET_ENV}` setting
   - Test execution commands

3. **Child Pipeline Trigger**: The second job takes this generated YAML and runs it as a child pipeline.

4. **Environment-Scoped Variables**: Because the child pipeline's job has the correct environment name, GitLab automatically injects all the environment-specific variables.

5. **Proper Failure Handling**: If testing shouldn't run on a particular day, I explicitly fail the first job (with `exit 1`), which prevents the child pipeline from running.

## Key Advantages

This approach provides several significant benefits:

1. **Environment Variable Isolation**: Each environment's variables are properly scoped and isolated.

2. **Schedule-Aware Testing**: The 2-week cycle logic determines exactly when and where tests should run.

3. **Clean Pipeline Structure**: The parent pipeline handles scheduling logic, while the child pipeline handles test execution.

4. **No Template Inheritance Issues**: The child pipeline has a complete, self-contained configuration without relying on parent templates.

5. **Proper Failure Reporting**: When tests shouldn't run on a particular day, the pipeline fails clearly and early.

## Gotchas and Lessons Learned

During implementation, I encountered a few subtle issues:

1. **Script + Trigger Conflict**: GitLab doesn't allow having both a `script` and a `trigger` in the same job, which is why I needed two separate jobs.

2. **Child Pipeline Template Access**: Child pipelines don't have access to the parent pipeline's templates (like `.tests`), so I had to include all configuration directly.

3. **Variable Escaping**: When creating the child pipeline YAML, environment variables need to be escaped properly (using backslashes) to prevent premature expansion.

4. **Artifact Management**: Always create the artifact file (even an empty one) to avoid "artifact not found" errors in downstream jobs.

## Conclusion

Dynamic environment selection with child pipelines is a powerful technique for complex CI/CD workflows. It allows you to make runtime decisions about where and when to run tests while still maintaining proper environment isolation and variable scoping.

This approach is particularly valuable when:

- You have multiple environments with different configurations
- Your test scheduling logic is complex
- You need to ensure proper access to environment-specific credentials
- You want to avoid running unnecessary tests

By leveraging GitLab's child pipeline feature, you can create truly dynamic, environment-aware testing workflows that adapt to your schedule and requirements while maintaining proper security and isolation.