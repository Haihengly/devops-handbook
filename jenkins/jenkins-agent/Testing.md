# Jenkins Agent — POC Testing

> **Purpose:** Evidence-based recommendation for CI/CD agent strategy

> **Topic:** Static Agent vs Docker Agent (Per Job & Per Stage)

---

## 1. Test Environment

| Component | Static Agent VM | Docker Agent VM 01 | Docker Agent VM 02 |
|---|---|---|---|---|
| VM Specs | Same across all VMs | Same across all VMs | Same across all VMs |
| Executors | 2 executors | 2 executors | 2 executors |
| Agent Type | Static (agent any) | Docker - Per Job | Docker - Per Stage |

---

## 2. Test Results

### Test 1 — Single Small Job: Static Agent vs Docker Agent

| Agent Type | Checkout | Build Image | CPU Load Stage | Total Time |
|---|---|---|---|---|
| Static Agent | ~0.7 sec | ~0.5 sec | ~1 min 2 sec | ~1 min 3 sec |
| Docker Agent | ~2.5 sec | ~2.1 sec | ~28 sec | **~35 sec ✅ Winner** |

> *Docker Agent is ~2x faster on single job. Static Agent CPU stage is slower due to shared resource contention on the VM.*

---

### Test 2 — Big 3 Jobs Simultaneously: Static Agent vs Docker Agent

| Agent Type | Job 1 & 2 Finish | Job 3 Finish | Job 3 Wait? | Total Time |
|---|---|---|---|---|
| Static Agent | ~20 min | ~40 min | Yes ⏳ waits for slot | ~40 min |
| Docker Agent | ~10 min | ~20 min | No ✅ starts immediately | **~20 min ✅ Winner** |

> *Docker Agent completes all 3 jobs in half the time. No queuing with Docker Agent since each job gets its own container.*

---

### Test 3 — Per Job vs Per Stage (2 Executors, 4 Jobs, Big Job, 4 Stages)

| Agent Type | Job 1 & 2 Finish | Job 3 & 4 Finish | Complexity | Total Time | Winner |
|---|---|---|---|---|---|
| Docker - Per Job | ~20 min | ~40 min | Simple ✅ | ~40 min | **✅ Per Job** |
| Docker - Per Stage | ~30 min | ~41 min | Complex ⚠️ | ~41 min | ❌ |

> *Queue (2) = Executor (2) → no free executor to sneak in between stages → Per Job wins.*

---

### Test 4 — Per Job vs Per Stage (2 Executors, 6 Jobs, Medium Job, 4 Stages)

| Agent Type | Job 1 & 2 Finish | Job 3 Finish | Complexity | Total Time | Winner |
|---|---|---|---|---|---|
| Docker - Per Job | ~5 min 30 sec | ~10 min | Simple ✅ | ~10 min | ❌ |
| Docker - Per Stage | ~8 min 30 sec | ~5 min 40 sec | Complex ⚠️ | ~8 min 30 sec | **✅ Per Stage** |

> *Queue (1) < Executor (2) → always free executor to sneak in between stages → Per Stage wins. But complexity is significantly higher.*

---

### Test 5 — Per Job vs Per Stage (2 Executors, 6 Jobs, Medium Job, 3 Stages)

| Agent Type | Job 1 & 2 Finish | Job 3 Finish | Complexity | Total Time | Winner |
|---|---|---|---|---|---|
| Docker - Per Job | ~5 min 30 sec | ~10 min | Simple ✅ | ~10 min | **✅ Per Job** |
| Docker - Per Stage | ~5 min 30 sec | ~10 min | Complex ⚠️ | ~10 min | ❌ Almost same |

> *Fewer stages = fewer chances to sneak in → Per Stage loses its advantage → Per Job wins again.*

---

## 3. Key Finding — When Per Job vs Per Stage Wins

| Condition | Queue vs Executor | Winner | Reason |
|---|---|---|---|
| Jobs fill all executors evenly | Queue = Executor | ✅ Per Job | No free executor to sneak in between stages |
| Jobs less than executors | Queue < Executor | ✅ Per Stage | Always free executor → jobs always sneak in between stages |
| Jobs more than executors | Queue > Executor | ⚠️ Depends | Depends on stage count and stage duration |
| Few stages regardless of queue | Any | ✅ Per Job | Fewer stages = fewer sneak-in chances = Per Stage overhead not worth it |

---

## 4. Overall Summary

| Metric | Static Agent | Docker - Per Job | Docker - Per Stage | Winner |
|---|---|---|---|---|
| Single Job Speed | ~1 min 3 sec | ~35 sec | ~35 sec + overhead | ✅ Docker Per Job |
| Parallel Jobs | Limited ❌ | Good ✅ | Better (odd jobs) | ✅ Docker Per Job (even) / Per Stage (odd & queue < executor) |
| Environment Isolation | ❌ No | ✅ Per job | ✅ Per stage | ✅ Docker Agent (both) |
| Environment Bleed | ❌ Risk | ✅ No bleed | ✅ No bleed | ✅ Docker Agent (both) |
| Configuration Complexity | Simple | Simple ✅ | Complex ⚠️ | ✅ Docker Per Job |
| Maintenance | Low | Low ✅ | High ⚠️ | ✅ Docker Per Job |

---

### When to Switch to Per Stage in the Future

| Condition | Action |
|---|---|
| Jobs consistently < Executors | Consider switching to Per Stage — free executors will always be available to sneak in |
| Pipeline grows to 6+ long stages | Per Stage becomes more beneficial — more chances to free executor between stages |
| Queue consistently > Executor | First increase executors or add more VMs before switching to Per Stage |

---

## Conclusion

Based on POC testing across multiple scenarios — **Docker Agent Per Job** is the recommended approach for environment.

It provides:
- ✅ Clean isolated builds per job
- ✅ No environment bleed between jobs
- ✅ Better performance than Static Agent (~2x faster)
- ✅ Simple configuration and low maintenance
- ✅ Scales well for expected load of 5-6 simultaneous jobs

The decision to switch to **Per Stage** in the future should be based on real workload data and the **Queue vs Executor ratio.**