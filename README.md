# CronJob - Refresh Connection - External Secrets to Hashicorp Vault
This project provides a solution to ensure the connection between Hashicorp Vault and External Secrets after the reboot of OpenShift or even instabilities.

# Problem
When OpenShift is restarted, two things happen that can cause connection problems:

- Hashicorp Vault becomes unavailable and to reactivate it, it is necessary to unseal all pods.
- External Secrets loses its connection with Vault and to reestablish it, it is necessary to add an annotation to all resources related to External Secrets, as this forces a reload of them.

# Solution
The solution to maintain the connection between Hashicorp Vault and External Secrets is a Helm Chart that contains a CronJob that runs every 5 minutes. The CronJob performs two actions:

1. Unseals the Vault pods if they are unready.
2. Adds an annotation to all resources related to External Secrets.

# To do
Create a customized ClusterRole, is not recommended to give cluster-admin to a service account like this.