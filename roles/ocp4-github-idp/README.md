- [GitHub IDP](#github-idp)
- [Group Sync Operator with GitHub Teams. OpenShift Projects & RBAC](#group-sync-operator-with-github-teams-openshift-projects--rbac)
- [References](#references)
  - [GitHub Identity Provider](#github-identity-provider)
  - [Ansible](#ansible)
  - [Ansible Vault and Tower](#ansible-vault-and-tower)
  - [Group Sync Operator](#group-sync-operator)
  - [Kubernetes Secrets](#kubernetes-secrets)
  - [RBAC](#rbac)

# Authentication

## GitHub IDP
The following procedure sets up from scratch OpenShift Container Platform 4 with GitHub Identity Provider (GitHub IDP). A github user account with privileged permissions on your GitHub Organization is required.

1. Setup your GitHub organization via Web UI with:
   - **<my_github_organization>** -> Settings -> Developer settings -> OAuth Apps -> **New OAuth App**
   - Application Name: **<my_base_domain.com>**
   - Homepage URL: **https://oauth-openshift.apps.<my_clustername>.<my_base_domain.com>**
   - Application description: OpenShift Container Platform Disaster Recovery Cluster
   - Authorization callback URL: **https://oauth-openshift.apps.<my_clustername>.<my_base_domain.com>/oauth2callback/githubidp/**

2. Grab the created **token values** for **Client ID** and **Client Secret**. They will be used in the following steps.

3. Setup vars.yml accordingly:
   - **<my_github_client_id>** refers to **client ID** obtained in **step #2**.
   - Fill in **github_teams** with the list of teams (groups) defined in your GitHub Organization (<my_github_organization>) that you want to grant access to your OCP Cluster. 

    ```yaml
    # OAUTH 
    #oauth-github-idp
    github_client_id: <my_github_client_id>
    github_teams:
      - <my_github_organization>/devops-cloud-admin
      - <my_github_organization>/devops-cloud-view
    ```

4. **Encrypt** the literal (non-encoded) **Client Secret** of **step #2** with **ansible-vault** and **vault-<my_clustername>** *vault-id* (one vault can contain several secrets, each of them referenced with a **vaultID**). 
   - You will be prompted by a password to encrypt your vault. Create a new password with [keypass](https://keepass.info/) and save it in our [shared keypass database](https://github.com/<my_github_organization>/devops-tools/tree/master/keypass). This password will be used later.

    ```bash
    ansible-vault encrypt_string --vault-id vault-<my_clustername>@prompt '<my_non_encoded_secret>' --name 'github_client_secret'
    New vault password (vault-<my_clustername>):
    Confirm new vault password (vault-<my_clustername>):
    ```

    - **Note:** **Pay attention to secret names in ansible!** **Use underscores in ansible var names** instead of middle dashes. The above ansible-vault command with *'github-client-secret'* would make your OCP unable to authenticate against GitHub.

5. Transform the outcome of the previous step
   - **Before:**

   ```yaml
   github_client_secret: !vault | 
       $ANSIBLE_VAULT;1.2;AES256;vault-<my_clustername>
             64386534373136626333343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             63303738373439333662343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             66343931613461633533393233616638356
   ```

   - **After:**

   ```yaml
   github_client_secret:
      __ansible_vault: |
             $ANSIBLE_VAULT;1.2;AES256;vault-<my_clustername>
             64386534373136626333343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             63303738373439333662343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             66343931613461633533393233616638356
   ```

6. Add the outcome of the previous step to **Ansible Tower** -> **Inventories** -> **OCP** inventory -> **VARIABLES** (YAML).
7. Add **OCP inventory** to your Ansible Template in Tower that runs **deploy_only_oauth.yml** playbook.
8.  Add your **Vault Password** generated in **step #5** to Tower -> Credentials -> **vault-<my_clustername>** credential:
    - NAME: **vault-<my_clustername>**
    - DESCRIPTION: <my_github_organization> GitHub Organization
    - ORGANIZATION: <My Organization Name>
    - CREDENTIAL TYPE: **Vault**
    - VAULT PASSWORD: your **Vault Password** generated in **step #5** 
    - VAULT IDENTIFIER: **vault-<my_clustername>** 
9.   Add already defined **vault-<my_clustername>** to your Tower -> Templates -> \<your template\> -> **CREDENTIALS**. Multiple credentials can be added simultaneously.
10. Launch you Ansible Template in order to setup OCP4 with <my_github_organization> GitHub Organization.
11. Check your deployed secret (optional):
    - Check the secret encoded with base64: 
        ```oc get secret github-client-secret -n openshift-config --template={{.data.clientSecret}}```
    - Check the decoding of github-client-secret matches the literal (non-encoded) secret of **step 2**: 
        ```oc get secret github-client-secret -n openshift-config --template={{.data.clientSecret}} | base64 -d```
    - Secret deletion (optional). The goal is to provision this secret with Ansible:
        ```oc delete secret github-client-secret -n openshift-config```

## Group Sync Operator with GitHub Teams. OpenShift Projects & RBAC
The following procedure deploys [Group Sync Operator](https://github.com/redhat-cop/group-sync-operator): Teams stored within a GitHub organization can be synchronized into OpenShift. 

**Features** added to this playbook:
-  Teams stored within **<my_github_organization>** GitHub organization are synchronized into OpenShift.
-  **Creation of Projects/Namespaces** (unless they already exist).
-  **RBAC** at the cluster level and namespace level.

**Note:** Some of the **below steps are already applied** during the setting up of GitHub IDP.

1. Create an [OAuth Personal Access Token in your github account](https://github.com/settings/tokens) to authenticate *Group Sync Operator* against GitHub: 

   - GitHub -> settings -> Personal access tokens -> Generate new token:  
    - Note: **group-sync-operator-<my_clustername>**
    - Select scopes: **read:org**
    
2. Copy the token generated in step #1 and save it in [our shared keypass database](https://github.com/<my_github_organization>/devops-tools/tree/master/keypass).

3. **Encrypt** the generated token with **ansible-vault** and **vault-<my_clustername>** *vault-id* (one vault can contain several secrets, each of them referenced with a **vaultID**). 
   - You will be prompted by a password to encrypt your vault. Create a new password with [keypass](https://keepass.info/) and save it in our [shared keypass database](https://github.com/<my_github_organization>/devops-tools/tree/master/keypass). This password will be used later.

    ```bash
    ansible-vault encrypt_string --vault-id vault-<my_clustername>@prompt '<my_non_encoded_token>' --name 'github_oauth_personal_access_token'
    New vault password (vault-<my_clustername>):
    Confirm new vault password (vault-<my_clustername>):
    ```

4. Transform the outcome of the previous step
   - **Before:**

   ```yaml
   github_oauth_personal_access_token: !vault | 
       $ANSIBLE_VAULT;1.2;AES256;vault-<my_clustername>
             64386534373136626333343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             63303738373439333662343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             66343931613461633533393233616638356
   ```

   - **After:**

   ```yaml
   github_oauth_personal_access_token:
      __ansible_vault: |
             $ANSIBLE_VAULT;1.2;AES256;vault-<my_clustername>
             64386534373136626333343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             63303738373439333662343965383634393261623034306532663132333261323566366466383263
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             6634393161346163353339323361663835613236643931360a396230663032313631643532656635
             66343931613461633533393233616638356
   ```

5. Add the outcome of the previous step to **Ansible Tower** -> **Inventories** -> **OCP** inventory -> **VARIABLES** (YAML).
6. Add **OCP inventory** to your Ansible Template in Tower that runs **deploy_only_oauth.yml** playbook.
7.  Add your **Vault Password** generated in **step #5** to Tower -> Credentials -> **vault-<my_clustername>** credential:
    - NAME: **vault-<my_clustername>**
    - DESCRIPTION: <my_github_organization> GitHub Organization
    - ORGANIZATION: <my_company>
    - CREDENTIAL TYPE: **Vault**
    - VAULT PASSWORD: your **Vault Password** generated in **step #5** 
    - VAULT IDENTIFIER: **vault-<my_clustername>** 
8.   Add already defined **vault-<my_clustername>** to your Tower -> Templates -> \<your template\> -> **CREDENTIALS**. Multiple credentials can be added simultaneously.
9.  Setup **site.yml** accordingly: Fill in **github_groupsync_organization** and **github_groupsync_teams** with the list of teams (groups) defined in your GitHub Organization that you want to synchronize against your OCP Cluster. **RBAC permissions** granted to each team/group (at the cluster level or Project level) are defined here as well:

    ```yaml
    # OAUTH  & RBAC & Projects
    # oauth-github-idp 
    github_client_id: 08e7a8689608e7a8689b
    github_teams:
      - <my_github_organization>/devops-cloud-admin
      - <my_github_organization>/devops-cloud-view
      - <my_github_organization>/project1-admin
      - <my_github_organization>/project1-view
      - <my_github_organization>/project1-edit
    # github-group-sync & rbac & projects
    github_groupsync_organization: <my_github_organization>
    github_groupsync_teams:
    - github_team: 'devops-cloud-admin'
      cluster_roles: # add-cluster-role-to-group
        - 'cluster-admin'           # 'cluster-admin' role
        - 'cluster-monitoring-view' # Enabling access to Grafana and Kibana via cluster-monitoring-view cluster role
      roles: []      # groups with 'cluster-admin' cluster role don't need project roles
    - github_team: 'devops-cloud-view'
      cluster_roles: # add-cluster-role-to-group
        - 'view'       # 'view' cluster role
        - 'cluster-monitoring-view' 
      roles: []      # groups with 'view' cluster role don't need project roles
    - github_team: 'project1-admin'
      project: project1   # project/namespace in OCP
      cluster_roles: 
        - 'cluster-monitoring-view' # Enabling access to Grafana and Kibana via cluster-monitoring-view cluster role
      roles: # add-role-to-group
        - 'admin'      # 'admin' role within the Project/Namespace scope
    - github_team: 'project1-edit'
      project: project1   # project/namespace in OCP
      cluster_roles: []
      roles: 
        - 'edit'       # 'edit' role within the Project/Namespace scope
    - github_team: 'project1-view'
      project: project1   # project/namespace in OCP
      cluster_roles:
        - 'cluster-monitoring-view' 
      roles: 
        - 'view'       # 'view' role within the Project/Namespace scope
    ```

10. Launch you Ansible Template in order to deploy **Group Sync Operator with GitHub Teams and RBAC**.

## References
### GitHub Identity Provider
- https://docs.openshift.com/container-platform/4.4/authentication/identity_providers/configuring-github-identity-provider.html

### Ansible
- https://serverfault.com/questions/610982/ansible-iterate-a-dictionary-with-lists

### Ansible Vault and Tower
- https://developers.redhat.com/blog/2020/01/30/vault-ids-in-red-hat-ansible-and-red-hat-ansible-tower/
- https://medium.com/@claudio.domingos/ansible-awx-from-scratch-to-rest-api-part-7-of-8-c84bc443de6e
- https://www.digitalocean.com/community/tutorials/how-to-use-vault-to-protect-sensitive-ansible-data-on-ubuntu-16-04

### Group Sync Operator
- https://github.com/redhat-cop/group-sync-operator

### Kubernetes Secrets
- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/concepts/configuration/secret/#creating-a-secret-manually
- https://opensource.com/article/19/6/introduction-kubernetes-secrets-and-configmaps
- https://stackoverflow.com/questions/56909180/decoding-kubernetes-secret 
- [Why is there an inconsistency in base64 output?](https://askubuntu.com/questions/680218/why-is-there-an-inconsistency-in-base64-output)

### RBAC
- https://docs.openshift.com/container-platform/4.4/authentication/using-rbac.html
- [Configure RBAC in Kubernetes Like a Boss](https://medium.com/trendyol-tech/configure-rbac-in-kubernetes-like-a-boss-665e2a8665dd)

