# Redis 集群源码分析

## 1. 概述

Redis 集群是 Redis 官方提供的分布式解决方案，通过数据分片、主从复制和自动故障转移实现高可用、可扩展的分布式数据存储。

## 2. 核心常量定义

```c
// 集群槽位数量
#define CLUSTER_SLOTS 16384

// 集群状态
#define CLUSTER_OK 0          /* 集群正常工作 */
#define CLUSTER_FAIL 1        /* 集群无法工作 */

// 节点名称长度（SHA1 十六进制长度）
#define CLUSTER_NAMELEN 40

// 集群端口偏移量
#define CLUSTER_PORT_INCR 10000 /* 集群端口 = 基础端口 + PORT_INCR */

// 重定向错误类型
#define CLUSTER_REDIR_NONE 0          /* 节点可以处理请求 */
#define CLUSTER_REDIR_CROSS_SLOT 1    /* 跨槽位请求 */
#define CLUSTER_REDIR_UNSTABLE 2      /* 槽位不稳定，需要重试 */
#define CLUSTER_REDIR_ASK 3           /* ASK 重定向 */
#define CLUSTER_REDIR_MOVED 4         /* MOVED 重定向 */
#define CLUSTER_REDIR_DOWN_STATE 5    /* 集群下线状态 */
#define CLUSTER_REDIR_DOWN_UNBOUND 6  /* 槽位未绑定 */
#define CLUSTER_REDIR_DOWN_RO_STATE 7 /* 集群下线，允许读操作 */
```

## 3. 核心数据结构

### 3.1 集群节点（clusterNode）

```c
typedef struct clusterNode {
    mstime_t ctime;                    /* 节点创建时间 */
    char name[CLUSTER_NAMELEN];        /* 节点名称，SHA1 十六进制字符串 */
    int flags;                         /* 节点标志位 */
    uint64_t configEpoch;              /* 配置纪元 */
    unsigned char slots[CLUSTER_SLOTS/8]; /* 该节点处理的槽位位图 */
    sds slots_info;                    /* 槽位信息字符串表示 */
    int numslots;                      /* 该节点处理的槽位数量 */
    int numslaves;                     /* 从节点数量（如果是主节点） */
    struct clusterNode **slaves;       /* 从节点指针数组 */
    struct clusterNode *slaveof;       /* 主节点指针（如果是从节点） */
    mstime_t ping_sent;                /* 最后发送 ping 的时间 */
    mstime_t pong_received;            /* 最后接收 pong 的时间 */
    mstime_t data_received;            /* 最后接收数据的时间 */
    mstime_t fail_time;                /* 设置 FAIL 标志的时间 */
    mstime_t voted_time;               /* 最后为该主节点的从节点投票的时间 */
    mstime_t repl_offset_time;         /* 接收该节点复制偏移量的时间 */
    mstime_t orphaned_time;            /* 孤儿主节点条件的开始时间 */
    long long repl_offset;             /* 该节点的最后已知复制偏移量 */
    char ip[NET_IP_STR_LEN];           /* 该节点的最新已知 IP 地址 */
    int port;                          /* 最新已知的客户端端口 */
    int pport;                         /* 最新已知的明文端口 */
    int cport;                         /* 最新已知的集群端口 */
    clusterLink *link;                 /* 与该节点的 TCP/IP 连接 */
    list *fail_reports;                /* 报告该节点故障的节点列表 */
} clusterNode;
```

### 3.2 集群连接（clusterLink）

```c
typedef struct clusterLink {
    mstime_t ctime;             /* 连接创建时间 */
    connection *conn;           /* 到远程节点的连接 */
    sds sndbuf;                 /* 数据包发送缓冲区 */
    char *rcvbuf;               /* 数据包接收缓冲区 */
    size_t rcvbuf_len;          /* rcvbuf 已使用大小 */
    size_t rcvbuf_alloc;        /* rcvbuf 分配大小 */
    struct clusterNode *node;   /* 与此连接相关的节点，或 NULL */
} clusterLink;
```

### 3.3 集群状态（clusterState）

