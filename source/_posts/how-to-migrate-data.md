---
title: 生产环境数据迁移实战指南：从策略设计到落地实践
date: 2022-10-23 18:25:12
tags: [数据库迁移, 高可用, 运维, MySQL, Redis]
categories: [系统架构, 数据库]
---

## 数据迁移场景与挑战

在生产环境中，数据迁移是一项高风险、高复杂度的运维操作。常见的迁移驱动因素包括：

### 迁移驱动因素
- **容量扩展**：业务快速增长导致存储容量不足，需要扩容或分库分表
- **性能优化**：单机性能瓶颈，需要迁移到更高配置的硬件或集群
- **成本优化**：降本增效，迁移到成本更低的存储方案
- **技术升级**：数据库版本升级、存储引擎切换
- **合规要求**：数据本地化、异地容灾等合规性需求

### 核心挑战
- **数据一致性**：确保迁移过程中数据不丢失、不重复
- **服务可用性**：最小化业务中断时间
- **性能影响**：避免迁移过程对线上服务造成性能冲击
- **回滚能力**：具备快速回滚机制应对异常情况

## 迁移策略选择

### 停机迁移
**适用场景**：
- 金融、支付等强一致性要求的核心业务
- 数据量相对较小，可接受短时间停机
- 迁移窗口期有明确的业务停机时间

**优势**：
- 实施简单，风险可控
- 数据一致性有保障
- 操作流程清晰

**劣势**：
- 业务中断时间较长
- 用户体验受影响

### 不停机迁移
**适用场景**：
- 7x24小时服务的互联网业务
- 大数据量迁移场景
- 对可用性要求极高的系统

**优势**：
- 业务无感知
- 可分阶段执行，风险分散
- 具备实时回滚能力

**劣势**：
- 技术复杂度高
- 需要完善的数据校验机制
- 迁移周期相对较长

## 迁移时机选择

### 前置条件验证
- **测试环境验证**：在与生产环境相同的数据规模下完成完整迁移演练
- **监控告警完备**：确保迁移过程中的关键指标可观测
- **回滚预案就绪**：制定详细的回滚策略和操作手册

### 最佳执行时间窗口
- **业务低峰期**：通常选择凌晨2-6点，此时QPS相对较低
- **非关键业务时段**：避开营销活动、结算等关键业务时间点
- **充足的处理时间**：确保有足够时间处理异常情况

## 不停机迁移实施方案

### 五阶段迁移策略

#### 第一阶段：初始状态
```
应用 ----写----> 旧库
     ----读----> 旧库
```
- **目标**：建立基线，确保系统稳定运行
- **关键指标**：记录迁移前的性能基线数据

#### 第二阶段：全量数据同步
```
应用 ----写----> 旧库
     ----读----> 旧库

旧库 ----全量同步----> 新库
```
- **MySQL迁移工具**：
  - `mysqldump`：适合中小型数据库（< 100GB）
  - `XtraBackup`：适合大型数据库，支持热备份
- **Redis迁移工具**：
  - `redis-shake`：支持全量+增量同步
  - `redis-port`：阿里云开源的Redis迁移工具

**全量同步关键配置**：
```bash
# MySQL XtraBackup示例
xtrabackup --backup --target-dir=/backup/full --datadir=/var/lib/mysql \
           --parallel=4 --compress --compress-threads=4

# Redis redis-shake示例
redis-shake -type=sync -source=127.0.0.1:6379 -target=127.0.0.1:6380 \
           -source.password_raw=xxx -target.password_raw=xxx
```

#### 第三阶段：双写旧读
```
应用 ----写----> 旧库 ----增量同步----> 新库
     ----写----> 新库
     ----读----> 旧库
```
- **实现方式**：业务代码修改，增加双写逻辑
- **核心要点**：
  - 优先写旧库，保证主路径稳定
  - 新库写入失败不影响旧库事务
  - 记录写入差异用于后续校验

**基于配置中心的动态双写实现**：

