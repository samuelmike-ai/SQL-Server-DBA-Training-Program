# SQL Server DBA Practice Exams

## Overview

This comprehensive practice exam resource contains 100+ questions across multiple difficulty levels and domains of SQL Server Database Administration. Each exam is designed to test practical knowledge and real-world scenarios that DBAs encounter in production environments.

## Exam Structure

### Certification Practice Exam 1: Core DBA Fundamentals (50 Questions)

**Duration:** 90 minutes | **Passing Score:** 80% | **Difficulty:** Intermediate

#### Question 1: Database Configuration and Installation
**Multiple Choice - Single Answer**

A company wants to deploy a new SQL Server instance with the following requirements:
- Support for up to 100 concurrent connections
- Maximum database size of 500GB
- High availability with automatic failover
- Encrypted data at rest and in transit

Which SQL Server edition and configuration would you recommend?

A) SQL Server Express Edition with Always On Basic Availability Groups
B) SQL Server Standard Edition with Always On Availability Groups
C) SQL Server Enterprise Edition with Always On Failover Cluster Instances
D) SQL Server Developer Edition with Database Mirroring

**Correct Answer:** B
**Explanation:** SQL Server Standard Edition supports Always On Availability Groups (up to 2 replicas, basic availability groups for read-scale), provides high availability with automatic failover, and supports encryption. Express Edition has connection limits and size restrictions that don't meet the requirements. Developer Edition is not licensed for production environments. Database Mirroring has been deprecated in favor of Always On technologies.

#### Question 2: Backup and Recovery Strategy
**Multiple Choice - Multiple Answers** (Select all that apply)

Your organization maintains critical financial databases with the following characteristics:
- 24/7 availability requirement
- Recovery Time Objective (RTO) of 15 minutes
- Recovery Point Objective (RPO) of 5 minutes
- Database size: 2TB
- Change rate: 10% daily

Which backup strategies would meet these requirements?

A) Full backup weekly, differential backup daily, log backup every 15 minutes
B) Full backup daily during maintenance window (2 hours available)
C) Full backup weekly, incremental backup every 4 hours, log backup every 5 minutes
D) Full backup monthly, differential backup daily, log backup every 30 minutes
E) Full backup daily, log backup every 5 minutes

**Correct Answers:** A, E
**Explanation:** Option A provides the required RPO through frequent log backups and RTO through differential backups. Option E meets both objectives with daily full backups and frequent log backups. Options B, C, and D don't meet the RPO requirement due to insufficient log backup frequency.

#### Question 3: Index Maintenance and Performance
**Scenario-based Question**

You notice that query performance has degraded over the past month. Investigation reveals:
- Average index fragmentation: 45%
- Page density: 68%
- Missing index requests: 25
- Duplicate indexes: 8
- Last index maintenance: 3 months ago

What is your recommended action plan?

A) Rebuild all indexes immediately during business hours
B) Schedule index maintenance during next maintenance window, implement missing indexes, remove duplicates
C) Only rebuild highly fragmented indexes (>40%), update statistics
D) Create new indexes for all missing index requests

**Correct Answer:** B
**Explanation:** Comprehensive index maintenance should include rebuilding fragmented indexes (45% avg fragmentation warrants attention), implementing missing indexes based on actual query performance, removing duplicate indexes to save space and maintenance time. Rebuilding during business hours could impact performance. Statistics should be updated as part of the maintenance process.

#### Question 4: Security and Access Management
**Practical Scenario**

You need to implement a security model for a healthcare application with these requirements:
- Database contains PHI (Protected Health Information)
- Multiple application tiers require different access levels
- Need to track all data access for audit purposes
- Must comply with HIPAA regulations
- Users should have minimum necessary access

Design the security implementation:

A) Create application-specific roles, implement row-level security, enable audit logging
B) Use Windows authentication only, disable SQL authentication, create database users with specific permissions
C) Implement transparent data encryption, use certificates for connections, create stored procedures for all data access
D) Enable Always Encrypted for sensitive columns, implement dynamic data masking, create custom roles with granular permissions

