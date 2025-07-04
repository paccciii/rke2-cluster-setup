# Bonus Tips and Productivity Hacks

This document contains optional but useful customizations for working with your RKE2 cluster efficiently.

---

## 🖥️ Custom Shell Aliases

Add the following aliases to your `~/.bashrc` or `~/.zshrc` for faster `kubectl` usage:

```bash
alias k=kubectl
alias kgp='kubectl get pods -A'
alias kgs='kubectl get services -A'
alias kgn='kubectl get nodes'
alias kaf='kubectl apply -f'
alias kdf='kubectl describe -f'


After adding them, run:

```bash
source ~/.bashrc