```go
// 迁移阶段枚举
type MigrationPhase int

const (
    PhaseReadOldWriteOld MigrationPhase = iota + 1  // 阶段1：读旧写旧
    PhaseReadOldWriteOldNew                         // 阶段3：先写旧再写新，读旧
    PhaseReadNewWriteNewOld                         // 阶段4：先写新再写旧，读新
    PhaseReadNewWriteNew                            // 阶段5：读新写新
)

// 配置中心接口
type ConfigCenter interface {
    GetMigrationPhase(service string) MigrationPhase
    GetWriteStrategy(service string) WriteStrategy
}

// 写策略配置
type WriteStrategy struct {
    PrimaryDB      string        // 主库标识：old/new
    SecondaryDB    string        // 副库标识：old/new  
    IsAsync        bool          // 是否异步写副库
    FailureAction  string        // 副库写失败处理：log/queue/ignore
    TimeoutMs      int           // 写入超时时间
}

// 数据访问层
type DataAccessLayer struct {
    oldDB        *sql.DB
    newDB        *sql.DB
    configCenter ConfigCenter
    serviceName  string
    logger       Logger
    failureQueue Queue
}

func (dal *DataAccessLayer) WriteData(data *Data) error {
    phase := dal.configCenter.GetMigrationPhase(dal.serviceName)
    strategy := dal.configCenter.GetWriteStrategy(dal.serviceName)
    
    switch phase {
    case PhaseReadOldWriteOld:
        return dal.writeToOldOnly(data)
        
    case PhaseReadOldWriteOldNew:
        return dal.writeOldThenNew(data, strategy)
        
    case PhaseReadNewWriteNewOld:
        return dal.writeNewThenOld(data, strategy)
        
    case PhaseReadNewWriteNew:
        return dal.writeToNewOnly(data)
        
    default:
        return fmt.Errorf("unknown migration phase: %d", phase)
    }
}

// 阶段1：只写旧库
func (dal *DataAccessLayer) writeToOldOnly(data *Data) error {
    return dal.oldDB.Create(data)
}

// 阶段3：先写旧库，再写新库
func (dal *DataAccessLayer) writeOldThenNew(data *Data, strategy WriteStrategy) error {
    // 主路径：写旧库，必须成功
    if err := dal.oldDB.Create(data); err != nil {
        return fmt.Errorf("write to old db failed: %w", err)
    }
    
    // 副路径：写新库
    return dal.writeSecondary(dal.newDB, data, strategy, "new")
}

// 阶段4：先写新库，再写旧库  
func (dal *DataAccessLayer) writeNewThenOld(data *Data, strategy WriteStrategy) error {
    // 主路径：写新库，必须成功
    if err := dal.newDB.Create(data); err != nil {
        return fmt.Errorf("write to new db failed: %w", err)
    }
    
    // 副路径：写旧库（用于回滚保障）
    return dal.writeSecondary(dal.oldDB, data, strategy, "old")
}

// 阶段5：只写新库
func (dal *DataAccessLayer) writeToNewOnly(data *Data) error {
    return dal.newDB.Create(data)
}

// 副库写入逻辑
func (dal *DataAccessLayer) writeSecondary(db *sql.DB, data *Data, strategy WriteStrategy, dbType string) error {
    writeFunc := func() error {
        ctx, cancel := context.WithTimeout(context.Background(), 
            time.Duration(strategy.TimeoutMs)*time.Millisecond)
        defer cancel()
        
        return db.CreateWithContext(ctx, data)
    }
    
    if strategy.IsAsync {
        // 异步写入
        go func() {
            if err := writeFunc(); err != nil {
                dal.handleSecondaryWriteFailure(data, err, strategy, dbType)
            }
        }()
        return nil
    } else {
        // 同步写入
        if err := writeFunc(); err != nil {
            dal.handleSecondaryWriteFailure(data, err, strategy, dbType)
            // 根据策略决定是否返回错误
            if strategy.FailureAction == "fail" {
                return fmt.Errorf("write to %s db failed: %w", dbType, err)
            }
        }
        return nil
    }
}

// 副库写入失败处理
func (dal *DataAccessLayer) handleSecondaryWriteFailure(data *Data, err error, strategy WriteStrategy, dbType string) {
    switch strategy.FailureAction {
    case "log":
        dal.logger.Error("secondary db write failed", 
            "db_type", dbType, "error", err, "data", data)
            
    case "queue":
        dal.failureQueue.Push(&FailureRecord{
            Data:      data,
            DBType:    dbType,
            Error:     err.Error(),
            Timestamp: time.Now(),
            Retries:   0,
        })
        
    case "ignore":
        // 静默忽略
        
    default:
        dal.logger.Error("unknown failure action", "action", strategy.FailureAction)
    }
}

// 读取数据
func (dal *DataAccessLayer) ReadData(id string) (*Data, error) {
    phase := dal.configCenter.GetMigrationPhase(dal.serviceName)
    
    switch phase {
    case PhaseReadOldWriteOld, PhaseReadOldWriteOldNew:
        return dal.readFromOld(id)
        
    case PhaseReadNewWriteNewOld, PhaseReadNewWriteNew:
        return dal.readFromNew(id)
        
    default:
        return nil, fmt.Errorf("unknown migration phase: %d", phase)
    }
}

func (dal *DataAccessLayer) readFromOld(id string) (*Data, error) {
    return dal.oldDB.FindByID(id)
}

func (dal *DataAccessLayer) readFromNew(id string) (*Data, error) {
    return dal.newDB.FindByID(id)
}
```

