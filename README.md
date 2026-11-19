# QPPELIG-5461 – AWS Backup Integrity and Recoverability Validation Report

**Project:** CMS QPP Eligibility  
**Jira Ticket:** QPPELIG-5461  
**Task:** [DevOps] Validate Backup Integrity and Recoverability (PI37)  
**Environment:** AWS Production (Read-Only Validation) and AWS Non-Production (Restore Testing)  
**AWS Region:** us-east-1  
**Validation Period:** November 2026  
**Status:** Infrastructure Restore Testing Completed

---

## 1. Executive Summary

The purpose of QPPELIG-5461 is to validate the integrity and recoverability of AWS backups supporting the CMS QPP Eligibility application.

The validation included reviewing backup configurations, schedules, retention policies, recovery points, backup completion history, encryption, and recovery capabilities.

A representative restore exercise was successfully performed in the AWS non-production environment using an automated Amazon RDS PostgreSQL snapshot.

The restored database instance successfully reached the `available` state, and AWS confirmed that storage encryption was enabled.

Following the infrastructure restore verification, the temporary test database was deleted successfully to prevent unnecessary resource utilization and ongoing AWS charges.

**Overall outcome:** The AWS backup configuration checks and DEV RDS infrastructure-level restore exercise were successful. No failures were identified in the reviewed backup records.

Database connectivity was subsequently reported as verified. Detailed database query results and restored-data comparisons were not included in the AWS CLI evidence.

---

## 2. Objectives

The primary objectives of the validation were to:

1. Review AWS backup configurations and verify backup schedules.
2. Validate backup frequency, retention, and completion history.
3. Identify representative recovery points for testing.
4. Review backup encryption and AWS KMS configurations.
5. Perform a representative restore exercise in a non-production environment.
6. Confirm that the restored infrastructure becomes available.
7. Verify database connectivity.
8. Review available evidence supporting Recovery Point Objective (RPO) and Recovery Time Objective (RTO).
9. Document validation results and findings.
10. Remove temporary testing resources after completion.

These activities support the recoverability requirements for the QPP Eligibility application.

---

## 3. Validation Scope

### 3.1 Production Environment

Production backup validation was performed using read-only AWS CLI commands.

The review included the following RDS databases:

- `qpp-elig-prod`
- `qpp-elig-prod-etl`

The validation focused on existing backup configurations, recovery points, backup plans, retention policies, and related AWS Backup information.

No production databases were restored, modified, restarted, or deleted as part of this exercise.

### 3.2 Non-Production Environment

The restore exercise was performed in the designated non-production AWS account.

| Configuration | Value |
|---|---|
| AWS Account ID | 387191992873 |
| AWS Region | us-east-1 |
| AWS CLI Profile | 387191992873_ct-ado-qpp-cis-devops |
| Source Database | qpp-elig-dev |
| Database Engine | PostgreSQL |
| Source Instance Class | db.r6g.large |
| Automated Backup Retention | 7 days |
| Temporary Restore Instance | qpp-elig-dev-restore-test |

The non-production environment was selected to avoid introducing changes to production infrastructure.

---

## 4. AWS Backup Configuration Validation

### 4.1 Backup Vault Verification

AWS Backup vaults were reviewed using the AWS CLI.

The review identified several existing vaults, including:

- `qpp-prod-backup-vault-us-east-1`
- `qpp-devpre-backup-vault-us-east-1`
- `qpp-dev-backup-vault-us-east-1`
- `qpp-ransomware-backup-vault-us-east-1`

The production backup vault contained existing recovery points.

The AWS CLI results showed that the production backup vault had approximately 1,636 recovery points at the time of validation.

**Result:** Backup vaults and stored recovery points were successfully identified.

### 4.2 Backup Plan Verification

Existing AWS Backup plans were reviewed.

The environment contained multiple backup policies supporting different schedules and retention requirements.

Examples of observed backup plans included:

| Backup Plan | Configured Retention Rules |
|---|---|
| Daily15_Weekly90 | Daily: 15 days; Weekly: 95 days |
| Daily90 | Daily: 90 days |
| 4Hours1_Daily7_Weekly35_Monthly90 | Hourly: 1 day; Daily: 7 days; Weekly: 35 days; Monthly: 90 days |
| Daily15_Weekly90_Monthly365_Yearly730 | Daily: 15 days; Weekly: 95 days; Monthly: 365 days; Yearly: 730 days |
| Daily15_Weekly90_Monthly365 | Daily: 15 days; Weekly: 95 days; Monthly: 365 days |
| Daily15_Weekly90_Monthly180 | Daily: 15 days; Weekly: 95 days; Monthly: 180 days |

The retention values above reflect the AWS Backup rule output. Actual resource protection depends on the applicable plan assignments and recovery points.

**Result:** Backup schedules and retention rules were reviewed successfully.

### 4.3 Backup Selection Verification

AWS Backup selections were inspected to understand how resources are associated with backup plans.

The review identified backup selections using resource tags, including:

- `app:backupplan`
- `aws:ResourceTag/AWS_Backup`

Existing selection configurations were successfully retrieved.

**Result:** AWS Backup selection rules and tag-based configurations were reviewed.

---

## 5. Production RDS Backup Validation

### 5.1 Production Eligibility Database

**Database Identifier:** `qpp-elig-prod`

The production Eligibility database was reviewed for automated backup configuration and recovery-point availability.

The AWS CLI output confirmed:

- Automated backup retention configured for 7 days.
- Existing completed AWS Backup recovery points.
- Recovery-point records available for review.
- Production database available during the relevant validation checks.

The reviewed AWS Backup recovery points showed successful completion statuses.

### 5.2 Production Eligibility ETL Database

**Database Identifier:** `qpp-elig-prod-etl`

The production ETL database was also reviewed.

The AWS CLI output confirmed:

- Automated backup retention configured for 7 days.
- Existing completed backup recovery points.
- Historical AWS Backup recovery records available.

### 5.3 Production Validation Results

| Validation Check | Result |
|---|---|
| Production RDS instances identified | Passed |
| Backup retention reviewed | Passed |
| AWS Backup recovery points identified | Passed |
| Completed recovery points reviewed | Passed |
| Backup plan configurations reviewed | Passed |
| Production database modifications | None performed |

No backup failures were identified in the reviewed records.

---

## 6. Backup Encryption and KMS Validation

Backup encryption and AWS Key Management Service (KMS) configurations were included in the validation.

The non-production source database configuration confirmed:

- Storage encryption enabled.
- AWS KMS key associated with the RDS instance.
- Private network configuration.
- Existing RDS security groups configured.

The restored RDS instance was also verified as encrypted.

The successful snapshot restore demonstrated that the AWS identity used for the operation had the permissions required to initiate the restore using the selected encrypted snapshot.

This does not independently establish that every application or database-level permission is configured correctly.

**Result:** RDS storage encryption and the successful use of the encrypted backup during restore testing were verified.

---

## 7. DEV Backup Snapshot Validation

The non-production Eligibility database was selected for representative restore testing.

**Source Database:** `qpp-elig-dev`

AWS CLI validation confirmed the following:

| Validation Item | Result |
|---|---|
| Database status | Available |
| Database engine | PostgreSQL |
| Automated backup retention | 7 days |
| Automated snapshots | Available |
| Snapshot history | November 10–17, 2026 |
| Snapshot status | Available |

Eight automated snapshots were displayed during validation.

The selected snapshot was:

`rds:qpp-elig-dev-2026-11-17-09-24`

The selected recovery point was available before the restore operation began.

**Result:** A representative automated DEV snapshot was successfully identified and selected.

---

## 8. DEV RDS Restore Testing

### 8.1 Restore Test Objective

The objective of the restore exercise was to confirm that an existing DEV PostgreSQL backup could be used to create a separate, operational RDS instance.

The exercise was performed without overwriting or modifying the original DEV database.

### 8.2 Restore Configuration

