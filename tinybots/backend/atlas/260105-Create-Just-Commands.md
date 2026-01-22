# 📋 ATLAS-001: 2026-01-05 - Tạo Just Commands cho Atlas Repository

## References

- Hướng dẫn: [devdocs/tinybots/devtools/OVERVIEW-opus.md](../devtools/OVERVIEW-opus.md)
- Template: [devdocs/agent/templates/create-plan.md](../../agent/templates/create-plan.md)
- CI Config: [atlas/ci/docker-compose.yml](../../../atlas/ci/docker-compose.yml)
- CI Test Script: [atlas/ci/test.sh](../../../atlas/ci/test.sh)
- Main Justfile: [devtools/tinybots/local/Justfile](../../../devtools/tinybots/local/Justfile)

## 🎯 Objective

Tích hợp Atlas repository vào hệ thống centralized testing infrastructure của devtools bằng cách:
1. Thêm service definition cho Atlas vào `devtools/docker-compose.yaml`
2. Tạo file `devtools/justfiles/atlas.just` với các commands: `start-atlas`, `test-atlas`, `log-atlas`
3. Import commands vào main Justfile

### ⚠️ Key Considerations

**Đặc điểm độc nhất của Atlas:**
- **Dual Database Setup**: Atlas cần 2 databases:
  - **TinyBots schema**: Dùng `mysql-typ-e-db` có sẵn (shared with other repos)
  - **Analytics schema**: Tạo mới `mysql-atlas-intelligence-db` riêng cho Intelligence
- **Intelligence Migration Service**: Cần service `intelligence` mới để migrate analytics database
- **Different Naming Convention**: Sử dụng `TYP_DB_RW_HOST` và `INT_DB_RW_HOST` thay vì naming pattern thông thường
- **No External Service Dependencies**: Không cần LocalStack, checkpoint, prowl, wonkers, hay service nào khác

**Approach:**
- **Reuse existing typ-e-db**: Dùng `mysql-typ-e-db` và `typ-e` migration runner có sẵn
- **Create only Intelligence infrastructure**: Chỉ cần tạo 2 services mới: `mysql-atlas-intelligence-db` và `intelligence`
- **Simple command set**: Chỉ cần 2 commands cơ bản: `test-atlas` và `start-atlas`

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

#### 1.1 Analyze Atlas CI Dependencies ✅
- **Current CI Setup**:
  ```yaml
  # atlas/ci/docker-compose.yml
  mysql-db:           # Port 1235 → TinyBots schema
  int-mysql-db:       # Port 1236 → Analytics schema  
  typ-e:              # Migration runner for TinyBots
  intelligence:       # Migration runner for Analytics
  node:               # Test runner
  ```

- **Startup Sequence** (from `ci/test.sh`):
  ```bash
  1. docker-compose up -d mysql-db typ-e
  2. Wait for typ-e migrations (docker attach)
  3. docker-compose up -d int-mysql-db intelligence  
  4. Wait for intelligence migrations (docker attach)
  5. docker-compose up -d (test runner)
  ```

- **Environment Variables**:
  ```bash
  TYP_DB_RW_HOST: mysql-db       # TinyBots database
  TYP_DB_PORT: 3306
  INT_DB_RW_HOST: int-mysql-db   # Analytics/Intelligence database
  INT_DB_PORT: 3306
  ```

- **Test Command**: `yarn test` (from `ci/node-verify.sh`)
  - Coverage thresholds: 85% statements, 85% functions, 75% branches, 85% lines

- **Outcome**: ✅ Atlas cần 2 databases với intelligence migration service