**配置中心四阶段配置示例**：
```yaml
# Apollo/Nacos/etcd 配置示例 - 完整的四阶段配置
migration:
  # ========== 阶段1：读旧写旧（初始状态）==========
  user_service_phase1:
    phase: 1                    # 阶段1：只读写旧库
    write_strategy:
      primary_db: "old"         # 只写旧库
      secondary_db: ""          # 无副库
      is_async: false          # 不涉及双写
      failure_action: "fail"    # 写入失败直接返回错误
      timeout_ms: 3000         # 写入超时3秒
    read_strategy:
      primary_db: "old"         # 只读旧库
      fallback_db: ""          # 无降级库
    
  # ========== 阶段2：全量同步阶段（后台进行）==========
  # 此阶段应用层配置不变，由同步工具执行全量拷贝
  
  # ========== 阶段3：双写旧读（先写旧再写新，读旧）==========
  user_service_phase3:
    phase: 3                    # 阶段3：双写旧读
    write_strategy:
      primary_db: "old"         # 主库：旧库（必须成功）
      secondary_db: "new"       # 副库：新库（允许失败）
      is_async: true           # 异步写副库，降低延迟
      failure_action: "queue"   # 副库失败入队重试
      timeout_ms: 1000         # 副库写入超时1秒
      max_retries: 3           # 最大重试次数
    read_strategy:
      primary_db: "old"         # 读旧库
      fallback_db: ""          # 暂无降级
    validation:
      enabled: true            # 开启数据校验
      sample_rate: 0.1         # 10%采样校验
      diff_threshold: 0.001    # 允许0.1%数据差异
    
  # ========== 阶段4：双写新读（先写新再写旧，读新）==========  
  user_service_phase4:
    phase: 4                    # 阶段4：双写新读
    write_strategy:
      primary_db: "new"         # 主库：新库（必须成功）
      secondary_db: "old"       # 副库：旧库（保障回滚）
      is_async: false          # 同步写副库，保证强一致性
      failure_action: "log"     # 副库失败记录日志
      timeout_ms: 500          # 副库写入超时500ms
      max_retries: 1           # 最多重试1次
    read_strategy:
      primary_db: "new"         # 读新库
      fallback_db: "old"       # 降级到旧库
      fallback_threshold: 0.95 # 新库成功率<95%时降级
    validation:
      enabled: true            # 继续数据校验
      sample_rate: 0.05        # 5%采样校验
      diff_threshold: 0.0001   # 允许0.01%数据差异
    circuit_breaker:
      enabled: true            # 开启熔断器
      failure_threshold: 10    # 连续10次失败触发熔断
      timeout_ms: 30000        # 熔断30秒后尝试恢复
      
  # ========== 阶段5：读新写新（最终状态）==========
  user_service_phase5:
    phase: 5                    # 阶段5：只读写新库
    write_strategy:
      primary_db: "new"         # 只写新库
      secondary_db: ""          # 无副库
      is_async: false          # 不涉及双写
      failure_action: "fail"    # 写入失败直接返回错误
      timeout_ms: 2000         # 写入超时2秒
    read_strategy:
      primary_db: "new"         # 只读新库
      fallback_db: "old"       # 紧急情况可降级到旧库
      fallback_enabled: false  # 默认不开启降级
    cleanup:
      old_db_retention_days: 30 # 旧库保留30天
      auto_cleanup: false      # 不自动清理，人工确认
      
# ========== 不同业务服务的配置示例 ==========
  # 订单服务（高一致性要求）
  order_service:
    phase: 4
    write_strategy:
      primary_db: "new"
      secondary_db: "old"
      is_async: false          # 同步双写，确保强一致性
      failure_action: "fail"    # 副库失败也要报错
      timeout_ms: 200          # 更短的超时时间
    read_strategy:
      primary_db: "new"
      fallback_db: "old"
      fallback_threshold: 0.99 # 更高的降级阈值
      
  # 用户画像服务（可接受最终一致性）
  profile_service:
    phase: 3
    write_strategy:
      primary_db: "old"
      secondary_db: "new"
      is_async: true           # 异步双写，性能优先
      failure_action: "ignore" # 忽略副库失败
      timeout_ms: 2000         # 更宽松的超时
    read_strategy:
      primary_db: "old"
      fallback_db: ""

# ========== 全局配置 ==========
global:
  migration:
    monitoring:
      metrics_interval_seconds: 30   # 指标采集间隔
      alert_threshold:
        error_rate: 0.01             # 错误率超过1%告警
        latency_p99_ms: 1000         # P99延迟超过1秒告警
        consistency_rate: 0.999      # 一致性低于99.9%告警
    auto_promotion:
      enabled: false                 # 是否开启自动阶段推进
      check_interval_minutes: 30     # 检查间隔30分钟
      stability_duration_hours: 2    # 稳定运行2小时后允许推进
```

