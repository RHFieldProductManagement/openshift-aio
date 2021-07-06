# OpenShift All-in-One Deployment

~~~bash
$ git clone https://github.com/RHFieldProductManagement/openshift-aio.git
$ cd openshift-aio
$ cp sample_vars.yml my_vars.yml
~~~

Now you can edit your variables to your configuration:

~~~bash
$ vi my_vars.yml
~~~

When you're ready to deploy a cluster, call main.yml making sure you specify your variables and the dynamic inventory (don't worry that it doesn't exist yet):

~~~bash
$ ansible-playbook playbooks/main.yml -e @my_vars.yml -i inventory.yml
~~~

To destroy your cluster, call destroy.yml, again making sure to specify your variables and dynamic inventory:

~~~bash
$ ansible-playbook playbooks/destroy.yml -e @my_vars.yml -i inventory.yml
~~~
