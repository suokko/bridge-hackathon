# Project / Org
GCP_PROJECT ?= globalbridge-app
GKE_ZONE ?= europe-west4-b

# Service / App
GKE_CLUSTER_NAME ?= prod-cluster
K8S_NS ?= prod-${COMPONENT_NAME}

# Docker Config
DOCKER_REPO ?= gcr.io/${GCP_PROJECT}
VERSION ?= $(shell git describe --always --dirty)
DOCKER_TAG = ${DOCKER_REPO}/${COMPONENT_NAME}:${VERSION}

EXTERNAL_ADDRESS ?= ${COMPONENT_NAME}.prod.globalbridge.app

# Default target for a component
release: push

# Generate .dockerignore from .gitignore. Git and Docker uses different
# different wildcard syntax. Syntaxes is similar enough to allow conversion
# using regular expression replace. Docker ignore improves build performance
# with a smaller build context.
.dockerignore: .gitignore
	sed 's#^[^/]#**/\0#' < $< > $@

# Build a docker image.
# submodule-update makes sure all submodules have correct source code for the
# build.
# .dockerignore makes sure build context includes only required source files.
build: submodule-update .dockerignore
	docker build -t ${DOCKER_TAG} .

# Publish docker image to the registery.
# build dependency makes sure local image is available.
push: build
	docker push ${DOCKER_TAG}

# Deploy image to selected cluster.
# set_gcp_context makes sure kubectl and helm send request to correct cluster.
# ensure_ns makes sure namespace exists for deployment.
# push makes sure image has been build and published.
deploy: set_gcp_context ensure_ns push
	helm upgrade --install ${COMPONENT_NAME} ./chart \
		--set image="${DOCKER_TAG}" \
		--set externalHostname="${EXTERNAL_ADDRESS}" \
		--namespace ${K8S_NS} \
		--history-max=10

# Remove component from the cluster.
# IMPORTANT: This disables service from the cluster which can be production
# cluster.
uninstall: set_gcp_context
	@echo Warning: Are you sure you want to delete this Production service ${COMPONENT_NAME} [N/y]
	read answer; [ "x$$answer" = "xy" ] && helm del ${COMPONENT_NAME} --namespace ${K8S_NS}

set_gcp_context:
	gcloud container clusters get-credentials ${GKE_CLUSTER_NAME} --zone ${GKE_ZONE} --project ${GCP_PROJECT}

ensure_ns: set_gcp_context
	kubectl create ns ${K8S_NS} || :