**迁移阶段动态控制器**：
```go
// 迁移控制器
type MigrationController struct {
    configCenter    ConfigCenter
    dataValidator   *DataValidator
    metrics        *MigrationMetrics
    logger         Logger
}

// 自动阶段推进
func (mc *MigrationController) AutoPromotePhase(serviceName string) error {
    currentPhase := mc.configCenter.GetMigrationPhase(serviceName)
    
    // 检查当前阶段是否满足推进条件
    canPromote, err := mc.checkPhasePromotionConditions(serviceName, currentPhase)
    if err != nil {
        return fmt.Errorf("check promotion conditions failed: %w", err)
    }
    
    if !canPromote {
        mc.logger.Info("phase promotion conditions not met", 
            "service", serviceName, "current_phase", currentPhase)
        return nil
    }
    
    // 推进到下一阶段
    nextPhase := currentPhase + 1
    if nextPhase > PhaseReadNewWriteNew {
        mc.logger.Info("migration completed", "service", serviceName)
        return nil
    }
    
    return mc.setMigrationPhase(serviceName, nextPhase)
}

// 检查阶段推进条件
func (mc *MigrationController) checkPhasePromotionConditions(serviceName string, phase MigrationPhase) (bool, error) {
    switch phase {
    case PhaseReadOldWriteOld:
        // 检查全量同步是否完成
        return mc.checkFullSyncCompleted(serviceName)
        
    case PhaseReadOldWriteOldNew:
        // 检查双写一致性
        return mc.checkDualWriteConsistency(serviceName)
        
    case PhaseReadNewWriteNewOld:
        // 检查新库稳定性
        return mc.checkNewDBStability(serviceName)
        
    case PhaseReadNewWriteNew:
        // 已是最终阶段
        return false, nil
        
    default:
        return false, fmt.Errorf("unknown phase: %d", phase)
    }
}

// 检查全量同步完成情况
func (mc *MigrationController) checkFullSyncCompleted(serviceName string) (bool, error) {
    // 检查数据行数是否一致
    oldCount, err := mc.getTableRowCount(serviceName, "old")
    if err != nil {
        return false, err
    }
    
    newCount, err := mc.getTableRowCount(serviceName, "new")
    if err != nil {
        return false, err
    }
    
    // 允许1%的误差（考虑到同步过程中的增量数据）
    threshold := float64(oldCount) * 0.01
    diff := math.Abs(float64(oldCount - newCount))
    
    return diff <= threshold, nil
}

// 检查双写一致性
func (mc *MigrationController) checkDualWriteConsistency(serviceName string) (bool, error) {
    metrics := mc.metrics.GetDualWriteMetrics(serviceName)
    
    // 检查写入成功率（要求99.9%以上）
    successRate := float64(metrics.SuccessCount) / float64(metrics.TotalCount)
    if successRate < 0.999 {
        return false, nil
    }
    
    // 检查数据一致性（要求99.99%以上）
    consistencyRate := float64(metrics.ConsistentCount) / float64(metrics.ValidatedCount)
    if consistencyRate < 0.9999 {
        return false, nil
    }
    
    return true, nil
}

// 紧急回滚机制
func (mc *MigrationController) EmergencyRollback(serviceName string) error {
    currentPhase := mc.configCenter.GetMigrationPhase(serviceName)
    
    mc.logger.Warn("executing emergency rollback", 
        "service", serviceName, "current_phase", currentPhase)
    
    // 根据当前阶段执行不同的回滚策略
    switch currentPhase {
    case PhaseReadOldWriteOldNew:
        // 回滚到阶段1：停止双写，只写旧库
        return mc.setMigrationPhase(serviceName, PhaseReadOldWriteOld)
        
    case PhaseReadNewWriteNewOld:
        // 回滚到阶段3：恢复读旧库
        return mc.setMigrationPhase(serviceName, PhaseReadOldWriteOldNew)
        
    case PhaseReadNewWriteNew:
        // 回滚到阶段4：恢复双写
        return mc.setMigrationPhase(serviceName, PhaseReadNewWriteNewOld)
        
    default:
        return fmt.Errorf("cannot rollback from phase: %d", currentPhase)
    }
}

// 设置迁移阶段
func (mc *MigrationController) setMigrationPhase(serviceName string, phase MigrationPhase) error {
    // 更新配置中心
    err := mc.configCenter.SetMigrationPhase(serviceName, phase)
    if err != nil {
        return fmt.Errorf("update config center failed: %w", err)
    }
    
    // 记录阶段变更日志
    mc.logger.Info("migration phase changed", 
        "service", serviceName, "new_phase", phase)
    
    // 发送告警通知
    mc.sendPhaseChangeAlert(serviceName, phase)
    
    return nil
}
```

