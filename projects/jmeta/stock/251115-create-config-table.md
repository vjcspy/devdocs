# 📋 [251115: 2025-11-15] - Create Stock Common Configuration entity, mapper, DTO, repository, service, controller

## User Requirements
> 1. add created_at and updated_at column to entity
> 2. Create mapper, dto, repository, service (use mapstruct, follow like in `packages/stock/src/main/java/com/vjcspy/stock/stockinfo/domain/tick`)
> 3. Create controller

## 🎯 Objective
> Introduce `StockCommonConfiguration` domain with timestamped entity and full CRUD stack (DTO/Mapper/Repository/Service/Controller), aligned with existing Quarkus + Panache + MapStruct patterns.

[Implement `StockCommonConfigurationEntity` with `created_at`, `updated_at`; wire MapStruct mapper and Lombok DTO; add Panache repository and application-scoped service; expose JAX-RS resource for CRUD/queries returning `ApiResponse` wrapped payloads.]

### ⚠️ Key Considerations
> This describes the extremely important points or reasons that need attention
- Use Quarkus + Panache repository pattern as in tick domain (`StockInfoTickRepository.java:10`)
- Use MapStruct with `componentModel="cdi"` (`StockInfoTickMapper.java:7`) and Lombok DTO style (`StockInfoTickDto.java:11`)
- JAX-RS resource style with `@Path`, `@Consumes`, `@Produces` (`TickResource.java:25-28`), returning `ApiResponse`
- Timestamp fields type `Instant`; set `created_at` with default `CURRENT_TIMESTAMP`; manage `updated_at` on update operations in service
- Table name: `stock_common_configuration`; avoid reserved keyword by quoting `"key"` column like current entity
- Keep JSON column mapping via `@JdbcTypeCode(SqlTypes.JSON)` for `value`
- Add Flyway migration under `projects/http/src/main/resources/db/migration` to create table and indexes

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [ ] Analyze detailed requirements
  - **Outcome**: Timestamped entity plus full CRUD stack modeled on tick domain
- [ ] Define scope and edge cases
  - **Outcome**: Basic CRUD; unique `key`; JSON `value`; pagination for list; 404 handling; optimistic update strategy for `updated_at`

### Phase 2: Implementation (File/Code Structure)
> Describe the proposed file/directory structure, including the purpose of each key component. Remember use status markers like ✅ (Implemented), 🚧 (To-Do), 🔄 (In Progress).

```
packages/stock/src/main/java/com/vjcspy/stock/common/
├── domain/
│   └── configuration/
│       ├── StockCommonConfigurationEntity.java   # 🔄 IN PROGRESS - add timestamps
│       ├── StockCommonConfigurationRepository.java # 🚧 TODO - Panache repository
│       ├── StockCommonConfigurationService.java   # 🚧 TODO - Business logic CRUD
│       ├── StockCommonConfigurationMapper.java    # 🚧 TODO - MapStruct mapper (cdi)
│       └── StockCommonConfigurationDto.java       # 🚧 TODO - Lombok DTO
└── controller/
    └── ConfigurationResource.java                 # 🚧 TODO - JAX-RS CRUD endpoints

projects/http/src/main/resources/db/migration/
└── V6.0.0__StockCommonConfiguration.sql           # 🚧 TODO - create table, indexes
```

Reference patterns to mirror:
- Mapper CDI: `packages/stock/src/main/java/com/vjcspy/stock/stockinfo/domain/tick/StockInfoTickMapper.java:7`
- DTO Lombok: `packages/stock/src/main/java/com/vjcspy/stock/stockinfo/domain/tick/StockInfoTickDto.java:11`
- Panache repository: `packages/stock/src/main/java/com/vjcspy/stock/stockinfo/domain/tick/StockInfoTickRepository.java:10`
- JAX-RS resource: `packages/stock/src/main/java/com/vjcspy/stock/stockinfo/controller/TickResource.java:25-28`

### Phase 3: Detailed Implementation Steps
1. Entity updates (`StockCommonConfigurationEntity.java`)
   - Add fields:
     - `@Column(name = "created_at", nullable = false)` `Instant createdAt` with `@ColumnDefault("CURRENT_TIMESTAMP")`
     - `@Column(name = "updated_at", nullable = false)` `Instant updatedAt`
   - Ensure existing fields remain:
     - `id` as `@Id @GeneratedValue(strategy = GenerationType.IDENTITY)`
     - `@Column(name = "\"key\"", nullable = false, unique = true, columnDefinition = "text") String key`
     - `@Column(name = "value", columnDefinition = "jsonb") @JdbcTypeCode(SqlTypes.JSON) JsonNode value`
   - Rationale: mirror timestamp approach used in `StockTradingAnalysisEntity` (`packages/stock/src/main/java/com/vjcspy/stock/stocktrading/domain/analysis/StockTradingAnalysisEntity.java:35-40`)

