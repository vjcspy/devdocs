# 📋 [251223] - Seed Script Execution Table with 1M Records

## References

- **Global Standard**: `devdocs/tinybots/OVERVIEW.md`
- **Repo Standard**: Not available (devtools OVERVIEW.md does not exist yet)
- **Related Documentation**:
  - `devdocs/tinybots/micro-manager/251206-store-trigger-script.md` - Trigger-based script execution implementation
  - `typ-e/src/main/resources/db/migration/V97__add_triggered_script_execution_support.sql` - Schema migration
- **Key Files**:
  - `devtools/src/seed/units/RobotAccountSeed.ts` - Example seed implementation
  - `devtools/src/seed/core/AbstractSeed.ts` - Base seeding framework
  - `devtools/src/seed/core/SeedContext.ts` - Seeding context with Prisma clients
  - `devtools/prisma/tinybots/schema.prisma` - Database schema (script_execution model)
  - `devtools/src/config/db.ts` - Database connection configuration

## User Requirements

**Goal:** Simulate the migration `V97__add_triggered_script_execution_support.sql` by creating seed data for the `script_execution` table in the local development environment.

**Requirements:**
1. Seed **1 million records** into the `script_execution` table
2. Leverage the existing devtools seeding framework pattern (AbstractSeed)
3. Simulate both **scheduled** and **triggered** executions based on the new schema:
   - Scheduled executions: `schedule_id NOT NULL`, `triggering_event_id NULL`
   - Triggered executions: `schedule_id NULL`, `triggering_event_id NOT NULL`
4. Follow the constraint: `(schedule_id IS NOT NULL AND triggering_event_id IS NULL) OR (schedule_id IS NULL AND triggering_event_id IS NOT NULL)`

**Data Distribution:**
- 50% scheduled executions (500k records)
- 50% triggered executions (500k records)

## 🎯 Objective

Create a performant seeding utility within the devtools repository that generates 1 million `script_execution` records efficiently, simulating both scheduled and trigger-based script executions as implemented in the micro-manager service.

### ⚠️ Key Considerations

1. **Performance at Scale**: Inserting 1M records requires batch processing. Prisma's `createMany()` should be used with appropriate batch sizes (e.g., 10,000 records per batch).

2. **Foreign Key Dependencies**: The `script_execution` table has foreign keys to:
   - `script_reference` (required)
   - `script_version` (required)
   - `task_schedule` (optional - for scheduled executions)
   - `event_trigger` (optional - for triggered executions via new column `triggering_event_id`)

3. **Schema Compatibility**: The Prisma schema at `devtools/prisma/tinybots/schema.prisma` may not yet reflect the V97 migration changes (nullable `schedule_id`, new `triggering_event_id` column). We must either:
   - Re-generate Prisma schema from the migrated database, OR
   - Use raw SQL queries via `$executeRaw` for maximum control

4. **Database Constraints**: The check constraint `chk_execution_source` ensures exactly one of `schedule_id` or `triggering_event_id` is NOT NULL.

5. **Realistic Test Data**: Generated records should have:
   - Valid `script_reference_id` and `script_version_id` from existing seed data
   - Realistic timestamps (`planned` for scheduled, `created_at` for all)
   - Valid foreign key references to `task_schedule` and `event_trigger` tables

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] **Verify Prisma Schema State**
  - Check if `devtools/prisma/tinybots/schema.prisma` reflects V97 migration
  - If not updated, run `npm run db:pull:tinybots` to pull latest schema
  - Verify `script_execution` model has nullable `schedule_id`, `planned`, and new `triggering_event_id` field
  - **Outcome**: Prisma schema matches production database structure

- [ ] **Identify Prerequisite Seed Data**
  - Determine required seed units for foreign key dependencies:
    - `script_reference` records (need at least 1)
    - `script_version` records (need at least 1)
    - `task_schedule` records (need multiple for variety in scheduled executions)
    - `event_trigger` records (need multiple for variety in triggered executions)
  - Check if these seed units exist; create if missing
  - **Outcome**: List of dependency seed units and their creation strategy