**Correct Answer:** D
**Explanation:** HIPAA compliance requires encryption at rest and in transit (Always Encrypted, certificates), data masking to limit exposure, granular access control (custom roles), and audit capabilities. Row-level security alone doesn't address encryption requirements. Windows authentication is good but not sufficient for compliance.

#### Question 5: Monitoring and Proactive Maintenance
**Performance Analysis Question**

You're reviewing server metrics from the past week:
- CPU utilization: 70-85% during business hours
- Memory pressure: Frequent paging, available memory <20%
- Disk I/O: Average queue length 8-12, latency 25-45ms
- Blocking: 15-25 blocked processes daily
- Deadlocks: 2-3 per day during peak hours

Identify the top 3 priorities for performance optimization:

A) Increase RAM, optimize queries causing blocking, implement index maintenance
B) Upgrade CPU, replace storage system, create additional indexes
C) Optimize blocking queries, adjust memory allocation, review I/O patterns
D) Increase concurrent connections, enable query optimizer improvements

**Correct Answer:** A
**Explanation:** The metrics indicate memory pressure (causing paging), significant blocking (affecting concurrency), and the need for index maintenance. These are the root causes affecting overall performance. Upgrading hardware without addressing query performance and blocking will provide limited improvement.

### Practice Exam 2: Advanced Administration Topics (50 Questions)

#### Question 6: High Availability and Disaster Recovery
**Complex Scenario**

Your organization runs a global e-commerce platform with these requirements:
- Primary data center in New York
- Secondary data center in London
- Automatic failover capability
- Read-scale workload distribution
- Recovery time <30 seconds
- RPO <1 minute
- Database size: 5TB
- Transaction rate: 50,000 TPS

Which HA/DR solution meets these requirements most cost-effectively?

A) Always On Failover Cluster Instances (FCI) with shared storage
B) Always On Availability Groups with synchronous commit
C) Always On Availability Groups with asynchronous commit and readable secondary
D) Database mirroring with high safety mode

**Correct Answer:** B
**Explanation:** Always On Availability Groups with synchronous commit provides the required RTO (<30 seconds) and RPO (<1 minute). Readable secondary replicas handle read-scale distribution. FCIs require shared storage which isn't suitable for geographically distributed setups. Asynchronous commit doesn't meet RPO requirements. Database mirroring is deprecated and doesn't support multiple secondary replicas.

#### Question 7: Transaction Log Management
**Troubleshooting Question**

A production database suddenly stops responding. Investigation reveals:
- Transaction log file: 100GB
- Available disk space: 50GB
- Last backup: 48 hours ago
- Active log records: 95% of log file
- Replication delay: 5 minutes
- Highest transaction ID: 152789456

What is the immediate action and long-term solution?

A) Expand transaction log file immediately, increase backup frequency
B) Backup transaction log, truncate log, increase log file size, implement regular log backups
C) Shrink log file to free space, run integrity checks
D) Perform full database backup, change recovery model to simple

**Correct Answer:** B
**Explanation:** The transaction log is full with active records. Immediate action: back up the log to release inactive log space. Long-term: implement regular log backups, ensure proper log file sizing, and monitor log space usage. Shrinking is temporary and can cause fragmentation. Simple recovery model may not meet business requirements.

#### Question 8: Capacity Planning and Resource Management
**Data Growth Analysis**

Based on historical data analysis:
- Current database size: 2.5TB
- Monthly growth rate: 15%
- Current users: 500
- Projected users in 12 months: 800
- Current peak transaction rate: 25,000 TPS
- Transaction growth rate: 25% annually

Calculate and recommend:

A) Database size in 12 months: 3.2TB, Recommended disk space: 5TB
B) Database size in 12 months: 3.8TB, Recommended disk space: 6TB with RAID 10
C) Database size in 12 months: 4.1TB, Recommended: Additional server with database sharding
D) Database size in 12 months: 3.5TB, Recommended: Cloud storage with auto-scaling

