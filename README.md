# Kubespray Installation Feedback
In this report, I will be listing the steps during the [SLATE cluster installation with Kubespray](https://slateci.io/docs/cluster/install-kubernetes-with-kubespray.html) in which I got stuck on. I used 2 VMs for the test installation. Below are the names I used to describe these 2 VMs: 
- host machine: the VM on which the cluster will be installed
- deployment machine: the VM on which I ran the installation procedure and install the cluster to the host machine




## Kubernetes Cluster Creation
- step 2:
    - the `node1:` tags are to be replaced by the host machine's DNS name. The host machine's DNS name can be looked up with `hostname`
    - `slate_group_name`: after discussion with Emerson, he mention the group name in this field should be a already created group in the cluster. I mistakenly thought that I could create a new group by giving an arbitrary name in this field
- step 3:
    - I remembered in the previous installation guide there is line number associate with the fields. Now it's not there, so it's a bit hard to find the field in the `.yml` file.
- step 5:
    - in previous installations, I kept getting error at `TASK [registration : Register cluster with SLATE]`. The snippet of the error message is copied below:
        ```bash
        TASK [registration : Register cluster with SLATE] ************************************************************************************
        fatal: [li-zhang-training]: FAILED! => {"ansible_job_id": "458509196631.10572", "changed": true, "cmd": "slate cluster create lz-kspray-cluster --group 'lz-kspary-group' --org 'University of Chicago' -y ", "delta": "0:00:00.116869", "end": "2020-11-23 22:51:19.470750", "finished": 1, "msg": "non-zero return code", "rc": 1, "start": "2020-11-23 22:51:19.353881", "stderr": "slate: Exception: Unable to list deployments in the kube-system namespace; this command needs to be run with kubernetes administrator privileges in order to create the correct environment (with limited privileges) for SLATE to use.\nKubernetes error: The connection to the server 128.135.235.190:6443 was refused - did you specify the right host or port?", "stderr_lines": ["slate: Exception: Unable to list deployments in the kube-system namespace; this command needs to be run with kubernetes administrator privileges in order to create the correct environment (with limited privileges) for SLATE to use.", "Kubernetes error: The connection to the server 128.135.235.190:6443 was refused - did you specify the right host or port?"], "stdout": "Checking NRP-controller status...", "stdout_lines": ["Checking NRP-controller status..."]}
        ```
    - although the error message says that the port `6443` at the host machine cannot be connect, the true reason that the installation is not working is because the systemd version on the host machine is out of date. This issue is fixed by running `sudo yum update -y` on the host machine
## SLATE Cluster Creation
- Deploy step 1:
    - I got error at `TASK [registration : Register cluster with SLATE]`. The error message is listed below:
        ```bash
        TASK [registration : Register cluster with SLATE] ****************************************************************************************************************************************************************************
        fatal: [li-zhang-training]: FAILED! => {"changed": false, "msg": "async task did not complete within the requested time - 180s"}
        ```
    - This error is caused by the setting of `MetalLB` in `addons.yml`. I previously set it up as the IP of the host machine `128.135.235.190/24`. However, MetalLB requires an entirely different IP address than the one on host machine. When I give it `128.135.235.190/24`, it either fails because it’s not in standard CIDR format or it’ll try using any IP address in the `128.135.235.0/24` subnet.
    - I can either setup the `metallb_ip_range` as `128.135.235.0/24` or choose not to run with the `MetalLB` setting
    - the problem was solved by running the following steps:
        1. reset the cluster
            ```bash
            git checkout master && git pull && ansible-playbook -i inventory/lz-kspray-cluster/hosts.yaml --become --become-user=root -u centos reset.yml
            ```
        2. switch git branch
            ```bash
            git checkout v2.14.2
            ```
        3. rerun the K8S cluster creation
            ```bash
            ansible-playbook -i inventory/lz-kspray-cluster/hosts.yaml --become --become-user=root -u centos cluster.yml

            ```
        4. rerun the SLATE cluster creation with the flag to disable the metalLB
            ```bash
            ansible-playbook -i ~/kubespray/inventory/lz-kspray-cluster/hosts.yaml -u centos --become --become-user=root \
            -e 'slate_cli_token=<token>' \
            -e 'slate_cli_endpoint=https://api.slateci.io:443' \
            -e 'slate_enable_ingress=false' \
            site.yml
            ```