| Configuration | Value |
|---|---|
| AWS Account | 387191992873 |
| AWS Region | us-east-1 |
| Source Database | qpp-elig-dev |
| Source Snapshot | rds:qpp-elig-dev-2026-11-17-09-24 |
| Temporary Database | qpp-elig-dev-restore-test |
| Database Engine | PostgreSQL |
| Instance Class | db.r6g.large |
| DB Subnet Group | qpp-elig-dev-db-subnet-group |
| Public Accessibility | Disabled |
| Multi-AZ | Disabled |
| Storage Encryption | Enabled |

The restore used the existing DEV network configuration and security groups.

### 8.3 Restore Execution

The restore operation was initiated using the AWS CLI command:

`aws rds restore-db-instance-from-db-snapshot`

AWS accepted the restore request and returned a new RDS instance record.

The initial status was:

`creating`

This confirmed that the restore request was successfully submitted.

### 8.4 Restore Completion Verification

The following AWS CLI waiter was used:

`aws rds wait db-instance-available`

The waiter completed successfully.

A subsequent RDS description confirmed:

- Database identifier: `qpp-elig-dev-restore-test`
- Database status: `available`
- RDS endpoint generated successfully
- Storage encryption: `True`

**Restore Test Result: PASSED**

The selected DEV automated snapshot successfully restored into an available, encrypted RDS instance.

---

## 9. Database Connectivity Verification

Following the restore exercise, database connectivity verification was coordinated with the development team.

Database connectivity was subsequently reported as verified.

The AWS CLI evidence independently confirmed that the temporary RDS instance reached the `available` state and had an endpoint.

The captured AWS CLI results did not include PostgreSQL login output, SQL query results, or table-level data comparisons.

**Reported Result:** Database connectivity verified.

**Evidence distinction:** The RDS infrastructure status was directly verified through AWS CLI. Database connectivity confirmation was reported separately.

---

## 10. RDS Restore Event Verification

AWS RDS event history was reviewed for the temporary restored instance.

The following events were identified:

| RDS Event | Verification |
|---|---|
| DB instance restarted | Event recorded |
| Restored from snapshot | Event recorded |
| Backing up DB instance | Event recorded |
| Finished DB instance backup | Event recorded |

The restore event history provided supporting evidence that AWS completed the snapshot restore process.

The events also confirmed that the restored instance subsequently participated in normal RDS backup activity.

**Result:** Restore-related RDS events were successfully reviewed.

---

## 11. RPO and RTO Review

### 11.1 Recovery Point Objective (RPO)

Recovery Point Objective represents the maximum acceptable amount of data loss measured in time.

For this exercise, an automated DEV RDS snapshot dated November 17, 2026, was selected.

The snapshot was available and successfully restored.

The validation demonstrated that a historical recovery point could be used to recover the DEV database infrastructure.

The exact RPO compliance result was not calculated because an approved application-specific RPO target and corresponding data comparison were not included in the captured evidence.

### 11.2 Recovery Time Objective (RTO)

Recovery Time Objective represents the maximum acceptable time required to restore service after an interruption.

The temporary RDS instance successfully transitioned from `creating` to `available`.

AWS RDS restore events were collected.

The screenshots establish successful infrastructure recovery, but do not provide a sufficiently precise end-to-end measurement of application recovery time.

### 11.3 Recovery Objectives Summary

| Validation | Result |
|---|---|
| Recovery point identified | Completed |
| Snapshot availability verified | Completed |
| RDS infrastructure restore | Passed |
| Restored instance available | Passed |
| Formal RPO target comparison | Not measured |
| Formal RTO target comparison | Not measured |

The restore exercise provides evidence of technical infrastructure recoverability.

---

## 12. Temporary Restore Instance Cleanup

Following completion of the infrastructure restore verification, the temporary RDS instance was deleted.

**Temporary Database:** `qpp-elig-dev-restore-test`

The deletion was initiated using:

`aws rds delete-db-instance`

The command specified:

- Skip final snapshot.
- Delete automated backups associated with the temporary instance.

AWS accepted the request and reported:

`DBInstanceStatus: deleting`

The deletion process was subsequently verified using:

`aws rds wait db-instance-deleted`

The waiter completed successfully without errors.

