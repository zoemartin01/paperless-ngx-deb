SHELL := /bin/bash
.ONESHELL:
.SHELLFLAGS := -euo pipefail -c

PKG           ?= paperless-ngx
DEBIAN_BRANCH ?= debian

TARBALL_DIR ?= ../tarballs
EXPORT_DIR  ?= ../build

TOPDIR ?= $(shell git rev-parse --show-toplevel 2>/dev/null || pwd)

VERSION ?= $(shell dpkg-parsechangelog -S Version | sed 's/-[^-]*$$//')
DEB_REV ?= 1
FULL_VERSION ?= $(VERSION)-$(DEB_REV)

DIST ?= $(shell dpkg-parsechangelog -S Distribution 2>/dev/null || echo "unstable")

UPSTREAM_GH_TAG ?= v$(VERSION)

# Output orig tarballs (what gbp/dpkg-source look for)
ORIG_TARBALL_GZ ?= $(TARBALL_DIR)/$(PKG)_$(VERSION).orig.tar.gz
ORIG_TARBALL_XZ ?= $(TARBALL_DIR)/$(PKG)_$(VERSION).orig.tar.xz

UPSTREAM_TARBALL ?= $(TARBALL_DIR)/$(PKG)-$(UPSTREAM_GH_TAG).tar.xz

ORIG_URL ?= https://github.com/paperless-ngx/paperless-ngx/releases/download/$(UPSTREAM_GH_TAG)/$(PKG)-$(UPSTREAM_GH_TAG).tar.xz

GBP_BUILD_OPTS ?= --git-debian-branch=$(DEBIAN_BRANCH) \
                 --git-export-dir=$(EXPORT_DIR) \
                 --git-tarball-dir=$(TARBALL_DIR) \
                 --git-overlay

GBP_BUILD_WC_OPTS ?= $(GBP_BUILD_OPTS) --git-export=WC

.PHONY: help
help:
	@printf '%s\n' \
	  'Targets:' \
	  '' \
	  '  make help' \
	  '      Show this help.' \
	  '' \
	  '  make sanity' \
	  '      Check required tools exist and that debian/changelog + debian/control are present.' \
	  '' \
	  '  make checkout' \
	  '      Switch to the Debian packaging branch: $(DEBIAN_BRANCH).' \
	  '' \
	  '  make clean-tree' \
	  '      Fail if there are unstaged or staged changes in the working tree.' \
	  '' \
	  '  make orig [VERSION=x.y.z]' \
	  '      Download the upstream GitHub release tarball if needed:' \
	  '        $(ORIG_URL)' \
	  '      Then repack it with mk-origtargz (gzip) into:' \
	  '        $(TARBALL_DIR)/$(PKG)_$(VERSION).orig.tar.gz' \
	  '      Finally remove .xz artifacts:' \
	  '        - $(TARBALL_DIR)/$(PKG)-v$(VERSION).tar.xz      (cached upstream download)' \
	  '        - $(TARBALL_DIR)/$(PKG)_$(VERSION).orig.tar.xz  (if created)' \
	  '      Notes:' \
	  '        - Runs from TOPDIR ($(TOPDIR)) so mk-origtargz can see ./debian/control.' \
	  '' \
	  '  make release' \
	  '      Create a release commit + tag:' \
	  '        - updates debian/changelog to $(VERSION)-$(DEB_REV) for $(DIST)' \
	  '        - commits + tags debian/$(VERSION)-$(DEB_REV)' \
	  '        - runs: make orig VERSION=$(VERSION)' \
	  '      Next:' \
	  '        make build VERSION=$(VERSION)' \
	  '' \
	  '  make build' \
	  '      Build overlay package from branch $(DEBIAN_BRANCH):' \
	  '        gbp buildpackage $(GBP_BUILD_OPTS)' \
	  '' \
	  '  make build-wc' \
	  '      Like build, but exports the working copy (includes uncommitted changes):' \
	  '        gbp buildpackage $(GBP_BUILD_WC_OPTS)' \
	  '' \
	  '  make source' \
	  '      Source-only overlay build:' \
	  '        gbp buildpackage $(GBP_BUILD_OPTS) -S' \
	  '' \
	  '  make clean' \
	  '      Remove export dir ($(EXPORT_DIR)); keep tarballs.' \
	  '' \
	  '  make distclean' \
	  '      Remove export dir and tarballs dir ($(TARBALL_DIR)).' \
	  '' \
	  'Variables (defaults shown):' \
	  '  PKG=$(PKG)' \
	  '  DEBIAN_BRANCH=$(DEBIAN_BRANCH)' \
	  '  TOPDIR=$(TOPDIR)' \
	  '  VERSION=$(VERSION)              Upstream version (from debian/changelog, without debian revision)' \
	  '  DEB_REV=$(DEB_REV)' \
	  '  FULL_VERSION=$(FULL_VERSION)' \
	  '  DIST=$(DIST)' \
	  '  UPSTREAM_GH_TAG=$(UPSTREAM_GH_TAG)' \
	  '  ORIG_URL=$(ORIG_URL)' \
	  '  TARBALL_DIR=$(TARBALL_DIR)' \
	  '  EXPORT_DIR=$(EXPORT_DIR)'


