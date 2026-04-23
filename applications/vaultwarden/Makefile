manifests_dir := $(shell pwd)/manifests
chart_archives := $(wildcard $(manifests_dir)/*.tgz)

ARGS = $(filter-out $@,$(MAKECMDGOALS))
%:
	@:

SHELL := /bin/bash
.SHELLFLAGS = -x +u

# Define the base path to your Helm charts directory
HELM_CHARTS_DIR = ./charts

# Define the path to your target KOTS directory
KOTS_DIR = ./manifests

# Replicated app slug (override with REPLICATED_APP env var)
APP_SLUG ?= $(REPLICATED_APP)

# Release channel (default: Unstable)
CHANNEL ?= Unstable

# Define the function to extract chartVersion from KOTS HelmChart CR
define get_kots_chart_version
	grep 'chartVersion:' $(1) | sed 's/.*chartVersion: //'
endef

# Function to get chart version from Chart.yaml
define get_helm_chart_version
	helm show chart $(1) | grep '^version:' | cut -d ' ' -f 2
endef

# Target to update chart dependencies
.PHONY: update-chart-dependencies
update-chart-dependencies:
	@for chart in $(HELM_CHARTS_DIR)/*; do \
		if [ -f "$$chart/Chart.yaml" ]; then \
			echo "Updating dependencies for $$chart"; \
			helm dependency update --skip-refresh $$chart; \
		fi \
	done

# Target to package charts and update versions in KOTS manifests
.PHONY: package-and-update
package-and-update: update-chart-dependencies
	@for chart in $(HELM_CHARTS_DIR)/*; do \
		if [ -f "$$chart/Chart.yaml" ]; then \
			echo "Packaging $$chart"; \
			helm package $$chart -d $(KOTS_DIR); \
			version=$$(eval $(call get_helm_chart_version,$$chart)); \
			chart_name=$$(basename $$chart); \
			echo "Updating chartVersion to $$version in $(KOTS_DIR)/$$chart_name-chart.yaml"; \
			sed -i.bak "s|chartVersion: [0-9a-zA-Z._-]*|chartVersion: $$version|g" $(KOTS_DIR)/$$chart_name-chart.yaml && rm -f $(KOTS_DIR)/$$chart_name-chart.yaml.bak; \
		fi \
	done

# Lint all charts
.PHONY: lint
lint:
	@for chart in $(HELM_CHARTS_DIR)/*; do \
		if [ -f "$$chart/Chart.yaml" ]; then \
			echo "Linting $$chart"; \
			helm lint $$chart; \
		fi \
	done

# Template all charts (dry-run validation)
.PHONY: template
template:
	@for chart in $(HELM_CHARTS_DIR)/*; do \
		if [ -f "$$chart/Chart.yaml" ]; then \
			echo "Templating $$chart"; \
			helm template test-release $$chart --dry-run; \
		fi \
	done

# Clean chart archives from manifests dir and dependency archives
.PHONY: clean
clean:
	rm -f $(KOTS_DIR)/*.tgz $(HELM_CHARTS_DIR)/*/charts/*.tgz

# Create a Replicated release and promote to channel
.PHONY: release
release: clean package-and-update
	replicated release create --yaml-dir $(KOTS_DIR) --promote $(CHANNEL) --auto -y

# Validate the full config contract (KOTS Config items vs HelmChart CR references)
.PHONY: validate
validate: lint
	@echo "Validating KOTS config contract..."
	@ERRORS=0; \
	CONFIG_ITEMS=$$(yq -r '.spec.groups[].items[].name' $(KOTS_DIR)/kots-config.yaml 2>/dev/null | sort -u); \
	HELMCHART_REFS=$$( \
		grep -oE 'ConfigOption "([^"]+)"' $(KOTS_DIR)/vaultwarden-chart.yaml 2>/dev/null | sed 's/ConfigOption "//;s/"//'; \
		grep -oE 'ConfigOptionEquals "([^"]+)"' $(KOTS_DIR)/vaultwarden-chart.yaml 2>/dev/null | sed 's/ConfigOptionEquals "//;s/"//' \
	); \
	HELMCHART_REFS=$$(echo "$$HELMCHART_REFS" | sort -u); \
	echo "=== KOTS Config items ==="; \
	echo "$$CONFIG_ITEMS"; \
	echo ""; \
	echo "=== ConfigOption references in HelmChart CR ==="; \
	echo "$$HELMCHART_REFS"; \
	echo ""; \
	for ref in $$HELMCHART_REFS; do \
		if ! echo "$$CONFIG_ITEMS" | grep -qx "$$ref"; then \
			echo "ERROR: HelmChart references ConfigOption '$$ref' but no matching Config item found"; \
			ERRORS=$$((ERRORS + 1)); \
		fi \
	done; \
	for item in $$CONFIG_ITEMS; do \
		if ! echo "$$HELMCHART_REFS" | grep -qx "$$item"; then \
			echo "WARNING: Config item '$$item' is not referenced by any HelmChart CR"; \
		fi \
	done; \
	if [ $$ERRORS -gt 0 ]; then \
		echo "FAILED: $$ERRORS error(s) found"; \
		exit 1; \
	else \
		echo "Config contract validation passed!"; \
	fi

.PHONY: help
help:
	@echo "Available targets:"
	@echo "  make release                - Clean, package charts, create & promote Replicated release"
	@echo "  make package-and-update     - Package Helm charts and sync versions to KOTS manifests"
	@echo "  make update-chart-dependencies - Update Helm chart dependencies"
	@echo "  make lint                   - Lint all Helm charts"
	@echo "  make template               - Template all Helm charts (dry-run)"
	@echo "  make validate               - Validate KOTS config contract"
	@echo "  make clean                  - Remove chart archives"
	@echo ""
	@echo "Environment variables:"
	@echo "  REPLICATED_APP  - Replicated app slug (required for release)"
	@echo "  CHANNEL         - Release channel (default: Unstable)"
