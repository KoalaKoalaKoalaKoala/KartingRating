---
starter_id: spring
package_manager: maven
project_name: karting-rating
hints:
 language_family: java
 team_size: solo
 deployment_target: fly
 ci_provider: github-actions
 ci_default_flow: auto-deploy-on-merge
 bootstrapper_confidence: verified
 path_taken: standard
 quality_override: false
 self_check_answers: null
 has_auth: true
 has_payments: false
 has_realtime: false
 has_ai: false
 has_background_jobs: false
---

## Why this stack

KartingRating is a small web app (about ten users) with login, two roles, result entry and several computed season tables. You asked for a Java backend with PostgreSQL, and Spring Boot is the vetted default for Java web projects. It is typed, convention-based, widely represented in training data and well documented, so an AI agent can work reliably in it. Scaffolding is verified end-to-end. It fits your Java background and your goal of building a full application on your own. Frontend remains open; PostgreSQL is a database preference to apply when the project is scaffolded (e.g. Spring Data JPA). Deployment defaults to Fly with GitHub Actions auto-deploy on merge.
