GIT_TAG ?= dirty-tag
GIT_VERSION ?= $(shell git describe --tags --always --dirty)
GIT_HASH ?= $(shell git rev-parse HEAD)
DATE_FMT = +'%Y-%m-%dT%H:%M:%SZ'
SOURCE_DATE_EPOCH ?= $(shell git log -1 --pretty=%ct)
ifdef SOURCE_DATE_EPOCH
    BUILD_DATE ?= $(shell date -u -d "@$(SOURCE_DATE_EPOCH)" "$(DATE_FMT)" 2>/dev/null || date -u -r "$(SOURCE_DATE_EPOCH)" "$(DATE_FMT)" 2>/dev/null || date -u "$(DATE_FMT)")
else
    BUILD_DATE ?= $(shell date "$(DATE_FMT)")
endif
GIT_TREESTATE = "clean"
DIFF = $(shell git diff --quiet >/dev/null 2>&1; if [ $$? -eq 1 ]; then echo "1"; fi)
ifeq ($(DIFF), 1)
    GIT_TREESTATE = "dirty"
endif

# PKG=github.com/Poluect/scss/main
PKG=main
LDFLAGS="-X $(PKG).GitVersion=$(GIT_VERSION) -X $(PKG).gitCommit=$(GIT_HASH) -X $(PKG).gitTreeState=$(GIT_TREESTATE) -X $(PKG).buildDate=$(BUILD_DATE)"

GO_BUILD_FLAGS := -trimpath -ldflags $(LDFLAGS)
COMMANDS       := src

.PHONY: help
help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-25s\033[0m %s\n", $$1, $$2}'

FORCE: ;

.PHONY: download
download: ## download dependencies via go mod
	go mod download

$(GO_PATH)/bin/goimports:
	go install golang.org/x/tools/cmd/goimports@latest

$(GO_PATH)/bin/golint:
	go install golang.org/x/lint/golint@latest

.PHONY: lint
lint: $(GO_PATH)/bin/goimports $(GO_PATH)/bin/golint ## runs linting
	@echo Linting using golint
	@golint -set_exit_status $(shell go list -f '{{ .Dir }}' ./...)
	@echo Linting imports
	@goimports -d -e -local github.com/philips-labs/slsa-provenance-action $(shell go list -f '{{ .Dir }}' ./...)

.PHONY: test
test: ## runs the tests
	go test -race -v -count=1 ./...

coverage.out:
	go test -race -v -count=1 -covermode=atomic -coverprofile=coverage.out ./... || true

.PHONY: coverage.out
coverage-out: coverage.out ## Ouput code coverage to stdout
	go tool cover -func=$<

.PHONY: coverage.out
coverage-html: coverage.out ## Ouput code coverage as HTML
	go tool cover -html=$<

.PHONY: build
build: 
	CGO_ENABLED=0 go build -mod=readonly $(GO_BUILD_FLAGS) -o bin/sscs .

.PHONY: image
image: ## build the binary in a docker image
	docker build \
		-t "skifdh/sscs:$(GIT_TAG)" \
		-t "skifdh/sscs:$(GIT_HASH)" .

$(GO_PATH)/bin/goreleaser:
	go install github.com/goreleaser/goreleaser@v0.182.1

.PHONY: snapshot-release
snapshot-release: $(GO_PATH)/bin/goreleaser ## creates a snapshot release using goreleaser
	LDFLAGS=$(LDFLAGS) goreleaser release --snapshot --rm-dist

.PHONY: release
release: $(GO_PATH)/bin/goreleaser ## creates a release using goreleaser
	LDFLAGS=$(LDFLAGS) goreleaser release

.PHONY: release-vars
release-vars: ## print the release variables for goreleaser
	@echo export LDFLAGS=\"$(LDFLAGS)\"
