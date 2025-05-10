# AWS Backup SAP HANA Pre-Check Automation - Design Doc

## 1. Executive Summary/Purpose

The AWS Backup SAP HANA Pre-Check Automation solution aims to reduce SSMSAP and AWS backup configuration issue identification time for SAP HANA systems from weeks to minutes through an automated SSM Document that executes pre-checks, reports issues, suggests mitigation, and collects logs. This solution provides a comprehensive validation tool that identifies configuration issues in SAP HANA environments before they impact backup operations, significantly improving customer experience and reducing support overhead.

### 1.1 BACKGROUND

SAP HANA backup using AWS Backup requires proper configuration across multiple components including the SAP HANA database, AWS Systems Manager Agent, network connectivity, IAM permissions, Python environment, storage configuration, and SAP Host Agent. Currently, when backups fail, troubleshooting requires manual verification of dozens of configuration elements across these components.

This manual process is time-consuming, error-prone, and often leads to extended resolution times, potentially taking weeks to resolve issues that could be identified within minutes using an automated approach.

### 1.2 PROBLEM STATEMENT

SAP backup and restore operations face configuration-related failures that impact customer operations. The current troubleshooting process requires manual log collection and analysis, extending resolution time to weeks and impacting customer backup schedules and potentially causing RPO breaches.

Support data indicates that the most common configuration issues involve:

- SAP HANA database accessibility and permissions
- AWS Backint Agent installation and configuration
- Network connectivity issues
- IAM permission problems
- Python environment configuration
- Storage configuration issues
- SAP Host Agent problems

### 1.3 CURRENT SITUATION

Currently, there is no automated validation solution to verify SAP HANA environments are properly configured for AWS Backup. When customers experience backup failures, the troubleshooting process relies on:

1. Manual checklists executed by engineers
2. Reactive troubleshooting only after failures have occurred
3. Multiple rounds of log collection and configuration verification
4. Siloed diagnostic approaches that vary by support engineer

This manual approach creates inefficiencies, extends resolution times, and leaves customers vulnerable to preventable backup failures. A systematic, automated pre-check solution would address these issues proactively.

## 2. Goals

The goal is to develop an AWS Systems Manager Automation document that performs comprehensive configuration checks on all components required for successful SAP HANA backups. This solution will be implemented following a phased approach as outlined in the project proposal, with this design focused on Phase 1 implementation while keeping extensibility for Phases 2 and 3 in mind.

The solution will run troubleshooting scripts, collect necessary logs, provide mitigation recommendations, and optionally execute additional diagnostic scripts like HANASupportInfo.sh to enhance troubleshooting capabilities.

### 2.1 USE CASES

1. **Pre-Implementation Validation**
   - A customer is setting up AWS Backup for SAP HANA and wants to verify their environment is properly configured
   - The automated pre-check identifies and helps resolve configuration issues before the first backup attempt

2. **Backup Failure Troubleshooting**
   - A customer reports SAP HANA backup failures to AWS Support
   - Support engineers instruct the customer to run the pre-check automation to quickly find configuration issues

3. **Periodic Health Checks**
   - A customer uses the pre-check as part of a scheduled maintenance routine
   - Automated validation identifies drift in configuration that could affect backup reliability

4. **Post-Change Validation**
   - After system patching or configuration changes, administrators verify that SAP HANA backup functionality remains intact
   - The pre-check automation identifies any unintended consequences of recent changes

### 2.2 ARCHITECTURE

The solution architecture consists of the following components shown in the workflow diagram:

**AWS Systems Manager Automation Document**
- Orchestrates the overall pre-check process through a series of sequential validation steps
- Provides parameters for customization (e.g., target instance, SID, S3 bucket location, link to S3 Uploader Tool provided by Premium Support team)
- Manages the workflow across different validation modules implemented as aws:runCommand actions
- Consists of the following sequential check steps:
  - **RetrieveCredentials** - Securely retrieves SAP HANA database credentials from AWS Secrets Manager
  - **PrepareEnvironment** - Sets up the directory structure and environment variables
  - **CheckEnvironmentandOSVersioning** - Validates OS environment and SAP directory structure
  - **CheckSAPHostAgent** - Validates SAP Host Agent installation and status
  - **NetworkingCheck** - Tests network connectivity, checks existence of recommended VPC endpoints, checks connectivity to AWS Backup service endpoint
  - **CheckClusterConfiguration** - Detects and verifies high-availability clustering for SAP HANA
  - **CheckHanaStatus** - Verifies SAP HANA database status and processes
  - **CheckBackupConfiguration** - Examines the SAP HANA database's backup configuration  
  - **CheckUserPermissions** - Validates necessary database permissions
  - **CheckBackintAgent** - Verifies AWS Backint Agent installation and configuration
  - **CheckLogFiles** - Validates log file structure, verifies presence of important SSM and SAP HANA logs, and status
  - **CheckMemorySettings** - Validates memory configuration
  - **CheckStorageAndResources** - Validates storage configuration and resources
  - **RunHANASupportInfo** - Optionally executes the SAP HANA Support Information tool
  - **CreateAndUploadArchive** - Creates and uploads one compressed file to the customer's designated S3 Bucket and/or S3 Uploader Tool Link from customer's Premium Support case

