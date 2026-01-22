# 📋 [PROD-XXX: 2025-11-25] - CI/CD Pipeline cho azi-3-status-check-jobs

## TL;DR

- **Objective**: Tạo complete CI/CD pipeline cho `azi-3-status-check-jobs` để build Docker image và deploy lên production server
- **Pattern**: Follow megazord-events và azi-3-status-check structure với điều chỉnh cho background job service
- **Key Updates**:
  - Package name: `azi-3-status-check-jobs` (đã được update)
  - Expose port 3000 cho health endpoint `/health` và debug endpoint `/internal/v1/monitoring/sessions`
  - Simplified test pipeline: chỉ cần service container, không cần DB/external deps
- **Deliverables**: 14 files (Dockerfile + 13 CI scripts/configs)

## References

- Source pattern: `/Users/kai/work/tinybots/tinybots/backend/megazord-events/ci/`
- Reference pattern: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check/ci/`
- Target repository: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/`
- Template document: `/Users/kai/work/tinybots/devdocs/agent/TEMPLATE.md`
- Global standard: `/Users/kai/work/tinybots/devdocs/tinybots/OVERVIEW.md`
- Repo standard: `/Users/kai/work/tinybots/devdocs/tinybots/backend/azi-3-status-check-jobs/OVERVIEW.md`

## User Requirements

Tạo CI/CD pipeline deployment cho service `azi-3-status-check-jobs` dựa trên pattern hiện có từ `megazord-events` và `azi-3-status-check`.

## 🎯 Objective

Triển khai đầy đủ CI/CD pipeline cho `azi-3-status-check-jobs` để build Docker image và deploy lên server production, đảm bảo consistency với các service khác trong TinyBots ecosystem.

### ⚠️ Key Considerations

1. **Consistency với existing services**: Pipeline phải follow pattern của `megazord-events` và `azi-3-status-check`
2. **Private repository access**: Cần DEPLOYMENT_KEY để truy cập Git dependencies (tiny-backend-tools, tiny-internal-services)
3. **Multi-stage Docker build**: Tách biệt builder và production image để optimize size
4. **Yarn 3 workspaces**: Service sử dụng Yarn 3.8.7 với workspace-tools plugin
5. **No schemas directory**: Khác với megazord-events, service này không có `schemas/` folder
6. **Background job service**: Service chạy cron jobs và SQS consumers, không expose HTTP port như azi-3-status-check

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Analyze pipeline requirements
  - **Outcome**: Cần tạo cấu trúc CI tương tự megazord-events/azi-3-status-check với các điều chỉnh phù hợp
- [x] Identify differences with reference services
  - **Outcome**:
    - Không có schemas directory (khác megazord-events)
    - **CÓ expose endpoint**: `/internal/v1/monitoring/sessions` (debug endpoint)
    - **CÓ expose port 3000**: Health check và monitoring endpoints
    - Package name đã được update: `azi-3-status-check-jobs` (consistent với folder)
    - Entry point là `dist/cmd` (khác azi-3-status-check là `dist/cmd/app`)
    - Test pipeline chỉ cần container duy nhất của service, không cần external dependencies
- [x] Review dependencies and build process
  - **Outcome**:
    - Yarn 3.8.7 với corepack
    - Build script: `yarn lint && tsc --project tsconfig.prod.json`
    - Start script: `node dist/cmd`

### Phase 2: Implementation (File/Code Structure)

Cấu trúc CI/CD files cần tạo:

```text
azi-3-status-check-jobs/
├── ci/                                    # 🚧 TODO - CI/CD directory
│   ├── build.sh                          # 🚧 TODO - Local Docker build script
│   ├── build-azi-3-status-check-jobs.yml # 🚧 TODO - Concourse build task definition
│   ├── concourse-build.sh                # 🚧 TODO - Concourse build wrapper
│   ├── concourse-test.sh                 # 🚧 TODO - Concourse test wrapper
│   ├── docker-compose.yml                # 🚧 TODO - Test environment (service container only)
│   ├── docker-entrypoint.sh              # 🚧 TODO - Container entrypoint
│   ├── local-test.sh                     # 🚧 TODO - Local test runner
│   ├── node-verify.sh                    # 🚧 TODO - Node setup and test execution
│   ├── reset.sh                          # 🚧 TODO - Cleanup Docker resources
│   ├── ssh.config                        # 🚧 TODO - SSH config for Git access
│   ├── test.sh                           # 🚧 TODO - Test runner with Docker Compose
│   └── test-azi-3-status-check-jobs.yml  # 🚧 TODO - Concourse test task
├── Dockerfile                             # 🚧 TODO - Multi-stage Docker build
└── .dockerignore                          # 🚧 TODO - Docker ignore patterns
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Tạo Dockerfile

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/Dockerfile`