.PHONY: sanity
sanity:
	@command -v gbp >/dev/null || { echo "ERROR: gbp not found (apt install git-buildpackage)"; exit 1; }
	@command -v dpkg-parsechangelog >/dev/null || { echo "ERROR: dpkg-parsechangelog not found (apt install devscripts)"; exit 1; }
	@command -v dch >/dev/null || { echo "ERROR: dch not found (apt install devscripts)"; exit 1; }
	@command -v mk-origtargz >/dev/null || { echo "ERROR: mk-origtargz not found (apt install devscripts)"; exit 1; }
	@git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "ERROR: not in a git repo"; exit 1; }
	@test -f "$(TOPDIR)/debian/changelog" || { echo "ERROR: debian/changelog missing"; exit 1; }
	@test -f "$(TOPDIR)/debian/control"   || { echo "ERROR: debian/control missing"; exit 1; }

.PHONY: checkout
checkout: sanity
	@cd "$(TOPDIR)"
	@git checkout "$(DEBIAN_BRANCH)" >/dev/null 2>&1 || git switch "$(DEBIAN_BRANCH)"

.PHONY: clean-tree
clean-tree: sanity
	@cd "$(TOPDIR)"
	@git diff --quiet || { echo "ERROR: working tree has unstaged changes"; git status --porcelain; exit 1; }
	@git diff --cached --quiet || { echo "ERROR: index has staged changes"; git status --porcelain; exit 1; }

.PHONY: orig
orig: sanity
	@cd "$(TOPDIR)"
	@mkdir -p "$(TARBALL_DIR)"

	@out_gz="$$(readlink -f "$(ORIG_TARBALL_GZ)")"; \
	out_xz="$$(readlink -f "$(ORIG_TARBALL_XZ)")"; \
	in_xz="$$(readlink -f "$(UPSTREAM_TARBALL)")"; \

	if [ -s "$$out_gz" ]; then \
		echo "Orig tarball already exists:"; \
		echo "  $$out_gz"; \
		rm -f "$$out_xz" "$$in_xz"; \
		exit 0; \
	fi; \

	if [ ! -s "$$in_xz" ]; then \
		echo "Downloading release tarball: $(ORIG_URL)"; \
		tmp="$$(mktemp)"; \
		if command -v curl >/dev/null; then \
			curl -fsSL "$(ORIG_URL)" -o "$$tmp"; \
		else \
			wget -qO "$$tmp" "$(ORIG_URL)"; \
		fi; \
		mv "$$tmp" "$$in_xz"; \
	else \
		echo "Using existing release tarball:"; \
		echo "  $$in_xz"; \
	fi; \

	echo "Repacking with mk-origtargz -> $$out_gz"; \
	mk-origtargz "$$in_xz" --compression gzip -C "$(TARBALL_DIR)" --repack; \

	rm -f "$$out_xz" "$$in_xz"; \
	echo "Done. Kept:"; \
	echo "  $$out_gz"

.PHONY: release
release: sanity checkout clean-tree
	@cd "$(TOPDIR)"
	@echo "Preparing release: $(PKG) $(FULL_VERSION) for $(DIST)…"
	@top="$$(dpkg-parsechangelog -S Version)"; \
	if [ "$$top" = "$(FULL_VERSION)" ]; then \
		echo "ERROR: debian/changelog already at version $(FULL_VERSION)"; \
		exit 1; \
	fi
	@dch --newversion "$(FULL_VERSION)" --distribution "$(DIST)" "New upstream release."
	@git add debian/changelog
	@git commit -m "Release $(FULL_VERSION)"
	@git tag -a debian/$(FULL_VERSION) -m "Release $(FULL_VERSION)"
	@$(MAKE) -f debian/Makefile orig VERSION="$(VERSION)"
	@echo "Release commit created. Next: make build VERSION=$(VERSION)"

.PHONY: build
build: sanity checkout clean-tree
	@cd "$(TOPDIR)"
	@echo "Building overlay package from branch '$(DEBIAN_BRANCH)'…"
	@mkdir -p "$(EXPORT_DIR)" "$(TARBALL_DIR)"
	@gbp buildpackage $(GBP_BUILD_OPTS)

.PHONY: build-wc
build-wc: sanity checkout
	@cd "$(TOPDIR)"
	@echo "Building overlay package (working copy export, includes uncommitted changes)…"
	@mkdir -p "$(EXPORT_DIR)" "$(TARBALL_DIR)"
	@gbp buildpackage $(GBP_BUILD_WC_OPTS)

.PHONY: source
source: sanity checkout clean-tree
	@cd "$(TOPDIR)"
	@echo "Source-only overlay build…"
	@mkdir -p "$(EXPORT_DIR)" "$(TARBALL_DIR)"
	@gbp buildpackage $(GBP_BUILD_OPTS) -S

.PHONY: clean
clean:
	@rm -rf "$(EXPORT_DIR)"

.PHONY: distclean
distclean: clean
	@rm -rf "$(TARBALL_DIR)"

