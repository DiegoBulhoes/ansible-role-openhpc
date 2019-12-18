[![Build Status](https://travis-ci.com/stackhpc/ansible-role-openhpc.svg?branch=master)](https://travis-ci.com/stackhpc/ansible-role-openhpc)

# stackhpc.openhpc

This Ansible role is used to install the necessary packages to have a fully functional OpenHPC cluster.

Role Variables
--------------

`openhpc_slurm_service_enabled`: checks whether `openhpc_slurm_service` is enabled

`openhpc_slurm_service`: name of the slurm service e.g. `slurmd`

`openhpc_slurm_control_host`: ansible host name of the controller e.g `"{{ groups['cluster_control'] | first }}"`

`openhpc_slurm_partitions`: list of one or more slurm partitions.  Each partition may contain the following values:
* `groups`: If there are multiple node groups that make up the partition, a list of group objects can be defined here.
  Otherwise, `groups` can be omitted and the following attributes can be defined in the partition object.
  * `name`: The name of the nodes within this group.
  * `cluster_name`: Optional.  An override for the top-level definition `openhpc_cluster_name`.
  * `min_nodes`: Minimum number (i.e. number of persistent) nodes within the group.
  * `max_nodes`: Maximum number of nodes within the group.
  * `ram_mb`: Optional.  The physical RAM available in each server of this group.
  Compute node hostnames are assumed to take the form: `cluster_name-group_name-{0..max_nodes-1}`
* `default`: Optional.  A boolean flag for whether this partion.  Valid settings are `YES` and `NO`.
* `maxtime`: Optional.  A partition-specific time limit in hours, minutes and seconds.  The default value is
  `openhpc_job_maxtime`, which defaults to `24:00:00`.

`openhpc_job_maxtime`: A maximum time job limit in hours, minutes and seconds.  The default is `24:00:00`.

`openhpc_cluster_name`: name of the cluster

`openhpc_packages`: additional OpenHPC packages to install

`openhpc_enable`:
* `control`: whether to enable control host
* `batch`: whether to enable compute nodes
* `runtime`: whether to enable OpenHPC runtime
* `drain`: whether to drain compute nodes
* `resume`: whether to resume compute nodes

Example Inventory
-----------------

And an Ansible inventory as this:

    [openhpc_login]
    openhpc-login-0 ansible_host=10.60.253.40 ansible_user=centos

    [openhpc_compute]
    openhpc-compute-0 ansible_host=10.60.253.31 ansible_user=centos
    openhpc-compute-1 ansible_host=10.60.253.32 ansible_user=centos

    [cluster_login:children]
    openhpc_login

    [cluster_control:children]
    openhpc_login

    [cluster_batch:children]
    openhpc_compute

Example Playbooks
----------------

To deploy, create a playbook which looks like this:

    ---
    - hosts:
      - cluster_login
      - cluster_control
      - cluster_batch
      become: yes
      roles:
        - role: openhpc
          openhpc_enable:
            control: "{{ inventory_hostname in groups['cluster_control'] }}"
            batch: "{{ inventory_hostname in groups['cluster_batch'] }}"
            runtime: true
          openhpc_slurm_service_enabled: true
          openhpc_slurm_control_host: "{{ groups['cluster_control'] | first }}"
          openhpc_slurm_partitions:
            - name: "compute"
              flavor: "compute-A"
              image: "CentOS7.5-OpenHPC"
              num_nodes: 8 # FIXME
              user: "centos"
          openhpc_cluster_name: openhpc
          openhpc_packages: []
    ...

Note that the "compute" of the openhpc_slurm_partition name and the
openhpc_cluster_name are used to generate the compute node in the
slurm config of openhpc-compute-[0:7]. Your inventory entries
for that partition must match that convention.

To drain nodes, for example, before scaling down the cluster to 6 nodes:

    ---
    - hosts: openstack
      gather_facts: false
      vars:
        partition: "{{ cluster_group.output_value | selectattr('group', 'equalto', item.name) | list }}"
        openhpc_slurm_partitions:
          - name: "compute"
            flavor: "compute-A"
            image: "CentOS7.5-OpenHPC"
            num_nodes: 6 # FIXME
            user: "centos"
        openhpc_cluster_name: openhpc
      roles:
        # Our stackhpc.cluster-infra role can be invoked in `query` mode which
        # looks up the state of the cluster by querying the Heat API.
        - role: stackhpc.cluster-infra
          cluster_name: "{{ cluster_name }}"
          cluster_state: query
          cluster_params:
            cluster_groups: "{{ cluster_groups }}"
      tasks:
        # Given that the original cluster that was created had 8 nodes and the
        # cluster we want to create has 6 nodes, the computed desired_state
        # variable stores the list of instances to leave untouched.
        - name: Count the number of compute nodes per slurm partition
          set_fact:
            desired_state: "{{ (( partition | first).nodes | map(attribute='name') | list )[:item.num_nodes] + desired_state | default([]) }}" # FIXME
          when: partition | length > 0
          with_items: "{{ openhpc_slurm_partitions }}"
        - debug: var=desired_state

    - hosts: cluster_batch
      become: yes
      vars:
        desired_state: "{{ hostvars['localhost']['desired_state'] | default([]) }}"
      roles:
        # Now, the stackhpc.openhpc role is invoked in drain/resume modes where
        # the instances in desired_state are resumed if in a drained state and
        # drained if in a resumed state.
        - role: stackhpc.openhpc
          openhpc_slurm_control_host: "{{ groups['cluster_control'] | first }}"
          openhpc_enable:
            drain: "{{ inventory_hostname not in desired_state }}"
            resume: "{{ inventory_hostname in desired_state }}"
    ...

