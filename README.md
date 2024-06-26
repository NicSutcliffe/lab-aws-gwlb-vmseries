# 1. VM-Series on AWS Gateway Load Balancer Lab


<img src="https://www.paloaltonetworks.com/content/dam/pan/en_US/images/logos/brand/primary-company-logo/Parent-logo.png" width=50% height=50%>

## 1.1. Overview

----------
This repository contains deployment code and lab guide for learning GWLB traffic flows with VM-Series. Some configuration and resources are intentionally ommitted to be left as troubleshooting excercises. 

***Do not use this for a production deployment or an easy demo environment!***

There are regularly maintained terraform modules for VM-Series deployments in AWS that are published on the [HashiCorp Registry](https://registry.terraform.io/modules/PaloAltoNetworks/vmseries-modules/aws/latest). The [vmseries_combined_with_gwlb_natgw](https://github.com/PaloAltoNetworks/terraform-aws-swfw-modules/tree/main/examples/combined_design) example closely matches the design from this learning lab.

----------


This lab will involve deploying a solution for AWS using Palo Alto Networks VM-Series in the Gateway Load Balancer (GWLB) topology.

The lab assumes an existing Panorama that the VM-Series will bootstrap to. Panorama assumptions:
- Accessible with public IP on TCP 3978
- Prepped with Template Stacks and Device Groups
- vm-auth-key generated on Panorama

The lab has been updated to include a Panorama deployment as the first task. You will then manually install the Software Licensing Plugin and configure it for bootstrapping.

This guide is intended to be used with a specific QwikLabs scenario, and some steps are specific to Qwiklabs. This could be easily adapted for other environments.

```
Manual Last Updated: 2024-05-02
Lab Last Tested: 2024-05-04
```

## 1.2. Lab Guide Syntax conventions

- Items with a bullet indicate actions that need to be taken to complete the lab. They are sometimes followed with a code block for copy / paste or reference.
```
Example Code block following an action item
```

> &#8505;  Items with info icon are additional context or details around actions performed in the lab

## 1.3. Table of Contents

- [1. VM-Series on AWS Gateway Load Balancer Lab](#1-vm-series-on-aws-gateway-load-balancer-lab)
  - [1.1. Overview](#11-overview)
  - [1.2. Lab Guide Syntax conventions](#12-lab-guide-syntax-conventions)
  - [1.3. Table of Contents](#13-table-of-contents)
- [2. Lab Topology](#2-lab-topology)
  - [2.1. Flow Diagrams](#21-flow-diagrams)
    - [2.1.1. Outbound Traffic Flows](#211-outbound-traffic-flows)
    - [2.1.2. Inbound Traffic Flows](#212-inbound-traffic-flows)
- [3. Lab Steps](#3-lab-steps)
  - [3.1. Initialize Lab](#31-initialize-lab)
    - [3.1.1. Find SSH Key Pair Name](#311-find-ssh-key-pair-name)
  - [3.2. Update IAM Policies](#32-update-iam-policies)
  - [3.3. Check Marketplace Subscriptions](#33-check-marketplace-subscriptions)
  - [3.4. Launch CloudShell](#34-launch-cloudshell)
  - [3.6. Download Terraform](#36-download-terraform)
  - [3.7. Clone Deployment Git Repository](#37-clone-deployment-git-repository)
  - [3.8. Deploy Panorama and TGW Infrastructure with Terraform](#38-deploy-panorama-and-tgw-infrastructure-with-terraform)
  - [3.9. Prepare Panorama](#39-prepare-panorama)
  - [3.10. Update Deployment Values in tfvars for VM-series](#310-update-deployment-values-in-tfvars-for-vm-series)
  - [3.11. Apply Terraform](#311-apply-terraform)
  - [3.12. Inspect deployed resources](#312-inspect-deployed-resources)
    - [3.12.1. Get VM-Series instance screenshot](#3121-get-vm-series-instance-screenshot)
    - [3.12.2. Check VM-Series instance details](#3122-check-vm-series-instance-details)
    - [3.12.3. Check cloudwatch bootstrap logs](#3123-check-cloudwatch-bootstrap-logs)
  - [3.13. Verify Bootstrap in Panorama](#313-verify-bootstrap-in-panorama)
  - [3.14. Access VM-Series Management](#314-access-vm-series-management)
  - [3.15. Check bootstrap logs](#315-check-bootstrap-logs)
  - [3.16. Fix GWLB Health Probes](#316-fix-gwlb-health-probes)
  - [3.17. Inbound Traffic Flows to App Spoke VPCs](#317-inbound-traffic-flows-to-app-spoke-vpcs)
    - [3.17.1. Verify/Update Spoke1 App VPC networking for Inbound inspection with GWLB](#3171-verifyupdate-spoke1-app-vpc-networking-for-inbound-inspection-with-gwlb)
    - [3.17.2. Verify/Update Spoke2 App VPC networking for Inbound inspection with GWLB](#3172-verifyupdate-spoke2-app-vpc-networking-for-inbound-inspection-with-gwlb)
    - [3.17.3. Test Inbound Traffic to Spoke Web Apps](#3173-test-inbound-traffic-to-spoke-web-apps)
    - [3.17.4. Test Outbound Traffic from Spoke1 Instance](#3174-test-outbound-traffic-from-spoke1-instance)
    - [3.17.5. Check Inbound Traffic Logs](#3175-check-inbound-traffic-logs)
    - [3.17.6. Check Outbound Traffic Logs](#3176-check-outbound-traffic-logs)
  - [3.18. Outbound and East / West (OBEW) Traffic Flows](#318-outbound-and-east--west-obew-traffic-flows)
    - [3.18.1. Update Spoke1 VPC for OB/EW routing with TGW](#3181-update-spoke1-vpc-for-obew-routing-with-tgw)
    - [3.18.2. Update Spoke2 VPC for OB/EW routing with TGW](#3182-update-spoke2-vpc-for-obew-routing-with-tgw)
    - [3.18.3. Update Transit Gateway (TGW) Route Tables](#3183-update-transit-gateway-tgw-route-tables)
  - [3.19. Test Traffic Flows](#319-test-traffic-flows)
    - [3.19.1. Test Outbound Traffic from Spoke1 Instances](#3191-test-outbound-traffic-from-spoke1-instances)
    - [3.19.2. Test Inbound Web Traffic to Spoke1 and Spoke2 Apps](#3192-test-inbound-web-traffic-to-spoke1-and-spoke2-apps)
    - [3.19.3. Test E/W Traffic from Spoke1 Instance to Spoke2 Instance](#3193-test-ew-traffic-from-spoke1-instance-to-spoke2-instance)
  - [3.20. Finished](#320-finished)

# 2. Lab Topology

![GWLB Topology](docs/images/topology.png)

## 2.1. Flow Diagrams

Reference these diagrams for a visual of traffic flows through this topology.
### 2.1.1. Outbound Traffic Flows
<img src="https://raw.githubusercontent.com/NicSutcliffe/lab-aws-gwlb-vmseries/main/docs/images/AWS%20Qwiklabs%20-%20GWLB%20-%20Traffic%20Flow%20-%20Outbound.png" width=30% height=30%>

### 2.1.2. Inbound Traffic Flows
<img src="https://raw.githubusercontent.com/NicSutcliffe/lab-aws-gwlb-vmseries/main/docs/images/AWS%20Qwiklabs%20-%20GWLB%20-%20Traffic%20Flow%20-%20Inbound.png" width=30% height=30%>

# 3. Lab Steps
## 3.1. Initialize Lab

- Download `Student Lab Details` File from Qwiklabs interface for later reference
- Click Open Console and authenticate to AWS account with credentials displayed in Qwiklabs
- Verify your selected region in AWS console (top right) matches the aws-gwlb-lab-secrets.txt
  
### 3.1.1. Find SSH Key Pair Name

- EC2 Console -> Key pairs
- Copy and record the name of the Key Pair that was generated by Qwiklabs, e.g. `qwikLABS-L17939-10286`
- In Qwiklabs console, download the ssh key for later use (PEM and PPK options available)

> &#8505; Any EC2 Instance must be associated with a SSH key pair, which is the default method of initial interactive login to EC2 instances. With successful bootstrapping, there should not be any need to connect to the VM-Series instances directly with this key, but it is usually good to keep this key securely stored for any emergency backdoor access. 
> 
> For this lab, we will use the key pair automatically generated by Qwiklabs. The key will also be used for connecting to the test web server instances.


## 3.2. Update IAM Policies


- Search for `IAM` in top search bar (IAM is global)
- In IAM dashboard select Users -> awsstudent
- Expand `default_policy`, Edit Policy -> Visual Editor
- Find the Deny Action for `Cloud Shell` and click `Remove` on the right
- Also remove the Deny actions for all actions containing `Marketplace`
- Select `Review policy`
- Select `Save changes`

---

<img src="https://user-images.githubusercontent.com/43679669/200521132-07ca60f0-2186-49cc-b6ac-4c3477de3abf.png" width=50% height=50%>


> &#8505; Qwiklabs has an explicit Deny for CloudShell. However, we have permissions to remove this deny policy. Take a look at the other Deny statements while you are here.

> &#8505; It is important to be familiar with IAM concepts for VM-Series deployments. Several features (such as bootstrap, custom metrics, cloudwatch logs, HA, VM Monitoring) require IAM permissions. You also need to consider IAM permissions in order to deploy with IaC or if using lambda for custom automation.

---

## 3.3. Check Marketplace Subscriptions

> &#8505; Before you can launch VM-Series or Panorama images in an account, the account must first have accepted the Marketplace License agreement for that product.

> &#8505; The QwikLabs accounts should already be subscribed to these offers, but we will need to verify and correct if required.

- Search for `AWS Marketplace Subscriptions` in top search bar
- Verify that there are active subscription for both of:
  - `VM-Series Next-Generation Firewall (BYOL)`
  - `Palo Alto Networks Panorama`

<img src="https://user-images.githubusercontent.com/43679669/210279563-6e313499-41fb-42b3-b516-636df544c6e6.gif" width=50% height=50%>

- If you have both subscriptions, continue to the next section
- If you are missing either subscription, select `Discover Products` and search for `palo alto`
- Select `VM-Series Next-Generation Firewall (BYOL)` or `Palo Alto Networks Panorama` as needed
- Continue to Subscribe
- Accept Terms
- Allow a few moments for the Subscription to be processed
- Repeat for the other Subscription if needed
- Exit out of the Marketplace
- Notify lab instructor if you have any issues

---

## 3.4. Launch CloudShell

- *Verify you are in the assigned region!*
- Search for `cloudshell` in top search bar
- Close out of the Intro Screen
- Allow a few moments for it to initialize

---

> &#8505; This lab will use cloudshell for access to AWS CLI and as a runtime environment to provision your lab resources in AWS using terraform. Cloudshell will have the same IAM role as your authenticated user and has some utilities (git, aws cli, etc) pre-installed. It is only available in limited regions currently.
>
> Anything saved in home directory `/home/cloudshell-user` will remain persistent if you close and relaunch CloudShell

---

## 3.6. Download Terraform 

- Make sure CloudShell home directory is clean

```
rm -rf ~/bin && rm -rf ~/lab-aws-gwlb-vmseries/
```

- Download Terraform in Cloudshell

```
mkdir /home/cloudshell-user/bin/ && wget https://releases.hashicorp.com/terraform/1.3.9/terraform_1.3.9_linux_amd64.zip && unzip terraform_1.3.9_linux_amd64.zip && rm terraform_1.3.9_linux_amd64.zip && mv terraform /home/cloudshell-user/bin/terraform
```

- Verify Terraform 1.3.9 is installed
```
terraform version
```

> &#8505; Terraform projects often have version constraints in the code to protect against potentially breaking syntax changes when new version is released. For this project, the [version constraint](https://github.com/PaloAltoNetworks/lab-aws-gwlb-vmseries/blob/main/terraform/vmseries/versions.tf) is:
> ```
> terraform {
>  required_version = ">=0.12.29, <2.0"
>}
>```
>
>Terraform is distributed as a single binary so isn't usually managed by OS package managers. It simply needs to be downloaded and put into a system `$PATH` location. For Cloudshell, we are using the `/home/cloud-shell-user/bin/` so it will be persistent if the sessions times out.


## 3.7. Clone Deployment Git Repository 

- Clone the Repository with terraform to deploy
  
```
git clone https://github.com/NicSutcliffe/lab-aws-gwlb-vmseries.git && cd lab-aws-gwlb-vmseries/terraform/panorama
```

## 3.8. Deploy Panorama and TGW Infrastructure with Terraform

- Make sure you are in the appropriate deployment directory

```
cd ~/lab-aws-gwlb-vmseries/terraform/panorama
```
- Initialize Terraform

```
terraform init
```

- Apply Terraform

```
terraform apply
```

- When Prompted for confirmation, type `yes`

- It should take a few minutes minutes for terraform to finish deploying all resources

- When complete, you will see an output containing the Panorama URL. **Copy these locally so you can reference them in later steps**

> &#8505; You can also come back to this directory in CloudShell later and run `terraform output` to view this information 

- It will be about 7 minutes from the time the Terraform finishes before you can access and authenticate to the Panorama.

## 3.9. Prepare Panorama

> &#8505; The Panorama was deployed from a partially prepped image that is licensed and has some baseline template, devicegroup, and logging configured. Additional steps will be completed here to prepare the Panorama for bootstrapping.

- Copy the `panorama_url` from the Terraform output and access it in a browser.
  
- Authenticate using the credentials from `aws-gwlb-lab-secrets.txt`

- Panorama Tab -> Plugins -> Check Now

- Search for `sw_fw` -> Locate the latest version -> Download -> Install

- Configure SW Firewall License Bootstrap Definition
  - Name: `aws-gwlb-lab`
  - Auth Code: Found in `aws-gwlb-lab-secrets.txt`

- Configure SW Firewall License Manager
  - Name: `aws-gwlb-lab`
  - Device Group: `AWS-GWLB-LAB`
  - Template Stack: `stack-aws-gwlb-lab`
  - Auto Deactivate: `Never`
  - Bootstrap Definition: `aws-gwlb-lab`

> &#8505; Auto Deactive is handy for automating the cleanup of devices that have been terminated, for example in an autoscaling VM-Series deployment. Caution should be used as it is possibility for false positives. For example, if there is an unrelated connectivity issue, Panorama could perceive that the devices are no longer active and initiate the deactivation based on the timer. This plugin can also be used for manually deactivating devices, or creating an event-driven workflow to make API call to Panorama for deactvation. For any deactivation functions to work, the Panorama must be configured with a [Licensing API Key ](https://docs.paloaltonetworks.com/vm-series/10-1/vm-series-deployment/license-the-vm-series-firewall/install-a-license-deactivation-api-key)that is generated from the Customer Support Portal. 

- Configure Template Stack to Automatically Push Content
  - Panorama -> Templates -> `stack-aws-gwlb-lab`
  - Check Box for `Automaically push content....`

> &#8505; This feature was added in 10.2 and is very useful for automated deployments, especially autoscaling. It ensures that as devices bootstrap, the Panorama will push down the current dynamic content before committing the configuration.

- Commit to Panorama and verify it completes successfully.

- Return to Panorama -> SW Firewall Licenses -> License Manager
  
- Select `Show Bootstrap Parameters`

- Copy the bootstrap parameters for use in the next step

## 3.10. Update Deployment Values in tfvars for VM-series


Most of the bootstrap parameters and environment-specific variable values have already been prepped in the Terraform code for this deployment. You will only need to update the `auth-key` value for your Panorama.

- **When bootstrapping, be very careful to ensure all of the parameters are set correctly. These values will be passed in as User Data for the VM-Series launch. If values are incorrect, the bootstrap will likely fail and you will need to redeploy!**

- Make sure you are in the appropriate directory

```
cd ~/lab-aws-gwlb-vmseries/terraform/vmseries
```

- Edit the file `student.auto.tfvars` to update the value of the `auth-key` variable that you copied from the bootstrap parameters in Panorama `aws-gwlb-lab-secrets.txt`

- Set the value inside of the empty quotes

```
auth-key        = "_AQ__xxxxxxxxxxxxxxxxxxx"
```

---
- ( Option 1 ) Use vi to update values in `student.auto.tfvars`

```
vi student.auto.tfvars
```
---
- ( Option 2 ) If you don't like vi, you can install nano editor:
```
sudo yum install -y nano
nano student.auto.tfvars
```
---

- Verify the contents of file have the correct value

```
cat ~/lab-aws-gwlb-vmseries/terraform/vmseries/student.auto.tfvars
```

> &#8505; This deployment is using a [newer feature for basic bootstrapping](https://docs.paloaltonetworks.com/plugins/vm-series-and-panorama-plugins-release-notes/vm-series-plugin/vm-series-plugin-20/vm-series-plugin-201/whats-new-in-vm-series-plugin-201.html) that does not require S3 buckets. Any parameters normally specified in init-cfg can now be passed directly to the instance via UserData. Prerequisite is the image you are deploying has plugin 2.0.1+ installed


## 3.11. Apply Terraform

- Make sure you are in the appropriate directory

```
cd ~/lab-aws-gwlb-vmseries/terraform/vmseries
```
- Initialize Terraform

```
terraform init
```

- Apply Terraform

```
terraform apply
```

- When Prompted for confirmation, type `yes`


<img src="https://user-images.githubusercontent.com/43679669/108799781-36671400-755f-11eb-9724-d18c7ea147bb.gif" width=50% height=50%>


- It should take 5-10 minutes for terraform to finish deploying all resources

- When complete, you will see a list of outputs. **Copy these locally so you can reference them in later steps**

- If you do get an error, first try to run `terraform apply` again to finish updating of any pending resources. Notify lab instructor if there are still issues.

> &#8505; You can also come back to this directory in CloudShell later and run `terraform output` to view this information 


## 3.12. Inspect deployed resources

All resources are now created in AWS, but it will be around 10 minutes until VM-Series are fully initialized and bootstrapped.

In the meantime, lets go look at what you built!


- EC2 Dashboard -> Instances -> Select `vmseries01` -> Actions -> Instance settings -> Edit user data

- Verify the values matches what was provided in your Lab Details

> &#10067; What needs to happen if you have a typo or missed a value for bootstrap when you deployed?

---
### 3.12.1. Get VM-Series instance screenshot

- EC2 Dashboard -> Instances -> Select `vmseries01` -> Actions -> Monitor and troubleshoot -> Get instance screenshot

> &#8505; This can be useful to get a view of the console during launch. It is not interactive and must be manually refreshed, but you can at least see some output related to bootstrap process or to troubleshoot if the VM-Series isn't booting properly or is in maintenance mode.

---

### 3.12.2. Check VM-Series instance details

- EC2 Dashboard -> Instances -> Select `vmseries01` -> Review info / tabs in bottom pane


> &#10067; What is the instance type? 

> &#10067; How many interfaces are associated to the VM-Series? 

> &#10067; Which interface is the default ENI for the instance? 

> &#10067; Which interface has a public IP associated?


---

### 3.12.3. Check cloudwatch bootstrap logs

- Search for `cloudwatch` in the top search bar
- Logs -> Log groups -> PaloAltoNetworksFirewalls
- Assuming enough time has passed since launch, verify that the bootstrap operations completed successfully.

> &#8505; It is normal for the VMs to briefly lose connectivity to Panorama after first joining.

> &#8505; This feature is only implemented for AWS currently.

> &#10067; What is required to enable these logs during boot process?

---


## 3.13. Verify Bootstrap in Panorama

> &#8505; For this lab, the Panorama and VM-Series mgmgt are publicly accessible. For production deployments, management should not be exposed to inbound Internet traffic as a general practice. If public inbound management access is required, make sure to use other controls (MFA, AWS security groups, PAN-OS permitted-ip lists).

- Login to Panorama web interface with student credentials
- Check Panorama -> Managed Devices -> Summary
- Verify your deployed VM-Series are connected and successfully bootstrapped
- Verify that the auto-commit push succeeded for Shared Policy and Template and that devices are "In sync"
- Check Panorama system logs to verify the content push and licensing was successful
  - Monitor -> Device Group Dropdown = All -> System
  - Search for the serial number of one of your VM-Series
    - `( description contains '00795xxxxxxx' )`
- Inspect Pre-Configured Interface, Zone, and Virtual Router configuration for your template
- Inspect Pre-Configured Security Policies and NAT Policies for your Device Group


## 3.14. Access VM-Series Management

- Most configurations will be done in Panorama, but we will use the local consoles for some steps and validation

- Refer to the output you copied from your terraform deployment for the public EIPs associated to each VM-Series management interface
  
```
vmseries_eips = {
  "vmseries01-mgmt" = "54.71.121.124"
  "vmseries02-mgmt" = "44.237.145.237"
}
```
- Refer to `aws-gwlb-lab-secrets.txt` from QwikLabs for local credentials for the VM-Series instances. This will be the same credentials you used for Panorama.

- Establish a connection to both VM-Series Web UI via HTTPS
- Establish a connection to both VM-Series CLI via SSH
  - You can use Cloud Shell for the SSH session if it is blocked on your local machine

## 3.15. Check bootstrap logs

> &#8505; It is common to have issues when initially setting up an environment for bootstrapping. Thus it is a good to know how to troubleshoot. When using the new basic bootstrapping with user-data, there is less potential for problems.
>
> Some things to keep in mind:
>- Default AWS ENI needs access to reach S3 bucket as well as path to Internet for licensing
>   - S3 access can be via Internet or with VPC endpoint
>   - IAM Instance Profile will need permissions to access S3 bucket
> - When using interface swap, the subnet for the second ENI will also need path to S3 and Internet
>   - Interface swap can be done with user-data or in init-cfg parameters. 
>   - Generally better to do via user-data
> - Template Stack and Device Group names must match **exactly** or they will never join Panorama (no indication of this in Panorama logs)
> - If there are any issues with licensing, VM-Series will not join Panorama (no indication of this in Panorama logs)

- From SSH session on either VM-Series, check the bootstrap summary
  
```
show system bootstrap status
```

- Check the bootstrap detail log

```
debug logview component bts_details
```

> &#8505; If you have Cloudwatch logs enabled, you can see most of this status without SSH session to VM-Series.


## 3.16. Fix GWLB Health Probes

- Check GWLB Target Group status
  - In EC2 Console -> Target Groups -> select `security-gwlb`
  - In the `Targets` tab, note that the status of both VM-Series is unhealthy
  - Switch to `Health Checks` tab to verify health check settings

> &#10067; What Protocol and Port is the Target Group Health Probe configured to use?

> &#10067; What Protocol and Port is the GWLB listening on and forwarding to the VM-Series instances?

> &#8505; You can check traffic logs either in Panorama or local device

> &#8505; **Reboot your VM-Series if you do not see any traffic logs!**
> 
- Check Traffic Logs for Health Probes
  - In Panorama UI -> Monitor -> Traffic
  - Analyze the traffic logs for the port 80 traffic
  - Enable Columns to view `Bytes Sent` and `Bytes Received`
  - Notice that the sessions matching allow policy for `student-gwlb-any` policy but aging out

> &#10067; Why are there two different source addresses in the traffic logs for these GWLB Health Probes?

<img src="https://user-images.githubusercontent.com/43679669/109860663-6af86100-7c2c-11eb-88f3-0aac14d256a8.gif" width=50% height=50%>

- Resolve the issue with the Health Probes:

> &#8505; We can see in the traffic logs that the health probes are being received. So we know they are being permitted by the AWS Security Group. They are being permitted by catch-all security policy but there is no return traffic (Notice 0 Bytes Received in the traffic logs). This indicates the VM-Series dataplane interface is not listening 

  - Create and add Interface Management profile to eth1/1
    - In Panorama select Network Tab -> Template `tpl-aws-gwlb-lab`
    - Create Interface Management Profile
      - Name: `gwlb-health-probe`
      - Services: HTTP
      - Permitted IP addresses: `10.100.0.16/28`, `10.100.1.16/28`
    - Select Interfaces -> ethernet1/1 -> Advanced -> Management Profile `gwlb-health-probe`

<img src="https://user-images.githubusercontent.com/43679669/109861732-9fb8e800-7c2d-11eb-87b7-98e794ce8131.gif" width=50% height=50%>

  - Create a specific security policies for these health probes to keep logging clean
    - In Panorama select Policies Tab -> Device Group `AWS-GWLB-LAB`
    - Create new security policy
      - Name: `gwlb-health-probe`
      - Source Zone: `gwlb`
      - Source Addresses: `10.100.0.16/28`, `10.100.1.16/28` (Can use predefined address objects)
      - Dest Zone: `gwlb`
      - Dest Addresses: `10.100.0.16/28`, `10.100.1.16/28` (Can use predefined address objects)
      - Application: `Any`
      - Serivce: `service-http`
    - Make sure new policy is before the existing catch-all `student-gwlb-any` policy

  - Commit and Push your changes

<img src="https://user-images.githubusercontent.com/43679669/109866175-f674f080-7c32-11eb-8c77-4e2c3c195f9a.gif" width=50% height=50%>


- Verify Traffic Logs have Bytes Received and are matching the appropriate security policy
- Return to AWS console to verify the Targets are now showing as healthy

<img src="https://user-images.githubusercontent.com/43679669/109867219-3b4d5700-7c34-11eb-8121-5ce90f2e1004.gif" width=50% height=50%>


## 3.17. Inbound Traffic Flows to App Spoke VPCs

- The deployed topology does not have all of the AWS routing in place for a working GWLB topology and you must fix it!
- Refer to the diagram and try to resolve before looking at the specific steps.

> &#8505; For GWLB Distributed model, Inbound traffic for public services comes directly into the App Spoke Internet Gateway. Ingress VPC route table directs traffic to the GWLB Endpoints in the App Spoke VPC.
> 
> Application owners can provision their external facing resources in their VPC (EIP, Public NLB / ALB, etc), but all traffic will be forwarded to Security VPC (via GWLB endpoint) for inspection prior to reaching the resource.
> 
> This inbound traffic flow uses AWS Private Link technology and does not involve the Transit Gateway at all.

- Tip: In the VPC Dashboard you can set a filter by VPC, which will apply to any other sections of the dashboard (subnets, route tables, etc)

<img src="https://user-images.githubusercontent.com/43679669/110424278-8b7f4b80-8070-11eb-91e2-87bab5882f21.gif" width=50% height=50%> 


### 3.17.1. Verify/Update Spoke1 App VPC networking for Inbound inspection with GWLB

- First investigate `spoke1-app-vpc` Route Tables in the VPC Dashboard and try to identify and fix what is missing. Refer to the diagram for guidance.
- For **inbound** traffic, no changes are required for the `web` route tables in the Spoke VPCs
- Refer to terraform output for GWLB Endpoint IDs (or identify them in VPC Dashboard)

- **To verify your routes, see below for specific steps. Note that many of these routes have been created for you by Terraform to reduce the time commitment required for this specific lab.**

Starting left to right on the diagram...

  - VPC Dashboard -> Filter by VPC -> `spoke1-app-vpc`
  - Route Tables -> `spoke1-vpc-igw-edge` -> Routes Tab (bottom panel)
  - Add Route (spoke1-vpc-alb1 subnet CIDR to spoke1-gwlbe1)
     - CIDR: 10.200.0.16/28
     - Target: Gateway Load Balancer Endpoint (ID of spoke1-vpc-inbound-gwlb-endpoint1 GWLBE)
  - Add Route (spoke1-vpc-alb2 subnet CIDR to spoke2-gwlbe2)
     - CIDR: 10.200.1.16/28
     - Target: Gateway Load Balancer Endpoint (ID of spoke1-vpc-inbound-gwlb-endpoint2 GWLBE)
  - Save Routes

---  
  - Route Tables -> `spoke1-vpc-gwlbe1` -> Routes Tab (bottom panel)
  - Add Route (default route to spoke1-IGW)
     - CIDR: 0.0.0.0/0
     - Target: Internet Gateway (only one per VPC)
  - Save Routes

---  
  - Route Tables -> `spoke1-vpc-gwlbe2` -> Routes Tab (bottom panel)
  - Add Route (default route to spoke1-IGW)
     - CIDR: 0.0.0.0/0
     - Target: Internet Gateway (only one per VPC)
  - Save Routes
  
---
  - Route Tables -> `spoke1-vpc-alb1` -> Routes Tab (bottom panel)
  - Add Route (default route to spoke1-gwlbe1)
     - CIDR: 0.0.0.0/0
     - Target: Gateway Load Balancer Endpoint (ID of spoke1-vpc-inbound-gwlb-endpoint1 GWLBE)
  - Save Routes

---  

  - Route Tables -> `spoke1-vpc-alb2` -> Routes Tab (bottom panel)
  - Add Route (default route to spoke1-gwlbe2)
     - CIDR: 0.0.0.0/0
     - Target: Gateway Load Balancer Endpoint (ID of spoke1-vpc-inbound-gwlb-endpoint2 GWLBE)
  - Save Routes

---  

### 3.17.2. Verify/Update Spoke2 App VPC networking for Inbound inspection with GWLB

- First investigate **`spoke2-app-vpc`** Route Tables in the VPC Dashboard and try to identify and fix what is missing. Refer to the diagram for guidance.
- For **inbound** traffic, no changes are needed for the `web` route tables in the App Spoke VPCs
- Refer to terraform output for GWLB Endpoint IDs (or identify them in VPC Dashboard)

- - **To verify your routes, see below for specific steps. Note that many of these routes have been created for you by Terraform to reduce the time commitment required for this specific lab.**

Only `spoke2-vpc-igw-edge` Route Table is missing routes for App2 Spoke

Starting left to right on the diagram...

  - VPC Dashboard -> Filter by VPC -> `spoke2-app-vpc`
  - Route Tables -> `spoke2-vpc-igw-edge` -> Routes Tab (bottom panel)
  - Add Route (spoke2-vpc-alb1 subnet CIDR to spoke2-gwlbe1)
     - CIDR: 10.250.0.16/28
     - Target: Gateway Load Balancer Endpoint (ID of spoke2-vpc-inbound-gwlb-endpoint1 GWLBE)
  - Add Route (spoke2-vpc-alb2 subnet CIDR to spoke2-gwlbe2)
     - CIDR: 10.250.1.16/28
     - Target: Gateway Load Balancer Endpoint (ID of spoke2-vpc-inbound-gwlb-endpoint2 GWLBE)
  - Save Routes

###  3.17.3. Test Inbound Traffic to Spoke Web Apps

Generate some HTTP traffic to the web apps using the DNS name of the Public NLB that is in front of the web compute instances. URL will be the FQDN of the NLBs from the terraform output

- Reference terraform output for `app_nlbs_dns`
- From your local machine browser, attempt connection to http://`spoke1-nlb`
- From your local machine browser, attempt connection to http://`spoke2-nlb`
- Refresh a few times

> &#8505; The inbound routing should now be in place, but you will not get a response yet as the instances are not yet running a web server.

> &#8505; The web instances are configured to update and install web server automatically with a user-data script, but they first must have a working outbound path to the Internet to retrieve packages.

###  3.17.4. Test Outbound Traffic from Spoke1 Instance

Access the spoke web servers console using the AWS Systems Manager connect

- Navigate to Instances view in the EC2 Console
- Select `spoke1-web-az1` and click Connect button in the top right
- Switch to Session Manager and Connect
- From the Shell, try to generate outbound traffic

```ping 8.8.8.8```

```curl http://ifconfig.me```

> &#8505; Session Manager relies on a package being installed in the OS that makes an outbound connection to the AWS SSM service. We do not have outbound internet currently, but there is a private endpoint for the SSM service configured. It is not possible to use SSM for CLI access to VM-Series, as it does not have the package installed


###  3.17.5. Check Inbound Traffic Logs

- Panorama -> Monitor Tab -> Traffic
- Filter for traffic *to* Spoke 1 `( addr.dst in 10.200.0.0/16 )`
- Notice that there is already many connection attempts from Internet scans to the App1 Public NLB and being permitted by VM-Series!
- Notice that VM-Series is able to see the original client source IP for connections to to App Spokes
- Try to identify your client source IP in the logs

###  3.17.6. Check Outbound Traffic Logs

> &#8505; Since outbound traffic was not working earlier, let's check to see if it it making it to VM-Series.

- Panorama -> Monitor Tab -> Traffic
- Filter for traffic *from* Spoke 1 `( addr.src in 10.200.0.0/16 )`
- Notice that there is no outbound traffic from the Spokes yet, so we are likely missing something in routing between App Spoke -> TGW -> GWLB.

> &#8505; We now have visibility and control for inbound connectivity but instances do not yet have a path outbound.


## 3.18. Outbound and East / West (OBEW) Traffic Flows

- The deployed topology does not have all of the AWS routing in place for a working GWLB topology and you must fix it!
- Refer to the diagram and try to resolve before looking at the specific steps
- We will work from left to right on the diagram

### 3.18.1. Update Spoke1 VPC for OB/EW routing with TGW

- First investigate `spoke1-app-vpc` Route Tables in the VPC Dashboard and try to identify and fix what is missing. Refer to the diagram for guidance.

- **To verify your routes, see below for specific steps. Note that some of these routes have been created for you by Terraform to redude the time commitment required for this specific lab.**

  - VPC Dashboard -> Filter by VPC -> `spoke1-app-vpc`
  - Route Tables -> `spoke1-vpc-web1` -> Routes Tab (bottom panel)
  - Add Route (default route to TGW)
     - CIDR: 0.0.0.0/0
     - Target: Transit Gateway (only one available)
     - Save Routes

---

  - Route Tables -> `spoke1-vpc-web2` -> Routes Tab (bottom panel)
  - Add Route (default route to TGW)
     - CIDR: 0.0.0.0/0
     - Target: Transit Gateway (only one available)
     - Save Routes
  

### 3.18.2. Update Spoke2 VPC for OB/EW routing with TGW

- First investigate `spoke2-app-vpc` Route Tables in the VPC Dashboard and try to identify and fix what is missing. Refer to the diagram for guidance.

- **To verify your routes, see below for specific steps. Note that some of these routes have been created for you by Terraform to redude the time commitment required for this specific lab.**

- Nothing is missing for `spoke2-app-vpc`! Web1 and Web2 Route Tables already have routes to TGW.


### 3.18.3. Update Transit Gateway (TGW) Route Tables


> &#8505; For GWLB Centralized Outbound, the TGW routing for Outbound and EastWest (OB/EW) traffic is the same as previous TGW models. Spokes send all traffic to TGW. Spoke TGW RT directs all traffic to Security VPC. Security TGW RT has routes to reach all spoke VPCs for return traffic.
>
>For OB/EW flows, the GWLB doesn't come into play until traffic comes into the Security VPC from TGW before being forwarded to a GWLB endpoint.

- First investigate Transit Gateway Route Tables in the VPC Dashboard and try to identify and fix what is missing. Refer to the diagram for guidance.

- **To verify your TGW routes, see below for specific steps**

  - VPC Dashboard -> Transit Gateway Route Tables -> Select `from-spoke-vpcs`
  -  Check `Associations` tab and verify the two spoke App VPCs are associated
  -  Check Routes tab and notice there is no default route
  -  Create Static Route (Default to security VPC)
     - CIDR: 0.0.0.0/0
     - Attachment: Security VPC (Name Tag = security-vpc)

- VPC Dashboard -> Transit Gateway Route Tables -> Select `from-security-vpc`
  -  Check `Associations` tab and verify the security VPC is associated
  -  Check `Routes` tab and notice there are no existing routes to reach the spoke VPCs
  -  Select Propagations Tab -> Create Propagation
  -  Select attachment with Name Tag `spoke1-vpc`
  -  Repeat for attachment with Name Tag `spoke2-vpc`
  -  Return to `Routes` tab and verify the table now has routes to reach the App VPCs (may need to refresh)


<img src="https://user-images.githubusercontent.com/43679669/109261520-e8a41300-77cd-11eb-9324-3b0dd1d85f7d.gif" width=50% height=50%>


## 3.19. Test Traffic Flows

At this point all routing should be in place for GWLB topology. Now we will verify traffic flows and check the logs.

> &#8505; Note that web instances in Spoke VPCs are configured to update and install web server automatically, now that you have provided an outbound path, this should have completed.

###  3.19.1. Test Outbound Traffic from Spoke1 Instances

- Using an AWS Sessions Manager, connect to an App1 web instance, test outbound traffic.
  
```ping 8.8.8.8```

```curl http://ifconfig.me```


> &#8505; ifconfig.me is a service that just returns your client's public IP (like google "what is my ip" or ipmonkey.com)

- Try the curl to ifconfig.me several times to see if you egress address changes.

- Identify these sessions in Panorama traffic logs
- Identify the sessions for outbound traffic for the automated web server install
  - Filter `( addr.src in 10.200.0.0/16 )`

###  3.19.2. Test Inbound Web Traffic to Spoke1 and Spoke2 Apps

- Reference terraform output for `app_nlbs_dns`
- From your local machine browser, attempt connection to http://`app1_nlb`
- From your local machine browser, attempt connection to http://`app2_nlb`
- Refresh a few times

- **Troubleshooting Steps if Inbound Traffic is not working**

  If your NLB is not responding and you see traffic in the Panorama logs, it is possible the script to install the web server didn't execute
  - From the Session Manager, verify if you can connect locally
    - curl http://localhost
  - If not, check the user data of the instance to see the bash script that was configured and run it manually
  - If the web service is responding locally, there is likely an issue in the spoke VPC routing

> &#8505; Local IP and VM Name in the response will show you which VM you are connected to behind the NLB. Session persistence may keep you pinned to a specific instances

- Identify these sessions in Panorama traffic logs


### 3.19.3. Test E/W Traffic from Spoke1 Instance to Spoke2 Instance

- Use EC2 Console to identify the Private IP address of `spoke1-web-az1`
- Using an Console Connection session on `spoke1-web-az1` instance, test traffic to `spoke2-web-az1`

```ping 10.250.0.x```

```curl http://10.250.0.x```

- Identify these sessions in Panorama traffic logs 

> &#8505; Backhaul traffic flows (VPN or Direct Connect Attachments to TGW) will follow this same general traffic flow as E/W between VPCs.

## 3.20. Finished

Congratulations! You have completed this lab