**Nội dung**: Multi-stage build với builder và production stages

**Đặc điểm**:

- Stage 1 (builder): Build TypeScript code với full devDependencies
- Stage 2 (production): Copy artifacts và install production dependencies only
- Sử dụng DEPLOYMENT_KEY build arg cho private Git repos
- Enable corepack cho Yarn 3
- Copy config directory cho environment-specific settings
- **KHÔNG copy schemas/** (service không có)
- **EXPOSE port 3000**: Health check endpoint `/health` và debug endpoint `/internal/v1/monitoring/sessions`
- Entrypoint: `/docker-entrypoint.sh`

**Template dựa trên**: `azi-3-status-check/Dockerfile` và `megazord-events/Dockerfile`

```dockerfile
FROM node:22-alpine AS builder
ARG DEPLOYMENT_KEY

# Install git, openssh
RUN apk update && apk add --no-cache git openssh

COPY ./ci/ssh.config /root/.ssh/config
COPY ./ci/docker-entrypoint.sh /docker-entrypoint.sh

RUN mkdir -p /root/.ssh && \
  echo "$DEPLOYMENT_KEY" > /root/.ssh/id_rsa && \
  chmod 600 /root/.ssh/id_rsa && \
  chmod 600 /root/.ssh/config

WORKDIR /app

COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn/releases .yarn/releases

ENV COREPACK_ENABLE_DOWNLOAD_PROMPT=0
RUN corepack enable

RUN yarn install

COPY . .

RUN yarn build

# production image
FROM node:22-alpine AS production

COPY --from=builder /docker-entrypoint.sh ./docker-entrypoint.sh

WORKDIR /app

COPY --from=builder /app/.yarn ./.yarn
COPY --from=builder /root/.yarn /root/.yarn
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/yarn.lock ./yarn.lock
COPY --from=builder /app/.yarnrc.yml ./.yarnrc.yml

COPY --from=builder /app/config ./config

RUN yarn plugin import workspace-tools

RUN yarn workspaces focus --production

RUN rm -rf ./.yarn/cache

EXPOSE 3000

ENTRYPOINT ["/docker-entrypoint.sh"]
```

#### Step 2: Tạo .dockerignore

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/.dockerignore`

**Nội dung**: Exclude unnecessary files khỏi build context

```text
node_modules
dist
coverage
.nyc_output
.git
.idea
*.log
test
.gitignore
.prettierrc.json
.prettierignore
devdocs
README.md
```

#### Step 3: Tạo CI directory và scripts

##### 3.1: ssh.config

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/ssh.config`

**Nội dung**: SSH configuration cho Bitbucket access

```ssh_config
host bitbucket.org
 HostName bitbucket.org
 IdentityFile /root/.ssh/id_rsa
 User git
 BatchMode yes
 StrictHostKeyChecking no
 
```

##### 3.2: docker-entrypoint.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/docker-entrypoint.sh`

**Nội dung**: Container entrypoint script

```bash
#!/bin/sh
set -e

yarn start
```

**Permissions**: Cần set executable: `chmod +x ci/docker-entrypoint.sh`

##### 3.3: build.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/build.sh`

**Nội dung**: Local build script cho Docker image

```bash
#!/bin/sh

trap '{
    VOLUMES=$(docker volume ls -q)
    if [ -n "$VOLUMES" ]
    then
        docker volume rm $VOLUMES
    fi
    }' EXIT

set -e -u

apk update
apk add git

docker login -u $DOCKER_ID -p $DOCKER_PASSWORD

echo $(git describe --tags --always) >../azi-3-status-check-jobs/version
VERSION=$(git describe --tags --always)

docker build -t azi-3-status-check-jobs:local --build-arg DEPLOYMENT_KEY="$DEPLOYMENT_KEY" --build-arg version=$VERSION .
```

**Permissions**: `chmod +x ci/build.sh`

##### 3.4: concourse-build.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/concourse-build.sh`

**Nội dung**: Concourse build wrapper

```bash
#!/bin/sh
set -e

source /docker-lib.sh
start_docker

ci/build.sh

docker save azi-3-status-check-jobs:local >../azi-3-status-check-jobs/azi-3-status-check-jobs-image.tar
```

**Permissions**: `chmod +x ci/concourse-build.sh`

##### 3.5: build-azi-3-status-check-jobs.yml

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/build-azi-3-status-check-jobs.yml`

**Nội dung**: Concourse task definition cho build

```yaml
---
platform: linux

image_resource:
  type: docker-image
  source:
    repository: xzilde/concourse-dind-pip
    tag: latest
    username: ((DOCKER_ID))
    password: ((DOCKER_PASSWORD))

inputs:
  - name: azi-3-status-check-jobs.git

outputs:
  - name: azi-3-status-check-jobs

run:
  dir: azi-3-status-check-jobs.git
  path: "ci/concourse-build.sh"
```

#### Step 4: Test Pipeline Files

**Note**: Test pipeline với isolated container setup:

- Docker Compose chỉ với container của service
- Không cần external dependencies (DB, Wonkers, etc.) - tests sử dụng mocks
- Tests run với `yarn test` trong isolated container
- Matching pattern từ megazord-events nhưng simplified

##### 4.1: docker-compose.yml

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/docker-compose.yml`

**Nội dung**: Minimal Docker Compose cho test environment

```yaml
services:
  node:
    image: node:22-alpine
    volumes:
      - ..:/usr/src/app
      - ./ssh.config:/root/.ssh/config
      - $DEPLOYMENT_KEY:/root/.ssh/id_rsa
    labels:
      - azi-3-status-check-jobs
    environment:
      NODE_ENV: test
      # Mock external service addresses for tests
      STATUS_QUEUE_ADDRESS: http://localhost:4566/000000000000/status-queue
      MEGAZORD_EVENTS_ADDRESS: http://localhost:8080
    working_dir: /usr/src/app
    entrypoint: ci/node-verify.sh
```

**Note**: Service này không cần MySQL, Wonkers, hay các service khác. Tests sử dụng mocks.

##### 4.2: node-verify.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/node-verify.sh`

**Nội dung**: Setup Node environment và run tests

```bash
#!/bin/sh

set -eux

apk update
apk add git openssh

eval $(ssh-agent)

npm config set shell sh
chown -R root /root/.ssh/id_rsa
chown -R root /root/.ssh/config
chmod 700 /root/.ssh/id_rsa
chmod 700 /root/.ssh/config
ssh-add /root/.ssh/id_rsa
ssh-add -l

export COREPACK_ENABLE_DOWNLOAD_PROMPT=0
corepack enable
yarn install --json

yarn test
```

**Permissions**: `chmod +x ci/node-verify.sh`

##### 4.3: test.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/test.sh`

**Nội dung**: Run tests với Docker Compose

```bash
#!/bin/sh

trap '{
    docker-compose -f ci/docker-compose.yml down
    VOLUMES=$(docker volume ls -q)
    if [ -n "$VOLUMES" ]
    then
        docker volume rm $VOLUMES
    fi
    }' EXIT

set -e

docker login -u $DOCKER_ID -p $DOCKER_PASSWORD
docker-compose -f ci/docker-compose.yml up -d
docker attach $(docker ps -q --filter=label=azi-3-status-check-jobs)
```

**Permissions**: `chmod +x ci/test.sh`

##### 4.4: local-test.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/local-test.sh`

**Nội dung**: Local test runner wrapper

```bash
#!/usr/bin/env bash

set -u -e

ci/test.sh
```

**Permissions**: `chmod +x ci/local-test.sh`

##### 4.5: reset.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/reset.sh`

**Nội dung**: Cleanup Docker resources

```bash
docker-compose -f ci/docker-compose.yml down
VOLUMES=$(docker volume ls -q)
if [ -n "$VOLUMES" ]
then
    docker volume rm $VOLUMES
fi
```

**Permissions**: `chmod +x ci/reset.sh`

##### 4.6: concourse-test.sh

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/concourse-test.sh`

**Nội dung**: Concourse test wrapper

```bash
#!/bin/sh

set -e

pip install awscli
apk update
apk add git

source /docker-lib.sh
start_docker

echo $DEPLOYMENT_KEY | sed -e 's/\(KEY-----\)\s/\1\n/g; s/\s\(-----END\)/\n\1/g' | sed -e '2s/\s\+/\n/g' >deployment_key

export DEPLOYMENT_KEY=$(pwd)/deployment_key
chmod 600 $DEPLOYMENT_KEY

ci/test.sh

echo $(git describe --tags --always) >../azi-3-status-check-jobs/azi-3-status-check-jobs
echo $(cat ../azi-3-status-check-jobs/version)
```

**Permissions**: `chmod +x ci/concourse-test.sh`

##### 4.7: test-azi-3-status-check-jobs.yml

**File**: `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/test-azi-3-status-check-jobs.yml`

```yaml
---
platform: linux

image_resource:
  type: docker-image
  source:
    repository: xzilde/concourse-dind-pip
    tag: latest
    username: ((DOCKER_ID))
    password: ((DOCKER_PASSWORD))

inputs:
  - name: git
  - name: db.version

outputs:
  - name: azi-3-status-check-jobs

run:
  dir: git
  path: "ci/concourse-test.sh"
```

#### Step 5: Environment Variables & Secrets

**Required secrets trong Concourse pipeline**:

1. **DOCKER_ID**: Docker Hub username
2. **DOCKER_PASSWORD**: Docker Hub password  
3. **DEPLOYMENT_KEY**: SSH private key để access private Bitbucket repos
   - Format: PEM private key với newlines
   - Used để clone: `tiny-backend-tools`, `tiny-internal-services`, `tiny-testing`

**Deployment key format example**:

```text
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
...
-----END RSA PRIVATE KEY-----
```

#### Step 6: Verification Checklist

Sau khi tạo xong files, verify:

- [ ] Tất cả shell scripts có execute permission (`chmod +x`)
- [ ] Dockerfile build thành công locally với:

  ```bash
  docker build -t azi-3-status-check-jobs:test \
    --build-arg DEPLOYMENT_KEY="$(cat ~/.ssh/id_rsa)" .
  ```

- [ ] Container start được với: `docker run azi-3-status-check-jobs:test`
- [ ] Config directory được copy vào production image
- [ ] Yarn workspaces focus hoạt động correct
- [ ] Entry point script executable và start service

#### Step 7: Integration với Concourse CI

**Concourse pipeline configuration** (ngoài scope của plan này, nhưng notes để reference):

```yaml
resources:
  - name: azi-3-status-check-jobs.git
    type: git
    source:
      uri: git@bitbucket.org:tinybots/azi-3-status-check-jobs.git
      branch: master
      private_key: ((GIT_PRIVATE_KEY))

jobs:
  - name: build-azi-3-status-check-jobs
    plan:
      - get: azi-3-status-check-jobs.git
        trigger: true
      - task: build
        file: azi-3-status-check-jobs.git/ci/build-azi-3-status-check-jobs.yml
        params:
          DOCKER_ID: ((DOCKER_ID))
          DOCKER_PASSWORD: ((DOCKER_PASSWORD))
          DEPLOYMENT_KEY: ((DEPLOYMENT_KEY))
      - put: azi-3-status-check-jobs-image
        params:
          image: azi-3-status-check-jobs/azi-3-status-check-jobs-image.tar
```

### Phase 4: Testing & Validation

#### Local Testing Steps

1. **Build Docker image locally**:

   ```bash
   cd /Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs
   
   # Set deployment key
   export DEPLOYMENT_KEY="$(cat ~/.ssh/id_rsa)"
   
   # Build image
   docker build -t azi-3-status-check-jobs:local \
     --build-arg DEPLOYMENT_KEY="$DEPLOYMENT_KEY" .
   ```

2. **Verify image size**:

   ```bash
   docker images azi-3-status-check-jobs:local
   # Expected: ~500MB (production image without dev deps)
   ```

3. **Test container startup**:

   ```bash
   docker run --rm \
     -e NODE_ENV=production \
     azi-3-status-check-jobs:local
   ```

4. **Verify production dependencies**:

   ```bash
   docker run --rm azi-3-status-check-jobs:local \
     yarn list --depth=0
   # Should NOT show devDependencies
   ```

#### Concourse Testing Steps

1. **Push to feature branch** và verify Concourse trigger
2. **Check build logs** trong Concourse UI
3. **Verify Docker image** được save thành công
4. **Test deploy** lên staging environment (if available)

## 📊 Summary of Results

Implementation completed on 2025-11-25

### ✅ Completed Achievements

- [x] Analyzed reference implementations (megazord-events, azi-3-status-check)
- [x] Identified service-specific requirements and differences
- [x] Designed complete CI/CD file structure matching TinyBots patterns
- [x] Dockerfile created with multi-stage build
- [x] All CI scripts created and executable
- [x] docker-compose.yml for isolated testing
- [x] Concourse task definitions created
- [ ] Local Docker build successful (ready to test)
- [ ] Container starts without errors (ready to test)
- [ ] Tests pass in Docker environment (ready to test)
- [ ] Production dependencies optimized (ready to verify)

### 📝 Implementation Checklist

**Files created** (14 files total):

1. ✅ `/Users/kai/work/tinybots/devdocs/tinybots/backend/azi-3-status-check-jobs/251125-Pipeline-Deployment.md` - This plan
2. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/Dockerfile`
3. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/.dockerignore`
4. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/ssh.config`
5. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/docker-entrypoint.sh`
6. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/build.sh`
7. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/concourse-build.sh`
8. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/build-azi-3-status-check-jobs.yml`
9. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/docker-compose.yml`
10. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/node-verify.sh`
11. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/test.sh`
12. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/local-test.sh`
13. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/reset.sh`
14. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/concourse-test.sh`
15. ✅ `/Users/kai/work/tinybots/tinybots/backend/azi-3-status-check-jobs/ci/test-azi-3-status-check-jobs.yml`

**All shell scripts have executable permissions set.**

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

1. **Test pipeline**: Test pipeline với Docker Compose có cần thiết ngay không? Service này chủ yếu là background jobs, có thể implement test pipeline sau.

2. **Health check endpoint**: Service expose health endpoint tại `/health` và debug endpoint tại `/internal/v1/monitoring/sessions`. Port 3000 đã được EXPOSE trong Dockerfile và có thể được sử dụng cho:
   - Health checks trong orchestration platform (Kubernetes/ECS)
   - Monitoring và debugging production sessions
   - Load balancer health probes

3. **Environment-specific configs**: Service sử dụng `config/` với `custom-environment-variables.json`. Cần confirm:
   - Production config values
   - Environment variables mapping
   - Secret management strategy

4. **Logging & Monitoring**:
   - Winston logger output format
   - Log aggregation setup
   - Metrics/monitoring integration (if needed)

5. **Database dependencies**: Service không có database migrations trong Dockerfile. Nếu cần access MySQL (typ-e hoặc wonkers-db), cần:
   - Network configuration
   - Connection strings trong production config

6. **SQS Queue configuration**:
   - Production queue URLs
   - IAM permissions cho container
   - AWS credentials injection strategy

### 📝 Next Steps (Post-Implementation)

1. Document deployment procedure trong `devdocs/tinybots/azi-3-status-check-jobs/DEPLOYMENT.md`
2. Create runbook cho troubleshooting common issues
3. Setup monitoring alerts cho job failures
4. Implement test pipeline với integration tests
5. Consider adding Docker Compose for local development environment
6. Setup automated versioning/tagging strategy

---

**Implementation Priority**: HIGH  
**Estimated Effort**: 2-4 hours (excluding testing pipeline)  
**Dependencies**: Access to Concourse, Docker Hub, Bitbucket deployment keys  
**Risk Level**: LOW (following established patterns)
