ifdef LIMIT ## Limit ansible hosts to manage
	LIMIT_OPTS=-l $(LIMIT)
else
	LIMIT_OPTS=
endif

ifdef VERBOSE ## Set ansible in verbose mode (bool)
	VERBOSE_OPTS=-vvv
endif

ANSIBLE_OPTS=-i ../inventory.ini $(LIMIT_OPTS) $(VERBOSE_OPTS)

ANSIBLE_PLAYBOOK=ansible-playbook $(ANSIBLE_OPTS)

WHICH=$(shell uname -s | fgrep -q Linux && echo "which" || echo "which -s")

##############################################################################
# Files to watch

../controlplane.yaml:
	$(WHICH) ansible-playbook
	$(WHICH) helm
	$(ANSIBLE_PLAYBOOK) playbooks/gen-conf.yml

../worker.yaml:
	$(WHICH) ansible-playbook
	$(WHICH) helm
	$(ANSIBLE_PLAYBOOK) playbooks/gen-conf.yml

../talosconfig:
	$(WHICH) ansible-playbook
	$(WHICH) helm
	$(ANSIBLE_PLAYBOOK) playbooks/gen-conf.yml

##############################################################################
# Create the cluster

.PHONY: bootstrap
bootstrap: ## Bootstraps the first etcd node
	$(WHICH) ansible-playbook
	$(ANSIBLE_PLAYBOOK) playbooks/bootstrap.yml
	@echo

.PHONY: apply-conf
apply-conf: ## Apply the configuration to the nodes
	$(WHICH) ansible-playbook
	$(ANSIBLE_PLAYBOOK) playbooks/apply-conf.yml
	@echo

.PHONY: gen-conf
gen-conf: ## Creates the configuration files
	$(ANSIBLE_PLAYBOOK) playbooks/gen-conf.yml
	@echo

.PHONY: health-playbook
health-playbook: ## Check cluster's health (using ansible)
	@echo "▷ Cluster Health"
	$(WHICH) ansible-playbook
	test -f ../talosconfig
	$(ANSIBLE_PLAYBOOK) playbooks/health.yml
	@echo

.PHONY: health
health: ## Check cluster's health
	@echo "▷ Cluster Health"
	test -f ../talosconfig
	for s in $$(cat ../inventory.ini| awk '!/\[controlplane/ && !/\[worker/') ; do talosctl --talosconfig ../talosconfig -n $$s -e $$s health ; done
	@echo

.PHONY: cluster-reboot
cluster-reboot: ## Reboot the cluster (one by one)
	@echo "▷ Cluster Reboot"
	$(WHICH) ansible-playbook
	$(ANSIBLE_PLAYBOOK) playbooks/reboot.yml
	@echo