**Correct Answer:** B
**Explanation:** Calculating 15% monthly growth over 12 months: 2.5TB × (1.15)^12 = 3.8TB. Adding headroom (50%), log files, indexes, and tempdb requires ~6TB total storage. RAID 10 provides performance and redundancy. Cloud auto-scaling could be considered but on-premises RAID 10 provides consistent performance. Sharding may be premature at this scale.

### Certification Practice Exam 3: Performance Tuning and Optimization (50 Questions)

#### Question 9: Query Performance Optimization
**Query Analysis Question**

This query is experiencing poor performance:

```sql
SELECT 
    c.CustomerName,
    o.OrderDate,
    o.TotalAmount,
    p.ProductName
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
INNER JOIN OrderDetails od ON o.OrderID = od.OrderID
INNER JOIN Products p ON od.ProductID = p.ProductID
WHERE o.OrderDate >= '2024-01-01'
    AND c.Region = 'North'
    AND p.Category = 'Electronics'
ORDER BY o.TotalAmount DESC;
```

Current execution time: 45 seconds. Expected: <5 seconds.

What optimizations would you implement?

A) Add non-clustered indexes on OrderDate, Region, and Category columns
B) Create covering indexes including all columns in SELECT and WHERE clauses
C) Add clustered indexes on all join columns, update statistics
D) Implement query hints to force specific join order

**Correct Answer:** B
**Explanation:** A covering index includes all columns referenced in the query (SELECT, WHERE, JOIN, ORDER BY) allowing the query to be satisfied from the index without table access. For this query, create indexes like: (Region, OrderDate) INCLUDE (CustomerName, TotalAmount, ProductName) and (Category, ProductID) INCLUDE (ProductName). Single column indexes won't be as effective for this complex query.

#### Question 10: Wait Statistics Analysis
**Performance Monitoring**

SQL Server wait statistics show:
- PAGEIOLATCH_SH: 35% of wait time
- CXPACKET: 25% of wait time
- LCK_M_S: 20% of wait time
- SOS_SCHEDULER_YIELD: 15% of wait time
- Other waits: 5%

What does this indicate and what actions should you take?

A) High I/O latency - upgrade storage, add indexes
B) Parallel query issues - adjust degree of parallelism, optimize queries
C) Blocking problems - identify and resolve blocking queries
D) CPU pressure - optimize queries, consider hardware upgrade

**Correct Answer:** A
**Explanation:** PAGEIOLATCH_SH is the dominant wait, indicating I/O subsystem issues. Storage performance is the primary bottleneck. Actions: verify storage performance (latency <20ms for OLTP), consider SSD storage, optimize queries to reduce I/O through better indexing and query optimization. CXPACKET waits suggest parallel execution but are secondary to I/O issues.

### Practice Exam 4: Cloud and Modern Technologies (50 Questions)

#### Question 11: Azure SQL Database Management
**Cloud Migration Scenario**

Migrating an on-premises SQL Server to Azure SQL Database. Current configuration:
- Database size: 800GB
- Peak DTU utilization: 95%
- Complex queries with multiple CTEs and window functions
- Real-time reporting requirements
- Need to maintain 99.9% uptime

Migration and sizing recommendations:

A) Azure SQL Database with 100 DTUs, enable query store
B) Azure SQL Database with 200 vCores, configure geo-replication
C) Azure SQL Managed Instance with 8 vCores, enable Active Directory authentication
D) SQL Server on Azure VMs with 16 vCores, implement Always On

**Correct Answer:** B
**Explanation:** Complex queries with real-time reporting requirements suggest high compute needs (200 vCores). Azure SQL Database provides 99.9% uptime SLA. Query Store helps with performance optimization. Managed Instance is for lift-and-shift scenarios but doesn't optimize as well for cloud-native. VMs require more management overhead.

#### Question 12: Always Encrypted Implementation
**Security Implementation**

Implementing Always Encrypted for sensitive customer data:
- Credit card numbers
- Social Security Numbers
- Email addresses
- Phone numbers