2. Flyway migration (`projects/http/src/main/resources/db/migration/V6.0.0__StockCommonConfiguration.sql`)
   - Create table `stock_common_configuration`:
     - `id BIGSERIAL PRIMARY KEY`
     - `"key" TEXT NOT NULL UNIQUE`
     - `value JSONB NULL`
     - `created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP`
     - `updated_at TIMESTAMPTZ NOT NULL`
   - Indexes:
     - `CREATE INDEX IF NOT EXISTS idx_stock_common_configuration_key ON stock_common_configuration ("key");`
   - Note: handle `updated_at` in service on updates; DB trigger optional for future improvement

3. DTO (`StockCommonConfigurationDto.java`)
   - Lombok annotations: `@Data @NoArgsConstructor @AllArgsConstructor`
   - Fields: `Long id`, `String key`, `JsonNode value`, `Instant createdAt`, `Instant updatedAt`

4. Mapper (`StockCommonConfigurationMapper.java`)
   - `@Mapper(componentModel = "cdi")`
   - Methods:
     - `StockCommonConfigurationDto toDto(StockCommonConfigurationEntity entity)`
     - `@Mapping(target = "id", ignore = true)`
       `@Mapping(target = "createdAt", ignore = true)`
       `@Mapping(target = "updatedAt", ignore = true)`
       `StockCommonConfigurationEntity toEntity(StockCommonConfigurationDto dto)`
   - Mirrors `StockInfoTickMapper` style (`packages/stock/src/main/java/com/vjcspy/stock/stockinfo/domain/tick/StockInfoTickMapper.java:7-13`)

5. Repository (`StockCommonConfigurationRepository.java`)
   - `@ApplicationScoped` and implements `PanacheRepository<StockCommonConfigurationEntity>`
   - CRUD helpers:
     - `findByKey(String key)`
     - `insert(String key, JsonNode value)`
     - `updateValueByKey(String key, JsonNode value)` (do not modify `created_at`)
     - `deleteByKey(String key)`

6. Service (`StockCommonConfigurationService.java`)
   - `@ApplicationScoped`; inject repository
   - Use `VirtualUni` pattern where appropriate for transactions, consistent with tick service
   - Methods:
     - `list(int page, int size)` returns `List<StockCommonConfigurationEntity>`
     - `get(String key)` returns single entity or `null`
     - `create(String key, JsonNode value)` sets `createdAt=now`, `updatedAt=now`
     - `update(String key, JsonNode value)` sets `updatedAt=now`
     - `delete(String key)`
   - Logging style: use `kv(...)` helper similar to `TickResource` (`packages/stock/src/main/java/com/vjcspy/stock/stockinfo/controller/TickResource.java:47`)

7. Controller (`ConfigurationResource.java`)
   - JAX-RS (`jakarta.ws.rs`) resource:
     - `@Path("/configurations")`
     - `@Consumes(MediaType.APPLICATION_JSON)` / `@Produces(MediaType.APPLICATION_JSON)`
   - Endpoints (return `ApiResponse` wrapped payloads):
     - `GET /configurations` → list with pagination (`page`, `size`), map to DTOs via mapper
     - `GET /configurations/{key}` → single by key, 404 returns domain error object
     - `POST /configurations` → create from DTO (`key`, `value`), set timestamps
     - `PUT /configurations/{key}` → update `value`, refresh `updated_at`
     - `DELETE /configurations/{key}` → delete
   - Follow `TickResource` patterns for Mutiny `Uni` chains and messages (`packages/stock/src/main/java/com/vjcspy/stock/stockinfo/controller/TickResource.java:43-55`)

8. Validation & errors
   - Enforce unique `key` at DB and check conflict in service; respond with error payload
   - 404 handling mirrors `StockInfoTickError.notFoundError(...)` usage style

9. Configuration
   - No additional properties required; Flyway already enabled (`projects/http/src/main/resources/application.properties:15-23`)

## 📊 Summary of Results

### ✅ Completed Achievements
- Planning document outlines entity timestamps, migration, full stack components, and REST endpoints aligned with existing patterns

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] Decide whether to manage `updated_at` via DB trigger vs service; current plan uses service

### 🔮 Future Improvements (Optional)
- [ ] Add audit trail (who changed configuration and when)
- [ ] Add caching layer for reads to reduce DB hits