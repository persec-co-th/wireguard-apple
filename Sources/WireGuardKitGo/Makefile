
ARCHS ?= x86_64 arm64
PLATFORM_NAME ?= macosx
SDKROOT ?= $(shell xcrun --sdk $(PLATFORM_NAME) --show-sdk-path)
CONFIGURATION_BUILD_DIR ?= $(CURDIR)/out
CONFIGURATION_TEMP_DIR ?= $(CURDIR)/.tmp

export PATH := $(PATH):/usr/local/bin:/opt/homebrew/bin
export CC ?= clang
LIPO ?= lipo
DESTDIR ?= $(CONFIGURATION_BUILD_DIR)
BUILDDIR ?= $(CONFIGURATION_TEMP_DIR)/wireguard-go-bridge


CFLAGS_PREFIX := $(if $(DEPLOYMENT_TARGET_CLANG_FLAG_NAME),-$(DEPLOYMENT_TARGET_CLANG_FLAG_NAME)=$($(DEPLOYMENT_TARGET_CLANG_ENV_NAME)),) -isysroot $(SDKROOT) -arch


IS_CATALYST :=
ifneq (,$(findstring maccatalyst,$(EFFECTIVE_PLATFORM_NAME)))
  IS_CATALYST := 1
endif


GOARCH_arm64  := arm64
GOARCH_x86_64 := amd64


ifeq ($(IS_CATALYST),1)
  GOOS_macosx := ios
else
  GOOS_macosx := darwin
endif
GOOS_iphoneos := ios
GOOS_iphonesimulator := ios


IOS_MACABI_DEPLOYMENT ?= 15.0

ifeq ($(IS_CATALYST),1)
  TARGET_arm64  := -target arm64-apple-ios$(IOS_MACABI_DEPLOYMENT)-macabi
  TARGET_x86_64 := -target x86_64-apple-ios$(IOS_MACABI_DEPLOYMENT)-macabi
  GO_BUILD_TAGS := -tags ios
else
  TARGET_arm64  :=
  TARGET_x86_64 :=
  GO_BUILD_TAGS :=
endif


build: $(DESTDIR)/libwg-go.a
version-header: $(DESTDIR)/wireguard-go-version.h


REAL_GOROOT := $(shell go env GOROOT 2>/dev/null)
export GOROOT := $(BUILDDIR)/goroot

$(GOROOT)/.prepared:
	[ -n "$(REAL_GOROOT)" ]
	mkdir -p "$(GOROOT)"
	rsync -a --delete --exclude=pkg/obj/go-build "$(REAL_GOROOT)/" "$(GOROOT)/"
	cat goruntime-*.diff | patch -p1 -f -N -r- -d "$(GOROOT)"
	touch "$@"


define libwg-go-a
$(BUILDDIR)/libwg-go-$(1).a: export CGO_ENABLED := 1
$(BUILDDIR)/libwg-go-$(1).a: export GOOS := $(GOOS_$(PLATFORM_NAME))
$(BUILDDIR)/libwg-go-$(1).a: export GOARCH := $(GOARCH_$(1))
$(BUILDDIR)/libwg-go-$(1).a: export CGO_CFLAGS := $(CFLAGS_PREFIX) $(1) $(TARGET_$(1))
$(BUILDDIR)/libwg-go-$(1).a: export CGO_LDFLAGS := $(CFLAGS_PREFIX) $(1) $(TARGET_$(1))
$(BUILDDIR)/libwg-go-$(1).a: $(GOROOT)/.prepared go.mod
	@mkdir -p "$(BUILDDIR)"
	go build -ldflags=-w -trimpath -v $(GO_BUILD_TAGS) -o "$(BUILDDIR)/libwg-go-$(1).a" -buildmode c-archive
	rm -f "$(BUILDDIR)/libwg-go-$(1).h"
endef
$(foreach ARCH,$(ARCHS),$(eval $(call libwg-go-a,$(ARCH))))


$(DESTDIR)/wireguard-go-version.h: go.mod $(GOROOT)/.prepared
	@mkdir -p "$(DESTDIR)"
	sed -E -n 's/.*golang\.zx2c4\.com\/wireguard +v[0-9.]+-[0-9]+-([0-9a-f]{8})[0-9a-f]{4}.*/#define WIREGUARD_GO_VERSION "\1"/p' "$<" > "$@"

$(DESTDIR)/libwg-go.a: $(foreach ARCH,$(ARCHS),$(BUILDDIR)/libwg-go-$(ARCH).a)
	@mkdir -vp "$(DESTDIR)"
	$(LIPO) -create -output "$@" $^


clean:
	rm -rf "$(BUILDDIR)" "$(DESTDIR)/libwg-go.a" "$(DESTDIR)/wireguard-go-version.h"

install: build

.PHONY: clean build version-header install
