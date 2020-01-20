# VAGRANT-KUBERNETES
Deploy simple Kubernetes cluster using single Vagrantfile.
Used https://github.com/ecomm-integration-ballerina/kubernetes-cluster as starting point, changed Ubuntu to Centos and made some slight modifications.

# Points of note
Tabs present in Vagrantfile are forwarded to files created during provision (kubernetes.repo and k8s-head.conf). Yum throws a parsing error when it detects tabs in kubernetes.repo file. Added a line 'sed -i 's/\t//g' /etc/yum.repos.d/kubernetes.repo' to remove tabs.