.PHONY: cluster-reset
cluster-reset: ## Reset the cluster (nodes are wiped, kubectl context removed)
	@echo "▷ Cluster Reset"
	$(WHICH) ansible-playbook
	test -f ../talosconfig
	$(ANSIBLE_PLAYBOOK) playbooks/reset.yml
	rm -f ../talosconfig ../worker.yaml ../controlplane.yaml ../*.host.yaml
	@echo

.PHONY: cluster-shutdown
cluster-shutdown: ## Stop the nodes but keep the current configuration
	@echo "▷ Cluster Shutdown"
	$(WHICH) ansible-playbook
	test -f ../talosconfig
	$(ANSIBLE_PLAYBOOK) playbooks/shutdown.yml
	@echo

.PHONY: cluster-create
cluster-create: ## Create a cluster
	@echo "▷ Cluster Create"
	$(WHICH) helm
	$(WHICH) kubectl
	$(WHICH) talosctl
	test -f ../group_vars/all.yml
	test -f ../group_vars/controlplane.yml
	test -f ../group_vars/worker.yml
	@$(MAKE) ../controlplane.yaml
	@$(MAKE) apply-conf
	@$(MAKE) bootstrap
	@$(MAKE) -C ../day-2 add-nodes
	@echo

##############################################################################
# LoadBalancer configuration
.PHONY: cilium-l2
cilium-l2: ## Deploy Cilium L2 CiliumLoadBalancerIPPool and CiliumL2AnnouncementPolicy
	@echo "▷ Cilium Layer 2"
	$(WHICH) ansible-playbook
	$(ANSIBLE_PLAYBOOK) playbooks/cilium-l2.yml
	@echo

##############################################################################
# Rook Ceph targets

.PHONY: rook-ceph-operator
rook-ceph-operator: ## Install Rook operator for Ceph
	@echo "▷ Rook Ceph Operator"
	$(WHICH) kubectl
	kubectl apply -f manifests/rook.common.yaml
	kubectl patch namespace rook-ceph -p '{ "metadata": { "labels": { "pod-security.kubernetes.io/enforce": "privileged" } } }'
	kubectl apply -f manifests/rook.crds.yaml -f manifests/rook.operator.yaml
	kubectl -n rook-ceph wait deployment/rook-ceph-operator --for=condition=Available  --timeout=1h > /dev/null
	@echo

.PHONY: rook-ceph-cluster
rook-ceph-cluster: rook-ceph-operator cilium-l2 ## Deploy a ceph cluster
	@echo "▷ Rook Ceph Cluster"
	kubectl krew list | fgrep -q rook-ceph
	$(WHICH) ansible-playbook
	$(ANSIBLE_PLAYBOOK) playbooks/ceph-cluster.yml
	kubectl rook-ceph health
	kubectl rook-ceph ceph status
	@echo

##############################################################################
# Cert Manager

.PHONY: cert-manager
cert-manager: ## Install cert manager
	@echo "▷ Cert Manager"
	kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
	@echo

##############################################################################
# Observability

## Quickwit

.PHONY: quickwit
quickwit: ## Install Quickwit log and tracing management tool
	@echo "▷ Quickwit" 
	$(WHICH) ansible-playbook
	$(ANSIBLE_PLAYBOOK) playbooks/quickwit.yml
	kubectl -n quickwit wait deployment/quickwit-obs-control-plane --for=condition=Available --timeout=1h > /dev/null
	kubectl -n quickwit wait deployment/quickwit-obs-janitor --for=condition=Available --timeout=1h > /dev/null
	kubectl -n quickwit wait deployment/quickwit-obs-metastore --for=condition=Available --timeout=1h > /dev/null
	@echo

##############################################################################
# Metal-as-a-Service targets

## Tinkerbell

/tmp/tink-charts:
	$(WHICH) git
	git clone https://github.com/tinkerbell/charts /tmp/tink-charts

.PHONY: tinkerbell
tinkerbell: cilium-l2 /tmp/tink-charts ## Install tinkerbell metal-as-a-service
	@echo "▷ Tinkerbell" 
	$(WHICH) ansible-playbook
	$(ANSIBLE_PLAYBOOK) playbooks/tinkerbell.yml
	@echo

##############################################################################
# SDN

.PHONY: tailscale
tailscale:
	@echo "▷ Tailscale" 
	$(ANSIBLE_PLAYBOOK) playbooks/tailscale.yml
	@echo

##############################################################################
# CP as a Service

.PHONY: local-provisioner
local-provisioner:
	@echo "▷ Local Provisioner"
	cd kustomize/local-provisioning && kustomize build | kubectl apply -f -
	@echo

.PHONY: kamaji-etcd
kamaji-etcd: local-provisioner ## Install etcd datastore for Kamaji
	@echo "▷ NS"
	kubectl get namespace kamaji-system || kubectl create namespace kamaji-system
	@echo

	@echo "▷ ETCD Datastore"
	helm repo add clastix https://clastix.github.io/charts
	helm repo update
	helm install kamaji-etcd clastix/kamaji-etcd -n kamaji-system --set datastore.enabled=true --set datastore.name=default --set replicas=1
	@echo

kubectl apply -f manifests/kamaji.yaml

##############################################################################
# Help

.PHONY: help
help: ## This help message
	@echo Global variables
	@echo ================
	@awk \
		'$$1 ~ /^ifdef/ && $$3 ~ /##/ {gsub(/##/,"\t") ; gsub("ifdef ","") ; print $$0}' \
		Makefile \
	| expand -t35 \
	| sort
	@awk \
		'$$1 ~ /^ifndef/ && $$3 ~ /##/ {gsub(/##/,"\t") ; gsub("ifndef ","") ; print $$0}' \
		Makefile \
	| expand -t35 \
	| sort
	@echo

	@echo Targets
	@echo =======
	@awk -F: \
		'$$1 ~/^[a-z\$$\.\/_]/ && $$2~/##/ {gsub(/: .*?\s*##/, "\t");print $$0}' \
		Makefile \
	| expand -t35 \
	| sort