- [ ] **Determine Batch Strategy**
  - Research Prisma `createMany()` batch size limits and performance characteristics
  - Decide batch size (recommended: 5,000 - 10,000 records per batch)
  - Calculate total batches needed: 1,000,000 / batch_size
  - **Outcome**: Batch configuration parameters

- [ ] **Design Data Generation Strategy**
  - Define timestamp ranges for `planned` and `created_at` fields
  - Design alternating pattern for scheduled vs. triggered executions
  - Plan randomization for `script_reference_id`, `script_version_id`, etc.
  - **Outcome**: Data generation algorithm pseudocode

### Phase 2: Implementation (File/Code Structure)

**File Structure:**
```
devtools/src/seed/
├── units/
│   ├── ScriptReferenceSeed.ts         # 🚧 TODO - Seed script_reference (if not exists)
│   ├── ScriptVersionSeed.ts           # 🚧 TODO - Seed script_version (if not exists)
│   ├── TaskScheduleSeed.ts            # 🚧 TODO - Seed task_schedule (if not exists)
│   ├── EventTriggerSeed.ts            # 🚧 TODO - Seed event_trigger (if not exists)
│   ├── ScriptExecutionSeed.ts         # 🚧 TODO - Main seed unit for 1M executions
│   └── RobotAccountSeed.ts            # ✅ EXISTS - Reference pattern
├── core/
│   ├── AbstractSeed.ts                # ✅ EXISTS - Base class
│   └── SeedContext.ts                 # ✅ EXISTS - Context with Prisma clients
└── run.ts                             # 🔄 TO UPDATE - Register new seed units
```

**Key Components:**

1. **ScriptExecutionSeed.ts**: Primary implementation
   - Extends `AbstractSeed<ScriptExecutionRefs>`
   - Accepts dependencies: script references, versions, schedules, event triggers
   - Implements `seed()` method with batch processing logic
   - Implements `clean()` method for cleanup

2. **Prerequisite Seed Units**: Create minimal viable seeds for:
   - `ScriptReferenceSeed` - creates 10 script references
   - `ScriptVersionSeed` - creates 10 script versions (linked to references)
   - `TaskScheduleSeed` - creates 100 task schedules (for variety)
   - `EventTriggerSeed` - creates 100 event triggers (for variety)

3. **Updated run.ts**: Register new seed units in dependency order

### Phase 3: Detailed Implementation Steps

#### Step 1: Update Prisma Schema (if needed)
```bash
# Pull latest database schema to reflect V97 migration
cd devtools
npm run db:pull:tinybots
npm run generate:tinybots
```

**Verification**: Check that `script_execution` model in `prisma/tinybots/schema.prisma` shows:
- `schedule_id BigInt? @db.UnsignedBigInt` (nullable)
- `planned DateTime? @db.DateTime(0)` (nullable)
- `triggering_event_id BigInt? @db.UnsignedBigInt` (nullable)

#### Step 2: Create Prerequisite Seed Units

**2.1 ScriptReferenceSeed.ts**
```typescript
import { AbstractSeed } from '../core/AbstractSeed'
import type { SeedContext } from '../core/SeedContext'

export type ScriptReferenceRefs = { 
  scriptReferenceIds: bigint[] 
}

export class ScriptReferenceSeed extends AbstractSeed<ScriptReferenceRefs> {
  constructor() {
    super('script_reference')
  }

  async seed(ctx: SeedContext): Promise<ScriptReferenceRefs> {
    const count = 10
    const ids: bigint[] = []
    
    for (let i = 1; i <= count; i++) {
      const ref = await ctx.tinybots.script_reference.upsert({
        where: { id: BigInt(i) },
        create: {
          id: BigInt(i),
          name: `seed-script-${i}`,
          description: `Seed script reference ${i}`,
          created_at: new Date(),
        },
        update: {},
      })
      ids.push(ref.id)
    }
    
    ctx.log(`✅ Seeded ${count} script_reference records`)
    return { scriptReferenceIds: ids }
  }

  async clean(ctx: SeedContext): Promise<void> {
    await ctx.tinybots.script_reference.deleteMany({
      where: { name: { startsWith: 'seed-script-' } }
    })
  }
}
```

