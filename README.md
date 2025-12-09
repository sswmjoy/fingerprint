graph TD
    %% 定义样式
    classDef storage fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef process fill:#f3e5f5,stroke:#4a148c,stroke-width:2px;
    classDef actor fill:#fff3e0,stroke:#e65100,stroke-width:2px;

    Client((外部调用方)):::actor

    subgraph "应用服务 (Java Service)"
        API[消耗接口 / 获取券码]:::process
        
        subgraph "核心调度模块"
            CacheMonitor[缓存监测任务]:::process
            DBMonitor[数据库监测任务]:::process
            Generator[生成器模块]:::process
        end
    end

    subgraph "数据存储层"
        Redis[(Redis 缓存码池)]:::storage
        DB[(Database 数据库码池)]:::storage
    end

    %% 流程连线
    
    %% 1. 消耗流程
    Client -->|1. 请求获取券码| API
    API -->|2. POP 取出可用码| Redis
    API -.->|3. 异步/同步标记为已使用| DB

    %% 2. 缓存回补流程
    CacheMonitor -->|定时监控阈值| Redis
    Redis -.->|反馈当前数量| CacheMonitor
    CacheMonitor -->|如果低于阈值: 拉取可用码| DB
    DB -->|返回可用码数据| CacheMonitor
    CacheMonitor -->|填充缓存| Redis

    %% 3. 数据库生产流程
    DBMonitor -->|定时监控总可用量| DB
    DB -.->|反馈可用库存| DBMonitor
    DBMonitor -->|如果低于阈值: 触发生成| Generator
    
    Generator -->|1. 按照规则生成随机码| Generator
    Generator -->|2. 查重 & 插入| DB
    DB -.->|如果重复/主键冲突| Generator
    Generator -->|3. 重新生成直到成功| DB

    %% 注释说明
    note1[缓存码池: 只存可用码<br/>纯内存操作, 高性能]
    note2[数据库码池: 全量数据<br/>包含可用/已用状态]
    
    Redis -.- note1
    DB -.- note2