#### 1.2 Architecture Decision ✅ CONFIRMED
- **Approach: Hybrid - Reuse TYP-E DB, Create Intelligence DB**
  - ✅ Dùng `mysql-typ-e-db` và `typ-e` migration runner có sẵn cho TinyBots schema
  - ✅ Tạo `mysql-atlas-intelligence-db` mới riêng cho Analytics schema
  - ✅ Tạo `intelligence` migration service mới trong devtools
  - **Pros**: 
    - Tận dụng infrastructure có sẵn (mysql-typ-e-db)
    - Chỉ tạo thêm services thực sự cần thiết
    - Intelligence schema được isolated riêng
  - **Cons**: 
    - TYP-E database shared state - nhưng OK vì các repos khác đã dùng shared approach

#### 1.3 Check Intelligence Service Availability
- [ ] Verify intelligence Docker image exists: `693338167548.dkr.ecr.eu-central-1.amazonaws.com/intelligence:latest`
- [ ] Check if we need AWS credentials to pull intelligence image
- [ ] Alternative: Build intelligence image locally if ECR access is an issue
- **Outcome**: [To be determined after checking]

#### 1.4 Edge Cases & Considerations
- [ ] **Database Port Conflicts**: Atlas CI dùng ports 1235, 1236 - devtools có thể dùng ports khác (1125, 1126?)
- [ ] **Intelligence Service**: Chưa có trong devtools, cần add service definition
- [ ] **Yarn Version**: Atlas dùng Yarn 1.22.22 (classic) - check compatibility với node:22-alpine
- [ ] **Coverage Reports**: `nyc` coverage với thresholds cao - đảm bảo paths đúng khi mount volume
- [ ] **Python Scripts**: Có `loadFlair.py` trong CI - check xem có cần Python trong container không

---

### Phase 2: Implementation Structure

#### File Structure
```
devtools/
├── docker-compose.yaml         # 🔄 TO UPDATE - Thêm atlas services
│   ├── mysql-atlas-tinybots-db     # New DB for TinyBots schema  
│   ├── mysql-atlas-intelligence-db  # New DB for Intelligence schema
│   ├── intelligence                 # New migration runner
│   └── atlas                        # New test runner service
│3 services mới
│   ├── mysql-typ-e-db              # ✅ REUSE - Existing TYP-E database
│   ├── typ-e                       # ✅ REUSE - Existing migration runner
│   ├── mysql-atlas-intelligence-db # ✅ NEW - Intelligence database
│   ├── intelligence                # ✅ NEW - Intelligence migration runner
│   └── atlas                       # ✅ NEW - Node.js test runner container
│
├── Justfile                    # 🔄 TO UPDATE - Import atlas.just
│
└── justfiles/
    └── atlas.just              # ✅ NEW FILE - 2 commands only

#### Step 3.1: Add Service Definitions to docker-compose.yaml
Intelligence Database Service**
```yaml
  # ------------------------------------- Atlas Intelligence Database -------------------------------------
  mysql-atlas-intelligence-db:
    image: mysql:8.0
    environment:
      MYSQL_USER: dbadmin
      MYSQL_PASSWORD: v7tL4VY6PEqnL5WH
      MYSQL_ROOT_PASSWORD: ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN
      MYSQL_DATABASE: analytics
    ports:
      - "1126:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 10
```

**3.1.2 Add Intelligence Migration Service**
```yaml
  intelligence:
    image: 693338167548.dkr.ecr.eu-central-1.amazonaws.com/intelligence:latest
    depends_on:
      mysql-atlas-intelligence-db:
        condition: service_healthy
    environment:
      DB_RW_HOST: mysql-atlas-intelligence-db:3306
      DB_HOST: mysql-atlas-intelligence-db:3306
      DB_RW_USERNAME: root
      DB_RW_PASSWORD: ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN
    labels:
      - "service.name=intelligence"
    volumes:
      - $HOME/.m2:/root/.m2  # Maven cache (if intelligence uses Maven)