#### 第四阶段：双写新读
```
应用 ----写----> 新库 ----反向同步----> 旧库
     ----写----> 旧库
     ----读----> 新库
```
- **关键操作**：切换读取数据源到新库
- **监控重点**：
  - 新库查询性能指标
  - 数据一致性校验
  - 业务功能正确性验证

#### 第五阶段：单写新读
```
应用 ----写----> 新库
     ----读----> 新库
```
- **清理工作**：移除双写逻辑，清理旧库资源
- **保留策略**：旧库数据保留一定周期用于应急回滚

## 数据校验与一致性保证

### 实时数据校验
```go
type DataValidator struct {
    oldDB    *sql.DB
    newDB    *sql.DB
    diffChan chan *DiffRecord
}

func (v *DataValidator) ValidateAsync(key string) {
    go func() {
        oldData := v.queryFromOldDB(key)
        newData := v.queryFromNewDB(key)
        
        if !v.isEqual(oldData, newData) {
            v.diffChan <- &DiffRecord{
                Key:     key,
                OldData: oldData,
                NewData: newData,
                Time:    time.Now(),
            }
        }
    }()
}
```

### 数据修复机制
- **自动修复**：对于简单的数据差异，自动执行修复逻辑
- **人工介入**：复杂差异需要人工分析和处理
- **修复日志**：记录所有修复操作，确保可追溯