**2.2 ScriptVersionSeed.ts**
```typescript
import { AbstractSeed } from '../core/AbstractSeed'
import type { SeedContext } from '../core/SeedContext'
import type { ScriptReferenceRefs } from './ScriptReferenceSeed'

export type ScriptVersionRefs = { 
  scriptVersionIds: bigint[] 
}

export class ScriptVersionSeed extends AbstractSeed<ScriptVersionRefs> {
  constructor(private refs: ScriptReferenceRefs) {
    super('script_version')
  }

  async seed(ctx: SeedContext): Promise<ScriptVersionRefs> {
    const ids: bigint[] = []
    
    // Create 1 version per reference
    for (const refId of this.refs.scriptReferenceIds) {
      const version = await ctx.tinybots.script_version.create({
        data: {
          script_reference_id: refId,
          version: 1,
          json: JSON.stringify({ steps: [] }),
          created_at: new Date(),
        },
      })
      ids.push(version.id)
    }
    
    ctx.log(`✅ Seeded ${ids.length} script_version records`)
    return { scriptVersionIds: ids }
  }

  async clean(ctx: SeedContext): Promise<void> {
    await ctx.tinybots.script_version.deleteMany({
      where: { 
        script_reference_id: { 
          in: this.refs.scriptReferenceIds.map(id => id) 
        } 
      }
    })
  }
}
```

**2.3 TaskScheduleSeed.ts**
```typescript
import { AbstractSeed } from '../core/AbstractSeed'
import type { SeedContext } from '../core/SeedContext'

export type TaskScheduleRefs = { 
  scheduleIds: bigint[] 
}

export class TaskScheduleSeed extends AbstractSeed<TaskScheduleRefs> {
  constructor() {
    super('task_schedule')
  }

  async seed(ctx: SeedContext): Promise<TaskScheduleRefs> {
    const count = 100
    const ids: bigint[] = []
    const now = new Date()
    
    for (let i = 1; i <= count; i++) {
      const schedule = await ctx.tinybots.task_schedule.create({
        data: {
          schedule_id: BigInt(i),
          robot_id: 1, // Assumes RobotAccountSeed has created robot_id = 1
          cron: '0 9 * * *',
          next_fire_time: new Date(now.getTime() + i * 3600000),
          status: 'ACTIVE',
          created_at: now,
        },
      })
      ids.push(schedule.id)
    }
    
    ctx.log(`✅ Seeded ${count} task_schedule records`)
    return { scheduleIds: ids }
  }

  async clean(ctx: SeedContext): Promise<void> {
    await ctx.tinybots.task_schedule.deleteMany({
      where: { schedule_id: { gte: BigInt(1), lte: BigInt(100) } }
    })
  }
}
```

**2.4 EventTriggerSeed.ts**
```typescript
import { AbstractSeed } from '../core/AbstractSeed'
import type { SeedContext } from '../core/SeedContext'

export type EventTriggerRefs = { 
  eventTriggerIds: bigint[] 
}

export class EventTriggerSeed extends AbstractSeed<EventTriggerRefs> {
  constructor() {
    super('event_trigger')
  }

  async seed(ctx: SeedContext): Promise<EventTriggerRefs> {
    const count = 100
    const ids: bigint[] = []
    
    for (let i = 1; i <= count; i++) {
      const trigger = await ctx.tinybots.event_trigger.create({
        data: {
          outgoing_event_id: BigInt(i), // Simplified - may need proper FK
          script_reference_id: BigInt(1),
          created_at: new Date(),
        },
      })
      ids.push(trigger.id)
    }
    
    ctx.log(`✅ Seeded ${count} event_trigger records`)
    return { eventTriggerIds: ids }
  }

  async clean(ctx: SeedContext): Promise<void> {
    await ctx.tinybots.event_trigger.deleteMany({
      where: { id: { gte: BigInt(1), lte: BigInt(100) } }
    })
  }
}
```

