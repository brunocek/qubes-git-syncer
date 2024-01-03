# qubes-git-syncer (Qubes sync git repositories with dom0)

This project is a pointer to codeberg where we collaborate and develop this: https://codeberg.org/brunoschroeder/qubes-git-syncer

As seen in the forum: (https://forum.qubes-os.org/t/sync-git-repositories-with-dom0-qubes-git-syncer/22739)

This solution is intended have git repos easily in sync with dom0.

The initial motivation is to reduce the toil on the development of SaltStack/Jinja states for Qubes.

The solution consists of a bash file (`dom0-syncer.sh` - a lib) with a function. It should be added somewhere in the project. 

On project root, the user should create one (or two) bash script "sourcing the lib" (bash's "importing") and calling the function (described in Suggested Usage).

Similar effort by **solene**:
- (https://dataswamp.org/~solene/2023-06-17-qubes-os-git-bundle.html)
- (https://forum.qubes-os.org/t/keep-a-git-repository-in-sync-between-dom0-and-an-appvm/19420)


## Under the hood

To sync a git repo **from** dom0 to vm:

```
git bundle create $DOM0DIRPATH/$FILENAME --all
sudo qvm-move-to-vm $VM_NAME $DOM0DIRPATH/$FILENAME
```

To sync a git repo **to** dom0:

```
sudo qvm-run --pass $VM_NAME 'cd ${VMPATH} && git bundle create - --all ' > $DOM0DIRPATH/$FILENAME
```

## Suggested Installation

On the VM where the project lives:

1. Have it in somewhere in the project. 
2. Also have on the project root, some bash script to call it (described on the next session - Suggested Usages).
3. git add and git commit all changes

On dom0:
1. Bring the project in home executing the bellow commands (substitute the bash variables (after`$` with actual text of where you have the files):

```
sudo qvm-run --pass $VM_NAME 'cd ${VM_PATH_TO_PROJECT} && git bundle create - --all ' > $DOM0DIRPATH/$FILENAME

cd ${DOM0_PATH_TO_GIT_CLONE_PROJECT}

git clone ${DOM0_PATH_TO_GIT_CLONE_PROJECT}/$FILENAME

```

Voilà, it's installed. From now on, just call your own script that lives on root and you created according to the next session (Suggested Usages). 


## Suggested Usage

On the root of your project, have a bash script to make the sync more convenient, e.g.i for a project on `~/bkp/conf-salt` and `dom0-syncer.sh` at `lib/`:

```
#!/bin/bash
# GNU GPL v3+, Bruno Schroeder
# https://codeberg.org/brunoschroeder/qubes-git-syncer

# convention: `LIB_PATH` (the path where `dom0-syncer.sh` lives, and it is the same for dom0 and VM - e.g. `~/src/project/lib/`)
LIB_PATH="~/bkp/conf-salt/lib/"

# `DOM0DIRPATH` (base path for pack files in dom0)
DOM0DIRPATH="~/bkp/in-out"

PACK_NAME="git-conf-salt.pack"

# `VMPATH` (project path in the VM)
VMPATH="/home/user/bkp/conf-salt"

# body
source ${LIB_PATH}/dom0-syncer.sh

if [ -z $1 ] || [ -z $2 ]; then
	echo "usage: bash sync.sh <direction> <vm_name>"
	echo -e "\t direction\t\t Either F (from dom0 to vm) or T (to dom0)."
	echo -e "\t vm_name\t\t Name of the VM."
	exit
fi
DIRECTION=$1 ; VM=$2

echo "Be sure to have commited all changes and have HEAD in place."
read T
dom0-sync ${DIRECTION} ${VM} ${DOM0DIRPATH} ${PACK_NAME} ${VMPATH}

git status

echo "`git reset --hard && pull` ? (Hit Enter to continue or Ctrl+c not to.)"
read T

git reset --hard

git pull ${DOM0DIRPATH}/${PACK_NAME}


```

One could have a script to sync on each direction. That would make it even more straightforward.

**To use the script above effectively:**

On dom0, typically:

- you run some salt states, and do little changes, debugs and tweaks
- you don't want to do all the work, as it is clunky, but you also prefer not losing the little changes
- when you get to a point where you can go to a VM for more productive work
- git commit all changes and have the branches and HEAD that you wish on the VM

Then run on `dom0`:

```
bash sync.sh F <name-of-the-VM>

``` 

On the VM, on the project path you will:

```
git pull ~/QubesIncoming/dom0/git-conf-salt.pack

```


On the VM you develop your project (say Salt states to be run on dom0):

- work naturally on the project
- git commit changes rationally, interact with other remote gits
- when you have something to be run on dom0, make sure you commit it to HEAD or the branch you wish to run

Then go on `dom0`, same project directory and run:

```
bash sync.sh T <name-of-the-VM>

```

You will be asked to hit Enter twice. Once in the begining, and once before pulling the contents of the `.pack` file.

The script runs a `git status` and offers a `git reset --hard` before the `git pull`.

**ATTENTION** `git reset --hard` with the `git pull` will get rid of any uncommitted changes you see on the `git status`, therefore, it is your responsibility to do something about these changes. **If you do not wish to have your changes in dom0 lost, hit Ctrl+c.** 

I find myself reviewing the changes on dom0, and manually adding them on the VM, committing there again and then running `sync.sh` ready to have it `git reset --hard`.


### Attention

1. Attention, it might be better to have all commits made and HEAD in place.
1. When moving from dom0, it might be better to have all commits made and HEAD in place.

### Context (History)
We are in Qubes 4.1.2. It's 2023-12. First contribution by Bruno Schroeder.

## Code for github

This project is a pointer to codeberg where we collaborate and develop this, and the code can be fetched: https://codeberg.org/brunoschroeder/qubes-git-syncer

But if you're in github and lazy to switch, here goes `dom0syncer.sh` :

```
#!/bin/bash
# GNU GPL v3 by Bruno Schroeder
# https://codeberg.org/brunoschroeder/qubes-git-syncer

dom0-sync(){
    if [ -z $1 ] || [ -z $2 ] || [ -z $3 ] || [ -z $4 ]; then
        echo "usage: dom0-sync <direction> <vm_name> <dom0dirpath> <file_name> [<vm_path>]"
        echo -e "\t direction\t\t Either F (from dom0 to vm) or T (to dom0)."
        echo -e "\t vm_name\t\t Name of the VM."
        echo -e "\t dom0dirpath\t\t Path to directory where git bundle pack file will be created."
        echo -e "\t file_name\t\t Name of the git bundle pack file to be createad."
		echo -e "\t vm_path\t\t (optional) Path on the VM to create the git bundle pack file.\n"
	exit
    fi
    DIRECTION=$1 ; VM=$2 ; DOM0DIRPATH=$3 ; FILENAME=$4 ; VMPATH=$5

    echo -e "dom0-sync\n"
    case "${DIRECTION}" in
    [f/F])
      echo "(From dom0 to vm) (ATTENTION Run on git root)" && echo ""
      echo "git bundle create ${DOM0DIRPATH}/${FILENAME} --all"
      echo "sudo qvm-move-to-vm ${VM} ${DOM0DIRPATH}/${FILENAME}" && echo ""
	  echo "Confirm (hit Enter) or cancel (Ctrl+c):"
      read T
      git bundle create $DOM0DIRPATH/$FILENAME --all
      sudo qvm-move-to-vm $VM $DOM0DIRPATH/$FILENAME
      ;;
    [t/T])
      if [ -z VMPATH ] ; then
          echo "VM path to git root on the VM?"
          read VMPATH
	  fi
      echo "(To dom0)"
      echo "sudo qvm-run --pass ${VM} 'cd ${VMPATH} && git bundle create - --all ' > ${DOM0DIRPATH}/${FILENAME}"
	  echo "Confirm (hit Enter) or cancel (Ctrl+c):"
      read T
      sudo qvm-run --pass $VM "cd ${VMPATH} && git bundle create - --all" > $DOM0DIRPATH/$FILENAME
      ;;
    *)
      echo ""
      ;;
    esac
}


```