Configuration requirements:
- Application doesn't need to decrypt data
- DBAs cannot view sensitive data
- Maintain query performance
- Support encrypted data in reporting

Implementation approach:

A) Use Always Encrypted with deterministic encryption for all columns
B) Use deterministic encryption for equality searches, randomized for complex operations
C) Implement column-level encryption with decrypt functions
D) Use row-level security instead of column encryption

**Correct Answer:** B
**Explanation:** Deterministic encryption allows equality operations (WHERE email = 'user@company.com') while randomized encryption provides stronger security but doesn't support equality. Mixed approach optimizes both security and functionality. Always Encrypted allows application-layer decryption while protecting data from DBAs.

### Practice Exam 5: Comprehensive Scenario Assessment (50 Questions)

#### Question 13: Production Issue Resolution
**Critical Incident Response**

At 2:00 AM, you receive alerts:
- Database unresponsive
- Transaction log 99% full
- Disk space critical (only 2GB remaining)
- 150 blocked processes
- Application timeout errors

Immediate assessment and actions:

A) Expand disk space, backup transaction log, kill blocking processes
B) Stop all applications, backup database, expand log file, restart services
C) Check for long-running transactions, backup log file, kill blockers, add disk space
D) Switch to secondary replica, perform emergency log backup, investigate root cause

**Correct Answer:** C
**Explanation:** Systematic approach: 1) Investigate root cause (long-running transactions holding log space), 2) Backup log to release space, 3) Kill blocking processes, 4) Add disk space. Immediate service disruption should be avoided if possible. Switching to secondary may not be possible if the issue is log-related.

#### Question 14: Database Design and Normalization
**Design Challenge**

Design a database schema for a multi-tenant SaaS application:

Requirements:
- Support 1000+ customers
- Each customer has multiple users
- Shared data with customer-specific data
- Support for tenant isolation
- Scalability to 10,000 customers
- Backup and recovery per customer

Recommended design:

A) Single database with customer ID column, row-level security
B) Separate database per customer
C) Schema per customer within single database
D) Hybrid approach with shared reference data and tenant-specific schemas

**Correct Answer:** D
**Explanation:** Hybrid approach provides best balance: shared reference data in common schemas, tenant-specific data in separate schemas. This allows efficient backup/restore per tenant, supports isolation, maintains performance, and scales well. Single database with row-level security is less maintainable. Per-database approach creates management overhead.

### Practice Exam 6: Compliance and Governance (50 Questions)

#### Question 15: GDPR Compliance Implementation
**Data Protection Regulation**

Implementing GDPR compliance for customer data:
- Right to be forgotten
- Data portability
- Consent tracking
- Audit logging
- Data breach notification
- Privacy by design

Technical implementation requirements:

A) Implement soft deletes, export functions, consent tables, audit triggers
B) Use table partitioning, implement data archiving, enable encryption
C) Create separate PII schemas, implement access logging, design deletion procedures
D) Enable row-level security, create data export views, implement consent management

**Correct Answer:** A
**Explanation:** GDPR requires specific technical controls: soft deletes for "right to be forgotten" (actual deletion after retention period), export functions for data portability, consent tracking tables, comprehensive audit logging. Privacy by design means implementing these controls from the start.

#### Question 16: Audit and Compliance Monitoring
**Continuous Compliance**

Establishing automated compliance monitoring:
- SOX compliance requirements
- PCI DSS for payment data
- HIPAA for healthcare data
- Regular audit reports
- Exception tracking
- Automated alerts for violations

Implementation strategy:

A) Enable SQL Server Audit, create compliance dashboards, implement automated reporting
B) Use third-party compliance tools, configure alerts, create audit procedures
C) Implement custom audit tables, create monitoring views, schedule compliance checks
D) Enable all audit features, configure log shipping to audit server

**Correct Answer:** A
**Explanation:** SQL Server Audit provides comprehensive audit capabilities. Creating dashboards and automated reporting ensures consistent compliance monitoring. Third-party tools can supplement but native audit provides the foundation. Automated alerts enable proactive violation detection.

### Practice Exam Answer Keys and Detailed Explanations