#### Step 3: Implement ScriptExecutionSeed.ts

**Full Implementation:**
```typescript
import { AbstractSeed } from '../core/AbstractSeed'
import type { SeedContext } from '../core/SeedContext'
import type { ScriptReferenceRefs } from './ScriptReferenceSeed'
import type { ScriptVersionRefs } from './ScriptVersionSeed'
import type { TaskScheduleRefs } from './TaskScheduleSeed'
import type { EventTriggerRefs } from './EventTriggerSeed'

export type ScriptExecutionRefs = { 
  totalSeeded: number 
}

export class ScriptExecutionSeed extends AbstractSeed<ScriptExecutionRefs> {
  private readonly TARGET_COUNT = 1_000_000
  private readonly BATCH_SIZE = 10_000
  
  constructor(
    private scriptRefs: ScriptReferenceRefs,
    private versionRefs: ScriptVersionRefs,
    private scheduleRefs: TaskScheduleRefs,
    private triggerRefs: EventTriggerRefs,
  ) {
    super('script_execution')
  }

  async seed(ctx: SeedContext): Promise<ScriptExecutionRefs> {
    const totalBatches = Math.ceil(this.TARGET_COUNT / this.BATCH_SIZE)
    const baseTime = new Date('2024-01-01T00:00:00Z').getTime()
    
    ctx.log(`🚀 Starting seed of ${this.TARGET_COUNT.toLocaleString()} script_execution records...`)
    ctx.log(`📦 Processing ${totalBatches} batches of ${this.BATCH_SIZE} records each`)
    
    let totalSeeded = 0
    
    for (let batchNum = 0; batchNum < totalBatches; batchNum++) {
      const batchData = []
      const batchStart = batchNum * this.BATCH_SIZE
      const batchEnd = Math.min(batchStart + this.BATCH_SIZE, this.TARGET_COUNT)
      const currentBatchSize = batchEnd - batchStart
      
      for (let i = 0; i < currentBatchSize; i++) {
        const globalIndex = batchStart + i
        const isScheduled = globalIndex % 2 === 0 // 50/50 split
        
        // Random selection from available IDs
        const scriptRefId = this.scriptRefs.scriptReferenceIds[
          globalIndex % this.scriptRefs.scriptReferenceIds.length
        ]
        const scriptVersionId = this.versionRefs.scriptVersionIds[
          globalIndex % this.versionRefs.scriptVersionIds.length
        ]
        
        // Generate timestamp (spread across 1 year)
        const timestamp = new Date(baseTime + (globalIndex * 31536)) // ~1 record per 31 seconds over a year
        
        if (isScheduled) {
          // Scheduled execution
          const scheduleId = this.scheduleRefs.scheduleIds[
            (globalIndex / 2) % this.scheduleRefs.scheduleIds.length
          ]
          
          batchData.push({
            script_reference_id: scriptRefId,
            script_version_id: scriptVersionId,
            schedule_id: scheduleId,
            planned: timestamp,
            triggering_event_id: null,
            created_at: new Date(timestamp.getTime() + 1000), // created 1s after planned
          })
        } else {
          // Triggered execution
          const triggerId = this.triggerRefs.eventTriggerIds[
            Math.floor(globalIndex / 2) % this.triggerRefs.eventTriggerIds.length
          ]
          
          batchData.push({
            script_reference_id: scriptRefId,
            script_version_id: scriptVersionId,
            schedule_id: null,
            planned: null,
            triggering_event_id: triggerId,
            created_at: timestamp,
          })
        }
      }
      
      // Insert batch using Prisma createMany
      await ctx.tinybots.script_execution.createMany({
        data: batchData,
        skipDuplicates: true,
      })
      
      totalSeeded += currentBatchSize
      
      // Progress logging every 10 batches
      if ((batchNum + 1) % 10 === 0 || batchNum === totalBatches - 1) {
        const progress = ((totalSeeded / this.TARGET_COUNT) * 100).toFixed(1)
        ctx.log(`📊 Progress: ${totalSeeded.toLocaleString()} / ${this.TARGET_COUNT.toLocaleString()} (${progress}%)`)
      }
    }
    
    ctx.log(`✅ Successfully seeded ${totalSeeded.toLocaleString()} script_execution records`)
    
    // Verify counts
    const scheduledCount = await ctx.tinybots.script_execution.count({
      where: { schedule_id: { not: null } }
    })
    const triggeredCount = await ctx.tinybots.script_execution.count({
      where: { triggering_event_id: { not: null } }
    })
    
    ctx.log(`📈 Verification:`)
    ctx.log(`   - Scheduled executions: ${scheduledCount.toLocaleString()}`)
    ctx.log(`   - Triggered executions: ${triggeredCount.toLocaleString()}`)
    ctx.log(`   - Total: ${(scheduledCount + triggeredCount).toLocaleString()}`)
    
    return { totalSeeded }
  }

  async clean(ctx: SeedContext): Promise<void> {
    ctx.log(`🧹 Cleaning script_execution seed data...`)
    
    const deleteResult = await ctx.tinybots.script_execution.deleteMany({
      where: {
        OR: [
          { schedule_id: { in: this.scheduleRefs.scheduleIds } },
          { triggering_event_id: { in: this.triggerRefs.eventTriggerIds } },
        ]
      }
    })
    
    ctx.log(`🗑️ Deleted ${deleteResult.count.toLocaleString()} script_execution records`)
  }
}
```