```c
typedef struct clusterState {
    clusterNode *myself;        /* 本节点 */
    uint64_t currentEpoch;      /* 当前纪元 */
    int state;                  /* 集群状态 */
    int size;                   /* 至少有一个槽位的主节点数量 */
    dict *nodes;                /* 名称到 clusterNode 结构的哈希表 */
    dict *nodes_black_list;     /* 我们暂时不重新添加的节点 */
    clusterNode *migrating_slots_to[CLUSTER_SLOTS];    /* 正在迁移到该节点的槽位 */
    clusterNode *importing_slots_from[CLUSTER_SLOTS];  /* 正在从该节点导入的槽位 */
    clusterNode *slots[CLUSTER_SLOTS];                 /* 槽位到节点的映射 */
    uint64_t slots_keys_count[CLUSTER_SLOTS];          /* 每个槽位的键数量 */
    rax *slots_to_keys;         /* 槽位到键的映射 */
    
    /* 故障转移相关字段 */
    mstime_t failover_auth_time;    /* 上次或下次选举时间 */
    int failover_auth_count;        /* 到目前为止收到的投票数 */
    int failover_auth_sent;         /* 是否已经请求投票 */
    int failover_auth_rank;         /* 当前认证请求中该从节点的排名 */
    uint64_t failover_auth_epoch;   /* 当前选举的纪元 */
    int cant_failover_reason;       /* 从节点当前无法故障转移的原因 */
    
    /* 手动故障转移状态 */
    mstime_t mf_end;                /* 手动故障转移时间限制 */
    clusterNode *mf_slave;          /* 执行手动故障转移的从节点 */
    long long mf_master_offset;     /* 从节点开始 MF 需要的主节点偏移量 */
    int mf_can_start;               /* 手动故障转移是否可以开始请求主节点投票 */
    
    /* 投票相关字段 */
    uint64_t lastVoteEpoch;         /* 上次投票的纪元 */
    int todo_before_sleep;          /* 在 clusterBeforeSleep() 中要做的事情 */
    
    /* 统计信息 */
    long long stats_bus_messages_sent[CLUSTERMSG_TYPE_COUNT];
    long long stats_bus_messages_received[CLUSTERMSG_TYPE_COUNT];
    long long stats_pfail_nodes;    /* PFAIL 状态的节点数量 */
} clusterState;
```

## 4. 关键算法实现

### 4.1 哈希槽计算

```c
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing between {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 0x3FFF;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 0x3FFF;
}
```

**算法说明：**
1. 如果键不包含 `{`，则对整个键进行 CRC16 哈希
2. 如果键包含 `{`，则查找对应的 `}`
3. 如果找到 `{` 和 `}`，则对它们之间的内容进行哈希
4. 最终结果对 16384 取模得到槽位号

### 4.2 节点查找和路由

```c
clusterNode *getNodeByQuery(client *c, struct redisCommand *cmd, robj **argv, int argc, int *hashslot, int *error_code) {
    clusterNode *n = NULL;
    robj *firstkey = NULL;
    int multiple_keys = 0;
    multiState *ms, _ms;
    multiCmd mc;
    int i, slot = 0, migrating_slot = 0, importing_slot = 0, missing_keys = 0;

    /* 检查所有键是否在同一个哈希槽中 */
    for (i = 0; i < ms->count; i++) {
        struct redisCommand *mcmd = ms->commands[i].cmd;
        robj **margv = ms->commands[i].argv;
        int margc = ms->commands[i].argc;
        int keyindex = 0;
        int numkeys = 0;

        /* 获取命令的键数量 */
        if (mcmd->getkeys_proc) {
            keyindex = mcmd->getkeys_proc(margc, margv, &numkeys);
        } else {
            keyindex = mcmd->firstkey;
            numkeys = mcmd->lastkey - mcmd->firstkey + 1;
        }

        /* 检查每个键的槽位 */
        for (int j = 0; j < numkeys; j++) {
            robj *thiskey = margv[keyindex + j];
            int thisslot = keyHashSlot(thiskey->ptr, sdslen(thiskey->ptr));
            
            if (slot == 0) {
                slot = thisslot;
                firstkey = thiskey;
            } else if (thisslot != slot) {
                /* 跨槽位操作 */
                if (error_code) *error_code = CLUSTER_REDIR_CROSS_SLOT;
                return NULL;
            }
        }
    }

    /* 获取负责该槽位的节点 */
    if (slot != 0) {
        n = server.cluster->slots[slot];
        if (n == NULL) {
            if (error_code) *error_code = CLUSTER_REDIR_DOWN_UNBOUND;
            return NULL;
        }
    }

    if (hashslot) *hashslot = slot;
    if (error_code) *error_code = CLUSTER_REDIR_NONE;
    return n;
}
```

