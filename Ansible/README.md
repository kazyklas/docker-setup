# Setup machine

This playbook will install all packages that are necesary for a survival of mine.
It will install k3s on top of it for other deployments done by argocd.


```bash
sudo pacman -S --needed ansible paru git
git clone <your-repo>
cd Ansible
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass
```
```
