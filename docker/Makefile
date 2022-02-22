PUPPERWARE_ANALYTICS_STREAM ?= dev
NAMESPACE ?= abhay900
git_describe = $(shell git describe)
vcs_ref := $(shell git rev-parse HEAD)
build_date := $(shell date -u +%FT%T)
hadolint_available := $(shell hadolint --help > /dev/null 2>&1; echo $$?)
hadolint_command := hadolint --ignore DL3008 --ignore DL3028
hadolint_container := ghcr.io/hadolint/hadolint:latest
export BUNDLE_PATH = $(PWD)/.bundle/gems
export BUNDLE_BIN = $(PWD)/.bundle/bin
export GEMFILE = $(PWD)/Gemfile
export DOCKER_BUILDKIT ?= 1
export PUPPETSERVER_IMAGE ?= puppet/puppetserver:edge

ifeq ($(IS_RELEASE),true)
	VERSION ?= $(shell echo $(git_describe) | sed 's/-.*//')
	# to work around failures that occur between when the repo is tagged and when the package
	# is actually shipped, see if this version exists in dujour
	PUBLISHED_VERSION ?= $(shell curl --silent 'https://updates.puppetlabs.com/?product=puppetdb&version=$(VERSION)' | jq '."version"' | tr -d '"')
	# For our containers built from packages, we want those to be built once then never changed
	# so check to see if that container already exists on dockerhub
	CONTAINER_EXISTS = $(shell DOCKER_CLI_EXPERIMENTAL=enabled docker manifest inspect $(NAMESPACE)/puppetdb:$(VERSION) > /dev/null 2>&1; echo $$?)
ifeq ($(CONTAINER_EXISTS),0)
	SKIP_BUILD ?= true
else ifneq ($(VERSION),$(PUBLISHED_VERSION))
	SKIP_BUILD ?= true
endif

	BUILD_TYPE ?= release
	LATEST_VERSION ?= latest
else
	BUILD_TYPE ?= edge
	VERSION ?= edge
	IS_LATEST := false
endif

prep:
	@git fetch --unshallow 2> /dev/null ||:
	@git fetch origin 'refs/tags/*:refs/tags/*'
ifeq ($(SKIP_BUILD),true)
	@echo "SKIP_BUILD is true, exiting with 1"
	@exit 1
endif

lint:
ifeq ($(hadolint_available),0)
	@$(hadolint_command) puppetdb/Dockerfile
else
	@docker pull $(hadolint_container)
	@docker run --rm -v $(PWD)/puppetdb/Dockerfile:/Dockerfile \
		-i $(hadolint_container) $(hadolint_command) Dockerfile
endif


push-image: prep
	docker buildx build --platform linux/arm64,linux/amd64 -t $(NAMESPACE)/puppetdb:$(VERSION) -f puppetdb/Dockerfile --push $(PWD)/..
ifeq ($(IS_LATEST),true)
	docker buildx build --platform linux/arm64,linux/amd64 -t $(NAMESPACE)/puppetdb:$(LATEST_VERSION) -f puppetdb/Dockerfile --push $(PWD)/..
endif


publish: push-image push-readme

.PHONY: prep lint build test publish push-image push-readme