**Shell Script Modules**
- Each check module is implemented as a shell script executed via aws:runCommand
- Performs specific validation tasks and produces detailed result files
- Generates structured output in both human-readable and machine-processable formats

**Log Collection Module**
- Collects relevant logs from SAP HANA, AWS Backint Agent, and AWS Systems Manager
- Implements intelligent log truncation for large files (only taking last 10,000 lines)
- Creates a consolidated archive containing all check results and relevant logs
- Handles secure transmission to customer's S3 bucket or AWS Support via S3 Uploader tool
- Ensures archives are deleted after 7 days and that its maximum size is 50 MB upon re-engagement

**Final Output and Reporting**
- Comprehensive YAML health report for programmatic analysis
- Human-readable text summaries of all findings
- HTML-formatted report for complex issue visualization (when AI remediation is enabled)
- S3 bucket storage of all artifacts for persistence and sharing

**Detailed Data Flow:**

1. Customer initiates pre-check via AWS Systems Manager console, CLI, or API with parameters:
   - InstanceId - EC2 instance running SAP HANA
   - SID - SAP system ID
   - SecretArn - AWS Secrets Manager ARN containing HANA credentials
   - S3BucketForLogs - S3 bucket for storing results (optional)
   - S3UploaderLink - Link for AWS Support case upload (optional)
   - HANASupportInfoS3Bucket - S3 bucket containing HANASupportInfo script (optional)

2. SSM Automation document orchestrates sequential check modules:
   - Each module produces specific validation results
   - Modules implement a fail-forward approach, continuing to the next check even if the current one fails
   - Results are stored in a temporary directory on the instance

3. Log collection module consolidates findings:
   - Generates a consolidated YAML health report
   - Collects relevant logs, truncating large files as needed
   - Creates a compressed archive of all data
   - Uploads to S3 bucket and/or AWS Support case via S3 Uploader Tool link

4. Final results are available to the customer via:
   - SSM Automation execution output
   - S3 bucket for full diagnostic data
   - AWS Support case if S3UploaderLink was provided

## 3. Design Details

### 3.1 FUNCTIONAL REQUIREMENTS

#### 3.1.1 Systems Manager Automation Document

The solution implements a comprehensive workflow with the following components:

1. **Parameter Handling**
   - Required parameters: InstanceId, SID
   - Optional parameters: SecretArn, S3BucketForLogs, S3UploaderLink
   - Generated parameters: Timestamp (automatically generated from execution ID)
   - Added parameter: HANASupportInfoS3Bucket (Optional S3 bucket containing HANASupportInfo.sh script)

2. **Validation Steps Sequence**

The document implements a fail-forward approach with the following sequence:

```
RetrieveCredentials
↓
PrepareEnvironment
↓
CheckEnvironmentandOSVersioning
↓
CheckSAPHostAgent
↓
NetworkingCheck
↓
CheckClusterConfiguration
↓
CheckHanaStatus
↓
CheckBackupConfiguration
↓
CheckUserPermissions
↓
CheckBackintAgent
↓
CheckLogFiles
↓
CheckMemorySettings
↓
CheckStorageAndResources
↓
RunHANASupportInfo (optional)
↓
CreateAndUploadArchive
↓
End
```

**Error Handling and Resilience**
- Each check step includes `onFailure` handlers to continue to the next step
- Comprehensive logging of both successful and failed validations
- Structured error codes and descriptive messaging

#### 3.1.2 Component Validation

The solution validates the following components essential for SAP HANA backups:

