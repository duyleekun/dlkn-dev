.PHONY: help k3s-help k3s-create-cluster k3s-install k3s-uninstall k3s-shell k3s-check-kubeconfig k3s-check _validate-cluster _validate-kubeconfig-file _validate-inventory

# Get the directory where this Makefile is located
MAKEFILE_DIR := $(dir $(abspath $(lastword $(MAKEFILE_LIST))))

# Cluster name (required, no default - must be specified for most targets)
CLUSTER ?=

# Inventory file location (will be empty if CLUSTER not set, validated in targets)
ifdef CLUSTER
  INVENTORY := inventory/cluster-$(CLUSTER).ini
else
  INVENTORY :=
endif

# Group vars file (default group vars - shared across all clusters)
GROUP_VARS := group_vars/k3s_cluster.yml

# Kubeconfig location (required, no default - must be specified)
KUBECONFIG_FILE ?=

ANSIBLE_PLAYBOOK := ansible-playbook

# Shell function to convert KUBECONFIG_FILE to absolute path (relative to PWD)
# This is evaluated at runtime, not parse time
GET_KUBECONFIG_ABS = if [ "$$(echo $(KUBECONFIG_FILE) | cut -c1)" = "/" ]; then \
	echo "$(KUBECONFIG_FILE)"; \
else \
	echo "$$(pwd)/$(KUBECONFIG_FILE)"; \
fi

# Helper target to validate CLUSTER is provided
_validate-cluster:
	@if [ -z "$(CLUSTER)" ]; then \
		echo "✗ Error: CLUSTER parameter is required"; \
		echo "  Usage: make <target> CLUSTER=<cluster-name>"; \
		exit 1; \
	fi

# Helper target to validate KUBECONFIG_FILE is provided
_validate-kubeconfig-file:
	@if [ -z "$(KUBECONFIG_FILE)" ]; then \
		echo "✗ Error: KUBECONFIG_FILE parameter is required"; \
		echo "  Usage: make <target> CLUSTER=$(CLUSTER) KUBECONFIG_FILE=<path>"; \
		exit 1; \
	fi

# Helper target to validate inventory file exists
_validate-inventory: _validate-cluster
	@if [ ! -f "$(MAKEFILE_DIR)$(INVENTORY)" ]; then \
		echo "✗ Error: Inventory file not found: $(INVENTORY)"; \
		echo "  Run 'make k3s-create-cluster CLUSTER=$(CLUSTER)' first"; \
		exit 1; \
	fi

# Default target
help: k3s-help

k3s-help:
	@echo "K3s Cluster Management"
	@echo ""
	@echo "Usage: make <target> CLUSTER=<cluster-name> KUBECONFIG_FILE=<path>"
	@echo ""
	@echo "Environment variables:"
	@echo "  CLUSTER          - Cluster name (required for most targets)"
	@echo "  KUBECONFIG_FILE  - Kubeconfig file path, relative to current directory (required for most targets)"
	@echo ""
	@echo "Available targets:"
	@echo "  make k3s-create-cluster CLUSTER=<name>  - Create a new cluster configuration"
	@echo "  make k3s-install CLUSTER=<name>         - Install K3s cluster"
	@echo "  make k3s-uninstall CLUSTER=<name>       - Uninstall K3s cluster"
	@echo "  make k3s-shell CLUSTER=<name>           - Enter a shell with KUBECONFIG set"
	@echo "  make k3s-check CLUSTER=<name>           - Check cluster status and show info"
	@echo ""
	@echo "Examples:"
	@echo "  make k3s-create-cluster CLUSTER=prod"
	@echo "  make k3s-install CLUSTER=prod KUBECONFIG_FILE=kubeconfig-prod"
	@echo "  make k3s-shell CLUSTER=prod KUBECONFIG_FILE=kubeconfig-prod"

# Create a new cluster configuration
k3s-create-cluster: _validate-cluster
	@if [ -f "$(MAKEFILE_DIR)$(INVENTORY)" ]; then \
		echo "✗ Error: Cluster '$(CLUSTER)' already exists at $(INVENTORY)"; \
		exit 1; \
	fi
	@echo "Creating new cluster configuration: $(CLUSTER)"
	@printf '[k3s_master]\nk3s-%s-master-01 ansible_host=CHANGE_ME ansible_user=ubuntu\n\n[k3s_agents]\n# k3s-%s-agent-01 ansible_host=CHANGE_ME ansible_user=ubuntu\n# Add more agent nodes as needed\n\n[k3s_cluster:children]\nk3s_master\nk3s_agents\n' $(CLUSTER) $(CLUSTER) > $(MAKEFILE_DIR)$(INVENTORY)
	@echo "✓ Inventory file created at $(INVENTORY)"
	@CLUSTER_VARS=$(MAKEFILE_DIR)group_vars/cluster_$(CLUSTER).yml; \
	if [ ! -f "$$CLUSTER_VARS" ]; then \
		printf '# Cluster-specific overrides for %s\n# This file overrides values from group_vars/k3s_cluster.yml\n# Uncomment and modify values as needed for this cluster\n\n---\n# K3s cluster configuration\n# k3s_version: v1.28.3+k3s1\n# k3s_token: "YourSecureTokenHere"\n\n# Registry configuration\n# registry_enabled: true\n# registry_port: 5000\n\n# K3s server config directory\n# k3s_config_dir: /etc/rancher/k3s\n' $(CLUSTER) > $$CLUSTER_VARS; \
		echo "✓ Group vars override file created at $$CLUSTER_VARS"; \
	fi
	@echo ""
	@echo "Next steps:"
	@echo "  1. Edit $(INVENTORY) and update the host addresses"
	@echo "  2. Edit group_vars/cluster_$(CLUSTER).yml to override default values if needed"
	@echo "  3. Run 'make k3s-install CLUSTER=$(CLUSTER)' to install the cluster"

# Install K3s cluster
k3s-install: _validate-cluster _validate-inventory _validate-kubeconfig-file
	@KUBECONFIG_ABS=$$($(GET_KUBECONFIG_ABS)); \
	echo "Installing K3s cluster: $(CLUSTER)"; \
	echo "Using inventory: $(INVENTORY)"; \
	echo "Kubeconfig will be saved to: $$KUBECONFIG_ABS"; \
	cd $(MAKEFILE_DIR) && $(ANSIBLE_PLAYBOOK) -i $(INVENTORY) -e cluster_name=$(CLUSTER) -e kubeconfig_file="$$KUBECONFIG_ABS" playbooks/k3s-install.yml
	@KUBECONFIG_ABS=$$($(GET_KUBECONFIG_ABS)); \
	echo ""; \
	echo "✓ K3s cluster '$(CLUSTER)' installed successfully!"; \
	if [ -f "$$KUBECONFIG_ABS" ]; then \
		echo "✓ Kubeconfig saved to: $$KUBECONFIG_ABS"; \
		echo ""; \
		echo "Next steps:"; \
		echo "  - Run 'make k3s-shell CLUSTER=$(CLUSTER) KUBECONFIG_FILE=$(KUBECONFIG_FILE)' to enter a shell with KUBECONFIG set"; \
		echo "  - Or manually: export KUBECONFIG=$$KUBECONFIG_ABS"; \
	fi

# Uninstall K3s cluster
k3s-uninstall: _validate-cluster _validate-inventory
	@echo "⚠ WARNING: This will uninstall K3s cluster '$(CLUSTER)' from all nodes!"
	@read -p "Are you sure you want to continue? [y/N] " confirm && [ "$$confirm" = "y" ] || exit 1
	cd $(MAKEFILE_DIR) && $(ANSIBLE_PLAYBOOK) -i $(INVENTORY) -e cluster_name=$(CLUSTER) playbooks/k3s-uninstall.yml
	@echo ""
	@echo "✓ K3s cluster '$(CLUSTER)' uninstalled successfully"

# Check if kubeconfig exists
k3s-check-kubeconfig: _validate-cluster _validate-kubeconfig-file
	@KUBECONFIG_ABS=$$($(GET_KUBECONFIG_ABS)); \
	if [ ! -f "$$KUBECONFIG_ABS" ]; then \
		echo "✗ Error: kubeconfig file not found at $$KUBECONFIG_ABS"; \
		echo "  Run 'make k3s-install CLUSTER=$(CLUSTER) KUBECONFIG_FILE=$(KUBECONFIG_FILE)' first to generate the kubeconfig"; \
		exit 1; \
	fi

# Enter shell with KUBECONFIG set
k3s-shell: k3s-check-kubeconfig
	@KUBECONFIG_ABS=$$($(GET_KUBECONFIG_ABS)); \
	echo "Entering shell with KUBECONFIG=$$KUBECONFIG_ABS"; \
	echo "Cluster: $(CLUSTER)"; \
	echo "Type 'exit' to leave the shell"; \
	echo ""; \
	PS1="[k3s:$(CLUSTER)] $$PS1" KUBECONFIG=$$KUBECONFIG_ABS $(SHELL)

# Check cluster status
k3s-check: k3s-check-kubeconfig
	@KUBECONFIG_ABS=$$($(GET_KUBECONFIG_ABS)); \
	echo "Cluster: $(CLUSTER)"; \
	echo "Cluster Information:"; \
	echo "===================="; \
	KUBECONFIG=$$KUBECONFIG_ABS kubectl cluster-info 2>/dev/null || echo "✗ Failed to connect to cluster"; \
	echo ""; \
	echo "Nodes:"; \
	echo "======"; \
	KUBECONFIG=$$KUBECONFIG_ABS kubectl get nodes -o wide 2>/dev/null || echo "✗ Failed to get nodes"; \
	echo ""; \
	echo "Namespaces:"; \
	echo "==========="; \
	KUBECONFIG=$$KUBECONFIG_ABS kubectl get namespaces 2>/dev/null || echo "✗ Failed to get namespaces"