## 生产实战案例

### MySQL分库分表迁移案例

**场景描述**：用户表从单表迁移到分库分表架构

#### 原表结构
```sql
CREATE TABLE users (
    id bigint NOT NULL AUTO_INCREMENT PRIMARY KEY,
    username varchar(64) NOT NULL UNIQUE,
    email varchar(128) NOT NULL,
    created_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_username (username),
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 目标分表结构
```sql
-- 分表规则：按user_id hash分16个表
CREATE TABLE users_0 (
    id bigint NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id bigint NOT NULL,
    username varchar(64) NOT NULL,
    email varchar(128) NOT NULL,
    created_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_id (user_id),
    INDEX idx_username (username),
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- users_1 到 users_15 结构相同
```

#### 分表路由逻辑
```go
func GetTableSuffix(userID int64) int {
    return int(userID % 16)
}

func GetTableName(userID int64) string {
    return fmt.Sprintf("users_%d", GetTableSuffix(userID))
}
```

### Redis集群迁移案例

**场景描述**：Redis单机迁移到Redis Cluster

#### 迁移配置
```yaml
# redis-shake配置
[source]
type = standalone
address = 127.0.0.1:6379
password = oldpassword

[target] 
type = cluster
address = 127.0.0.1:7000,127.0.0.1:7001,127.0.0.1:7002
password = newpassword

[filter]
# 过滤临时数据
key.whitelist = user:*,session:*,cache:*
```

## 风险控制与应急预案

### 关键风险点识别
1. **数据丢失风险**：同步延迟导致的数据丢失
2. **性能抖动风险**：迁移过程对线上服务的性能影响
3. **数据不一致风险**：并发写入导致的数据不一致
4. **依赖服务风险**：上下游服务的兼容性问题

### 应急预案
```go
type MigrationController struct {
    state       MigrationState
    rollbackFn  func() error
    checkpoints []Checkpoint
}

// 快速回滚机制
func (mc *MigrationController) EmergencyRollback() error {
    logger.Warn("executing emergency rollback")
    
    // 1. 停止所有迁移任务
    mc.stopAllTasks()
    
    // 2. 恢复到最近的检查点
    lastCheckpoint := mc.getLastCheckpoint()
    if err := mc.restoreCheckpoint(lastCheckpoint); err != nil {
        return fmt.Errorf("rollback failed: %w", err)
    }
    
    // 3. 验证回滚结果
    return mc.validateRollback()
}
```

## 总结与最佳实践

### 核心要点
1. **充分测试**：在测试环境进行多轮完整迁移演练
2. **渐进式迁移**：分阶段执行，每个阶段充分验证后再进入下一阶段
3. **完善监控**：建立全面的监控体系，及时发现和处理异常
4. **应急预案**：制定详细的回滚预案，确保出现问题时能快速恢复

### 技术选型建议
- **小数据量（< 10GB）**：mysqldump + 应用层双写
- **中数据量（10GB - 1TB）**：XtraBackup + DTS工具
- **大数据量（> 1TB）**：分批迁移 + 专业迁移工具

### 成功的关键因素
- **团队协作**：开发、运维、测试多团队紧密配合
- **时间规划**：预留充足的时间处理异常情况
- **风险意识**：始终保持对风险的敬畏心理
- **技术储备**：迁移前确保团队具备足够的技术能力

通过遵循上述实践指南，可以大大降低数据迁移的风险，确保业务系统的稳定运行。