1. **Credential Retrieval** (RetrieveCredentials)
   - Securely retrieves SAP HANA database credentials from AWS Secrets Manager
   - Uses the AWS CLI to call the `get-secret-value` API
   - Validates username and password were successfully retrieved
   - Stores credentials securely for use in subsequent steps

2. **Environment Configuration** (CheckEnvironmentandOSVersioning)
   - Validates OS environment and Python setup
   - Verifies required directories exist (e.g., `/usr/sap/$SID`, /hana/data/$SID)
   - Checks for SAP system user (${SID,,}adm)
   - Validates Python and pip installation and versions
   - Verifies hostname configuration and presence in /etc/hosts
   - Checks kernel parameter settings that affect SAP HANA performance

3. **SAP Host Agent** (CheckSAPHostAgent)
   - Verifies installation and correct version via /usr/sap/hostctrl/exe/saphostexec -version
   - Checks running SAP instances via /usr/sap/hostctrl/exe/saphostctrl
   - Validates service status for running instances

4. **Network Connectivity** (NetworkingCheck)
   - Tests connectivity to AWS Backup service endpoint
   - Verifies VPC endpoints configuration (S3, Secrets Manager)
   - Checks DNS resolution for Backup service endpoint
   - Validates connectivity to the internet (if needed)

5. **Cluster Configuration** (CheckClusterConfiguration)
   - Identifies presence of clustering software (crm for SUSE, pcs for RHEL)
   - Verifies SAP HANA cluster resources exist
   - Determines if the system is a multi-node deployment
   - Validates proper integration with the cluster manager

6. **SAP HANA Database** (CheckHanaStatus)
   - Verifies database installation and accessibility
   - Checks HANA processes status (hdbnameserver, hdbdaemon)
   - Validates system replication status via HDBSettings.sh systemReplicationStatus.py
   - Checks tenant database status by querying SYSTEMDB

7. **Backup Configuration** (CheckBackupConfiguration)
   - Queries the M_BACKUP_CATALOG view to check for existing successful backups
   - Examines global.ini configuration file for log backup settings
   - Verifies log_mode is not set to "overwrite" (which would prevent log backups)
   - Checks auto_log_backup settings and log_backup_interval values

8. **Database Permissions** (CheckUserPermissions)
   - Validates required backup privileges (BACKUP ADMIN, CATALOG READ, RECOVERY ADMIN)
   - Checks for necessary backup/recovery roles
   - Examines permissions on both system and tenant databases
   - Provides remediation guidance for missing permissions

9. **AWS Backint Agent** (CheckBackintAgent)
   - Verifies agent binary at /usr/bin/aws-backint
   - Validates configuration files in `/usr/sap/$SID/SYS/global/hdb/opt/`
   - Checks SAP HANA global.ini [backint] section configuration
   - Verifies file permissions and ownership

10. **Log Files** (CheckLogFiles)
    - Validates presence and status of critical log files
    - Checks SAP HANA log segments status
    - Examines AWS Backint Agent logs
    - Analyzes log volume and retention

11. **Memory Settings** (CheckMemorySettings)
    - Analyzes system memory configuration and usage
    - Compares against SAP HANA database size
    - Validates instance type memory capacity against database requirements
    - Checks Backint Agent concurrency settings

12. **Storage and Resources** (CheckStorageAndResources)
    - Validates required mount points (`/hana/data`, `/hana/log`, `/hana/shared`, /usr/sap)
    - Checks XFS filesystem for data and log volumes
    - Analyzes storage capacity and usage
    - Tests I/O performance

13. **HANASupportInfo Collection** (RunHANASupportInfo)
    - Optionally executes the HANASupportInfo script to gather enhanced diagnostic data
    - Collects comprehensive system state information
    - Preserves output for inclusion in the diagnostic archive

#### 3.1.3 Log Collection

The `CreateAndUploadArchive` step implements a comprehensive log collection process:

1. **Collection Structure**
   - Organized by component categories
   - Truncation of large log files (last 10,000 lines) for portability
   - Detailed collection summary and statistics