```

**3.1.3 Add Atlas Node.js Test Runner Service**
```yaml
  atlas:
    image: node:22-alpine
    volumes:
      - /Users/kai/work/tinybots/tinybots/backend/atlas:/usr/src/app
    labels:
      - "atlas-service"
    environment:
      TYP_DB_RW_HOST: mysql-typ-e-db
      TYP_DB_PORT: 3306
      INT_DB_RW_HOST: mysql-atlas-intelligence-db
      INT_DB_PORT: 3306
      # ATLAS_QUERY_LIMIT: "10"  # Optional - uncomment if needed
    working_dir: /usr/src/app
    entrypoint: ["sh", "-c", "corepack enable && yarn install && yarn test"]
    depends_on:
      typ-e:
        condition: service_completed_successfully
      intelligence:
        condition: service_completed_successfully
```

**Notes:**
- ✅ **Reuse mysql-typ-e-db**: Environment variable `TYP_DB_RW_HOST` points to existing `mysql-typ-e-db`
- ✅ **New Intelligence DB**: `INT_DB_RW_HOST` points to new `mysql-atlas-intelligence-db`
- ✅ **Depends on existing typ-e**: Uses existing `typ-e` migration runner, not creating new one
- ✅ **Node.js container**: Added `atlas` service with node:22-alpine image
- Sử dụng `corepack enable` để support Yarn 1.22.22
- Dependencies đảm bảo migrations chạy xong trước khi start testschạy xong trước khi start tests
- Volume mount đúng path tuyệt đối

---

#### Step 3.2: Create devtools/justfiles/atlas.just

```just
# ------------------------------------- Atlas -------------------------------------

# Start all Atlas dependencies (for development/debugging)
start-atlas:
    {{compose}} up -d \
      mysql-typ-e-db typ-e mysql-atlas-intelligence-db intelligence
    -{{compose}} logs -f \
      mysql-typ-e-db typ-e mysql-atlas-intelligence-db intelligence

# Run Atlas integration tests (resets DBs for clean state)
test-atlas:
    {{compose}} rm -sf mysql-typ-e-db mysql-atlas-intelligence-db
    {{compose}} up -d \
      mysql-typ-e-db typ-e mysql-atlas-intelligence-db intelligence
    {{compose}} run --rm --use-aliases atlas
```

**Command Breakdown:**
- `start-atlas`: Khởi động dependencies để debug/develop locally
  - Starts: mysql-typ-e-db (reuse), typ-e (reuse), mysql-atlas-intelligence-db (new), intelligence (new)
  - Tails logs của tất cả services
  
- `test-atlas`: **Main test command** - Chạy full integration tests
  - Removes both databases: mysql-typ-e-db (shared) và mysql-atlas-intelligence-db (dedicated) for clean state
  - Starts all dependencies and waits for migrations to complete
  - Runs atlas test runner container

**Key Points:**
- ✅ **Simple**: Chỉ 2 commands cơ bản như yêu cầu
- ✅ **Reuses typ-e infrastructure**: Dùng mysql-typ-e-db và typ-e có sẵn
- ✅ **Removes typ-e-db**: Reset shared database để đảm bảo clean state cho Atlas tests
- ✅ **Removes intelligence-db**: Reset intelligence database riêng
- ✅ **No external services**: Không cần localstack, checkpoint, prowl, etc.
   ```

3. **Run Atlas Tests**
   ```bash
   just test-atlas
   # Should run and pass all tests with correct coverage
   ```

4. **Test Cleanup**
   ```bash
   docker compose rm -sf mysql-typ-e-db mysql-atlas-intelligence-db
   # Verify containers are removed
   ```

5. **Verify No Port Conflicts**
   ```bash
   lsof -i :1123  # Should show mysql-typ-e-db when running (existing)
   lsof -i :1126  # Should show mysql-atlas-intelligence-db when running (new)
   ```2: Dedicated Intelligence Database ✅ ADOPTED
**Approach**: Tạo `mysql-atlas-intelligence-db` riêng cho Atlas
- ✅ **Adopted**: Intelligence schema không được dùng bởi repos khác
- **Benefit**: Isolated, không ảnh hưởng các repos khác

