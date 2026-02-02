# Apache Airflow on Virtual Machines - Data Pipeline Orchestration Case Study

## Project Overview

Implementation of Apache Airflow as a data pipeline orchestration platform deployed on virtual machines. The system manages complex ETL workflows, data processing pipelines, and scheduled automation tasks across multiple data sources and destinations.

## Business Problem

The organization faced several data orchestration challenges:
- Multiple disconnected batch processes
- No centralized scheduling or monitoring
- Difficult to track data lineage
- Manual intervention required for failures
- No dependency management between jobs
- Limited visibility into pipeline health

## Technical Architecture

### Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Load Balancer                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
        ┌───────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐
        │   Airflow    │ │  Airflow   │ │  Airflow   │
        │   Webserver  │ │  Webserver │ │  Webserver │
        │   (VM-1)     │ │   (VM-2)   │ │   (VM-3)   │
        └──────────────┘ └────────────┘ └────────────┘
                │               │               │
                └───────────────┼───────────────┘
                                │
                        ┌───────▼────────┐
                        │   Scheduler    │
                        │   (VM-4)       │
                        │   (Active)     │
                        └───────┬────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
        ┌───────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐
        │   Worker     │ │  Worker    │ │  Worker    │
        │   Pool-1     │ │  Pool-2    │ │  Pool-3    │
        │   (VM-5)     │ │  (VM-6)    │ │  (VM-7)    │
        └──────────────┘ └────────────┘ └────────────┘
                                │
                        ┌───────▼────────┐
                        │   PostgreSQL   │
                        │   Metadata DB  │
                        │   (VM-8)       │
                        └────────────────┘
```

### System Components

1. **Web Servers (Multi-VM)**
   - User interface access
   - API endpoint exposure
   - Load balanced for HA
   - Session management

2. **Scheduler (Active-Standby)**
   - DAG parsing and scheduling
   - Task queue management
   - Dependency resolution
   - Heartbeat monitoring

3. **Worker Pools**
   - Celery executor workers
   - Task execution environment
   - Isolated processing
   - Horizontal scalability

4. **Metadata Database**
   - PostgreSQL backend
   - Configuration storage
   - Execution history
   - User management

5. **Message Broker**
   - RabbitMQ/Redis
   - Task queue management
   - Worker communication
   - Result backend

## Process Flow

```
1. DAG Definition
   ├── Python scripts define workflows
   ├── DAGs stored in shared directory
   ├── Version controlled in Git
   └── Auto-discovery by scheduler
          │
          ▼
2. Scheduling
   ├── Scheduler parses DAGs
   ├── Creates task instances
   ├── Checks dependencies
   └── Queues ready tasks
          │
          ▼
3. Execution
   ├── Workers pull tasks from queue
   ├── Execute in isolated environments
   ├── Handle retries and failures
   └── Update metadata database
          │
          ▼
4. Monitoring & Alerts
   ├── Real-time status in UI
   ├── Email/Slack notifications
   ├── Metrics collection
   └── Log aggregation