## 5. 集群通信协议

### 5.1 消息类型

```c
#define CLUSTERMSG_TYPE_PING 0          /* Ping */
#define CLUSTERMSG_TYPE_PONG 1          /* Pong (回复 Ping) */
#define CLUSTERMSG_TYPE_MEET 2          /* Meet "让我们加入" 消息 */
#define CLUSTERMSG_TYPE_FAIL 3          /* 标记节点为故障 */
#define CLUSTERMSG_TYPE_PUBLISH 4       /* 发布/订阅传播 */
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST 5 /* 我可以故障转移吗？ */
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 6     /* 是的，你有我的投票 */
#define CLUSTERMSG_TYPE_UPDATE 7        /* 另一个节点的槽位配置 */
#define CLUSTERMSG_TYPE_MFSTART 8       /* 暂停客户端进行手动故障转移 */
#define CLUSTERMSG_TYPE_MODULE 9        /* 模块集群 API 消息 */
```

### 5.2 消息结构

```c
typedef struct {
    char sig[4];        /* 签名 "RCmb" (Redis Cluster message bus) */
    uint32_t totlen;    /* 此消息的总长度 */
    uint16_t ver;       /* 协议版本，当前设置为 1 */
    uint16_t port;      /* TCP 基础端口号 */
    uint16_t type;      /* 消息类型 */
    uint16_t count;     /* 仅用于某些类型的消息 */
    uint64_t currentEpoch;  /* 发送节点的纪元 */
    uint64_t configEpoch;   /* 如果是主节点则为配置纪元，如果是从节点则为最后广播的纪元 */
    uint64_t offset;    /* 如果是主节点则为主节点复制偏移量，如果是从节点则为处理的复制偏移量 */
    char sender[CLUSTER_NAMELEN]; /* 发送节点名称 */
    unsigned char myslots[CLUSTER_SLOTS/8]; /* 该节点处理的槽位 */
    char slaveof[CLUSTER_NAMELEN]; /* 主节点名称（如果是从节点） */
    char myip[NET_IP_STR_LEN];    /* 发送者 IP */
    char notused1[32];  /* 32 字节保留供将来使用 */
    uint16_t pport;      /* 发送者 TCP 明文端口 */
    uint16_t cport;      /* 发送者 TCP 集群总线端口 */
    uint16_t flags;      /* 发送者节点标志 */
    unsigned char state; /* 从发送者角度看集群状态 */
    unsigned char mflags[3]; /* 消息标志 */
    union clusterMsgData data; /* 消息数据 */
} clusterMsg;
```

## 6. 故障转移机制

### 6.1 故障检测

```c
void markNodeAsFailingIfNeeded(clusterNode *node) {
    int failures;
    int needed_quorum = (server.cluster->size / 2) + 1;

    if (!nodeTimedOut(node)) return; /* 节点没有超时 */
    if (nodeFailed(node)) return;   /* 节点已经标记为故障 */

    failures = clusterNodeFailureReportsCount(node);
    /* 如果故障报告数量达到法定人数，标记节点为故障 */
    if (failures >= needed_quorum) {
        serverLog(LL_NOTICE,
            "Marking node %.40s as failing (quorum reached).",
            node->name);
        node->flags &= ~CLUSTER_NODE_PFAIL;
        node->flags |= CLUSTER_NODE_FAIL;
        node->fail_time = mstime();
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
    }
}
```

### 6.2 故障转移选举