2. **Collected Log Files**
   ```
   SSM Agent Logs
   ○ /var/log/amazon/ssm/amazon-ssm-agent.log
   ○ /var/log/amazon/ssm/errors.log
   ○ /var/log/amazon/ssm/AmazonSSMAgent-update.txt
   
   SSM4SAP Logs
   ○ /usr/bin/ssm-sap/logs/backup_hana.log
   ○ /usr/bin/ssm-sap/logs/backup_hana_retrieve_metadata.log
   ○ Installation logs for AWS Backint Agent
   
   AWS Backint Agent Logs
   ○ AWS Backint Agent executables and configuration
   ○ AWS Backint Agent logs and catalog
   
   CDPRO/CDGLO Logs
   ○ SAP profiles and configuration
   ○ Backint configuration files
   ○ global.ini and tenant configurations
   
   System DB Backup Logs
   ○ Backup logs for system database
   ○ Backint logs for system database
   
   Tenant System DB and HDB Backup Logs
   ○ Dynamically discovered tenant databases
   ○ Backup and backint logs for each tenant
   
   System Logs
   ○ /var/log/messages
   ○ /var/log/audit/audit.log
   ○ Other system logs relevant to SAP HANA operation
   ```

3. **Output Handling**
   - Compressed archive with all collected logs and check results
   - Upload to customer-specified S3 bucket
   - Optional upload to AWS Support via S3 Uploader link
   - Comprehensive report of collection process

### 3.2 VALIDATION LOGIC FORMALIZATION

The complete validation process can be expressed as a series of logical predicates that define system behavior. This formal approach allows us to reason about the execution flow, dependencies, and failure handling in a precise manner.

#### Define:
- *V(c) = "Component c is valid"*
- *W(c) = "Component c has warnings"*
- *E(c) = "Component c has errors"*
- *S(s) = "Step s in the workflow"*
- *Comp(s) = "Set of components validated in step s"*
- *Exec(s) = "Step s is executed"*
- *Succ(s) = "Step s completes successfully"*
- *Fail(s) = "Step s fails"*
- *Next(s₁, s₂) = "Step s₂ follows step s₁ in normal flow"*
- *OnFail(s₁, s₂) = "If step s₁ fails, go to step s₂"*

#### Component Categories (C):
```
C₁ = {credentials_retrieval, secret_manager_access} // Credentials
C₂ = {utility_functions, environment_variables, directory_structure} // Environment Setup
C₃ = {python_env, pip_installation, dependencies, directories, user_accounts, os_version} // OS Environment
C₄ = {sap_hostagent_installation, hostagent_version, saphostexec, saphostctrl} // SAP Host Agent
C₅ = {backup_endpoint_connection, vpc_endpoints, dns_resolution, network_connectivity} // Networking
C₆ = {cluster_detection, saphana_resources, multi_node_detection} // Cluster Configuration
C₇ = {hana_installation, hana_services, hdbnameserver, hdbdaemon, system_replication} // HANA Status
C₈ = {backup_catalog, log_mode, auto_log_backup, backint_agent_config} // Backup Configuration
C₉ = {backup_admin_privilege, catalog_read_privilege, recovery_admin_privilege, tenant_privileges} // Permissions
C₁₀ = {backint_agent_binary, backint_config_file, params_file, global_ini_config, permissions} // Backint Agent
C₁₁ = {trace_logs, backup_logs, agent_logs, system_logs, log_backup_segments} // Log Files
C₁₂ = {memory_allocation, total_memory, db_size_ratio, concurrency_settings} // Memory Settings
C₁₃ = {mountpoints, filesystem_type, capacity, io_performance} // Storage Resources
C₁₄ = {script_execution, output_collection} // HANASupportInfo
C₁₅ = {archive_creation, s3_upload, report_generation} // Archive and Upload
```

#### Steps (S):
```
S = {
  s₁: RetrieveCredentials,
  s₂: PrepareEnvironment,
  s₃: CheckEnvironmentandOSVersioning,
  s₄: CheckSAPHostAgent,
  s₅: NetworkingCheck,
  s₆: CheckClusterConfiguration,
  s₇: CheckHanaStatus,
  s₈: CheckBackupConfiguration,
  s₉: CheckUserPermissions,
  s₁₀: CheckBackintAgent,
  s₁₁: CheckLogFiles,
  s₁₂: CheckMemorySettings,
  s₁₃: CheckStorageAndResources,
  s₁₄: RunHANASupportInfo,
  s₁₅: CreateAndUploadArchive,
  s₁₆: End
}
```