#### Step 4: Update run.ts to Register Seed Units

**Modify devtools/src/seed/run.ts:**
```typescript
// ... existing imports ...
import { ScriptReferenceSeed } from './units/ScriptReferenceSeed'
import { ScriptVersionSeed } from './units/ScriptVersionSeed'
import { TaskScheduleSeed } from './units/TaskScheduleSeed'
import { EventTriggerSeed } from './units/EventTriggerSeed'
import { ScriptExecutionSeed } from './units/ScriptExecutionSeed'

async function main() {
  const ctx = await createContext('dev')
  
  try {
    // ... existing seeds ...
    
    // Script execution seeding with dependencies
    const scriptRefSeed = new ScriptReferenceSeed()
    const scriptRefRefs = await scriptRefSeed.seed(ctx)
    
    const scriptVersionSeed = new ScriptVersionSeed(scriptRefRefs)
    const scriptVersionRefs = await scriptVersionSeed.seed(ctx)
    
    const taskScheduleSeed = new TaskScheduleSeed()
    const taskScheduleRefs = await taskScheduleSeed.seed(ctx)
    
    const eventTriggerSeed = new EventTriggerSeed()
    const eventTriggerRefs = await eventTriggerSeed.seed(ctx)
    
    const scriptExecutionSeed = new ScriptExecutionSeed(
      scriptRefRefs,
      scriptVersionRefs,
      taskScheduleRefs,
      eventTriggerRefs
    )
    const scriptExecutionRefs = await scriptExecutionSeed.seed(ctx)
    
    ctx.log('✨ All seeds completed successfully')
  } catch (error) {
    ctx.log('❌ Seeding failed:', error)
    throw error
  } finally {
    await destroyContext()
  }
}

main()
```

#### Step 5: Execution & Verification

**Run the seed:**
```bash
cd devtools
npm run seed
```