```c
void clusterHandleSlaveFailover(void) {
    mstime_t data_age;
    mstime_t auth_age = mstime() - server.cluster->failover_auth_time;
    mstime_t auth_timeout, auth_retry_time;

    /* 检查从节点是否可以开始故障转移 */
    if (server.cluster->mf_end) {
        clusterHandleManualFailover();
        return;
    }

    /* 检查数据年龄 */
    if (server.master) {
        data_age = (mstime_t)(server.unixtime - server.master->repl_put_online_on) * 1000;
    } else {
        data_age = (mstime_t)(server.unixtime - server.cluster->failover_auth_time) * 1000;
    }

    /* 如果数据太旧，不能故障转移 */
    if (data_age > server.cluster_node_timeout) {
        clusterLogCantFailover(CLUSTER_CANT_FAILOVER_DATA_AGE);
        return;
    }

    /* 检查是否已经发送了故障转移认证请求 */
    if (server.cluster->failover_auth_sent) {
        clusterHandleSlaveFailover();
        return;
    }

    /* 发送故障转移认证请求 */
    server.cluster->failover_auth_sent = 1;
    server.cluster->failover_auth_time = mstime();
    server.cluster->failover_auth_count = 0;
    server.cluster->failover_auth_rank = clusterGetSlaveRank();
    server.cluster->failover_auth_epoch = server.cluster->failover_auth_epoch;

    serverLog(LL_WARNING,
        "Starting a failover election for epoch %llu.",
        (unsigned long long) server.cluster->failover_auth_epoch);

    clusterRequestFailoverAuth();
}
```

## 7. 槽位管理

### 7.1 槽位分配

```c
int clusterAddSlot(clusterNode *n, int slot) {
    if (server.cluster->slots[slot]) return C_ERR;
    if (clusterNodeSetSlotBit(n,slot)) {
        server.cluster->slots[slot] = n;
        return C_OK;
    }
    return C_ERR;
}
```

### 7.2 槽位迁移

```c
int clusterSetSlot(clusterManagerNode *node1,
                   clusterManagerNode *node2,
                   int slot, const char *status, char **err) {
    redisReply *reply = CLUSTER_MANAGER_COMMAND(node1, "CLUSTER "
                                                "SETSLOT %d %s %s",
                                                slot, status,
                                                (char *) node2->name);
    if (err != NULL) *err = NULL;
    if (!reply) return 0;
    int success = 1;
    if (reply->type == REDIS_REPLY_ERROR) {
        success = 0;
        if (err != NULL) {
            *err = zmalloc((reply->len + 1) * sizeof(char));
            strcpy(*err, reply->str);
        } else CLUSTER_MANAGER_PRINT_REPLY_ERROR(node1, reply->str);
        goto cleanup;
    }
cleanup:
    freeReplyObject(reply);
    return success;
}
```

## 8. 集群初始化

### 8.1 集群配置加载

```c
int clusterLoadConfig(char *filename) {
    FILE *fp = fopen(filename,"r");
    struct stat sb;
    char *line;
    int maxline, j;

    if (fp == NULL) {
        if (errno == ENOENT) {
            return C_ERR;
        } else {
            serverLog(LL_WARNING,
                "Loading the cluster node config from %s: %s",
                filename, strerror(errno));
            exit(1);
        }
    }

    /* 检查文件是否为空 */
    if (fstat(fileno(fp),&sb) != -1 && sb.st_size == 0) {
        fclose(fp);
        return C_ERR;
    }

    /* 解析文件 */
    maxline = 1024+CLUSTER_SLOTS*128;
    line = zmalloc(maxline);
    while(fgets(line,maxline,fp) != NULL) {
        int argc;
        sds *argv;
        clusterNode *n, *master;

        /* 跳过空行 */
        if (line[0] == '\n' || line[0] == '\0') continue;

        /* 分割行参数 */
        argv = sdssplitargs(line,&argc);
        if (argv == NULL) goto fmterr;

        /* 处理特殊变量行 */
        if (strcasecmp(argv[0],"vars") == 0) {
            if (!(argc % 2)) goto fmterr;
            for (j = 1; j < argc; j += 2) {
                if (strcasecmp(argv[j],"currentEpoch") == 0) {
                    server.cluster->currentEpoch = strtoull(argv[j+1],NULL,10);
                } else if (strcasecmp(argv[j],"lastVoteEpoch") == 0) {
                    server.cluster->lastVoteEpoch = strtoull(argv[j+1],NULL,10);
                }
            }
            sdsfreesplitres(argv,argc);
            continue;
        }

        /* 创建节点 */
        n = clusterLookupNode(argv[0]);
        if (!n) {
            n = createClusterNode(argv[0],0);
            clusterAddNode(n);
        }

        /* 解析地址和端口 */
        // ... 地址解析代码 ...

        /* 解析标志 */
        // ... 标志解析代码 ...

        /* 解析槽位 */
        // ... 槽位解析代码 ...
    }

    zfree(line);
    fclose(fp);
    return C_OK;

fmterr:
    serverLog(LL_ERR,"Unrecoverable error: corrupted cluster config file.");
    zfree(line);
    if (fp) fclose(fp);
    exit(1);
}
```