```

## Technical Decisions

### 1. Deployment Model
**Decision**: Virtual machines instead of containers/Kubernetes
**Rationale**:
- Existing VM infrastructure and expertise
- Simpler operational model for team
- Better isolation for sensitive data processing
- Easier compliance with security policies
- Lower learning curve for operations team

### 2. Executor Choice
**Decision**: Celery Executor with worker pools
**Rationale**:
- Horizontal scalability
- Fine-grained resource allocation
- Support for heterogeneous workers
- Better for mixed workload types
- Mature and well-tested

### 3. High Availability Strategy
**Decision**: Multi-webserver with single active scheduler
**Rationale**:
- Webserver stateless, easy to load balance
- Scheduler active-standby prevents split-brain
- Database as single source of truth
- Simplified failover logic

### 4. Storage Strategy
**Decision**: Shared NFS mount for DAG files
**Rationale**:
- Central DAG repository
- Consistent view across all components
- Easy deployment (copy files)
- Version control integration

## Key Challenges & Solutions

### Challenge 1: Network Latency Between VMs
**Problem**: Inter-VM communication latency affected task scheduling performance
**Solution**:
- Optimized network configuration
- Colocated related VMs in same subnet/availability zone
- Tuned Celery worker prefetch settings
- Implemented local caching where appropriate

### Challenge 2: Resource Contention
**Problem**: Heavy workloads on shared workers caused performance degradation
**Solution**:
- Created specialized worker pools (CPU-intensive, memory-intensive, IO-intensive)
- Implemented queue-based routing
- Resource limits per task type
- Dynamic worker scaling based on queue depth

### Challenge 3: Monitoring and Observability
**Problem**: Distributed system made troubleshooting difficult
**Solution**:
- Centralized logging (ELK stack)
- Custom metrics and dashboards
- Distributed tracing for complex workflows
- Health check endpoints
- Alert escalation framework

### Challenge 4: DAG Deployment
**Problem**: Manual DAG deployment was error-prone and slow
**Solution**:
- CI/CD pipeline for DAG validation
- Automated testing of DAG syntax
- Blue-green deployment strategy
- Rollback capability
- DAG versioning

### Challenge 5: Database Performance
**Problem**: Metadata database became bottleneck under high load
**Solution**:
- PostgreSQL tuning (connection pooling, query optimization)
- Regular maintenance (vacuum, analyze)
- Read replicas for reporting queries
- Database connection pooling
- Archival of old execution data

## Infrastructure Specifications

### VM Sizing
- **Webservers**: 4 vCPU, 8GB RAM (3 instances)
- **Scheduler**: 8 vCPU, 16GB RAM (1 active + 1 standby)
- **Workers**: 16 vCPU, 32GB RAM (variable pool size)
- **Database**: 8 vCPU, 32GB RAM, SSD storage

### Network Configuration
- Private subnet for backend communication
- VPN access for administrative tasks
- Firewall rules for service isolation
- Load balancer for webserver access

## Key Metrics & Results

- **Pipeline Execution**: 500+ DAGs running daily
- **Task Volume**: 50,000+ tasks executed per day
- **Reliability**: 99.9% scheduler uptime
- **Performance**: < 5 second task scheduling latency
- **Scalability**: Successfully scaled from 10 to 500 DAGs
- **Resource Efficiency**: 70% average worker utilization

## Technologies Used

- **Orchestration**: Apache Airflow 2.x
- **Executor**: Celery
- **Message Broker**: RabbitMQ
- **Metadata DB**: PostgreSQL 13+
- **Infrastructure**: VMware vSphere / cloud VMs
- **Monitoring**: Prometheus, Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Load Balancer**: HAProxy / cloud load balancer

## Operational Patterns

- **Blue-Green Deployments**: For DAG updates
- **Circuit Breaker**: For external service calls
- **Retry with Exponential Backoff**: For transient failures
- **Health Checks**: For all components
- **Graceful Shutdown**: For maintenance windows

## Lessons Learned

1. **Start Small**: Begin with core use cases, expand gradually
2. **Monitoring is Critical**: Invest in observability from day one
3. **Resource Isolation**: Separate worker pools prevent noisy neighbor problems
4. **Database Maintenance**: Regular database tuning is essential
5. **Documentation**: Clear DAG documentation improves maintainability
6. **Testing**: Automated DAG testing catches errors early
7. **Security**: Implement least privilege access from the start

## Best Practices Implemented

- DAG idempotency for reliable reruns
- Sensors with appropriate poke intervals
- Task timeout configurations
- SLA monitoring and alerting
- Connection management via Airflow connections
- Variables for environment-specific configuration
- Task dependencies clearly defined
- Meaningful task/DAG naming conventions

## Future Enhancements

- Migration to container-based deployment (Kubernetes)
- Enhanced auto-scaling capabilities
- Advanced data lineage tracking
- Machine learning for performance optimization
- Integration with data catalog
- Enhanced security (secrets management, RBAC)

## Business Impact

- **Time Savings**: 80% reduction in manual data pipeline management
- **Reliability**: 95% decrease in pipeline failures
- **Visibility**: Complete transparency into data workflows
- **Agility**: New pipelines deployed in hours instead of weeks
- **Cost Efficiency**: Better resource utilization
- **Compliance**: Improved audit trail for data processing

---

*Note: Specific configurations, DAG implementations, and infrastructure details are generalized to respect confidentiality constraints.*