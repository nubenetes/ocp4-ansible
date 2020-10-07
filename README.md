# Configuration Management of OpenShift 4 with Ansible Roles and Ansible Tower. GitHub Identity Provider with IAM/RBAC on OCP4. Project Quota Management & Limits
My ansible roles for OpenShift 4:
- [ocp4-github-idp](roles/ocp4-github-idp/README.md)
- [ocp4-project-quota-management](roles/ocp4-project-quota-management/README.md)

The above roles were developed and integrated with a clone of [this repo](https://github.com/rcarrata/ocp4-auto-install) while collaborating within a devops team and the owner of the mentioned repo (a Red Hat architect). This collaboration based on automation tasks lasted for only 2-3 weeks, where I learnt how to develop and run code with the existing Ansible Tower and an existing code base (both implemented and sometimes controlled by Red Hat for a Customer with **OCP4 on AWS** ([ref1](https://github.com/openshift/installer/blob/master/docs/user/aws/README.md) , [ref2](https://aws.amazon.com/blogs/opensource/openshift-4-on-aws-quick-start/)).

This shared code is kind of simple, although to be honest it took me a while to find the working solution considering my lack of ansible projects for the last two years. It was also my first significant experience with Ansible Tower (I am more familiar with Jenkins to even manage ansible or terraform jobs). We had no access to the server hosting Ansible Tower and a bit of reverse engineering was needed to identify things like the right file paths or a valid kubeconfig.

**Ansible k8s module** was implemented following the good tips of the team leader (the mentioned red hat consultant).

I remembered how [variables](site.yml) with nested items (lists and dictionaries) can be defined in a data structure to define and setup a major issue not yet implemented by the customer: and **IDP like GitHub with IAM/RBAC permissions**. 

The design of this data structure and how to recover nested values from it with **ansible loop** or **ansible with_subelements** was also a needed achievement to learn how to setup and automate other complex tasks. See ansible task ['Apply Roles to Groups'](roles/ocp4-github-idp/tasks/rbac.yml) & ansible task ['Apply Default Quota to All Projects that END with a string'](roles/ocp4-project-quota-management/tasks/project-quota.yml).

Another interesting point is how to recover and filter out data from OCP with i.e. an oc command, data (a list of kubernetes resources) that is then saved and processed in a json variable. See ansible task ['Apply Default Quota to All Projects that END with a string'](roles/ocp4-project-quota-management/tasks/project-quota.yml).

**Security in Cloud technologies like OpenShift is a must. Please avoid using cluster-admin unless absolutely necessary. Likewise, please don't run Proof of Concepts in production clusters.**

Lastly, **provisioning secrets** with **Ansible Tower Vault** was also a requirement to make sure no secrets where saved in a repository. See the documented procedure [here](roles/ocp4-github-idp/README.md).

## Improvements
There's room for improvement on this code: 
- *K8S_AUTH_KUBECONFIG* & ansible k8s module *kubeconfig* should be defined globally or per playbook instead of per ansible task (they couldn't be implemented by some constraints with the base code or Ansible Tower's config).
- ansible with_subelements is deprecated and "ansible loop subelements" should be implemented instead.
- github_teams was defined & developed before github_groupsync_teams in site.yml. To avoid data duplication github_teams could be removed by updating the corresponding jinja template along with the documentation. 
- [ocp4-project-quota-management](roles/ocp4-project-quota-management/README.md) has been designed and tested against a single OCP cluster. The existing code could evolve to more complex solutions where each enviroment (dev, qa, ua, prod, etc) and corresponding namespaces are setup in dedicated OCP clusters (dev, qa, ua, prod, etc).

## Team synchronization across GitHub and your master LDAP
[GitHub IDP on OCP4](https://docs.openshift.com/container-platform/4.5/authentication/identity_providers/configuring-github-identity-provider.html) requires groups (GitHub Teams) and users to be setup manually via GitHub Web UI.

The following solutions can help to achieve an automated way of managing github teams and users:

- Automated management of github teams and users with GitHub API Rest: a custom development with [GitHub API Rest](https://docs.github.com/en/rest/reference/teams) can be a good approach in case you need an automated and scalable solution or a sync against your i.e. on-premises LDAP. Hopefully [github cli](https://cli.github.com/) will also provide this functionality in the near future.  
- Automated management of github teams and users with 3rd party solutions:
    - [Team synchronization across GitHub and Azure Active Directory](https://github.blog/2019-05-06-team-synchronization-across-github-and-azure-active-directory/)
    - https://github.com/github/ldap-teamsync : Sync GitHub teams to groups in Active Directory when any authentication method for GitHub. The original target was SAML, but is not restricted to this authentication method.
    - etc

## How to export kubernetes resources without metadata to be used as templates
```oc/kubectl --export``` is deprecated (click [here](https://stackoverflow.com/questions/43941772/get-yaml-for-deployed-kubernetes-services)). Setup one of the following [kubectl plugins](https://github.com/kubernetes-sigs/krew-index/blob/master/plugins.md) in order to export kubernetes resources without metadata. You will obtain a valid kubernetes manifest that can be applied as a template in your ansible tasks:
- [kubectl-neat](https://github.com/itaysk/kubectl-neat) Remove clutter from Kubernetes manifests to make them more readable.
- [kubectl-eksporter](https://github.com/Kyrremann/kubectl-eksporter) is a kubectl plugin designed to export Kubernetes resources, and remove pre-defined set of fields. You can finally export resources with `kubectl get pod <pod-id> -o yaml` and skip the `status` field

## Alternatives to Ansible Tower (in case you don't want to learn a new tool)
[Ansible Tower](https://www.ansible.com/products/tower) or [Ansible AWX](https://github.com/ansible/awx) are not the only solutions to run ansible playbooks and roles:
- [Jenkins](https://www.jenkins.io/): My favourite one since it provides a large number of plugins and connectors. Plugins like OpenShift, Kubernetes, Terraform, Ansible, Packer, vaults and others can boost your productivity with the most popular CI tool among developers and devops engineers. Traceability of triggered jobs is easily achieved with logs collected on each jenkins job. Ansible Tower also provides logs but I find them with limited functionality when compared to jenkins (only the last 10 logs are kept per ansible job/template, perhaps this can be customized accordingly).
- [Foreman](https://www.theforeman.org/)
- [Rundeck](https://www.rundeck.com/ansible)
- etc

## Namespace Configuration & Teams onboarding
Regarding the automation of Namespace Configuration & teams onboarding, the following solutions can be a good approach if you don't want to reinvent the wheel while saving time, money and headaches:
- https://github.com/redhat-cop/namespace-configuration-operator
- https://github.com/redhat-cop/openshift-toolkit/tree/master/networkpolicy

Automate repetitive and critical tasks like a boss and move to the next level. You don't need the qualifications of a NASA engineer to manage OpenShift! Happy coding!