#### Scoring Guide
- **Expert Level (90-100%):** Demonstrates mastery of all DBA concepts
- **Advanced Level (80-89%):** Strong DBA skills with minor knowledge gaps
- **Intermediate Level (70-79%):** Solid foundation with room for improvement
- **Basic Level (60-69%):** Fundamental understanding, needs more study
- **Below Standard (<60%):** Requires significant additional preparation

#### Performance Analysis
After completing each practice exam, review:

1. **Conceptual Understanding:** Are you clear on the principles behind each question?
2. **Practical Application:** Can you apply concepts to real-world scenarios?
3. **Best Practices:** Do your answers reflect industry best practices?
4. **Troubleshooting:** Can you diagnose and resolve complex issues?
5. **Strategic Thinking:** Do you consider long-term implications?

#### Remediation Strategy
For questions answered incorrectly:

1. **Research Phase:** Study relevant documentation and best practices
2. **Hands-on Practice:** Create test environments to validate concepts
3. **Peer Discussion:** Engage with DBA communities for different perspectives
4. **Official Documentation:** Review Microsoft SQL Server documentation
5. **Retest:** Complete similar questions to verify understanding

### Advanced Practice Scenarios

#### Emergency Response Scenarios
Test your ability to respond to critical incidents:

**Scenario 1: Ransomware Attack**
Database files encrypted, ransom note present, backup system compromised
**Scenario 2: Data Center Failure**
Primary data center offline, secondary systems activated
**Scenario 3: Performance Degradation**
Sudden performance drop affecting business operations
**Scenario 4: Security Breach**
Unauthorized access detected, potential data exfiltration

#### Business Continuity Planning
Test strategic planning capabilities:

**Scenario 5: Merger and Acquisition**
Integrating acquired company's database systems
**Scenario 6: Cloud Migration**
Moving critical systems to cloud infrastructure
**Scenario 7: Compliance Audit**
Preparing for regulatory compliance examination
**Scenario 8: Scalability Planning**
Planning for 10x growth in data volume and user base

### Exam Tips and Strategies

#### Time Management
- Spend 1.5-2 minutes per question on average
- Flag difficult questions and return to them
- Don't overthink simple questions
- Leave time for review (10-15 minutes)

#### Question Analysis Techniques
- **Read carefully:** Understand what's being asked
- **Eliminate wrong answers:** Remove obviously incorrect options
- **Look for clues:** Keywords in questions often indicate approach
- **Consider context:** Real-world constraints and requirements
- **Think systematically:** Use logical problem-solving approach

#### Common Pitfalls to Avoid
- Choosing solutions based on incomplete information
- Ignoring business requirements and constraints
- Selecting overly complex solutions when simple ones suffice
- Missing performance and scalability considerations
- Overlooking security and compliance requirements

#### Confidence Building
- Complete all practice exams multiple times
- Join study groups and discussion forums
- Participate in SQL Server community events
- Gain hands-on experience in test environments
- Read SQL Server blogs and expert recommendations

### Additional Practice Resources

#### Recommended Practice Platforms
- **SQL Server Documentation:** Hands-on examples and tutorials
- **Microsoft Learn:** Interactive SQL Server learning paths
- **SQL Server Central:** Community-driven practice questions
- **Brent Ozar Unlimited:** Performance tuning scenarios
- **Redgate SQL Server Central:** Tool-specific demonstrations

#### Hands-on Lab Exercises
1. **Installation and Configuration:** Practice different installation scenarios
2. **Backup and Recovery:** Simulate various failure scenarios
3. **Performance Tuning:** Optimize sample databases with known issues
4. **High Availability:** Set up and test Always On configurations
5. **Security Implementation:** Configure various security models
6. **Monitoring:** Create comprehensive monitoring solutions

This comprehensive practice exam resource provides thorough preparation for SQL Server DBA certification while building practical skills applicable to real-world database administration challenges. Each question is designed to test both theoretical knowledge and practical application, ensuring candidates are well-prepared for certification success and professional excellence.