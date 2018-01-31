BEAT_NAME={beat}
BEAT_INDEX_PREFIX={beat}
BEAT_PATH={beat_path}
BEAT_GOPATH=$(firstword $(subst :, ,${GOPATH}))
BEAT_URL=https://${BEAT_PATH}
BEAT_DOC_URL=https://path/to/{beat}/doc
COPYRIGHT_TXT=My Company
COPYRIGHT_YEAR={now.year}
VERSION=0.1.0

SYSTEM_TESTS=true
TEST_ENVIRONMENT=true
ES_BEATS?=./_beats
PREFIX?=.
BEATS_VERSION?=master # For specific version: BEATS_VERSION?=6.1
NOTICE_FILE=NOTICE.txt
LICENSE_FILE=LICENSE.txt
NOW=$(shell date -u '+%Y-%m-%dT%H:%M:%S')
GOBUILD_FLAGS=-i -ldflags "-s -X $(BEAT_PATH)/version.buildTime=$(NOW) -X $(BEAT_PATH)/version.commit=$(COMMIT_ID)"

# Path to the libbeachmt Makefile
-include $(ES_BEATS)/libbeat/scripts/Makefile

setup:
	$(MAKE) update-dependencies
	$(MAKE) set_version
	git init

# updates {beat} dependencies in the vendor folder
vendor:
	mkdir -p vendor
	govendor init

update-dependencies: vendor
	$(MAKE) update-libbeats
	$(MAKE) update-{beat}-dependencies

update-libbeats:
	@chmod +x scripts/clean_vendor.sh
	@chmod +x scripts/patch_vendorjson.py
	@chmod +x scripts/update_beats.sh
	@chmod +x scripts/update_govendor_deps.py
	rm -rf vendor/github.com/elastic/beats
	govendor fetch github.com/elastic/beats/...@$(BEATS_VERSION)
	govendor fetch github.com/elastic/beats/libbeat/kibana/@$(BEATS_VERSION)
	scripts/patch_vendorjson.py
	BEATS_VERSION=$(BEATS_VERSION) scripts/update_beats.sh
	scripts/patch_vendorjson.py --done
	BEATS_VERSION=$(BEATS_VERSION) scripts/update_beats.sh
	$(MAKE) update
	@echo --- Use this commit message: Update beats framework to `cat vendor/vendor.json | python -c 'import sys, json; print([p["revision"] for p in json.load(sys.stdin)["package"] if p["path"] == "github.com/elastic/beats/libbeat/beat"][0][:7])'`

update-{beat}-dependencies:
	govendor fetch github.com/olivere/elastic@v6.1.2

# This is called by the beats packer before building starts
.PHONY: before-build
before-build:

# Collects all dependencies and then calls update
.PHONY: collect
collect: notice

# Start manual testing environment with agents
start-env:
	@docker-compose -f tests/docker-compose.yml build
	@docker-compose -f tests/docker-compose.yml up -d

# Stop manual testing environment with agents
stop-env:
	@docker-compose -f tests/docker-compose.yml down -v

check-full: check
	@# Validate that all updates were committed
	@$(MAKE) update
	@$(MAKE) check
	@git diff | cat
	@git update-index --refresh
	@git diff-index --exit-code HEAD --

.PHONY: notice
notice: python-env
	@echo "Generating NOTICE.txt"
	@$(PYTHON_ENV)/bin/python ${ES_BEATS}/dev-tools/generate_notice.py . -e '_beats' -s "./vendor/github.com/elastic/beats" -b "$(BEAT_NAME)" -c $(COPYRIGHT_TXT) -y $(COPYRIGHT_YEAR)