## 9. 集群命令处理

### 9.1 CLUSTER 命令

```c
void clusterCommand(client *c) {
    if (!strcasecmp(c->argv[1]->ptr,"info") && c->argc == 2) {
        /* CLUSTER INFO */
        sds info = sdsempty();
        info = sdscatprintf(info,
            "cluster_state:%s\r\n",
            server.cluster->state == CLUSTER_OK ? "ok" : "fail");
        info = sdscatprintf(info,
            "cluster_slots_assigned:%d\r\n",
            server.cluster->slots_assigned);
        info = sdscatprintf(info,
            "cluster_slots_ok:%d\r\n",
            server.cluster->slots_ok);
        info = sdscatprintf(info,
            "cluster_slots_pfail:%d\r\n",
            server.cluster->slots_pfail);
        info = sdscatprintf(info,
            "cluster_slots_fail:%d\r\n",
            server.cluster->slots_fail);
        info = sdscatprintf(info,
            "cluster_known_nodes:%d\r\n",
            dictSize(server.cluster->nodes));
        info = sdscatprintf(info,
            "cluster_size:%d\r\n",
            server.cluster->size);
        info = sdscatprintf(info,
            "cluster_current_epoch:%llu\r\n",
            (unsigned long long) server.cluster->currentEpoch);
        info = sdscatprintf(info,
            "cluster_my_epoch:%llu\r\n",
            (unsigned long long) server.cluster->myself->configEpoch);
        info = sdscatprintf(info,
            "cluster_stats_messages_sent:%llu\r\n",
            server.cluster->stats_bus_messages_sent[0] +
            server.cluster->stats_bus_messages_sent[1] +
            server.cluster->stats_bus_messages_sent[2] +
            server.cluster->stats_bus_messages_sent[3] +
            server.cluster->stats_bus_messages_sent[4] +
            server.cluster->stats_bus_messages_sent[5] +
            server.cluster->stats_bus_messages_sent[6] +
            server.cluster->stats_bus_messages_sent[7] +
            server.cluster->stats_bus_messages_sent[8] +
            server.cluster->stats_bus_messages_sent[9]);
        info = sdscatprintf(info,
            "cluster_stats_messages_received:%llu\r\n",
            server.cluster->stats_bus_messages_received[0] +
            server.cluster->stats_bus_messages_received[1] +
            server.cluster->stats_bus_messages_received[2] +
            server.cluster->stats_bus_messages_received[3] +
            server.cluster->stats_bus_messages_received[4] +
            server.cluster->stats_bus_messages_received[5] +
            server.cluster->stats_bus_messages_received[6] +
            server.cluster->stats_bus_messages_received[7] +
            server.cluster->stats_bus_messages_received[8] +
            server.cluster->stats_bus_messages_received[9]);
        addReplyBulkSds(c,info);
    } else if (!strcasecmp(c->argv[1]->ptr,"nodes") && c->argc == 2) {
        /* CLUSTER NODES */
        sds nodes = clusterGenNodesDescription(0, 1);
        addReplyBulkSds(c,nodes);
    } else if (!strcasecmp(c->argv[1]->ptr,"slots") && c->argc == 2) {
        /* CLUSTER SLOTS */
        clusterReplyMultiBulkSlots(c);
    } else if (!strcasecmp(c->argv[1]->ptr,"keyslot") && c->argc == 3) {
        /* CLUSTER KEYSLOT */
        int slot = getSlotOrReply(c,c->argv[2]);
        if (slot == -1) return;
        addReplyLongLong(c,slot);
    } else if (!strcasecmp(c->argv[1]->ptr,"countkeysinslot") && c->argc == 3) {
        /* CLUSTER COUNTKEYSINSLOT */
        int slot = getSlotOrReply(c,c->argv[2]);
        if (slot == -1) return;
        addReplyLongLong(c,countKeysInSlot(slot));
    } else if (!strcasecmp(c->argv[1]->ptr,"getkeysinslot") && c->argc == 4) {
        /* CLUSTER GETKEYSINSLOT */
        int slot = getSlotOrReply(c,c->argv[2]);
        if (slot == -1) return;
        long count = getLongFromObjectOrReply(c,c->argv[3],NULL);
        if (count == -1) return;
        addReplyMultiBulkLen(c,count);
        for (int j = 0; j < count; j++) {
            robj *key = getKeyFromSlot(slot);
            if (key) {
                addReplyBulk(c,key);
                decrRefCount(key);
            }
        }
    } else if (!strcasecmp(c->argv[1]->ptr,"setslot") && c->argc >= 4) {
        /* CLUSTER SETSLOT */
        int slot = getSlotOrReply(c,c->argv[2]);
        if (slot == -1) return;
        if (!strcasecmp(c->argv[3]->ptr,"importing") && c->argc == 5) {
            clusterNode *n = clusterLookupNode(c->argv[4]->ptr);
            if (!n) {
                addReplyErrorFormat(c,"Unknown node %s", (char*)c->argv[4]->ptr);
                return;
            }
            server.cluster->importing_slots_from[slot] = n;
        } else if (!strcasecmp(c->argv[3]->ptr,"migrating") && c->argc == 5) {
            clusterNode *n = clusterLookupNode(c->argv[4]->ptr);
            if (!n) {
                addReplyErrorFormat(c,"Unknown node %s", (char*)c->argv[4]->ptr);
                return;
            }
            server.cluster->migrating_slots_to[slot] = n;
        } else if (!strcasecmp(c->argv[3]->ptr,"stable") && c->argc == 4) {
            server.cluster->importing_slots_from[slot] = NULL;
            server.cluster->migrating_slots_to[slot] = NULL;
        } else if (!strcasecmp(c->argv[3]->ptr,"node") && c->argc == 5) {
            clusterNode *n = clusterLookupNode(c->argv[4]->ptr);
            if (!n) {
                addReplyErrorFormat(c,"Unknown node %s", (char*)c->argv[4]->ptr);
                return;
            }
            /* 设置槽位所有者 */
            if (server.cluster->slots[slot]) {
                clusterDelSlot(slot);
            }
            clusterAddSlot(n,slot);
        } else {
            addReplyError(c,"Invalid CLUSTER SETSLOT action. Try IMPORTING, MIGRATING, STABLE or NODE");
            return;
        }
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"setslot") && c->argc >= 4) {
        /* CLUSTER SETSLOT (alias for backward compatibility) */
        // ... 处理逻辑同上 ...
    } else if (!strcasecmp(c->argv[1]->ptr,"delslots") && c->argc >= 3) {
        /* CLUSTER DELSLOTS */
        int j, slot;
        for (j = 2; j < c->argc; j++) {
            if ((slot = getSlotOrReply(c,c->argv[j])) == -1) return;
            if (server.cluster->slots[slot] == NULL) {
                addReplyErrorFormat(c,"Slot %d is already unassigned", slot);
                return;
            }
            clusterDelSlot(slot);
        }
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"addslots") && c->argc >= 3) {
        /* CLUSTER ADDSLOTS */
        int j, slot;
        for (j = 2; j < c->argc; j++) {
            if ((slot = getSlotOrReply(c,c->argv[j])) == -1) return;
            if (server.cluster->slots[slot]) {
                addReplyErrorFormat(c,"Slot %d is already busy", slot);
                return;
            }
            clusterAddSlot(server.cluster->myself,slot);
        }
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"replicate") && c->argc == 3) {
        /* CLUSTER REPLICATE */
        clusterNode *n = clusterLookupNode(c->argv[2]->ptr);
        if (!n) {
            addReplyErrorFormat(c,"Unknown node %s", (char*)c->argv[2]->ptr);
            return;
        }
        if (n == server.cluster->myself) {
            addReplyError(c,"Can't replicate myself");
            return;
        }
        if (nodeIsSlave(n)) {
            addReplyError(c,"I can only replicate a master, not a replica.");
            return;
        }
        clusterSetMaster(n);
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"meet") && c->argc == 4) {
        /* CLUSTER MEET */
        long port;
        if (getLongFromObjectOrReply(c,c->argv[3],&port,NULL) != C_OK) return;
        clusterStartHandshake(c->argv[2]->ptr,port,port+CLUSTER_PORT_INCR);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"forget") && c->argc == 3) {
        /* CLUSTER FORGET */
        clusterNode *n = clusterLookupNode(c->argv[2]->ptr);
        if (!n) {
            addReplyErrorFormat(c,"Unknown node %s", (char*)c->argv[2]->ptr);
            return;
        }
        if (n == server.cluster->myself) {
            addReplyError(c,"I tried to forget myself...");
            return;
        }
        clusterDelNode(n);
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"flushslots") && c->argc == 2) {
        /* CLUSTER FLUSHSLOTS */
        clusterCloseAllSlots();
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"reset") && (c->argc == 2 || c->argc == 3)) {
        /* CLUSTER RESET */
        int hard = 0;
        if (c->argc == 3) {
            if (!strcasecmp(c->argv[2]->ptr,"hard")) {
                hard = 1;
            } else if (!strcasecmp(c->argv[2]->ptr,"soft")) {
                hard = 0;
            } else {
                addReplyError(c,"CLUSTER RESET requires HARD or SOFT argument");
                return;
            }
        }
        clusterReset(hard);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"failover") && (c->argc == 2 || c->argc == 3)) {
        /* CLUSTER FAILOVER */
        int force = 0;
        if (c->argc == 3) {
            if (!strcasecmp(c->argv[2]->ptr,"force")) {
                force = 1;
            } else if (!strcasecmp(c->argv[2]->ptr,"takeover")) {
                force = 2;
            } else {
                addReplyError(c,"CLUSTER FAILOVER requires FORCE or TAKEOVER argument");
                return;
            }
        }
        if (nodeIsMaster(server.cluster->myself)) {
            addReplyError(c,"You should send CLUSTER FAILOVER to a replica");
            return;
        }
        if (server.cluster->myself->slaveof == NULL) {
            addReplyError(c,"I'm a replica but my master is unknown to me");
            return;
        }
        if (force == 2) {
            /* TAKEOVER */
            server.cluster->failover_auth_time = mstime();
            server.cluster->failover_auth_count = 0;
            server.cluster->failover_auth_sent = 0;
            server.cluster->failover_auth_rank = 0;
            server.cluster->failover_auth_epoch = server.cluster->currentEpoch;
            clusterRequestFailoverAuth();
        } else if (force == 1) {
            /* FORCE */
            server.cluster->failover_auth_time = mstime();
            server.cluster->failover_auth_count = 0;
            server.cluster->failover_auth_sent = 0;
            server.cluster->failover_auth_rank = 0;
            server.cluster->failover_auth_epoch = server.cluster->currentEpoch;
            clusterRequestFailoverAuth();
        } else {
            /* NORMAL */
            server.cluster->failover_auth_time = mstime();
            server.cluster->failover_auth_count = 0;
            server.cluster->failover_auth_sent = 0;
            server.cluster->failover_auth_rank = 0;
            server.cluster->failover_auth_epoch = server.cluster->currentEpoch;
            clusterRequestFailoverAuth();
        }
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"set-config-epoch") && c->argc == 3) {
        /* CLUSTER SET-CONFIG-EPOCH */
        long long epoch;
        if (getLongLongFromObjectOrReply(c,c->argv[2],&epoch,NULL) != C_OK) return;
        if (epoch < 0) {
            addReplyError(c,"Invalid config epoch");
            return;
        }
        if (server.cluster->myself->configEpoch != 0) {
            addReplyError(c,"Node config epoch is already non-zero");
            return;
        }
        server.cluster->myself->configEpoch = epoch;
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } else if (!strcasecmp(c->argv[1]->ptr,"bumpepoch") && c->argc == 2) {
        /* CLUSTER BUMPEPOCH */
        if (clusterBumpConfigEpochWithoutConsensus() == C_OK) {
            addReply(c,shared.ok);
        } else {
            addReplyError(c,"Node config epoch is already the maximum among the cluster");
        }
    } else if (!strcasecmp(c->argv[1]->ptr,"saveconfig") && c->argc == 2) {
        /* CLUSTER SAVECONFIG */
        if (clusterSaveConfig(1) == C_OK) {
            addReply(c,shared.ok);
        } else {
            addReplyError(c,"Error saving the cluster configuration");
        }
    } else if (!strcasecmp(c->argv[1]->ptr,"links") && c->argc == 2) {
        /* CLUSTER LINKS */
        sds info = sdsempty();
        dictIterator *di;
        dictEntry *de;
        di = dictGetSafeIterator(server.cluster->nodes);
        while((de = dictNext(di)) != NULL) {
            clusterNode *node = dictGetVal(de);
            if (node->link) {
                info = sdscatprintf(info,
                    "%s %s:%d@%d %s %s %lld %lld\r\n",
                    node->name,
                    node->ip, node->port, node->cport,
                    node->link->conn ? "connected" : "disconnected",
                    node->link->conn ? "connected" : "disconnected",
                    (long long) node->link->ctime,
                    (long long) node->link->ctime);
            }
        }
        dictReleaseIterator(di);
        addReplyBulkSds(c,info);
    } else {
        addReplySubcommandSyntaxError(c);
    }
}
```

