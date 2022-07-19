# Ansible Refactoring & Static Assignments (Imports and Roles)

This project builds upon the previous project by making improvements of the code and working with the ansible-config-mgt repository. The tasks in this project include refactoring the code, creating assignments and learning to use the imports functionality. Imports allow to effectively re-use previously created playbooks in a new playbook - it allows to organize tasks and reuse them when needed.

For better understanding or Ansible artifacts re-use, refer to [ansible docs](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse.html).


## Code Refactoring

Refactoring is a general term in computer programming. It means making changes to the source code without changing expected behaviour of the software. The main idea of refactoring is to enhance code readability, increase maintainability and extensibility, reduce complexity, add proper comments without affecting the logic.

In this project, the primary objective is to improve the Ansible code by reorganizing things around a little bit but the overall state of the infrastructure remains the same.



## Step 1 - Jenkins job enhancement

With the configuration so far, every change made creates a separate directory on Jenkins, which can get tedious and also consume more space. In this step, a new Jenkins job will be created to enhance the configuration.

1. On the jenkins-ansible server, create a new directory called 'ansible-config-artifact'. This directory will store the artifacts from all subsequent builds:

```
sudo mkdir /home/ubuntu/ansible-config-artifact
```

Change permissions to this directory, so Jenkins could save files there:

```
chmod -R 0777 /home/ubuntu/ansible-config-artifact
```

2. On Jenkins web console -> `Manage Jenkins` -> `Manage Plugins` -> on Available tab search for `Copy Artifact` and install this plugin without restarting Jenkins.

3. Create a Freestyle project and name it `save_artifacts`. Configure the project to be triggered by completion of the existing ansible project.

The main idea of save_artifacts project is to save artifacts into `/home/ubuntu/ansible-config-artifact` directory. To achieve this, create a Build step and choose `Copy artifacts from other project`, specify ansible as a source project and `/home/ubuntu/ansible-config-artifact` as a target directory.


![Artifact](./media/setartifact.png)

![Artifact](./media/setartifact2.png)

![Artifact](./media/setartifact3.png)


You can configure number of builds to keep in order to save space on the server, for example, you might want to keep only last 2 or 5 build results.

4. Test the set up by making some change in README.MD file inside the ansible-config-mgt repository (right inside main branch). If both Jenkins jobs have completed one after another - all the files will be inside /home/ubuntu/ansible-config-artifact directory and it will be updated with every commit to the main branch.

![Build successful](./media/artifactup.png)


![Artifact saved](./media/artifactsaved.png)

Now the Jenkins pipeline is neater.

## Step 2 - Refactor Ansible code by importing other playbooks into site.yml

Here, we're breaking tasks up into different files so as to organise sets of tasks and reuse them. Now, we'll try code re-use by importing other playbooks.

Checkout to a new branch called refactor:

```
git checkout -b refactor
```

1. Within `playbooks folder`, create a new file and name it `site.yml` which would be an entry point into the entire infrastructure configuration. Other playbooks will be included here as a reference. This means `site.yml` will become a parent to all other playbooks that will be developed.
2. Create a new folder in root of the repository and name it `static-assignments`. The `static-assignments` folder is where all other child playbooks will be stored. This is merely for easy organisation of our work.
3. Move `common.yml` file into the newly created static-assignments folder.

![Refactoring](./media/refactorbr3.png)


Our folder structure should look like this:

```
├── static-assignments
│   └── common.yml
├── inventory
    └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
```


4. Inside `site.yml` file, import `common.yml` playbook with the following code:

```
---
- hosts: all
- import_playbook: ../static-assignments/common.yml
```

5. Run ansible-playbook command against the `dev` environment.

We would need to apply some tasks to our `dev` servers. Since wireshark is already installed, we can create another playbook under `static-assignments` and name it `common-del.yml`. In this playbook, we would configure deletion of wireshark utility:

```
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark-qt
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes
```

In `/etc/ansible/ansible.cfg` file uncomment `inventory string` and provide a full path to the `inventory directory = /home/ubuntu/ansible/ansible-config/inventory`, so Ansible could know where to find configured roles.

![Inventory path](./media/inventorypath.png)



6. Update site.yml` with:

```
import_playbook: ../static-assignments/common-del.yml
```

instead of `common.yml` and run it against `dev` servers:

```
ansible-playbook -i /home/ubuntu/ansible/ansible-config-mgt/inventory/uat.yml /home/ubuntu/ansible/ansible-config-mgt/playbooks/site.yml
```





