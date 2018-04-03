# Automate the Launch of AWS Instances for a multi-DC DSE cluster with OpsCenter

# Quick start summary of the following steps:

1. Set all params and cluster topology in terraform_extended/variables.tf
2. Set all port rules and security in terraform_extended/ec2.tf
3. Set the cluster_names in genansinv_extended.sh file
4. Set all paths and vars in ansible/group_vars/all
5. Run /runterra_extended.sh and check AWS instances that will be created - accept and run the plan
6. Run /genansinv_extended.sh (it will generate the /ansible/hosts)
7. Run /runansi.sh (it will run osparm_change, dse_install and opsc_install playbooks, it expects your key to be: ~/.ssh/id_rsa_aws)

# Basic processes: 

The scripts in this repository have 3 major parts:
1. Terraform scripts to launch the required AWS resources (EC2 instances, security groups, etc.) based on the target DSE cluster toplogy.
2. Ansible playbooks to install and configure DSE and OpsCenter on the provisioned AWS EC2 instances.
3. Linux bash scripts to 
   1. generate the ansible host inventory file (required by the ansible playbooks) out of the terraform state output
   2. lauch the terraform scripts and ansible playbooks

## 1. Terraform Introduction and Cluster Topology

Terraform is a great tool to plan, create, and manage infrastructure as code. Through a mechanism called ***providers***, it offers an agonstic way to manage various infrastructure resources (e.g. physical machines, VMs, networks, containers, etc.) from different underlying platforms such as AWS, Azure, OpenStack, and so on. In this respository, I focus on using Terraform to launch AWS resources due to its popularity and 1-year free-tier access. For more information about Terraform itself, please check HashiCorp's document space for Terraform at https://www.terraform.io/docs/.

The infrastructure resources to be lauched is ultimately determined by the target DSE cluster topology. In this repository, a cluster topology like below is used for explanation purpose:
 