## 10、Possip协议

### 10.1. 什么是 Gossip 协议？

Gossip 协议是一种分布式系统中常用的信息传播协议。其核心思想是：每个节点定期随机选择其他节点交换自身已知的集群信息，从而实现整个集群状态的最终一致。

在 Redis 集群中，gossip 协议用于节点间同步集群拓扑、节点状态（如 PFAIL/FAIL）、槽位分配等信息。

------

### 10.2. Redis Gossip 协议流程

1. 定期通信

​	每个节点会定期（如每秒）向集群中的其他节点发送 PING 消息。

2. 消息内容

​	PING 消息中包含了发送者已知的部分其他节点的状态信息（gossip section），如节点名、IP、端口、状态、最后一次 PING/PONG 时间等。

3. 信息传播

​	被 PING 的节点收到消息后，会更新本地的集群视图，并回复 PONG 消息，PONG 也带有 gossip section。

4. 状态收敛

​	通过不断的 PING/PONG 交换，集群中所有节点最终都会获得一致的集群状态信息。

------

### 10.3. Gossip 消息结构（源码片段）

在 src/cluster.h 中，gossip 结构体如下：

```c
typedef struct {
    char nodename[CLUSTER_NAMELEN];
    uint32_t ping_sent;
    uint32_t pong_received;
    char ip[NET_IP_STR_LEN];
    uint16_t port;
    uint16_t cport;
    uint16_t flags;
    uint16_t pport;
    uint16_t notused1;
} clusterMsgDataGossip;
```

