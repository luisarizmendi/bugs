# rpm-ostree issue

## The issue 

I faced the issue while preparing a [demo guide about RHEL OSTree](https://github.com/luisarizmendi/edge-demos/tree/main/demos/upgrade-and-rollback) using the [Infra OSBuild Ansible Collection](https://github.com/redhat-cop/infra.osbuild).

In that demo I deploy a RHEL OSTree system with the ostree remote pointing to a repo published in an HTTP server running on the Image Builder.

After "publishing" a new OSTree files with the upgraded image in the remote HTTP server, the OSTree system cannot run `rpm-ostree upgrade --check` or `rpm-ostree upgrade --preview`, this error is shown:

```
[root@localhost ~]# rpm-ostree upgrade --check
2 metadata, 0 content objects fetched; 402 B transferred in 0 seconds; 0 bytes content written
error: Bus owner changed, aborting. This likely means the daemon crashed; check logs with `journalctl -xe`.
```

If you run just `rpm-ostree upgrade` the system gets the upgrade and then you can use again `rpm-ostree upgrade --check` and `rpm-ostree upgrade --preview` as can be seen in this capture:



![](files/rpm-ostree%20check%20issue.gif)




*If you "publish" the new image by just copying the new files into the repository (HTTP server) everything works ok*, but the collection does not just copy the files, it pull the commit using the `ostree` command, as you can see [in this ansible block](https://github.com/redhat-cop/infra.osbuild/blob/main/roles/builder/tasks/main.yml#L84), so im my scripts [I had to use the "just copy files" approach.](https://github.com/luisarizmendi/edge-demos/blob/main/common/playbooks/publish-image.yml)
 

## Files

[journalctl -u rpm-ostreed.service -b](files/journalctl)

[coredumpctl dump](files/coredumpctl) (no info)


## How to reproduce using RHEL

### Preparation

* You need a couple of VMs running on your laptop, the first one will be used as Image Builder where RHEL 9 must be pre-installed, the second one the OSTree system that is an "empty" (no SO installed) VM. The pre-installed (minimal install is enough) RHEL system (image builder VM) needs to be subscribed [Red Hat Enterprise Linux 9](https://access.redhat.com/downloads/content/479/ver=/rhel---9/9.1/x86_64/product-software) with at least 2 vCPUs, 4 GB memory and 50 GB disk.

* In your laptop install ansible and the infra osbuild Ansible collection:

```
ansible-galaxy collection install -f git+https://github.com/redhat-cop/infra.osbuild
```

* Modify the Ansible `inventory` file with your values pointing to the Image builder

* Copy your public SSH key into the Image Builder system, so you can open passwordless SSH sessions with the user that you configured in your Ansible inventory. Also be sure that the user is in sudoers with no password.


### Deploying the image v1

Run the first Ansible Playbook:

```
ansible-playbook -vvi inventory playbooks/01-version1_and_ISO.yml
```

It will:
* Install Image Builder in the pre-installed RHEL VM
* Create the OSTree Image v1 and publish it using an HTTP server (in the same server)
* Create a RHEL custom ISO that will be used to deploy the RHEL OSTree edge system pointing to the OSTree repo published in the Image Builder (v1)

The ISO is customized with this line in the kernel args: `inst.ks=http://<image_builder_IP>/kickstart.ks`.

The kickstart installs the OSTree repo along with other customizations (you can remove them): [kickstart.ks](templates/kickstart_demo-upgrade.j2)


Once the Ansible Playbook is finished, you will see the URL where the *custom* ISO is published in the last Ansible `debug` message. Download it to the system where you will create the Edge device VM.

Create a new VM () with 1 vCPU, 1.5GB memory, 20GB disk and one NIC in a network from where it has access to the Image Builder. Boot from the ISO and wait until the auto-installation is done.

Connect to the new system by SSH using the user `admin`. You shouldn't need password if you used the same laptop from where you ran the demo preparation since the SSH public key was injected into the OSTree image, otherwise you can use the password configured in the Blueprint (`R3dh4t1!`).


Now you can check `rpm-ostree upgrade --check` and `rpm-ostree upgrade --preview`, everything shoud be fine.


### Reproducing the issue by creating a new image

Run the first Ansible Playbook:

```
ansible-playbook -vvi inventory playbooks/02-version2.yml
```

It will:
* Create a new image (adds `zsh` package)
* Publish the new image in the HTTP server


Now you can run again `rpm-ostree upgrade --check` and `rpm-ostree upgrade --preview` and see the issue.