**Expected output:**
```
🚀 Starting seed of 1,000,000 script_execution records...
📦 Processing 100 batches of 10,000 records each
📊 Progress: 100,000 / 1,000,000 (10.0%)
📊 Progress: 200,000 / 1,000,000 (20.0%)
...
📊 Progress: 1,000,000 / 1,000,000 (100.0%)
✅ Successfully seeded 1,000,000 script_execution records
📈 Verification:
   - Scheduled executions: 500,000
   - Triggered executions: 500,000
   - Total: 1,000,000
```

**Verify in database:**
```sql
-- Check total count
SELECT COUNT(*) FROM script_execution;

-- Check scheduled vs triggered split
SELECT 
  CASE 
    WHEN schedule_id IS NOT NULL THEN 'scheduled'
    WHEN triggering_event_id IS NOT NULL THEN 'triggered'
  END AS execution_type,
  COUNT(*) AS count
FROM script_execution
GROUP BY execution_type;

-- Verify constraint (should return 0 rows with violations)
SELECT * FROM script_execution
WHERE NOT (
  (schedule_id IS NOT NULL AND triggering_event_id IS NULL) OR
  (schedule_id IS NULL AND triggering_event_id IS NOT NULL)
);
```

**Clean up if needed:**
```bash
cd devtools
npm run seed:clean
```

### Phase 4: Optimization & Performance Tuning

- [ ] **Measure Performance**
  - Time the full 1M record seed operation
  - Monitor memory usage during execution
  - **Outcome**: Baseline performance metrics

- [ ] **Optimize Batch Size** (if needed)
  - If performance is poor, experiment with batch sizes: 5k, 10k, 20k
  - Consider using raw SQL for maximum performance
  - **Alternative approach**: Use `$executeRawUnsafe` with bulk INSERT
  - **Outcome**: Optimal batch size configuration

- [ ] **Add Progress Indicators**
  - Implement percentage completion logging
  - Add estimated time remaining
  - **Outcome**: Better visibility during long-running seed operations

## 📊 Summary of Results
> (To be filled after implementation)

### ✅ Completed Achievements
- TBD

### 📈 Performance Metrics
- TBD

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

- [ ] **Prisma Schema Update**: The Prisma schema may not reflect V97 migration. Need to verify and run `db:pull` if necessary.

- [ ] **Foreign Key Data**: The prerequisite seed units (`task_schedule`, `event_trigger`) may themselves have foreign key dependencies that need to be seeded first:
  - `task_schedule` requires `robot_id` (from `robot_account`)
  - `event_trigger` requires `outgoing_event_id` (from `outgoing_event`)
  - We may need to create additional seed units or use placeholder data

- [ ] **Performance Considerations**: For 1M records, Prisma `createMany()` may be slower than raw SQL. If performance is critical, consider:
  ```typescript
  // Alternative: Raw SQL bulk insert
  await ctx.tinybots.$executeRawUnsafe(`
    INSERT INTO script_execution 
      (script_reference_id, script_version_id, schedule_id, planned, triggering_event_id, created_at)
    VALUES ${batchData.map(row => `(${row.script_reference_id}, ${row.script_version_id}, ...)`).join(',')}
  `)
  ```

- [ ] **Disk Space**: 1M records will consume significant disk space. Estimate: ~100MB depending on index overhead. Ensure sufficient disk space in local dev environment.

- [ ] **Index Performance**: After seeding, database query performance may require indexes to be analyzed/rebuilt:
  ```sql
  ANALYZE TABLE script_execution;
  ```

### 🔄 Next Steps After Implementation

1. Verify the seed completes successfully with 1M records
2. Run queries against seeded data to test micro-manager API endpoints
3. Measure query performance with realistic dataset size
4. Document any findings or schema improvements needed
5. Consider adding seed variants for testing edge cases (e.g., very old executions, specific time ranges)