每条 PING/PONG 消息中会带有若干个这样的 gossip 结构体。

------

### 4. Gossip 协议例子

假设有 3 个节点：A、B、C。

1. 初始状态

- A 只知道自己

- B 只知道自己

- C 只知道自己

1. A 向 B 发送 PING

- A 发送 PING，gossip section 只包含自己（A）

- B 收到后，知道了 A 的存在

1. B 向 C 发送 PING

- B 发送 PING，gossip section 包含 B 和 A

- C 收到后，知道了 B 和 A 的存在

1. C 向 A 发送 PING

- C 发送 PING，gossip section 包含 C、B、A

- A 收到后，知道了 C 的存在

经过几轮交换，所有节点都能知道集群中所有节点的状态。

------

### 5. 代码流程简要

- 发送 PING：clusterSendPing

- 处理 gossip section：clusterProcessGossipSection

- 更新节点状态：clusterNode 结构体的 flags 字段

Redis 集群通过 gossip 协议实现节点间的状态同步和故障检测，保证了集群拓扑和节点状态的最终一致性。

例子总结：

A → B（A告知B自己）

B → C（B告知C自己和A）

C → A（C告知A自己、B、A）

最终所有节点都知道了彼此的存在和状态

## 10. 总结

Redis 集群通过以下核心机制实现分布式数据存储：

1. **数据分片**：使用 CRC16 哈希算法将数据分散到 16384 个槽位中
2. **节点管理**：通过 clusterNode 结构管理集群中的所有节点
3. **槽位分配**：每个主节点负责处理一部分哈希槽位
4. **故障检测**：通过心跳机制和故障报告检测节点故障
5. **故障转移**：从节点自动升级为主节点，保证高可用性
6. **集群通信**：使用自定义的二进制协议进行节点间通信
7. **配置管理**：通过 nodes.conf 文件持久化集群配置

这种设计使得 Redis 集群能够提供高可用、可扩展的分布式数据存储服务，同时保持与单机 Redis 的兼容性。 