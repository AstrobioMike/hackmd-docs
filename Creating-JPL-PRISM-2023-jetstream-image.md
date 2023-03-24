---
tags: JetStream
title: Creating 2023 PRISM image
---

# Creating 2023 PRISM image

[toc]

GUI used was Jetstream2 exosphere: https://jetstream2.exosphere.app/

## Launching initial instance we are building ours on 

Ubuntu 22.04 (latest)

![](https://i.imgur.com/ZtvqK91.png)

M3.large for what we're doing here

![](https://i.imgur.com/0axgkgN.png)

Click Create.

Will go through stages, e.g., says "Building" when it starts:

![](https://i.imgur.com/HNkUjbP.png)

Likely should only take a few minutes.

Once launched (it will say "Ready").

## Setting up a shared file-system for reference dbs

Doing with [manila](https://docs.jetstream-cloud.org/ui/horizon/manila/). 

Logged into Horizon as detailed [here](https://docs.jetstream-cloud.org/ui/horizon/login/). 

Then followed the steps [here](https://docs.jetstream-cloud.org/ui/horizon/manila/):

![](https://i.imgur.com/demLxNY.png)

Manage Rules -> Add Rule:

![](https://i.imgur.com/zzV9AHM.png)

Back on Project / Share / Shares page, can select the share and see the metadata, key things are "Path", and "Access Key".

Then onto [here](https://docs.jetstream-cloud.org/general/manilaVM/) after logging in below.



## Logging in with ssh
Back on [jetstream2/exosphere](https://jetstream2.exosphere.app/) where we launched the instance, if we go to the instance page we can get the username and pass phrase info at the bottom:

![](https://i.imgur.com/TrKffKU.png)


And ssh in with that, e.g.:

```bash
ssh exouser@149.165.154.133
```

And once in, entering sudo mode:

```bash
sudo bash
```
### Connecting to fileshare started above

Following [here](https://docs.jetstream-cloud.org/general/manilaVM/):

```bash
mkdir /opt/reference-dbs
```

Created this file (name is based on what I named the "Access Rule" above), and inside it put the "Access Key" I got above:

```bash
nano /etc/ceph/ceph.client.BIO230010-JPL-PRISM-share.keyring
```

And put in this, but the actual "Acess Key" (name this also the same as the "Access Rule"):
```
[client.BIO230010-JPL-PRISM-share]
    key = ...
```

Ensuring settings:
```bash
chmod 600 /etc/ceph/ceph.client.BIO230010-JPL-PRISM-share.keyring
```

Editing `/etc/fstab` file:

```bash
nano /etc/fstab
```

To be this, but where `$path` is replace with the information int "Path" from the share creation above, and `name=` holds what I named the client just above in the file where I put the key:

```
$path /opt/reference-dbs ceph name=BIO230010-JPL-PRISM-share,x-systemd.device-timeout=30,x-systemd.mount-timeout=30,noatime,_netdev,rw 0 2
```

Mounting:

```bash
mount -a
```
May get an error for some other mount not succeeding, but can check the one we wanted did mount (if we named it above like we're `grep`-ing here):

```bash
df -h | grep '/opt/reference-dbs'
149.165.158.38:6789,149.165.158.22:6789,149.165.158.54:6789,149.165.158.70:6789,149.165.158.86:6789:/volumes/_nogroup/062b96d9-029a-4bd0-820d-60fea3010d06/f6629d57-fcf4-4ea5-9bd7-15b0426b3657  400G     0  400G   0% /opt/reference-dbs
```

### Setting timezone to where 2 of the workshops will be

```bash
timedatectl set-timezone America/Los_Angeles
```

### Modifying system-wide bashrc and skel bashrc files 

#### Modifying /etc/bash.bashrc
This is the bashrc profile that is copied over to new users. Only adding conda info:


```bash
cat >> /etc/bash.bashrc << 'EOF'

# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/opt/miniconda3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/opt/miniconda3/etc/profile.d/conda.sh" ]; then
        . "/opt/miniconda3/etc/profile.d/conda.sh"
    else
        export PATH="/opt/miniconda3/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<

# adding bakta db variable
export BAKTA_DB=/opt/reference-dbs/bakta/db

# adding gtdb-tk db variable
export GTDBTK_DATA_PATH=/opt/reference-dbs/gtdbtk

EOF
```

#### Modifying /etc/skel/.bashrc
This is the system-wide bashrc profile. Here we are changing the prompt and adding stuff to the end. Just overwriting the whole file here cause it's easier to just copy and paste this entire codeblock:

```bash
cat > /etc/skel/.bashrc << 'EOF'

# ~/.bashrc: executed by bash(1) for non-login shells.
# see /usr/share/doc/bash/examples/startup-files (in the package bash-doc)
# for examples

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac

# don't put duplicate lines or lines starting with space in the history.
# See bash(1) for more options
HISTCONTROL=ignoreboth

# append to the history file, don't overwrite it
shopt -s histappend

# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)
HISTSIZE=1000
HISTFILESIZE=2000

# check the window size after each command and, if necessary,
# update the values of LINES and COLUMNS.
shopt -s checkwinsize

# If set, the pattern "**" used in a pathname expansion context will
# match all files and zero or more directories and subdirectories.
#shopt -s globstar

# make less more friendly for non-text input files, see lesspipe(1)
[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"

# set variable identifying the chroot you work in (used in the prompt below)
if [ -z "${debian_chroot:-}" ] && [ -r /etc/debian_chroot ]; then
    debian_chroot=$(cat /etc/debian_chroot)
fi

# set a fancy prompt (non-color, unless we know we "want" color)
case "$TERM" in
    xterm-color|*-256color) color_prompt=yes;;
esac

# uncomment for a colored prompt, if the terminal has the capability; turned
# off by default to not distract the user: the focus in a terminal window
# should be on the output of commands, not on the prompt
#force_color_prompt=yes

if [ -n "$force_color_prompt" ]; then
    if [ -x /usr/bin/tput ] && tput setaf 1 >&/dev/null; then
	# We have color support; assume it's compliant with Ecma-48
	# (ISO/IEC-6429). (Lack of such support is extremely rare, and such
	# a case would tend to support setf rather than setaf.)
	color_prompt=yes
    else
	color_prompt=
    fi
fi

# getting externally accessible IP address
accessible_IP=$(dig +short myip.opendns.com @resolver1.opendns.com)

if [ "$color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;34m\]\u@${accessible_IP}\[\033[00m\]:\[\033[01;35m\]\w\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@${accessible_IP}:\w\$ '
fi
unset color_prompt force_color_prompt

# If this is an xterm set the title to user@host:dir
case "$TERM" in
xterm*|rxvt*)
    PS1="\[\e]0;${debian_chroot:+($debian_chroot)}\u@${accessible_IP}: \w\a\]$PS1"
    ;;
*)
    ;;
esac

# enable color support of ls and also add handy aliases
if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
    alias ls='ls --color=auto'
    #alias dir='dir --color=auto'
    #alias vdir='vdir --color=auto'

    alias grep='grep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias egrep='egrep --color=auto'
fi

# colored GCC warnings and errors
#export GCC_COLORS='error=01;31:warning=01;35:note=01;36:caret=01;32:locus=01:quote=01'

# some more ls aliases
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'

# Add an "alert" alias for long running commands.  Use like so:
#   sleep 10; alert
alias alert='notify-send --urgency=low -i "$([ $? = 0 ] && echo terminal || echo error)" "$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert$//'\'')"'

# Alias definitions.
# You may want to put all your additions into a separate file like
# ~/.bash_aliases, instead of adding them here directly.
# See /usr/share/doc/bash-doc/examples in the bash-doc package.

if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

# enable programmable completion features (you don't need to enable
# this, if it's already enabled in /etc/bash.bashrc and /etc/profile
# sources /etc/bash.bashrc).
if ! shopt -oq posix; then
  if [ -f /usr/share/bash-completion/bash_completion ]; then
    . /usr/share/bash-completion/bash_completion
  elif [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
  fi
fi

# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/opt/miniconda3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/opt/miniconda3/etc/profile.d/conda.sh" ]; then
        . "/opt/miniconda3/etc/profile.d/conda.sh"
    else
        export PATH="/opt/miniconda3/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<

# adding bakta db variable
export BAKTA_DB=/opt/reference-dbs/bakta/db

# adding gtdb-tk db variable
export GTDBTK_DATA_PATH=/opt/reference-dbs/gtdbtk

EOF
```

### Installing miniconda3

```bash
curl -O -L https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```
**During above interactive install steps**
- set install location to `/opt/miniconda3` (user directories not maintained with image creation)
- enter "yes" to initialize at end of install

#### Sourcing to activate new environment

```bash
source ~/.bashrc
```

### Installing mamba and wanted programs/envs

```bash
conda install -y -c conda-forge mamba

mamba install -y -c conda-forge -c bioconda -c defaults \
              jupyterlab=3.6.0 bash_kernel=0.9.0 coreutils=9.1
              
mamba create -y -n qc -c conda-forge -c bioconda -c defaults \
             fastqc=0.11.9 multiqc=1.12 fastp=0.23.2

mamba create -y -n assembly -c conda-forge -c bioconda -c defaults -c astrobiomike \
             spades=3.15.5 bit=1.8.57

mamba create -y -n checkm2 -c conda-forge -c bioconda -c defaults checkm2=1.0.1

mamba create -y -n gtdbtk -c conda-forge -c bioconda -c defaults gtdbtk=2.2.6

mamba create -y -n bakta -c conda-forge -c bioconda -c defaults bakta=1.7

mamba create -y -n gtotree -c conda-forge -c bioconda -c defaults gtotree=1.7.08
```

### Setting up any needed ref dbs

```bash
# checkm2 (this only takes a minute or two)
conda activate checkm2

checkm2 database --download --path /opt/reference-dbs/checkm2


# bakta (took about 30 minutes, so done in a screen)
screen -R bakta-setup

conda activate bakta

bakta_db download --output /opt/reference-dbs/bakta --type full

# gtdb (can take about 3 hours, so done in a screen)
screen -R gtdb-setup

conda activate gtdbtk

download-db.sh /opt/reference-dbs/gtdbtk

## setting up mash db (i think i need a genome to run this on to set it up at first...)
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/230/915/GCF_000230915.1_ASM23091v1/GCF_000230915.1_ASM23091v1_genomic.fna.gz

gunzip GCF_000230915.1_ASM23091v1_genomic.fna.gz

time gtdbtk classify_wf --genome_dir ./ --out_dir gtdb-tax --mash_db /opt/reference-dbs/gtdbtk/gtdb-tk-r207.msh -x fna --cpus 16
```

<!--
### Setting IRKernal to appropriate R
**This isn't retained, need to find another way, boot scripts weren't working for me either...**
```bash
R
```

```R
IRkernel::installspec(user = FALSE)
quit("no")
```
-->

### Creating Jupyter boot script in /opt/
This sets the jupyter notebook password and launches jupyter lab when a new instance is created with this image, it is run when the user instance is booted for the first time. 

We have to put in a hashed passwod, here is an example of how we can create it with an example password: 

```python
python

from notebook.auth import passwd
passwd("pw123", algorithm = "sha1")
# 'sha1:e985a3b764c2:ad258b3ca7c3d7fe86283d87731eaa92e87f5206'
```

The output of whatever the real password you put in there would replace what's in the below codeblock at that sha1 spot, following the `u`.

Got some of that info from [here](https://jupyter-notebook.readthedocs.io/en/stable/public_server.html#preparing-a-hashed-password).

```bash
cat > /opt/jupyter-boot.sh << 'EOF'

#!/bin/bash

rm -rf ~/.jupyter

# making default config files
/opt/miniconda3/bin/jupyter server --generate-config

# setting some things
printf "

c = get_config()
c.NotebookApp.ip = '*'
c.NotebookApp.open_browser = False
c.NotebookApp.password = u'sha1:35cda50d6565:72139b541910ca4b546955d5dc27a1354f8d86d0'
c.NotebookApp.port = 8000

" >> ~/.jupyter/jupyter_server_config.py

# launching jupyterlab
cd ~/
nohup /opt/miniconda3/bin/jupyter lab > ~/.jupyter/log 2>&1 &

EOF
```

### Setting so uneditable markdown cells can't be un-rendered in Jupyter

```bash
mkdir -p ${CONDA_PREFIX}/share/jupyter/lab/settings/

printf '{
  "@jupyterlab/notebook-extension:tracker": {
    "showEditorForReadOnlyMarkdown" : false}
}
' >> ${CONDA_PREFIX}/share/jupyter/lab/settings/overrides.json
```


## Now "burning" this image

Don't do this too fast after making the above changes, i think it actually needs a few minutes or the last file-changes don't propagate to the burned image (weird, I know, but happened several times).

Can exit the `ssh` connection, and on the instance page on exosphere on the right side, select "Actions" then "Image":

![](https://i.imgur.com/P1sGuiA.png)

![](https://i.imgur.com/NrMYzeV.png)

## Creating instance from this image

### Where to find images in exosphere GUI

These seem to be created virtually immediately now (well takes a few minutes still, but don't need to be approved/launched by anyone, which is great). Go to the main allocation page and click on the Images block:

![](https://i.imgur.com/ongvOXw.png)


Then apply filter for this project:

![](https://i.imgur.com/jcT5AIh.png)

![](https://i.imgur.com/m3XroxT.png)


### Create Instance

**Add name, select m3.large (for this one), click to show Advanced Options**

- set to "Assign a public IP address to this instance"
- replace what's in the "Boot Script" block with the below after modifying if needed (including user names and passwords as described below)

#### Modifying cloud-init config 
These changed with JetStream2 (or maybe exosphere) from deploy scripts. Start-up stuff is handled by a config now (https://docs.jetstream-cloud.org/ui/exo/create_instance/#advanced-options).

Info on boot commands here: https://cloudinit.readthedocs.io/en/latest/topics/examples.html#run-commands-on-first-boot

**New JS2 way**

Modifying the cloud-init config yaml, adding in a 'mike' user (as a sudo I can access on any instance) and a 'jpl' user/passwords, and jupyter lab launch script execution on to be run on initial boot-up. With a different user name, be sure to change the `bash /opt/jupyter-boot.sh` line mixed in there to the new user name.

Here's an example of making a 'salted' password, which is how we should put them in the config as done below:

```bash
openssl passwd -1 'pwd123'
# $1$n9TqfAQL$CqERWJdggOBq5cMAGBNCR.
```
That output from a real password would be placed below where appropriate, and adjust the usernames if wanted, then copy and paste into Boot Script window (note that this includes a public ssh key for my comp).



```
#cloud-config
users:
  - default
  - name: mike
    shell: /bin/bash
    groups: sudo, admin, users
    sudo: ['ALL=(ALL) NOPASSWD:ALL']{ssh-authorized-keys}
    lock_passwd: false
    passwd: $1$lS58A.Ee$WIVvQjNvHYojn/iPpuVYj0
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC6P9HXOJooxMDwGdGsrTAfMsf8+m28EoSpUJoecfcb8x2D19IG2DBdnfM0Tbg7ofwWDW77bd+XZbIMBbDo/+4HwdNTSDjURsL/rtjZxQrmz+ecoQtnl+J8gdWUy/EHV/kwg5D25kzKCJQpV4ktrA7I3sI2AClAafF5y6glJTV3e6vhgyqGNVH3olo3nYi8pZHJiKt2hmgihq8NxEnsVLqnCS+I6SDR+icPinttqp0nOZgzVIWza82az8LuQRHfQVWJhYp/rVrvgC+9v06/xrNGqvU8WeTKuvq0hKzEAuRpVNu5FPkAfV8aBGntyc3D8uXvMishG/0Bbh9JsU5BuVstKpXa4RgBwRY6QM7tVT0dKFKrPAtD7XM+LAQ+BTK1MXjqnZgTUTLTb91i8isf+fu+vhwYcoFll3ExUq6VcpXmEtUeST79oc2n1Ko16YbUf+dtYu8WocTCt8B9AvTaHax+QvCTeuw8sJzkcyJZZ5tglurg0y+dymyutgXbt1F4zQk= mdlee4@ARLAL0122021055
  - name: jpl
    shell: /bin/bash
    groups: users
    lock_passwd: false
    passwd: $1$0TXnLUJw$znmh8sMX9oXiApsFHI01p/
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC6P9HXOJooxMDwGdGsrTAfMsf8+m28EoSpUJoecfcb8x2D19IG2DBdnfM0Tbg7ofwWDW77bd+XZbIMBbDo/+4HwdNTSDjURsL/rtjZxQrmz+ecoQtnl+J8gdWUy/EHV/kwg5D25kzKCJQpV4ktrA7I3sI2AClAafF5y6glJTV3e6vhgyqGNVH3olo3nYi8pZHJiKt2hmgihq8NxEnsVLqnCS+I6SDR+icPinttqp0nOZgzVIWza82az8LuQRHfQVWJhYp/rVrvgC+9v06/xrNGqvU8WeTKuvq0hKzEAuRpVNu5FPkAfV8aBGntyc3D8uXvMishG/0Bbh9JsU5BuVstKpXa4RgBwRY6QM7tVT0dKFKrPAtD7XM+LAQ+BTK1MXjqnZgTUTLTb91i8isf+fu+vhwYcoFll3ExUq6VcpXmEtUeST79oc2n1Ko16YbUf+dtYu8WocTCt8B9AvTaHax+QvCTeuw8sJzkcyJZZ5tglurg0y+dymyutgXbt1F4zQk= mdlee4@ARLAL0122021055
ssh_pwauth: true
package_update: true
package_upgrade: {install-os-updates}
packages:
  - python3-virtualenv
  - git{write-files}
runcmd:
  - sudo -u jpl -H sh -c "bash /opt/jupyter-boot.sh"
  - echo on > /proc/sys/kernel/printk_devkmsg || true  # Disable console rate limiting for distros that use kmsg
  - sleep 1  # Ensures that console log output from any previous command completes before the following command begins
  - >-
    echo '{"status":"running", "epoch": '$(date '+%s')'000}' | tee --append /dev/console > /dev/kmsg || true
  - chmod 640 /var/log/cloud-init-output.log
  - {create-cluster-command}
  - |-
    (which virtualenv && virtualenv /opt/ansible-venv) || (which virtualenv-3 && virtualenv-3 /opt/ansible-venv) || python3 -m virtualenv /opt/ansible-venv
    . /opt/ansible-venv/bin/activate
    pip install ansible-core
    ansible-pull --url "{instance-config-mgt-repo-url}" --checkout "{instance-config-mgt-repo-checkout}" --directory /opt/instance-config-mgt -i /opt/instance-config-mgt/ansible/hosts -e "{ansible-extra-vars}" /opt/instance-config-mgt/ansible/playbook.yml
  - ANSIBLE_RETURN_CODE=$?
  - if [ $ANSIBLE_RETURN_CODE -eq 0 ]; then STATUS="complete"; else STATUS="error"; fi
  - sleep 1  # Ensures that console log output from any previous commands complete before the following command begins
  - >-
    echo '{"status":"'$STATUS'", "epoch": '$(date '+%s')'000}' | tee --append /dev/console > /dev/kmsg || true
mount_default_fields: [None, None, "ext4", "user,exec,rw,auto,nofail,x-systemd.makefs,x-systemd.automount", "0", "2"]
mounts:
  - [ /dev/sdb, /media/volume/sdb ]
  - [ /dev/sdc, /media/volume/sdc ]
  - [ /dev/sdd, /media/volume/sdd ]
  - [ /dev/sde, /media/volume/sde ]
  - [ /dev/sdf, /media/volume/sdf ]
  - [ /dev/vdb, /media/volume/vdb ]
  - [ /dev/vdc, /media/volume/vdc ]
  - [ /dev/vdd, /media/volume/vdd ]
  - [ /dev/vde, /media/volume/vde ]
  - [ /dev/vdf, /media/volume/vdf ]
```

Then click "Create".


## The goods

With that all set up as done above, launching an instance with that image will have users mike (with sudo) and jpl (as a user, no sudo). The public IP address can be found on the instance page once it is ready. 

- Jupyter hub is accessed at:
    - \<ip\>:8000/lab
<!--- Rstudio is accessed at: 
    - \<ip>:8787
-->


When thinking of storing notebooks on starting image, look here for inspiration: https://hackmd.io/G6IyzOp6Q0W3pBb7gB2dIw#Deploy-script-to-be-given-to-the-deployment-of-new-instances-of-this-image

  - though, maybe downloading with boot script is better than packing with the image, so i wouldn't need to change the image at all if i just change the notebooks, so likely just put the download commands in boot (and be sure to modify permissions)

