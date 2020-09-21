# Configuration Management of OpenShift 4 with Ansible Roles and Ansible Tower
My ansible roles for OpenShift 4:
- [ocp4-github-idp](roles/ocp4-github-idp/README.md)
- [ocp4-project-quota-management](roles/ocp4-project-quota-management/README.md)

The above roles were developed and integrated with [this repo](https://github.com/rcarrata/ocp4-auto-install) while collaborating within a devops team and the owner of the mentioned repo (a Red Hat architect). This collaboration based on automation tasks lasted for only 2-3 weeks, where I learnt how to develop and run code with the existing Ansible Tower and an existing code base (both implemented and sometimes controlled by Red Hat for a Customer with OCP4 on AWS). 

This shared code is kind of simple, although to be honest it took me a while to find the working solution considering my lack of ansible projects for the last two years. It was also my first significant experience with Ansible Tower (I am more familiar with Jenkins to even manage ansible or terraform jobs). 

Ansible's k8s module was implemented following the tips of the team leader (red hat consultant).

I remembered how [variables](site.yml) can be defined in a data structure to define and setup a major issue not yet implemented by the customer: and **IDP like GitHub with RBAC permissions**. 

The design of this data structure and how to recover values from it with ansible's loop or ansible's with_subelements was also a needed achievement to learn how to setup and automate other complex tasks. See ansible task ['Apply Roles to Groups'](roles/ocp4-github-idp/tasks/rbac.yml) & ansible task ['Apply Default Quota to All Projects that END with a string'](roles/ocp4-project-quota-management/tasks/project-quota.yml).

Another interesting point is how to recover and filter out data from OCP with i.e. an oc command, data (a list of kubernetes resources) that is then saved and processed in a json variable. See ansible task ['Apply Default Quota to All Projects that END with a string'](roles/ocp4-project-quota-management/tasks/project-quota.yml).

**Security in Cloud technologies like OpenShift is a must. Please avoid using cluster-admin unless absolutely necessary.**

Lastly, **provisioning secrets** with **Ansible Tower's vault** was also a requirement to make sure no secrets where saved in a repository. See the documented procedure [here](roles/ocp4-github-idp/README.md).

There's room for improvement on this code: 
- *K8S_AUTH_KUBECONFIG* & ansible k8s' *kubeconfig* should be defined globally or per playbook instead of per ansible task (they couldn't be implemented by some constraints with the base code or Ansible Tower's config).
- ansible's with_subelements is deprecated and "loop" should be implemented instead.

## Alternatives to Ansible Tower (in case you don't want to learn a new tool)
[Ansible Tower](https://www.ansible.com/products/tower) or [Ansible AWX](https://github.com/ansible/awx) are not the only solutions to run ansible playbooks and roles:
- [Jenkins](https://www.jenkins.io/): My favourite one since it provides a large number of plugins and connectors. Plugins like OpenShift, Kubernetes, Terraform, Ansible, Packer, vaults and others can boost your productivity with the most popular CI tool among developers and devops engineers. Traceability of triggered jobs is easily achieved with logs collected on each jenkins job. Ansible Tower also provides logs but I find them with limited functionality when compared to jenkins (perhaps this can be customized accordingly).
- [Foreman](https://www.theforeman.org/)
- etc

## Namespace Configuration & Teams onboarding
Regarding the automation of Namespace Configuration & teams onboarding, the following solutions can be a good approach if you don't want to reinvent the wheel while saving time, money and headaches:
- https://github.com/redhat-cop/namespace-configuration-operator
- https://github.com/redhat-cop/openshift-toolkit/tree/master/networkpolicy

