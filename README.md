# Running WRF with CycleCloud
blabla
## Pre-requisites

## Installation

### 1. Step 1

### 2. Step 2

Before installing Terraform is worth reading some of the [Terraform](https://learn.hashicorp.com/terraform) documentation.

**Ubuntu/Debian Installation**
```
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform
```
**Centos/RedHat Installation**
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install terraform
```
**Verify installation**

* [Azure Cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed

