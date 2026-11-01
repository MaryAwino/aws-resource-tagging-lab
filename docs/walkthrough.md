# AWS Managing Resources with Tagging – Detailed Walkthrough

## 1. Introduction

In this lab, I explored how AWS resource tagging can be used to organize cloud resources and automate infrastructure management.

I worked with Amazon EC2 instances using the AWS Command Line Interface (AWS CLI) and the AWS SDK for PHP.

The main objective was to understand how resource tags can help identify resources, update metadata, and enforce compliance through automation.

### Lab Environment

| Component | Configuration |
|---|---|
| Cloud provider | Amazon Web Services |
| Service | Amazon EC2 |
| Region | `us-west-2` |
| Operating environment | Linux CommandHost |
| Command-line tool | AWS CLI |
| Programming language | PHP |
| SDK | AWS SDK for PHP |
| Project tag | `ERPSystem` |

---

## 2. Task 1: Find and Update Resources Using Tags

### Step 1: Verify AWS identity

I started by checking the AWS identity associated with my lab environment.

```bash
aws sts get-caller-identity
```

**What this command does:**

- `aws` invokes the AWS CLI.
- `sts` accesses AWS Security Token Service.
- `get-caller-identity` displays the identity associated with the current credentials.

This helped confirm that the AWS CLI was authenticated.

### Step 2: Identify EC2 instances using tags

The objective was to find instances belonging to the `ERPSystem` project.

I used the following command:

```bash
aws ec2 describe-instances \
--filters "Name=tag:Project,Values=ERPSystem" \
--query 'Reservations[*].Instances[*].[InstanceId,Placement.AvailabilityZone,State.Name]' \
--output table
```

**Explanation:**

- `describe-instances` retrieves EC2 instance information.
- `--filters` limits the results to instances with the specified tag.
- `Name=tag:Project` identifies the tag key.
- `Values=ERPSystem` specifies the required tag value.
- `--query` selects the information to display.
- `--output table` presents the results in a readable table.

I identified seven EC2 instances associated with the `ERPSystem` project.

### Step 3: Filter development instances

I then narrowed the results to instances with the `Environment=development` tag.

```bash
aws ec2 describe-instances \
--filters "Name=tag:Project,Values=ERPSystem" \
"Name=tag:Environment,Values=development" \
--query 'Reservations[*].Instances[*].InstanceId' \
--output text
```

This returned two development instances:

- `i-0d0390ef9762b0e58`
- `i-00759b59b3fed3963`

### Step 4: Understand the tagging script

I navigated to the `aws-tools` directory and inspected the script.

```bash
cd aws-tools
ls
```

The directory contained:

- `composer.json`
- `composer.phar`
- `composer.lock`
- `stopinator.php`
- `terminate-instances.php`
- `vendor/`

I inspected the resource-tagging script:

```bash
cat change-resource-tags.sh
```

The script contained:

```bash
#!/bin/bash

ids=$(aws ec2 describe-instances \
--filter "Name=tag:Project,Values=ERPSystem" \
"Name=tag:Environment,Values=development" \
--query 'Reservations[*].Instances[*].InstanceId' \
--output text)

aws ec2 create-tags --resources $ids \
--tags 'Key=Version,Value=1.1'
```

**How the script works:**

1. Retrieves EC2 instances belonging to the ERPSystem project.
2. Filters the instances by the development environment.
3. Extracts their instance IDs.
4. Uses `aws ec2 create-tags` to update their Version tag to `1.1`.

I made the script executable:

```bash
chmod +x change-resource-tags.sh
```

Then executed it:

```bash
./change-resource-tags.sh
```

### Step 5: Verify the updated tags

I checked the development instances again:

```bash
aws ec2 describe-instances \
--filters "Name=tag:Project,Values=ERPSystem" \
"Name=tag:Environment,Values=development" \
--query 'Reservations[*].Instances[*].{ID:InstanceId,Version:Tags[?Key==`Version`] | [0].Value}' \
--output table
```

**Result:**

Both development instances were successfully updated from Version `1.0` to `1.1`.

This demonstrated how AWS CLI scripts can automate repetitive resource-tagging operations.

---

## 3. Task 2: Stop Instances Using Resource Tags

The next task introduced the `stopinator.php` script.

The objective was to use PHP and the AWS SDK to identify and stop EC2 instances based on their resource tags.

### Step 1: Inspect the PHP script

I inspected the script:

```bash
cat stopinator.php
```

The script used the AWS SDK for PHP to:

- Discover AWS Regions.
- Describe EC2 instances.
- Filter resources using tag criteria.
- Stop matching EC2 instances.

The tag filter used was:

```text
Project=ERPSystem;Environment=development
```

### Step 2: Execute the script

I ran:

```bash
./stopinator.php -t"Project=ERPSystem;Environment=development"
```

### Step 3: Investigate the error

The script failed when attempting to access the `ap-south-1` Region.

The AWS response returned:

```text
UnauthorizedOperation
```

The error indicated that an AWS Organizations Service Control Policy (SCP) explicitly denied the `ec2:DescribeInstances` operation.

### Step 4: Investigate regional access

I checked the configured AWS CLI Region:

```bash
aws configure get region
```

The configured Region was:

```text
us-west-2
```

I then tested EC2 access directly in that Region:

```bash
aws ec2 describe-instances --region us-west-2 \
--filter "Name=tag:Project,Values=ERPSystem" \
"Name=tag:Environment,Values=development" \
--query 'Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,Version:Tags[?Key==`Version`] | [0].Value}'
```

The two development instances were still running and had Version `1.1`.

I also tested PHP SDK access in `us-west-2`:

```bash
php -r 'require "aws-tools/vendor/autoload.php"; $ec2 = Aws\Ec2\Ec2Client::factory(array("region"=>"us-west-2")); $r = $ec2->describeInstances(); echo "PHP SDK access to us-west-2 successful\n";'
```

The test succeeded.

**Why did the original script fail?**

The script iterated through all available AWS Regions. Although PHP SDK access to `us-west-2` worked, the script encountered an organizational restriction when accessing `ap-south-1`.

**Result:** Task 2 could not be completed using the original script because of the SCP restriction.

This demonstrated that AWS Organizations policies can restrict API operations even when access to another Region is permitted.

---

## 4. Task 3: Challenge – Terminate Non-Compliant Instances

The final task focused on implementing a tag-or-terminate policy.

The scenario was that an organization wanted to automatically terminate EC2 instances that did not have the required `Environment` tag.

### Step 1: Inspect the termination script

I inspected the script:

```bash
cat terminate-instances.php
```

The script accepted two arguments:

- `-region`: The AWS Region.
- `-subnetid`: The subnet containing the instances to inspect.

It used two main EC2 describe operations.

**First operation: Find tagged instances**

```php
$tagArgs = array();

array_push($tagArgs, array(
    'Name' => 'tag-key',
    'Values' => array('Environment')
));

$ec2DescribeArgs['Filters'] = $tagArgs;

$result = $ec2->describeInstances($ec2DescribeArgs);
```

This filter identified instances that had an `Environment` tag, regardless of its value.

The instance IDs were stored in an array called `$goodInstances`.

**Second operation: Find instances in the selected subnet**

The script filtered instances using the subnet ID.

It then compared the instances in the subnet against the tagged-instance list.

If an instance was missing from `$goodInstances`, it was added to `$terminateInstances`.

Finally, the script called:

```php
$ec2->terminateInstances(array(
    "InstanceIds" => array_keys($terminateInstances),
    "Force" => true
));
```

This submitted the termination request for the identified instances.

### Step 2: Correct the argument-checking issue

During inspection, I identified this line:

```php
if (count($argv>0)) {
```

The intended condition was:

```php
if (count($argv)>0) {
```

I corrected the expression so that the script could properly check whether command-line arguments were supplied.

I verified the change:

```bash
grep -n 'if (count' terminate-instances.php
```

### Step 3: Identify the private subnet

I listed the available subnets:

```bash
aws ec2 describe-subnets --region us-west-2 \
--query 'Subnets[*].[SubnetId,VpcId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' \
--output table
```

I identified the following subnets in the lab VPC:

| Subnet | CIDR | Public IP assignment |
|---|---|---|
| `subnet-0762b43f8959c22eb` | `10.5.0.0/24` | True |
| `subnet-057a2eb1e65d92655` | `10.5.1.0/24` | False |

The subnet selected for the exercise was:

```text
subnet-057a2eb1e65d92655
```

Its automatic public IPv4 assignment setting was disabled, identifying it as the private-subnet candidate for this lab.

### Step 4: List instances in the private subnet

I used:

```bash
aws ec2 describe-instances --region us-west-2 \
--filters "Name=subnet-id,Values=subnet-057a2eb1e65d92655" \
--query 'Reservations[].Instances[].[InstanceId,State.Name]' \
--output table
```

The command returned seven running EC2 instances.

I then inspected their tags:

```bash
aws ec2 describe-instances --region us-west-2 \
--filters "Name=subnet-id,Values=subnet-057a2eb1e65d92655" \
--query 'Reservations[].Instances[].[InstanceId,Tags]' \
--output json
```

All seven instances initially had the `Environment` tag.

### Step 5: Remove the Environment tag from two instances

For the challenge, I selected two staging instances:

```text
i-0f8cafc1ac7bcb023
i-0acad603cb6c50e88
```

I removed only their `Environment` tags:

```bash
aws ec2 delete-tags --region us-west-2 \
--resources i-0f8cafc1ac7bcb023 i-0acad603cb6c50e88 \
--tags Key=Environment
```

**Important:** This command removed the specified tag only. It did not terminate the instances.

### Step 6: Verify tag removal

I checked both instances:

```bash
aws ec2 describe-instances --region us-west-2 \
--instance-ids i-0f8cafc1ac7bcb023 i-0acad603cb6c50e88 \
--query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Environment`]]' \
--output json
```

The output showed empty tag lists for both instances:

```json
[
    [
        "i-0f8cafc1ac7bcb023",
        []
    ],
    [
        "i-0acad603cb6c50e88",
        []
    ]
]
```

This confirmed that the two instances no longer had the required tag.

### Step 7: Execute the tag-or-terminate script

I ran:

```bash
./terminate-instances.php -region us-west-2 -subnetid subnet-057a2eb1e65d92655
```

The script checked all seven instances and returned:

```text
Checking i-0f8cafc1ac7bcb023
Checking i-0acad603cb6c50e88
Checking i-01bf9b232b9506da7
Checking i-0d0390ef9762b0e58
Checking i-00759b59b3fed3963
Checking i-053d7dd9fbe52618c
Checking i-07990b2b5da13a8d7
Terminating instances...
Instances terminated.
```

The script identified the two instances without the `Environment` tag and submitted the termination request.

### Step 8: Verify termination

I used:

```bash
aws ec2 describe-instances --region us-west-2 \
--instance-ids i-0f8cafc1ac7bcb023 i-0acad603cb6c50e88 \
--query 'Reservations[].Instances[].[InstanceId,State.Name]' \
--output table
```

**Final output:**

```text
|          DescribeInstances          |
+----------------------+--------------+
|  i-0f8cafc1ac7bcb023 |  terminated  |
|  i-0acad603cb6c50e88 |  terminated  |
+----------------------+--------------+
```

Both instances successfully reached the `terminated` state.

---

## 5. Troubleshooting and Lessons Learned

### Issue 1: AWS Organizations SCP restriction

**Problem:** The stopinator script failed with `UnauthorizedOperation` when accessing `ap-south-1`.

**Investigation:** I checked the configured Region and tested PHP SDK access directly in `us-west-2`.

**Lesson:** A script that loops through multiple Regions can encounter organizational restrictions even if the intended Region is accessible.

### Issue 2: PHP timezone warning

**Problem:** The PHP SDK test displayed a warning about the system timezone.

**Investigation:** The PHP runtime did not have a configured timezone.

**Lesson:** PHP scripts can use `date_default_timezone_set()` to set an explicit timezone. The warning did not prevent the SDK connectivity test from succeeding.

### Issue 3: AWS CLI table formatting error

**Problem:** An AWS CLI query that included nested tag data returned `list index out of range`.

**Solution:** I simplified the query to retrieve instance IDs and states, then used JSON output to inspect tags separately.

**Lesson:** Simplifying AWS CLI queries can make troubleshooting easier, especially when working with nested response data.

### Issue 4: PHP argument-checking expression

**Problem:** The termination script contained an incorrect argument-checking expression.

**Solution:** I corrected:

```php
if (count($argv>0)) {
```

to:

```php
if (count($argv)>0) {
```

**Lesson:** Inspecting the code before execution helps identify potential scripting errors.

---

## 6. Key Technical Concepts Learned

| Concept | Explanation |
|---|---|
| AWS resource tags | Key-value pairs used to organize and identify resources |
| EC2 filtering | Retrieving instances based on selected conditions |
| `create-tags` | Creating or updating resource tags |
| `delete-tags` | Removing specified resource tags |
| AWS CLI | Managing AWS services through terminal commands |
| AWS SDK for PHP | Interacting with AWS services programmatically |
| Tag-based automation | Using resource metadata to drive automated actions |
| AWS Organizations SCP | Controlling the maximum permissions available within an organization |
| Tag-or-terminate | Identifying and terminating resources that do not meet a tagging requirement |

## 7. Lab Summary

| Task | Outcome |
|---|---|
| Task 1 – Find and update resources | Completed |
| Task 2 – Stop instances using tags | Blocked by SCP restriction |
| Task 3 – Terminate non-compliant instances | Completed |

## 8. Conclusion

This lab provided practical experience in AWS resource management, tagging, automation, and cloud governance.

I used AWS CLI commands to discover and update EC2 resources, inspected PHP scripts that interact with AWS through the SDK, and implemented a tag-or-terminate approach to identify non-compliant instances.

The exercise also helped me understand the importance of AWS Organizations policies, resource governance, and validation before executing destructive infrastructure operations.

**Key takeaway:** Resource tagging is more than an organizational practice. When combined with automation, it can support security controls, compliance checks, and consistent cloud infrastructure management.