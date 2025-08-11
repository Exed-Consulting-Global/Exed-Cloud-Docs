# Diagramas de Arquitetura Detalhados - AppFlowy-Cloud AWS

## 📋 Índice

1. [Arquitetura de Rede Completa](#arquitetura-de-rede-completa)
2. [Distribuição de Recursos por AZ](#distribuição-de-recursos-por-az)
3. [Fluxo de Dados Detalhado](#fluxo-de-dados-detalhado)
4. [Arquitetura de Segurança](#arquitetura-de-segurança)
5. [Monitoramento e Observabilidade](#monitoramento-e-observabilidade)
6. [Disaster Recovery](#disaster-recovery)

## 🌐 Arquitetura de Rede Completa

### VPC e Subnets Detalhadas

```mermaid
graph TB
    subgraph "VPC: 10.0.0.0/16 - AppFlowy Production"
        subgraph "Public Subnets"
            subgraph "us-east-1a (10.0.1.0/24)"
                ALB1[Application Load Balancer<br/>10.0.1.10]
                NAT1[NAT Gateway<br/>10.0.1.20]
                BASTION1[Bastion Host<br/>10.0.1.30]
            end
            subgraph "us-east-1b (10.0.2.0/24)"
                ALB2[Application Load Balancer<br/>10.0.2.10]
                NAT2[NAT Gateway<br/>10.0.2.20]
                BASTION2[Bastion Host<br/>10.0.2.30]
            end
        end
        
        subgraph "Private Subnets - Application"
            subgraph "us-east-1a (10.0.10.0/24)"
                ECS1[ECS Tasks<br/>10.0.10.10-50]
                ECS2[ECS Tasks<br/>10.0.10.51-100]
                ECS3[ECS Tasks<br/>10.0.10.101-150]
            end
            subgraph "us-east-1b (10.0.11.0/24)"
                ECS4[ECS Tasks<br/>10.0.11.10-50]
                ECS5[ECS Tasks<br/>10.0.11.51-100]
                ECS6[ECS Tasks<br/>10.0.11.101-150]
            end
        end
        
        subgraph "Private Subnets - Database"
            subgraph "us-east-1a (10.0.20.0/24)"
                RDS1[RDS Primary<br/>10.0.20.10]
                RDS2[RDS Read Replica<br/>10.0.20.20]
            end
            subgraph "us-east-1b (10.0.21.0/24)"
                RDS3[RDS Read Replica<br/>10.0.21.10]
                RDS4[RDS Read Replica<br/>10.0.21.20]
            end
        end
        
        subgraph "Private Subnets - Cache"
            subgraph "us-east-1a (10.0.30.0/24)"
                REDIS1[ElastiCache Primary<br/>10.0.30.10]
                REDIS2[ElastiCache Replica<br/>10.0.30.20]
            end
            subgraph "us-east-1b (10.0.31.0/24)"
                REDIS3[ElastiCache Replica<br/>10.0.31.10]
                REDIS4[ElastiCache Replica<br/>10.0.31.20]
            end
        end
    end
    
    subgraph "Internet Gateway"
        IGW[Internet Gateway<br/>igw-12345678]
    end
    
    subgraph "Route Tables"
        RT_PUBLIC[Public Route Table<br/>0.0.0.0/0 → IGW]
        RT_PRIVATE_A[Private Route Table A<br/>0.0.0.0/0 → NAT1]
        RT_PRIVATE_B[Private Route Table B<br/>0.0.0.0/0 → NAT2]
    end
    
    IGW --> ALB1
    IGW --> ALB2
    IGW --> NAT1
    IGW --> NAT2
    
    ALB1 --> ECS1
    ALB1 --> ECS2
    ALB1 --> ECS3
    ALB2 --> ECS4
    ALB2 --> ECS5
    ALB2 --> ECS6
    
    ECS1 --> RDS1
    ECS2 --> RDS1
    ECS3 --> RDS1
    ECS4 --> RDS1
    ECS5 --> RDS1
    ECS6 --> RDS1
    
    ECS1 --> REDIS1
    ECS2 --> REDIS1
    ECS3 --> REDIS1
    ECS4 --> REDIS1
    ECS5 --> REDIS1
    ECS6 --> REDIS1
```

## 🏗️ Distribuição de Recursos por AZ

### Especificações Detalhadas por Availability Zone

```mermaid
graph TB
    subgraph "Availability Zone: us-east-1a"
        subgraph "ECS Cluster - AZ A"
            TASK1[appflowy_cloud<br/>2 vCPU, 4GB RAM<br/>Task ID: task-1a]
            TASK2[gotrue<br/>1 vCPU, 2GB RAM<br/>Task ID: task-2a]
            TASK3[appflowy_collaborate<br/>1 vCPU, 2GB RAM<br/>Task ID: task-3a]
            TASK4[appflowy_worker<br/>1 vCPU, 2GB RAM<br/>Task ID: task-4a]
            TASK5[ai_service<br/>2 vCPU, 4GB RAM<br/>Task ID: task-5a]
        end
        
        subgraph "Database Layer - AZ A"
            RDS_PRIMARY[db.r6g.xlarge<br/>4 vCPU, 32GB RAM<br/>500GB GP3<br/>Primary Instance]
            RDS_REPLICA1[db.r6g.large<br/>2 vCPU, 16GB RAM<br/>500GB GP3<br/>Read Replica 1]
        end
        
        subgraph "Cache Layer - AZ A"
            REDIS_PRIMARY[cache.r6g.large<br/>2 vCPU, 16GB RAM<br/>Primary Node]
            REDIS_REPLICA1[cache.r6g.large<br/>2 vCPU, 16GB RAM<br/>Replica Node 1]
        end
    end
    
    subgraph "Availability Zone: us-east-1b"
        subgraph "ECS Cluster - AZ B"
            TASK6[appflowy_cloud<br/>2 vCPU, 4GB RAM<br/>Task ID: task-1b]
            TASK7[gotrue<br/>1 vCPU, 2GB RAM<br/>Task ID: task-2b]
            TASK8[appflowy_collaborate<br/>1 vCPU, 2GB RAM<br/>Task ID: task-3b]
            TASK9[admin_frontend<br/>0.5 vCPU, 1GB RAM<br/>Task ID: task-4b]
            TASK10[appflowy_web<br/>0.5 vCPU, 1GB RAM<br/>Task ID: task-5b]
        end
        
        subgraph "Database Layer - AZ B"
            RDS_REPLICA2[db.r6g.large<br/>2 vCPU, 16GB RAM<br/>500GB GP3<br/>Read Replica 2]
            RDS_REPLICA3[db.r6g.large<br/>2 vCPU, 16GB RAM<br/>500GB GP3<br/>Read Replica 3]
        end
        
        subgraph "Cache Layer - AZ B"
            REDIS_REPLICA2[cache.r6g.large<br/>2 vCPU, 16GB RAM<br/>Replica Node 2]
            REDIS_REPLICA3[cache.r6g.large<br/>2 vCPU, 16GB RAM<br/>Replica Node 3]
        end
    end
    
    subgraph "Load Balancer"
        ALB[Application Load Balancer<br/>Multi-AZ Distribution<br/>Health Checks: /health]
    end
    
    subgraph "Global Services"
        S3[S3 Bucket<br/>appflowy-production-storage<br/>500GB + Lifecycle Rules]
        BEDROCK[Amazon Bedrock<br/>Titan Embeddings + Claude 3<br/>1M tokens/month]
    end
    
    ALB --> TASK1
    ALB --> TASK2
    ALB --> TASK3
    ALB --> TASK6
    ALB --> TASK7
    ALB --> TASK8
    ALB --> TASK9
    ALB --> TASK10
    
    TASK1 --> RDS_PRIMARY
    TASK6 --> RDS_PRIMARY
    TASK2 --> RDS_PRIMARY
    TASK7 --> RDS_PRIMARY
    
    TASK1 --> REDIS_PRIMARY
    TASK6 --> REDIS_PRIMARY
    TASK2 --> REDIS_PRIMARY
    TASK7 --> REDIS_PRIMARY
    
    TASK1 --> S3
    TASK6 --> S3
    TASK5 --> BEDROCK
```

## 🔄 Fluxo de Dados Detalhado

### Arquitetura de Dados e Comunicação

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web Browser]
        MOBILE[Mobile App]
        DESKTOP[Desktop App]
    end
    
    subgraph "CDN & Edge"
        CLOUDFRONT[CloudFront<br/>Global CDN<br/>Edge Locations]
        ROUTE53[Route 53<br/>DNS + Health Checks<br/>Failover Routing]
    end
    
    subgraph "Load Balancer Layer"
        ALB[Application Load Balancer<br/>SSL Termination<br/>Path-based Routing]
    end
    
    subgraph "Application Layer"
        subgraph "API Services"
            API1[appflowy_cloud<br/>Port: 8000<br/>2 vCPU, 4GB RAM]
            API2[appflowy_cloud<br/>Port: 8000<br/>2 vCPU, 4GB RAM]
        end
        
        subgraph "Authentication"
            AUTH1[gotrue<br/>Port: 9999<br/>1 vCPU, 2GB RAM]
            AUTH2[gotrue<br/>Port: 9999<br/>1 vCPU, 2GB RAM]
        end
        
        subgraph "Real-time Collaboration"
            COLLAB1[appflowy_collaborate<br/>Port: 8001<br/>1 vCPU, 2GB RAM]
            COLLAB2[appflowy_collaborate<br/>Port: 8001<br/>1 vCPU, 2GB RAM]
        end
        
        subgraph "Background Processing"
            WORKER[appflowy_worker<br/>Port: 4001<br/>1 vCPU, 2GB RAM]
            AI[ai_service<br/>Port: 5001<br/>2 vCPU, 4GB RAM]
        end
        
        subgraph "Frontend Services"
            ADMIN[admin_frontend<br/>Port: 3000<br/>0.5 vCPU, 1GB RAM]
            WEB_UI[appflowy_web<br/>Port: 3001<br/>0.5 vCPU, 1GB RAM]
        end
    end
    
    subgraph "Data Layer"
        subgraph "Primary Database"
            RDS_PRIMARY[PostgreSQL Primary<br/>db.r6g.xlarge<br/>4 vCPU, 32GB RAM]
        end
        
        subgraph "Read Replicas"
            RDS_REPLICA1[PostgreSQL Replica 1<br/>db.r6g.large<br/>2 vCPU, 16GB RAM]
            RDS_REPLICA2[PostgreSQL Replica 2<br/>db.r6g.large<br/>2 vCPU, 16GB RAM]
            RDS_REPLICA3[PostgreSQL Replica 3<br/>db.r6g.large<br/>2 vCPU, 16GB RAM]
        end
        
        subgraph "Cache Layer"
            REDIS_PRIMARY[Redis Primary<br/>cache.r6g.large<br/>2 vCPU, 16GB RAM]
            REDIS_REPLICA1[Redis Replica 1<br/>cache.r6g.large<br/>2 vCPU, 16GB RAM]
            REDIS_REPLICA2[Redis Replica 2<br/>cache.r6g.large<br/>2 vCPU, 16GB RAM]
            REDIS_REPLICA3[Redis Replica 3<br/>cache.r6g.large<br/>2 vCPU, 16GB RAM]
        end
        
        subgraph "Storage Layer"
            S3[S3 Bucket<br/>appflowy-production-storage<br/>500GB + Lifecycle]
        end
    end
    
    subgraph "External Services"
        BEDROCK[Amazon Bedrock<br/>AI/ML Services]
        SES[SES<br/>Email Service]
        SNS[SNS<br/>Notifications]
    end
    
    WEB --> CLOUDFRONT
    MOBILE --> CLOUDFRONT
    DESKTOP --> CLOUDFRONT
    
    CLOUDFRONT --> ROUTE53
    ROUTE53 --> ALB
    
    ALB --> API1
    ALB --> API2
    ALB --> AUTH1
    ALB --> AUTH2
    ALB --> ADMIN
    ALB --> WEB_UI
    
    API1 --> RDS_PRIMARY
    API2 --> RDS_PRIMARY
    AUTH1 --> RDS_PRIMARY
    AUTH2 --> RDS_PRIMARY
    WORKER --> RDS_PRIMARY
    
    API1 --> REDIS_PRIMARY
    API2 --> REDIS_PRIMARY
    AUTH1 --> REDIS_PRIMARY
    AUTH2 --> REDIS_PRIMARY
    COLLAB1 --> REDIS_PRIMARY
    COLLAB2 --> REDIS_PRIMARY
    
    API1 --> S3
    API2 --> S3
    WORKER --> S3
    AI --> S3
    
    AI --> BEDROCK
    WORKER --> SES
    API1 --> SNS
    API2 --> SNS
    
    RDS_PRIMARY --> RDS_REPLICA1
    RDS_PRIMARY --> RDS_REPLICA2
    RDS_PRIMARY --> RDS_REPLICA3
    
    REDIS_PRIMARY --> REDIS_REPLICA1
    REDIS_PRIMARY --> REDIS_REPLICA2
    REDIS_PRIMARY --> REDIS_REPLICA3
```

## 🔒 Arquitetura de Segurança

### Security Groups e IAM Policies

```mermaid
graph TB
    subgraph "Security Groups"
        subgraph "ALB Security Group (sg-alb)"
            ALB_SG[ALB Security Group<br/>sg-12345678]
            ALB_HTTP[Port 80: 0.0.0.0/0]
            ALB_HTTPS[Port 443: 0.0.0.0/0]
        end
        
        subgraph "ECS Security Group (sg-ecs)"
            ECS_SG[ECS Security Group<br/>sg-87654321]
            ECS_APP[Port 8000: sg-alb]
            ECS_AUTH[Port 9999: sg-alb]
            ECS_COLLAB[Port 8001: sg-alb]
        end
        
        subgraph "Database Security Group (sg-db)"
            DB_SG[Database Security Group<br/>sg-11223344]
            DB_POSTGRES[Port 5432: sg-ecs]
        end
        
        subgraph "Cache Security Group (sg-cache)"
            CACHE_SG[Cache Security Group<br/>sg-44332211]
            CACHE_REDIS[Port 6379: sg-ecs]
        end
    end
    
    subgraph "IAM Roles"
        subgraph "ECS Task Role"
            ECS_TASK_ROLE[appflowy-ecs-task-role]
            ECS_S3_POLICY[S3 Access Policy]
            ECS_SECRETS_POLICY[Secrets Manager Policy]
            ECS_BEDROCK_POLICY[Bedrock Access Policy]
        end
        
        subgraph "ECS Execution Role"
            ECS_EXEC_ROLE[appflowy-ecs-execution-role]
            ECS_LOGS_POLICY[CloudWatch Logs Policy]
            ECS_ECR_POLICY[ECR Access Policy]
        end
        
        subgraph "RDS Monitoring Role"
            RDS_MONITOR_ROLE[rds-monitoring-role]
            RDS_CLOUDWATCH_POLICY[CloudWatch Policy]
        end
    end
    
    subgraph "Secrets Manager"
        SECRETS[Secrets Manager]
        DB_SECRET[appflowy/production/database]
        REDIS_SECRET[appflowy/production/redis]
        S3_SECRET[appflowy/production/s3]
        BEDROCK_SECRET[appflowy/production/bedrock]
    end
    
    subgraph "KMS Keys"
        KMS[KMS Service]
        RDS_KMS[rds-encryption-key]
        S3_KMS[s3-encryption-key]
        SECRETS_KMS[secrets-encryption-key]
    end
    
    ALB_SG --> ECS_SG
    ECS_SG --> DB_SG
    ECS_SG --> CACHE_SG
    
    ECS_TASK_ROLE --> SECRETS
    ECS_TASK_ROLE --> KMS
    ECS_EXEC_ROLE --> KMS
    RDS_MONITOR_ROLE --> KMS
```

## 📊 Monitoramento e Observabilidade

### CloudWatch Dashboard e Métricas

```mermaid
graph TB
    subgraph "CloudWatch Dashboard"
        subgraph "ECS Metrics"
            ECS_CPU[CPU Utilization<br/>Target: <70%<br/>Alarm: >85%]
            ECS_MEMORY[Memory Utilization<br/>Target: <80%<br/>Alarm: >90%]
            ECS_RUNNING[Running Tasks<br/>Expected: 10<br/>Alarm: <8]
        end
        
        subgraph "RDS Metrics"
            RDS_CPU[CPU Utilization<br/>Target: <60%<br/>Alarm: >80%]
            RDS_CONNECTIONS[Database Connections<br/>Target: <150<br/>Alarm: >180]
            RDS_MEMORY[Freeable Memory<br/>Target: >8GB<br/>Alarm: <4GB]
            RDS_IOPS[IOPS<br/>Target: <3000<br/>Alarm: >5000]
        end
        
        subgraph "ElastiCache Metrics"
            REDIS_CPU[CPU Utilization<br/>Target: <50%<br/>Alarm: >70%]
            REDIS_MEMORY[Memory Usage<br/>Target: <80%<br/>Alarm: >90%]
            REDIS_CONNECTIONS[Connections<br/>Target: <1000<br/>Alarm: >1200]
            REDIS_HIT_RATIO[Cache Hit Ratio<br/>Target: >90%<br/>Alarm: <80%]
        end
        
        subgraph "S3 Metrics"
            S3_SIZE[Bucket Size<br/>Current: 500GB<br/>Alarm: >1TB]
            S3_OBJECTS[Number of Objects<br/>Current: 1M<br/>Alarm: >5M]
            S3_REQUESTS[Request Count<br/>Target: <10K/min<br/>Alarm: >20K/min]
        end
    end
    
    subgraph "CloudWatch Alarms"
        subgraph "Critical Alarms"
            CRITICAL_CPU[ECS CPU >85%<br/>SNS: Critical Alerts]
            CRITICAL_DB[RDS Connections >180<br/>SNS: Critical Alerts]
            CRITICAL_REDIS[Redis Memory >90%<br/>SNS: Critical Alerts]
        end
        
        subgraph "Warning Alarms"
            WARNING_CPU[ECS CPU >70%<br/>SNS: Warning Alerts]
            WARNING_DB[RDS CPU >60%<br/>SNS: Warning Alerts]
            WARNING_REDIS[Redis CPU >50%<br/>SNS: Warning Alerts]
        end
    end
    
    subgraph "X-Ray Tracing"
        XRAY[X-Ray Service Map]
        API_TRACE[API Request Tracing]
        DB_TRACE[Database Query Tracing]
        CACHE_TRACE[Cache Operation Tracing]
    end
    
    subgraph "Log Groups"
        ECS_LOGS[/ecs/appflowy-cloud<br/>Retention: 30 days]
        RDS_LOGS[/aws/rds/instance/production-appflowy-db<br/>Retention: 7 days]
        ALB_LOGS[/aws/applicationloadbalancer/production-alb<br/>Retention: 30 days]
    end
    
    ECS_CPU --> CRITICAL_CPU
    ECS_CPU --> WARNING_CPU
    RDS_CONNECTIONS --> CRITICAL_DB
    RDS_CPU --> WARNING_DB
    REDIS_MEMORY --> CRITICAL_REDIS
    REDIS_CPU --> WARNING_REDIS
    
    API_TRACE --> XRAY
    DB_TRACE --> XRAY
    CACHE_TRACE --> XRAY
```

## 🔄 Disaster Recovery

### Multi-Region Setup

```mermaid
graph TB
    subgraph "Primary Region: us-east-1"
        subgraph "Production Environment"
            ALB_EAST[Application Load Balancer]
            ECS_EAST[ECS Cluster<br/>10 Tasks]
            RDS_EAST[RDS Primary<br/>db.r6g.xlarge]
            REDIS_EAST[ElastiCache Primary<br/>cache.r6g.large]
            S3_EAST[S3 Bucket<br/>appflowy-production-storage]
        end
    end
    
    subgraph "Secondary Region: us-west-2"
        subgraph "DR Environment"
            ALB_WEST[Application Load Balancer]
            ECS_WEST[ECS Cluster<br/>2 Tasks]
            RDS_WEST[RDS Read Replica<br/>db.r6g.large]
            REDIS_WEST[ElastiCache Replica<br/>cache.r6g.large]
            S3_WEST[S3 Bucket<br/>Cross-Region Replication]
        end
    end
    
    subgraph "Global Services"
        ROUTE53[Route 53<br/>Health Checks<br/>Failover Routing]
        CLOUDFRONT[CloudFront<br/>Global CDN<br/>Edge Caching]
        SNS_GLOBAL[SNS<br/>Cross-Region Notifications]
    end
    
    subgraph "Backup Strategy"
        RDS_BACKUP[RDS Automated Backups<br/>7 days retention<br/>Cross-region copy]
        S3_BACKUP[S3 Cross-Region Replication<br/>Real-time sync<br/>99.9% SLA]
        ECS_BACKUP[ECS Task Definitions<br/>ECR Image Replication<br/>Blue-Green Deployment]
    end
    
    subgraph "Recovery Procedures"
        RTO[RTO: 15 minutes<br/>RPO: 5 minutes<br/>Automated Failover]
        MANUAL[Manual Failover<br/>DNS Update<br/>Health Check Validation]
    end
    
    ROUTE53 --> ALB_EAST
    ROUTE53 --> ALB_WEST
    CLOUDFRONT --> S3_EAST
    CLOUDFRONT --> S3_WEST
    
    RDS_EAST -.-> RDS_WEST
    REDIS_EAST -.-> REDIS_WEST
    S3_EAST --> S3_WEST
    
    RDS_EAST --> RDS_BACKUP
    S3_EAST --> S3_BACKUP
    ECS_EAST --> ECS_BACKUP
    
    RTO --> MANUAL
```

## 📈 Auto Scaling Configuration

### Scaling Policies e Thresholds

```mermaid
graph TB
    subgraph "ECS Auto Scaling"
        subgraph "Target Tracking"
            CPU_TARGET[CPU Target: 70%<br/>Scale Out: >70%<br/>Scale In: <50%]
            MEMORY_TARGET[Memory Target: 80%<br/>Scale Out: >80%<br/>Scale In: <60%]
        end
        
        subgraph "Scaling Limits"
            MIN_TASKS[Minimum: 2 tasks<br/>Maximum: 20 tasks<br/>Desired: 6 tasks]
        end
        
        subgraph "Cooldown Periods"
            SCALE_OUT_COOLDOWN[Scale Out: 300s<br/>Scale In: 300s<br/>Stabilization: 60s]
        end
    end
    
    subgraph "RDS Auto Scaling"
        subgraph "Storage Scaling"
            STORAGE_MIN[Min: 100GB<br/>Max: 1000GB<br/>Auto-increment: 10GB]
        end
        
        subgraph "Read Replicas"
            REPLICA_MIN[Min: 1 replica<br/>Max: 5 replicas<br/>Based on: CPU >60%]
        end
    end
    
    subgraph "ElastiCache Auto Scaling"
        subgraph "Node Scaling"
            NODE_MIN[Min: 2 nodes<br/>Max: 10 nodes<br/>Based on: Memory >80%]
        end
    end
    
    subgraph "S3 Auto Scaling"
        subgraph "Storage Classes"
            STANDARD_AUTO[Standard → IA<br/>30 days<br/>Automatic]
            IA_AUTO[IA → Glacier<br/>90 days<br/>Automatic]
            GLACIER_AUTO[Glacier → Deep Archive<br/>365 days<br/>Automatic]
        end
    end
    
    CPU_TARGET --> MIN_TASKS
    MEMORY_TARGET --> MIN_TASKS
    SCALE_OUT_COOLDOWN --> MIN_TASKS
    
    STORAGE_MIN --> REPLICA_MIN
    NODE_MIN --> REPLICA_MIN
```

## 💰 Custos por Componente

### Breakdown Detalhado de Custos

```mermaid
graph TB
    subgraph "Monthly Costs - Production"
        subgraph "Compute Costs ($255)"
            ECS_MAIN[ECS Main Tasks<br/>6 x $30 = $180]
            ECS_SECONDARY[ECS Secondary Tasks<br/>4 x $15 = $60]
            ECS_FRONTEND[ECS Frontend Tasks<br/>2 x $7.50 = $15]
        end
        
        subgraph "Database Costs ($180)"
            RDS_PRIMARY_COST[RDS Primary<br/>db.r6g.xlarge = $120]
            RDS_REPLICA_COST[RDS Read Replica<br/>db.r6g.large = $60]
        end
        
        subgraph "Cache Costs ($60)"
            REDIS_PRIMARY_COST[Redis Primary<br/>cache.r6g.large = $30]
            REDIS_REPLICA_COST[Redis Replica<br/>cache.r6g.large = $30]
        end
        
        subgraph "Storage & Network ($40)"
            S3_COST[S3 Storage<br/>500GB = $5]
            ALB_COST[Load Balancer<br/>2 AZs = $25]
            TRANSFER_COST[Data Transfer<br/>100GB = $10]
        end
        
        subgraph "Management & Monitoring ($28)"
            CLOUDWATCH_COST[CloudWatch<br/>Logs + Metrics = $15]
            ROUTE53_COST[Route 53<br/>DNS + Queries = $5]
            SECRETS_COST[Secrets Manager<br/>4 secrets = $8]
        end
        
        subgraph "AI Services ($10)"
            BEDROCK_COST[Amazon Bedrock<br/>1M tokens = $10]
        end
        
        subgraph "NAT Gateway ($90)"
            NAT_COST[NAT Gateway<br/>2 AZs x $45 = $90]
        end
    end
    
    subgraph "Cost Optimizations"
        subgraph "Reserved Instances (-$72)"
            RDS_RI[RDS Reserved 1yr<br/>30% savings = -$54]
            REDIS_RI[Redis Reserved 1yr<br/>30% savings = -$18]
        end
        
        subgraph "Spot Instances (-$10)"
            WORKER_SPOT[Worker Tasks Spot<br/>70% savings = -$10]
        end
        
        subgraph "Lifecycle Optimization (-$2.50)"
            S3_LIFECYCLE[S3 Lifecycle Rules<br/>50% savings = -$2.50]
        end
    end
    
    ECS_MAIN --> RDS_PRIMARY_COST
    ECS_SECONDARY --> RDS_REPLICA_COST
    REDIS_PRIMARY_COST --> REDIS_REPLICA_COST
    S3_COST --> ALB_COST
    CLOUDWATCH_COST --> ROUTE53_COST
    BEDROCK_COST --> NAT_COST
    
    RDS_RI --> RDS_PRIMARY_COST
    REDIS_RI --> REDIS_PRIMARY_COST
    WORKER_SPOT --> ECS_SECONDARY
    S3_LIFECYCLE --> S3_COST
```

---

**Nota**: Estes diagramas fornecem uma visão detalhada e específica da arquitetura AWS para o AppFlowy-Cloud, incluindo especificações exatas de recursos, distribuição geográfica, fluxos de dados e configurações de segurança. Use-os como referência para implementação e manutenção da infraestrutura.
