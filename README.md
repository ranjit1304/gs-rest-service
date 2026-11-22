# Jenkins Practice Exercise

Goal: build a complete CI/CD pipeline for a Spring Boot app that builds, tests,
packages a Docker image, smoke-tests it, and releases it with manual approval.
This mirrors the "build, test, release, reproducible delivery" topics of the role.

## Prerequisites
- Docker Desktop or Docker Engine with Compose v2
- A Git repo you can push to (a fork of the sample app works well)

## 1. Start Jenkins
```
docker compose up -d --build
docker compose logs jenkins | grep -A2 "initial admin"
```
Open http://localhost:8080, paste the password, choose "Install suggested plugins" or skip,
and create an admin user.

## 2. Prepare the app
Fork https://github.com/spring-guides/gs-rest-service and copy `Jenkinsfile.starter`
into the repo root as `Jenkinsfile`. The app lives in the `complete/` folder.

## 3. Create the job
New Item -> Pipeline -> "Pipeline script from SCM" -> Git -> your fork URL -> Script Path: `Jenkinsfile`.

## 4. Work through the TODOs (11 in total)
Do not look at `Jenkinsfile.solution` until you are stuck. Run the job after every 1-2 TODOs.

## 5. Break it on purpose (this is where you learn the most)
1. Make a unit test fail. Is the failure visible in the test report and does the pipeline stop?
2. Point the smoke test at the wrong port. Does the container get cleaned up?
3. Abort at the approval step. What state does the build end in?
4. Run the build twice. Is the second faster (Maven cache)?
5. Change the base image tag to one that doesn't exist. Where does it fail and how do you read the log?

## 6. Stretch goals (the "senior" part)
- Add JaCoCo and fail the build below 70% coverage (quality gate).
- Add an image vulnerability scan stage (Trivy as a container).
- Move the stages into a Jenkins Shared Library.
- Add a `when { branch 'main' }` condition so Release only runs on main (needs a multibranch job).
- Pin the base images by digest and explain why that makes builds more reproducible.
- Write a short note: how would you adapt this for an on-prem hospital delivery (versioned tar, checksums, rollback)?

## Self-check questions (the interviewer may ask these)
- Why `agent none` at the top and agents per stage?
- Why stash/unstash instead of rebuilding in each stage?
- Why build once and promote the same artifact?
- What does `fingerprint: true` give you?
- What are the risks of mounting the Docker socket into Jenkins? (practice only, never in production)
- Declarative vs scripted pipeline: when would you need `script {}`?

## Troubleshooting
- Docker agent fails with "permission denied" on the socket: make sure the compose file's `user: root` is applied.
- Smoke test can't resolve the container name: confirm the network is `jenkins-practice_default` (`docker network ls`).
- Slow first build: Maven downloads dependencies once, later builds use the `maven-repo` volume.
