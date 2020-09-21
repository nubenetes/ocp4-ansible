# ocp4-ansible
My ansible roles for OpenShift 4:
- [ocp4-github-idp](roles/ocp4-github-idp/README.md)
- [ocp4-project-quota-management](roles/ocp4-project-quota-management/README.md)

The above roles were developed and integrated with [this repo](https://github.com/rcarrata/ocp4-auto-install) while collaborating within a devops team and the owner of the mentioned repo (a Red Hat architect). This collaboration based on automation tasks lasted only for 2-3 weeks, where I learnt how to develop and run code with the existing Ansible Tower and an existing code base (both implemented and sometimes controlled by Red Hat for a Customer with OCP4 on AWS). 

This shared code is kind of simple, although to be honest it took me a while to find the working solution considering my lack of ansible projects for the last two years. It was also my first significant experience with Ansible Tower (I am more familiar with Jenkins to even manage ansible or terraform jobs). 

Ansible's k8s module was applied following the tips of the team leader (red hat consultant).

I remembered how [variables](site.yml) can be defined in a data structure to define and setup a major issue not yet implemented by the customer: and **IDP like GitHub with RBAC permissions**. 

The design of this data structure and how to recover values from it with ansible's loop or ansible's with_subelements (see ansible task ['Apply Roles to Groups'](roles/ocp4-github-idp/tasks/rbac.yml) & ansible task ['Apply Default Quota to All Projects that END with a name'](roles/ocp4-project-quota-management/tasks/project-quota.yml)) was also a needed achievement to learn how to setup and automate other complex tasks. 

Another interesting point is how to recover and filtered out data from OCP with i.e. an oc command, data that is then saved and processes in a json variable (see ansible task ['Apply Default Quota to All Projects that END with a name'](roles/ocp4-project-quota-management/tasks/project-quota.yml)).

**Security in Cloud technologies like OpenShift is a must.**. Please avoid using cluster-admin unless absolutely necessary.

Regarding the automation of Namespace Configuration & teams onboarding, the following solutions can be a good approach if you don't want to reinvent the wheel while saving time, money and headaches:
- https://github.com/redhat-cop/namespace-configuration-operator
- https://github.com/redhat-cop/openshift-toolkit/tree/master/networkpolicy