This confirmed that the temporary restored RDS instance had been deleted.

### Cleanup Results

| Check | Result |
|---|---|
| Temporary RDS deletion initiated | Passed |
| Deletion completion confirmed | Passed |
| Final snapshot | Skipped |
| Temporary automated backup deletion | Requested |
| Original DEV database targeted for deletion | No |
| Production resources targeted | No |

**Cleanup Result: PASSED**

---

## 13. Validation Results Summary

| Test ID | Validation Activity | Result |
|---|---|---|
| TC-01 | Review AWS Backup vaults | Passed |
| TC-02 | Review backup plans and retention | Passed |
| TC-03 | Review backup selections | Passed |
| TC-04 | Verify production RDS backup records | Passed |
| TC-05 | Review completed recovery points | Passed |
| TC-06 | Verify RDS encryption configuration | Passed |
| TC-07 | Identify DEV automated snapshots | Passed |
| TC-08 | Restore DEV RDS snapshot | Passed |
| TC-09 | Verify restored RDS availability | Passed |
| TC-10 | Verify restored instance encryption | Passed |
| TC-11 | Review RDS restore events | Passed |
| TC-12 | Database connectivity | Reported verified |
| TC-13 | Review RPO/RTO evidence | Reviewed; formal targets not measured |
| TC-14 | Delete temporary restore instance | Passed |
| TC-15 | Confirm deletion completion | Passed |

---

## 14. Findings

The following findings were recorded during the validation:

1. AWS Backup vaults and backup plans were accessible for review.
2. Existing backup retention configurations were identified.
3. Production Eligibility databases had completed recovery points available.
4. The DEV Eligibility PostgreSQL database had multiple available automated snapshots.
5. A representative automated snapshot was successfully restored.
6. The restored RDS instance reached the `available` state.
7. RDS storage encryption remained enabled on the restored instance.
8. AWS recorded the restore event in the RDS event history.
9. Database connectivity was subsequently reported as verified.
10. The temporary restore instance was successfully deleted.
11. No failures were identified in the completed infrastructure-level validation checks.

No remediation tasks were created from the successful infrastructure-level checks.

---

## 15. Supporting Evidence

The following screenshots can be attached to this report.

**Evidence 1 – AWS Backup Vaults and Plans**

Insert the screenshot showing AWS Backup vault names, recovery-point counts, and configured backup plans.

**Evidence 2 – Production Recovery Points**

Insert the screenshot showing completed recovery points for `qpp-elig-prod` and `qpp-elig-prod-etl`.

**Evidence 3 – DEV Automated Snapshots**

Insert the screenshot showing available snapshots for `qpp-elig-dev`, including the November 17, 2026 snapshot.

**Evidence 4 – Restore Configuration**

Insert the screenshot showing the temporary RDS restore request and initial `creating` status.

**Evidence 5 – Successful Restore**

Insert the screenshot showing:

- `qpp-elig-dev-restore-test`
- `available`
- Storage encryption: `True`

**Evidence 6 – RDS Restore Events**

Insert the screenshot showing the `Restored from snapshot` event and related RDS events.

**Evidence 7 – Cleanup Confirmation**

Insert the screenshot showing successful completion of:

`aws rds wait db-instance-deleted`

These screenshots provide the primary AWS CLI evidence supporting the infrastructure validation results.

---

## 16. Conclusion

AWS backup validation and the representative DEV RDS infrastructure restore exercise were completed successfully for QPPELIG-5461.

The validation confirmed that existing AWS Backup configurations and recovery points could be reviewed, an available DEV PostgreSQL snapshot could be restored, and the restored RDS instance could reach an operational infrastructure state with encryption enabled.

Database connectivity was subsequently reported as verified.

The temporary restored instance was successfully removed after testing, and no production infrastructure changes were performed.

**Final Infrastructure Validation Result: PASSED**

**Jira Reference:** QPPELIG-5461  
**Report Type:** Backup Integrity and Recoverability Validation  
**Environment:** AWS Non-Production Restore Test / Production Read-Only Backup Review  
**Validation Period:** November 2026
