# qubes-git-syncer (Sync git repositories through Qubes doms (VMs))

This project is a pointer to codeberg where we collaborate and develop this: https://codeberg.org/brunoschroeder/qubes-git-syncer

### Objectives

1. To be able to sync git (`git clone`, `git pull`, `git push`) repositories through different doms (VMs), so that users can have an infra-tools vault VM with git bares to serve as remote.
1. Having SaltStack states that create infra-tools vault and easily set's up desired git bares. It would be good to do it having a VM to interact with online git communities (codeberg, gitlab, github, etc..) and having infra-tools vault clone from there.
1. Integrate infra-tools to Qubes main development branch.

### Context (History)
We are in Qubes 4.1.2. It's 2023-12. First contribution by Bruno Schroeder.

### Current Status

This project is currently a dom0 bash script to generate git pack files and sync (move) from dom0 to a named VM or to dom0.

There are:

- [Rudd-O's git-remote-qubes](https://github.com/Rudd-O/git-remote-qubes): unable to install due to [issue](https://github.com/Rudd-O/git-remote-qubes/issues/5) on `make rpm`.
- [Woju's qubes-app-split-git](https://github.com/woju/qubes-app-split-git): under development, lacking capability to git push, pull, clone.

## Summary

To sync a git repo from dom0 to vm:

```
git bundle create $DOM0DIRPATH/$FILENAME --all
sudo qvm-move-to-vm $VM_NAME $DOM0DIRPATH/$FILENAME
```

To sync a git repo to dom0:

```
sudo qvm-run --pass $VM_NAME 'cd ${VMPATH} && git bundle create - --all ' > $DOM0DIRPATH/$FILENAME
```

## Suggested Usage

At the moment, I like creating an sh `dom0-sync.sh` on each project I wish to sync. E.g.:

```
#!/bin/bash
# GNU GPL v3 by Bruno Schroeder, 2023-12
# https://codeberg.org/brunoschroeder/qubes-git-syncer

FILENAME="dotfiles.pack"
DOM0DIRPATH="~/bkp/in-out"

source ~/src/lib/dom0-syncer.sh

if [ -z $1 ] || [ -z $2 ]; then
	echo "usage: dom0-sync <direction> <vm_name>" 
	echo "\t direction\t\t Either F (from dom0 to vm) or T (to dom0)."
	echo "\t vm_name\t\t Name of the VM."
    echo "Dom0 Path: ${DOM0DIRPATH}/${FILENAME}"
fi
DIRECTION=$1
VM=$2

dom0-sync $DIRECTION $VM $DOM0DIRPATH/$FILENAME

```

As seen above, I use some conventions on my dom0:

- `~/bkp/in-out` is where all git bundle pack files will be stored.
- `~/src/lib` is where all sh scripts will be stored.
- Naming the pack files with the same name of the git repository.

## Installation

First time you sync a project to dom0: 

1. On the VM, include the file `dom0-syncer.sh` somewhere in the project.
1. On dom0: `sudo qvm-run --pass $VMNAME 'cd ${VMPATH} && git bundle create - --all ' > $DOM0DIRPATH/$FILENAME`
1. `git clone $DOM0DIRPATH/$FILENAME`
1. Now move `dom0-syncer.sh` somewhere else.
1. You can create sh scripts as the one on section "Suggested Usage" (above) in each of your projects.

### Attention

1. Attention, it might be better to have all commits made and HEAD in place.
1. When moving to dom0, if the project is to be used in dom0, the `git clone` must be executed.
1. When using dom0 as a bridge between two VMs, after running `dom0-sync` to dom0, one can run `sudo qvm-move-to-vm $VM_NAME $DOM0DIRPATH $FILENAME`
1. When moving from dom0, it might be better to have all commits made and HEAD in place.

## Code for github

This project is a pointer to codeberg where we collaborate and develop this, and the code can be fetched: https://codeberg.org/brunoschroeder/qubes-git-syncer

But if you're in github and lazy to switch, here goes `dom0syncer.sh` :

```

#!/bin/bash
# GNU GPL v3 by Bruno Schroeder
# https://codeberg.org/brunoschroeder/qubes-git-syncer

dom0-sync(){
    if [ -z $1 ] || [ -z $2 ] || [ -z $3 ] || [ -z $4 ]; then
        echo "usage: dom0-sync <direction> <vm_name> <dom0dirpath> <file_name>"
        echo "\t direction\t\t Either F (from dom0 to vm) or T (to dom0)."
        echo "\t vm_name\t\t Name of the VM."
        echo "\t dom0dirpath\t\t Path to directory where git bundle pack file will be created."
        echo "\t file_name\t\t Name of the git bundle pack file to be createad."
    fi
    DIRECTION=$1 ; VM=$2 ; DOM0DIRPATH=$3 ; FILENAME=$4

    case "${DIRECTION}" in
    [f/F])
      echo "(From dom0 to vm) Please confirm: "
      echo "mv ${DOM0DIRPATH}/${FILENAME} ${DOM0DIRPATH}/z-${FILENAME}"
      echo "git bundle create ${DOM0DIRPATH}/${FILENAME} --all"
      echo "sudo qvm-move-to-vm ${VM} ${DOM0DIRPATH}/${FILENAME}"
      read T
      mv $DOM0DIRPATH/$FILENAME $DOM0DIRPATH/z-$FILENAME
      git bundle create $DOM0DIRPATH/$FILENAME --all
      sudo qvm-move-to-vm $VM $DOM0DIRPATH/$FILENAME
      ;;
    [t/T])
      echo "VM path where git pack file will be created?"
      read VMPATH
      echo "(To dom0) Please confirm: "
      echo "mv ${DOM0DIRPATH}/${FILENAME} ${DOM0DIRPATH}/z-${FILENAME}"
      echo "sudo qvm-run --pass ${VM} 'cd ${VMPATH} && git bundle create - --all ' > ${DOM0DIRPATH}/${FILENAME}"
      echo "Attention, it might be better to have all commits made and HEAD in place."
      read T
      mv $DOM0DIRPATH/$FILENAME $DOM0DIRPATH/z-$FILENAME
      sudo qvm-run --pass $VM 'cd ${VMPATH} && git bundle create - --all' > $DOM0DIRPATH/$FILENAME
      ;;
    *)
      echo ""
      ;;
    esac
}


```


