include ./Makefile.Common

BASE_SPACE:=$(shell pwd)
BUILD_SPACE:=$(BASE_SPACE)/build
BUCKET_NAME:=lambda-artifacts-$(shell dd if=/dev/random bs=8 count=1 2>/dev/null | od -An -tx1 | tr -d ' \t\n')
LAYER_NAME:=otel-collector

VERSION=$(shell cat VERSION)
GIT_SHA=$(shell git rev-parse HEAD)
GOARCH ?= amd64
GOBUILD=GO111MODULE=on CGO_ENABLED=0 installsuffix=cgo go build -trimpath
BUILD_INFO_IMPORT_PATH=main

LDFLAGS=-ldflags "-s -w -X $(BUILD_INFO_IMPORT_PATH).GitHash=$(GIT_SHA) -X $(BUILD_INFO_IMPORT_PATH).Version=$(VERSION) \
-X github.com/open-telemetry/opentelemetry-collector-contrib/exporter/awsxrayexporter.collectorDistribution=opentelemetry-collector-lambda"

TOOLS_MOD_DIR := ./internal/tools
ALL_GO_MOD_DIRS := $(shell find . -type f -name 'go.mod' -exec dirname {} \; | sort)
OTEL_GO_MOD_DIRS := $(filter-out $(TOOLS_MOD_DIR), $(ALL_GO_MOD_DIRS))
GOMODULES = $(OTEL_GO_MOD_DIRS) $(PWD)

# Define a delegation target for each module
.PHONY: $(GOMODULES)
$(GOMODULES):
	@echo "Running target '$(TARGET)' in module '$@'"
	$(MAKE) -C $@ $(TARGET)

# Triggers each module's delegation target
.PHONY: for-all-target
for-all-target: $(GOMODULES)

clean:
	rm -rf build/

build: clean
	@echo Building otel collector extension
	mkdir -p $(BUILD_SPACE)/extensions
	GOOS=linux $(GOBUILD) $(LDFLAGS) -o $(BUILD_SPACE)/extensions .

package: build
	@echo Package zip file for collector extension layer
	mkdir -p $(BUILD_SPACE)/collector-config
	cp config* $(BUILD_SPACE)/collector-config
	cd $(BUILD_SPACE) && zip -r collector-extension.zip collector-config extensions

publish-layer: package
	@echo Publishing collector extension layer...
	aws s3 mb s3://$(BUCKET_NAME)
	aws s3 cp $(BUILD_SPACE)/collector-extension.zip s3://$(BUCKET_NAME)
	aws lambda publish-layer-version --layer-name $(LAYER_NAME) --content S3Bucket=$(BUCKET_NAME),S3Key=collector-extension.zip --compatible-runtimes nodejs12.x nodejs14.x nodejs16.x java11 python3.8 python3.9 --query 'LayerVersionArn' --output text
	@echo Clearing cached files...
	aws s3 rm s3://$(BUCKET_NAME)/collector-extension.zip
	aws s3 rb s3://$(BUCKET_NAME)
	@echo OpenTelemetry Collector layer published.

.PHONY: gotidy
gotidy:
	@$(MAKE) for-all-target TARGET="tidy"

.PHONY: gotest
gotest:
	@$(MAKE) for-all-target TARGET="test"

.PHONY: goporto
goporto: install-tools
	porto -w --include-internal ./

.PHONY: golint
golint: install-tools
	@$(MAKE) for-all-target TARGET="lint"

.PHONY: goimpi
goimpi: install-tools
	@$(MAKE) for-all-target TARGET="impi"

.PHONY: gofmt
gofmt:
	@$(MAKE) for-all-target TARGET="fmt"

.PHONY: install-tools
install-tools:
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/client9/misspell/cmd/misspell
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/golangci/golangci-lint/cmd/golangci-lint
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/google/addlicense
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/ory/go-acc
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/pavius/impi/cmd/impi
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/tcnksm/ghr
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/wadey/gocovmerge
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install go.opentelemetry.io/build-tools/checkdoc
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install go.opentelemetry.io/build-tools/chloggen
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install go.opentelemetry.io/build-tools/semconvgen
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install golang.org/x/exp/cmd/apidiff
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install golang.org/x/tools/cmd/goimports
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install github.com/jcchavezs/porto/cmd/porto
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install go.opentelemetry.io/build-tools/multimod
	cd $(TOOLS_MOD_DIR) && $(GOCMD) install go.opentelemetry.io/build-tools/crosslink