#### Step-Component Mapping:
```
Comp(s₁) = C₁  // RetrieveCredentials validates credential components
Comp(s₂) = C₂  // PrepareEnvironment validates environment setup components
Comp(s₃) = C₃  // CheckEnvironmentandOSVersioning validates OS environment components
Comp(s₄) = C₄  // CheckSAPHostAgent validates SAP Host Agent components
Comp(s₅) = C₅  // NetworkingCheck validates networking components
Comp(s₆) = C₆  // CheckClusterConfiguration validates cluster components
Comp(s₇) = C₇  // CheckHanaStatus validates HANA status components
Comp(s₈) = C₈  // CheckBackupConfiguration validates backup configuration components
Comp(s₉) = C₉  // CheckUserPermissions validates permission components
Comp(s₁₀) = C₁₀ // CheckBackintAgent validates Backint Agent components
Comp(s₁₁) = C₁₁ // CheckLogFiles validates log file components
Comp(s₁₂) = C₁₂ // CheckMemorySettings validates memory setting components
Comp(s₁₃) = C₁₃ // CheckStorageAndResources validates storage resource components
Comp(s₁₄) = C₁₄ // RunHANASupportInfo validates script execution components
Comp(s₁₅) = C₁₅ // CreateAndUploadArchive validates archive and upload components
```

#### Normal Workflow Sequence:
```
∀i ∈ {1, 2, ..., 14}: Next(sᵢ, sᵢ₊₁)
Next(s₁₅, s₁₆)
```

#### Fail-Forward Behavior (OnFailure Handlers):
```
∀i ∈ {1, 2, ..., 14}: OnFail(sᵢ, sᵢ₊₁)
OnFail(s₁₅, s₁₆)
```

#### Component Validation Logic:
```
∀c ∈ C: V(c) ⟺ ¬W(c) ∧ ¬E(c)  // Component is valid iff it has no warnings or errors
∀c ∈ C: E(c) ⟹ report_error(c) // Any component error is reported
∀c ∈ C: W(c) ⟹ report_warning(c) // Any component warning is reported
```

#### Step Validation Logic:
```
∀s ∈ S: Step_Valid(s) ⟺ ∀c ∈ Comp(s): V(c)  // Step is valid iff all its components are valid
∀s ∈ S: Step_Warning(s) ⟺ (∃c ∈ Comp(s): W(c)) ∧ ¬(∃c ∈ Comp(s): E(c))  // Step has warnings iff at least one component has warnings but none have errors
∀s ∈ S: Step_Error(s) ⟺ ∃c ∈ Comp(s): E(c)  // Step has errors iff at least one component has errors
```

#### Step Execution Status:
```
∀s ∈ S: Step_Error(s) ⟹ Fail(s)  // Step fails if it has errors
∀s ∈ S: Step_Warning(s) ⟹ Fail(s)  // Step fails if it has warnings
∀s ∈ S: Step_Valid(s) ⟹ Succ(s)  // Step succeeds if it's valid
```

#### Workflow Execution Logic:
```
∀i,j ∈ {1, 2, ..., 15}: (Exec(sᵢ) ∧ Next(sᵢ, sⱼ) ∧ Succ(sᵢ)) ⟹ Exec(sⱼ)  // Normal flow: if current step succeeds, execute next step
∀i,j ∈ {1, 2, ..., 15}: (Exec(sᵢ) ∧ OnFail(sᵢ, sⱼ) ∧ Fail(sᵢ)) ⟹ Exec(sⱼ)  // Failure flow: if current step fails, execute onFailure step
```

#### Exit Code Determination:
```
∀s ∈ S: Step_Error(s) ⟹ exit_code(s) = 1  // Exit code 1 (failure) if step has errors
∀s ∈ S: Step_Warning(s) ⟹ exit_code(s) = 1  // Exit code 1 (failure) if step has warnings
∀s ∈ S: Step_Valid(s) ⟹ exit_code(s) = 0  // Exit code 0 (success) if step is valid
```

#### Dependencies Between Steps:
```
Exec(s₂) ⟹ ∃c ∈ Comp(s₁): V(c)  // PrepareEnvironment requires some valid credentials from RetrieveCredentials
Exec(s₇) ⟹ ∃c ∈ Comp(s₄): V(c)  // CheckHanaStatus requires some valid SAP Host Agent components
Exec(s₉) ⟹ ∃c ∈ Comp(s₇): V(c)  // CheckUserPermissions requires some valid HANA Status components
Exec(s₁₅) ⟹ ∀i ∈ {1, 2, ..., 14}: Exec(sᵢ)  // CreateAndUploadArchive requires all previous steps to be executed
```

#### HANASupportInfo Integration Logic:

**Define:**
- *H = "HANASupportInfoS3Bucket parameter has a value"*
- *D(s) = "Download script s succeeds"*
- *R(s) = "Running script s succeeds"*
- *P(s₁₄) = "Proceed with RunHANASupportInfo step"*
- *Skip(s₁₄) = "Skip RunHANASupportInfo step"*

**Decision Flow:**
```
H ⟹ attempt_download()  // If bucket specified, attempt download
¬H ⟹ Skip(s₁₄)  // If no bucket specified, skip entirely
H ∧ D(HANASupportInfo) ⟹ P(s₁₄)  // If download succeeds, proceed with step
H ∧ ¬D(HANASupportInfo) ⟹ report_error() ∧ Skip(s₁₄)  // If download fails, report error and skip
P(s₁₄) ∧ R(HANASupportInfo) ⟹ collect_output()  // If run succeeds, collect output
P(s₁₄) ∧ ¬R(HANASupportInfo) ⟹ report_error()  // If run fails, report error
(P(s₁₄) ∨ Skip(s₁₄)) ⟹ Exec(s₁₅)  // In either case, proceed to the next step
```

#### Key Properties:
1. The workflow implements a fail-forward approach where execution continues to the next step regardless of the current step's success or failure
2. Component validation results determine step success/failure status
3. The system maintains comprehensive counters for errors and warnings
4. Each step focuses on a specific set of components essential for SAP HANA backups
5. HANASupportInfo execution is optional and conditionally executed
6. The final archive consolidates results regardless of individual step outcomes

## 4. Appendices

### 4.1 TENETS

1. **Customer Obsession** - Prioritize validation of components most frequently associated with backup failures to address the most common customer pain points.
2. **Frugality** - Minimize operational overhead for both customers and support teams.
3. **Earn Trust** - The solution operates with minimal permissions and protects sensitive customer data.

### 4.2 FAQ

**Q: Will this automation make changes to the customer's environment?**
A: No, this is a read-only validation tool that will not modify any configurations.

**Q: How long will the pre-check automation take to run?**
A: Typically less than 5 minutes, depending on the size and complexity of the SAP HANA environment.

**Q: Can this be scheduled to run periodically?**
A: Yes, customers can schedule the automation to run at regular intervals using Systems Manager Maintenance Windows.

**Q: What data is being collected, and how is it handled?**
A: The automation first creates a temporary directory to store the results of our diagnostic checks which will be referenced in the "health_check_report.yaml" file which contains the concatenated results of all the diagnostic output contained in this directory for easy reference. It also collects full configuration files, SAP HANA logs, SSM Agent logs, and AWS Backint Agent logs (limited to the most recent 10,000 lines for portability purposes.) These logs and diagnostic results are sent to a single compressed file and uploaded to the customer's designated S3 Bucket and/or the AWS Support S3 Uploader Tool link provided in the customer's related support case. Logs and diagnostic outputs are sent to be archived upon subsequent runs of this automation. Archives are deleted after 7 days. Archive directory maintains a 50 MB size maximum.

**Q: Does this replace the need for AWS Support in troubleshooting backup failures?**
A: While this tool significantly reduces the need for support intervention, it complements rather than replaces AWS Support. Complex issues may still require support assistance, but the pre-check report will accelerate resolution.

**Q: How does the HANASupportInfo.sh script integration work?**
A: If the customer provides an S3 bucket containing the HANASupportInfo.sh script via the optional parameter, the automation will download and execute this script to gather enhanced diagnostic information. The results will be included in the same diagnostic archive produced by the pre-check automation.

### 4.3 REFERENCES

1. AWS Systems Manager for SAP documentation
2. SAP HANA backup on Amazon EC2 documentation
3. AWS Backint Agent for SAP HANA Playbook
4. Support escalation procedure for SAP HANA backups

### 4.4 GLOSSARY

| Term | Definition |
|------|------------|
| AWS Backup | A fully managed backup service that centralizes and automates data backup across AWS services |
| Backint | SAP-certified backup interface for database integration with third-party backup tools |
| AWS Backint Agent | AWS implementation of the SAP Backint interface for SAP HANA backup |
| SAP HANA | High-performance in-memory database from SAP |
| SAP Host Agent | Agent software that enables infrastructure monitoring and management of SAP systems |
| Systems Manager Agent | AWS agent that enables Systems Manager capabilities on EC2 instances |
| Systems Manager Automation | AWS service that provides predefined workflows for common maintenance tasks |
| HANASupportInfo.sh | SAP HANA diagnostic script that gathers comprehensive system information |