#### Decision 3: Simple Command Set ✅ ADOPTED
**Approach**: Chỉ 2 commands (`start-atlas`, `test-atlas`)
- ✅ **Adopted**: Keeps interface simple and consistent with other repos
- **Note**: Có thể thêm commands khác sau nếu cần (unit-test, performance-test, dev)

#### Backup Plan: Local Intelligence Build
**If ECR access fails:**
- Build intelligence image locally từ source code
- Cần intelligence repository và Dockerfile
- Update image tag in docker-compose.yaml

---

## 📊 Summary of Results
> Sẽ update sau khi implementation hoàn thành

### ✅ Completed Achievements
- [ ] Intelligence database service added (mysql-atlas-intelligence-db)
- [ ] Intelligence migration service integrated
- [ ] Atlas Node.js test runner container added
- [ ] atlas.just file created with 2 commands (start-atlas, test-atlas)
- [ ] Commands imported in main Justfile
- [ ] Tests run successfully via `just test-atlas`
- [ ] Verified reuse of existing mysql-typ-e-db and typ-e services

---

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

1. **Intelligence Image Access**
   - [ ] Verify ECR access for intelligence:latest image
   - [ ] If ECR blocked, need alternative solution (local build or public registry)
   - **Impact**: Blocking - cannot run migrations without intelligence

2. **Architecture Decision** ✅ RESOLVED
   - ✅ Confirmed: Reuse mysql-typ-e-db + Create dedicated mysql-atlas-intelligence-db
   - ✅ This approach balances simplicity and isolation
   - **Impact**: Low - optimal balance achieved

3. **Python Dependency**
   - [ ] Check if `loadFlair.py` script is used in tests
   - [ ] If yes, need Python in node:22-alpine or separate service
   - **Impact**: Low - might only be used in build, not tests

4. **Yarn Classic Compatibility**
   - [ ] Verify `corepack enable` works correctly with Yarn 1.22.22 in node:22-alpine
   - [ ] Test yarn install runs without issues
   - **Impact**: Medium - affects test execution

5. **Coverage Report Paths**
   - [ ] Verify nyc coverage reports generate correctly with volume mounts
   - [ ] Paths in .nyc_output and coverage/ folders should be accessible
   - **Impact**: Low - coverage still runs, just reports might need adjustment

---

## 🎯 Next Steps

1. **Verify Intelligence Access**: Check ECR permissions or find alternative
2. **Implement Service Definitions**: Add 3 services to docker-compose.yaml:
   - mysql-atlas-intelligence-db (new database)
   - intelligence (new migration runner)
   - atlas (new Node.js test runner)
3. **Create atlas.just**: Write 2 commands (start-atlas, test-atlas)
4. **Import in Justfile**: Add `import 'justfiles/atlas.just'`
5. **Test Execution**: Run `just test-atlas` and verify results
6. **Document**: Update Atlas OVERVIEW.md with devtools integration info (if exists)

---

## 📝 Notes

- **Simplified Approach**: Reuse existing typ-e infrastructure, only create Intelligence-specific services
- **Only 3 new services**: mysql-atlas-intelligence-db, intelligence, atlas (Node.js container)
- **2 commands only**: start-atlas và test-atlas (giữ interface đơn giản)
- Intelligence service có thể được reuse nếu repos khác cần analytics schema trong tương lai
- Consider documenting intelligence service trong devtools/OVERVIEW-opus.md sau khi implement

## 📋 Implementation Summary

**Services to Add:**
1. ✅ mysql-atlas-intelligence-db - Analytics database
2. ✅ intelligence - Analytics migration runner  
3. ✅ atlas - Node.js test runner container

**Services to Reuse:**
1. ✅ mysql-typ-e-db - TinyBots database (existing)
2. ✅ typ-e - TinyBots migration runner (existing)

**Just Commands to Create:**
1. ✅ start-atlas - Start dependencies
2. ✅ test-atlas - Run tests with DB reset