![cluster topology](https://github.com/yabinmeng/terradse/blob/master/resources/cluster.topology.png)

By this topology, there are 2 DSE clusters. 
* One cluster is a multi-DC (2 DC in the example) DSE cluster dedicated for application usage.
* Another cluster is a single-DC DSE cluster dedicated for monitoring purpose through DataStax OpsCenter.

Currently, the number of nodes per DC is configurable through Terraform variables. The number of DCs per cluster is fixed at 2 for application cluster and 1 for the monitoring cluster. However, it can be easily expanded to other settings depending on your unique application needs.

The reason of setting up a different monitoring cluster other than the application cluster is to follow the field best practice of physically separating the storage of monitoring metrics data in a different DSE cluster in order to avoid the hardware resource contentions that could happen when manaing the metrics data and application data together. 


## 2. Use Terraform to Launch Infrastructure Resources

**NOTE:** a linux bash script, ***runterra.sh***, is provided to automate the execution the terraform scripts.

### 2.1. Pre-requisites

In order to run the terraform script sucessfully, the following procedures need to be executed in advance:

1. Install Terraform software on the computer to run the script
2. Install and configure AWS CLI properly. Make sure you have an AWS account that have the enough privilege to create and configure AWS resources.
3. Create a SSH key-pair. The script automatically uploads the public key to AWS (to create an AWS key pair resource), so the launched AWS EC2 instances can be connected through SSH. The names of the SSH key-pair, by default, should be “id_rsa_aws and id_rsa_aws.pub”. If you choose other names, please make sure to update the Terraform configuration variable accordingly.

### 2.2. Provision AWS Resources 

#### 2.2.1. EC2 Count and Type

The number and type of AWS EC2 instances are determined at DataCenter (DC) level through terraform variable mappings, with each DC has its own instance type and count as determined by the target DSE cluster topology. The example for the example cluster topology is as below:
```
variable "instance_count" {
   type = "map"
   default = {
      opsc      = 2
      cassandra = 3
      solr      = 3
      spark     = 0
      graph     = 0
   }
}

variable "instance_type" {
   type = "map"
   default = {
      // t2.2xlarge is the minimal DSE requirement
      opsc      = "t2.2xlarge"
      cassandra = "t2.2xlarge"
      solr      = "t2.2xlarge"
      spark     = "t2.2xlarge"
      graph     = "t2.2xlarge"
   }
}
```

When provisioning the required AWS EC2 instances for a specific DC, the type and count is determined through a map search as in the example below:
```
#
# EC2 instances for DSE cluster, "DSE Search" DC
# 
resource "aws_instance" "dse_search" {
   ... ...
   instance_type   = "${lookup(var.instance_type, var.dse_search_type)}"
   count           = "${lookup(var.instance_count, var.dse_search_type)}"
   ... ...
}
```

#### 2.2.2. AWS Key-Pair

The script also creates an AWS key-pair resource that can be associated with the EC2 instances. The AWS key-pair resource is created from a locally generated SSH public key and the corresponding private key can be used to log into the EC2 instances.
```
resource "aws_key_pair" "dse_terra_ssh" {
    key_name = "${var.keyname}"
    public_key = "${file("${var.ssh_key_localpath}/${var.ssh_key_filename}.pub")}"
}

resource "aws_instance" "dse_search" {
   ... ...
   key_name        = "${aws_key_pair.dse_terra_ssh.key_name}"
   ... ... 
}
```

#### 2.2.3. Security Group

In order for the DSE cluster and OpsCenter to work properly, certain ports on the ec2 instances have to be open, as per the following DataStax documents:
* [Securing DataStax Enterprise ports](https://docs.datastax.com/en/dse/5.1/dse-admin/datastax_enterprise/security/secFirewallPorts.html)
* [OpsCenter ports reference](https://docs.datastax.com/en/opscenter/6.1/opsc/reference/opscLcmPorts.html)

The script does so by creating the following AWS security group resources:
1. sg_ssh: allows SSH access from public
2. sg_opsc_web: allows web Access from public, such as for OpsCenter Web UI
3. sg_opsc_node: allows OpsCenter related communication, such as between OpsCenter server and datastax-agent
4. sg_dse_node: allows DSE node specific communication

The code snippet below describes how a security group resource is defined and associated with EC2 instances.
```
resource "aws_security_group" "sg_ssh" {
   name = "sg_ssh"

   ingress {
      from_port = 22
      to_port = 22
      protocol = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
   }

   egress {
      from_port = 0
      to_port = 0
      protocol = "-1"
      cidr_blocks = ["0.0.0.0/0"]
   }
}

... ... // other security group definitions

resource "aws_instance" "dse_search" {
   ... ...
   vpc_security_group_ids = ["${aws_security_group.sg_internal_only.id}","${aws_security_group.sg_ssh.id}","${aws_security_group.sg_dse_node.id}"]
   ... ...
}
```

#### 2.2.4. User Data

One of the key requirements to run DSE cluster is to enable NTP service. The script achieves this through EC2 instance user data. which is provided through a terraform template file. 
```
data "template_file" "user_data" {
   template = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install python-minimal -y
              apt-get install ntp -y
              apt-get install ntpstat -y
              ntpq -pcrv
              EOF
}

resource "aws_instance" "dse_search" {
   ... ...
   user_data = "${data.template_file.user_data.rendered}"
   ... ...
}
```

Other than NTP service, python (minimal version) is also installed in order for Ansible to work properly. Please note that Java, as another essential software required by DSE and OpsCenter software, is currently installed through Ansible and therefore not listed here as part of the User Data installation. 

### 2.3. Limitation 

The terraform script presented in this section only focuses on the most fundamental AWS resources for DSE and OpsCenter installation and operation, such as EC2 instances and security groups in particular, For other AWS resources such as VPC, IP Subnet, and so on, we just rely on the default as provided by AWS. But the script should be very easy to extend to include other customized AWS resources.


## 3. Generate Ansible Inventory File Automatically

After the infrastructure instances have been provisioned, we need to install and configure DSE and OpsCenter and these instances accordingly, which is through the Ansible framework that I presented before at [here](https://github.com/yabinmeng/dseansible). One key item in the Ansible framework is the Ansible inventory file which determines key DSE node characteristics such as node IP, seed node, VNode, workload type, and so on. 

Now since we have provisioned the instances using terraform script, it is possible to generate the Ansible inventory file programmatically from terraform output state. Basically the idea is as below:
1. Generate terraform output state in a text file:
```
  terraform show terraform_extended/terraform.tfstate > $TFSTATE_FILE
```

2. Scan the terraform output state text file to generate a file that contains each instance's target DC tag, public IP, and private IP. An example is provided in this repository at: [dse_ec2IpList](https://github.com/thompson42/terradse/blob/master/dse_ec2IpList)

3. The same IP list information can also be used to generate the required Ansible inventory file. In the script, the first node in any DSE DC is automatically picked as the seed node. An example of the generated Ansible inventory file is provided in this repository: [dse_ansHosts](https://github.com/thompson/terradse/blob/master/dse_ansHosts)

A linux script file, ***genansinv_extended.sh***, is providied for this purpose. The script has 3 configurable parameters via input parameters. These parameters will impact the target DSE cluster topology information (as presented in the Ansible inventory file) a bit. Please adjust accordingly for your own case.

1. Script input argument: number of seed nodes per DC, default at 1
```
  genansinv_extended.sh [<number_of_seeds_per_dc>] [<dse_appcluster_name>] [<dse_opsccluster_name>]
```
2. Script input argument: the name of the application DSE cluster: 
```
  "MyAppClusterName"
```
3. Script input argument: the name of the OpsCenter monitoring cluster:
```
   "MyOpscClusterName"
```
4. e.g.:
```
   ./genansinv_extended.sh 1 dse_appcluster_name dse_opsccluster_name
```


## 4. This base forked version features:

1. *dse_install.yml*: installs and configures a multi-DC DSE cluster. This is the same functionality as the previous version.
2. *opsc_install.yml": installs OpsCenter server, datastax-agents, and configures accordingly to allow proper communication between OpsCenter server and datastax-agents.
3. *osparm_change.yml*: configures OS/Kernel parameters on each node where DSE is installed, as per [Recommended production settings](https://docs.datastax.com/en/dse/5.1/dse-admin/datastax_enterprise/config/configRecommendedSettings.html) from DataStax documentation.

For operational simplicity, a linux script file, ***runansi.sh***, is provided to execute these Ansible playbooks.

## 5. Additional features introduced by this fork

### 5.1 Creation of independently runnable Security_xyz playbooks under ansible/roles to configure:

1. Client -> node encryption
2. Node -> node encryption
3. Opscenter HTTPS access
4. Opscenter -> agent encryption

### 5.2 Introduction of spark and graph DSE datacenter types 

1. Extended versions of terraform file, use: terraform_extended.sh
2. Extended versions of .sh scripts to handle the new DC types


### 5.3 Added dse_set_heap role to automate setting HEAP for jvm.options file

1. See new params in group_vars/all: [heap_xms] and [heap_xmx] - always set them both to the same value to avoid runtime memory allocation issues.




