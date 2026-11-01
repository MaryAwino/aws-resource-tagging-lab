# AWS Managing Resources with Tagging Lab

## Overview

This lab explores how AWS resource tagging can be used to organize resources, manage infrastructure, and enforce cloud governance policies.

Using the AWS Command Line Interface (AWS CLI) and AWS SDK for PHP, I practised identifying EC2 instances based on tags, updating resource metadata, and implementing an automated tag-or-terminate policy.

## Objectives

- Identify EC2 instances using AWS resource tags.
- Filter AWS resources using the AWS CLI.
- Update resource tags programmatically.
- Explore AWS SDK for PHP.
- Understand how automation can enforce resource tagging policies.
- Identify and terminate EC2 instances that do not have a required `Environment` tag.

## Technologies and Services

- Amazon EC2
- AWS CLI
- AWS SDK for PHP
- PHP
- AWS Identity and Access Management (IAM)
- AWS Organizations – Service Control Policies (SCPs)
- Linux Command Line

## Lab Tasks

### Task 1: Find and Update Resources Using Tags

- Identified EC2 instances using the `Project=ERPSystem` tag.
- Filtered instances by their `Environment` tag.
- Identified development instances.
- Updated their `Version` tag from `1.0` to `1.1`.
- Verified the updated tags using AWS CLI.

### Task 2: Stop Instances Using Resource Tags

- Inspected the `stopinator.php` script.
- Explored how the PHP SDK can identify and stop EC2 instances based on tag filters.
- Encountered an AWS Organizations Service Control Policy (SCP) restriction when the script attempted to describe instances in another AWS Region.
- Confirmed that the two development instances in `us-west-2` remained running.

**Result:** The task was partially completed. The script was blocked by an organizational permission restriction.

### Task 3: Challenge – Terminate Non-Compliant Instances

- Inspected the `terminate-instances.php` script.
- Identified the private subnet and its EC2 instances.
- Removed the `Environment` tag from two selected staging instances.
- Corrected an argument-checking issue in the PHP script.
- Executed the script using the AWS SDK for PHP.
- Successfully identified and terminated the two instances without the required tag.
- Verified that both instances reached the `terminated` state.

## Key Learning Outcomes

- Using tags to organize and identify AWS resources.
- Applying AWS CLI filters to retrieve specific EC2 instances.
- Managing resource metadata with `create-tags` and `delete-tags`.
- Using PHP scripts and the AWS SDK to automate infrastructure management.
- Understanding tag-based governance and compliance.
- Understanding how AWS Organizations SCPs can restrict API operations.
- Implementing a tag-or-terminate approach to resource compliance.

## Security Considerations

Resource tagging can support cloud governance by ensuring that resources follow organizational standards.

A tag-or-terminate policy can help identify resources that do not meet required tagging standards. However, termination is destructive and should be implemented with appropriate safeguards, validation, and authorization.

## Lab Results

| Task | Result |
|---|---|
| Task 1 – Resource tagging | Completed |
| Task 2 – Stopinator | Blocked by SCP restriction |
| Task 3 – Tag-or-terminate | Completed |

## Conclusion

This lab provided practical experience in managing AWS resources through tagging and automation.

I gained hands-on experience using AWS CLI commands and the AWS SDK for PHP to identify, update, and manage EC2 instances.

The exercise also demonstrated how resource tags can support automated cloud governance and how organizational security policies can affect infrastructure management operations.

---

**Environment:** AWS Academy Lab  
**Region:** `us-west-2`  
**Project:** AWS Managing Resources with Tagging