
# Variables

MAKE ?= make
SHELL := /bin/bash
QMAKE := ${MAKE} --no-print-directory

PLATFORM := $(shell ./scripts/detect-platform)

DIR := initrd/${PLATFORM}
INITRD_PATH := $(shell readlink -f '${DIR}' 2>/dev/null || realpath '${DIR}')

DATE := $(shell date +'%Y%m%d')
TIMESTAMP := $(shell date +'%Y-%m-%d %H:%M:%S')

SYSTEM_ID ?= medic-os
SYSTEM_NAME ?= Medic OS
SYSTEM_VERSION_STRING ?= 1.1.0

COMPILER_VERSION ?= 4.8.2
COMPILER_ROOT ?= /srv/software/compiler/v${COMPILER_VERSION}/${PLATFORM}

ifeq (${RELEASE},)
  SYSTEM_IMAGE_ID := ${SYSTEM_ID}-v${SYSTEM_VERSION_STRING}-${DATE}-${PLATFORM}
else
  SYSTEM_IMAGE_ID := ${SYSTEM_ID}-v${SYSTEM_VERSION_STRING}-${PLATFORM}
endif


# Meta-targets

all: packages images

iso: build-iso-image
xen: build-xen-image

initrd: build-initrd
native: build-native-tarball

images: build-iso-image build-xen-image compress-xen-image build-native-tarball build-x86-image-nopae
packages: strip-binaries medic-core-pkg concierge-pkg java-pkg system-services-pkg vm-tools-pkg gardener-pkg medic-transport-pkg


# Public targets

compile:
	@if [ -z '${SKIP_COMPILER}' ]; then \
	  ${QMAKE} compiler || exit "$$?"; \
	fi
	@(cd source && ../scripts/prepend-path '${COMPILER_ROOT}' \
	    ${QMAKE} PLATFORM='${PLATFORM}' COMPILER_ROOT='${COMPILER_ROOT}')

copy:
	@(cd source && \
	  ${QMAKE} copy PLATFORM='${PLATFORM}')

compiler:
	@(cd source && ${QMAKE} compiler \
	  PLATFORM='${PLATFORM}' COMPILER_ROOT='${COMPILER_ROOT}')

clean:
	rm -rf output/* && \
	rm -rf staging/tarball

distclean: clean clean-source clean-staging
	for platform in x86 x64 armv6; do \
	  rm -rf "images/$$platform/iso/packages"/* && \
	  rm -rf "initrd/$$platform/lib/modules"/* && \
	  for type in iso xen; do \
	    rm -f "images/$$platform/$$type/boot/kernel" \
	      "images/$$platform/$$type/boot/image.xz" \
	        "images/$$platform/$$type/packages"/* || exit "$$?"; \
	  done || exit "$$?"; \
	done

clean-compiler:
	(cd source && ${MAKE} clean-compiler)


# Private targets

clean-source:
	(cd source && ${MAKE} clean)

clean-staging:
	rm -rf staging/packages && \
	rm -rf packages/vm-tools/software && \
	rm -rf packages/medic-core/software && \
	rm -rf 'staging/software/${PLATFORM}'/*

build-iso-image: build-initrd
	@echo -n 'Creating ISO image... ' && \
	\
	cd 'images/${PLATFORM}/iso' && mkisofs -J -R \
	  -V '${SYSTEM_NAME} v${SYSTEM_VERSION_STRING}' \
	  -o '../../../output/${SYSTEM_IMAGE_ID}${ISO_EXTRA}-virtual.iso' \
	  -boot-load-size 4 -boot-info-table \
	  -no-emul-boot -b boot/isolinux/isolinux.bin \
	  -c boot/isolinux/boot.cat . &>/dev/null && \
	\
	echo 'done.'

build-xen-image:
	@echo -n 'Creating Xen image... ' && \
	\
	loop_path="staging/loopback" && \
	image_path='output/${SYSTEM_IMAGE_ID}-xen' && \
	ext4_options='none,extent,large_file,has_journal' && \
	\
	dd if=/dev/zero of="$$image_path" \
	  bs=1048576 seek=1023 count=1 &>/dev/null && \
	\
	mkfs.ext4 -O "$$ext4_options" -qF "$$image_path" >/dev/null && \
	mount -o loop "$$image_path" "$$loop_path" && \
	cp -a 'images/${PLATFORM}/xen'/* "$$loop_path/" && \
	mkdir -p "$$loop_path/packages" && \
	\
	cp -a 'images/${PLATFORM}/iso/packages'/* \
	  "$$loop_path/packages/" && \
	\
	sync && umount "$$loop_path" && sync && \
	echo 'done.'

build-ami-image:
	@echo -n 'Building AMI instance-store image... ' && \
	\
	source 'config/aws/settings' && \
	export PATH="$$EC2_HOME/bin:$$PATH" && \
	\
	case '${PLATFORM}' in \
	  x86) architecture='i386';; \
	  x64) architecture='x86_64';; \
	  *) echo 'Fatal: Unsupported platform' >&2; exit 1 ;; \
	esac && \
	\
	rm -rf 'output/${SYSTEM_IMAGE_ID}-ami' && \
	mkdir -p 'output/${SYSTEM_IMAGE_ID}-ami' && \
	ln -f 'output/${SYSTEM_IMAGE_ID}-xen' 'output/image' && \
	\
	ec2-bundle-image \
	  -c "$$EC2_CERTIFICATE" -k "$$EC2_PRIVATE_KEY" \
	  -u "$$AWS_ID" -d 'output/${SYSTEM_IMAGE_ID}-ami' \
	  -i 'output/image' -r "$$architecture" >/dev/null || exit "$$?"; \
	\
	rm -f output/image; \
	echo 'done.'

build-native-tarball:
	@echo -n 'Creating native tarball... ' && \
	\
	base_dir='staging/tarball/${PLATFORM}' && \
	staging_dir="$$base_dir/${SYSTEM_IMAGE_ID}-native" && \
	\
	mkdir -p "$$staging_dir" && \
	cp -a '${INITRD_PATH}'/* "$$staging_dir/" && \
	(cd "$$staging_dir" && mkdir -p srv mnt/startup) && \
	\
	cp -a 'images/${PLATFORM}/iso/packages' \
	  "$$staging_dir/mnt/startup/" && \
	\
	(cd "$$staging_dir/../" && tar -cf - .) \
	  | xz -9ec > "output/${SYSTEM_IMAGE_ID}-native.tar.xz" && \
	\
	echo 'done.'

compress-xen-image:
	@echo -n 'Compressing Xen image... ' && \
	\
	xz -0qcf 'output/${SYSTEM_IMAGE_ID}-xen' \
	  > 'output/${SYSTEM_IMAGE_ID}-xen.xz' && \
	\
	echo 'done.'

upload: upload-ami-image

upload-ami-image: build-ami-image
	@echo -n 'Uploading AMI instance-store image... ' && \
	\
	source 'config/aws/settings' && \
	export PATH="$$EC2_HOME/bin:$$PATH" && \
	\
	ec2-upload-bundle \
	  -a "$$AWS_ACCESS_KEY" \
	  -s "$$AWS_SECRET_KEY" -b "$$S3_BUCKET" \
	  -m 'output/${SYSTEM_IMAGE_ID}-ami/image.manifest.xml' \
	    || exit "$$?"; \
	\
	echo 'done.'

check-compiler-root:
	@if [ -z '${COMPILER_ROOT}' ]; then \
	  echo 'Warning: Compiler root not specified' >&2; \
	  echo 'Warning: Resulting image may be incomplete' >&2; \
	fi

copy-compiler-libraries: check-compiler-root
	@echo -n 'Copying compiler libraries...' && \
	\
	mkdir -p '${INITRD_PATH}/lib' && \
	export PATH="${COMPILER_ROOT}/bin:$$PATH" && \
	compiler_root="$$(dirname "$$(which gcc)")/../" && \
	\
	cd "$$compiler_root/lib" && \
	\
	for lib in ssp gcc_s stdc++; do \
	  cp -a "lib$$lib.so" "lib$$lib.so."* \
	    '${INITRD_PATH}/lib' || exit "$$?"; \
	done && \
	\
	echo 'done.'

prepare-initrd:
	@echo -n 'Preparing initrd files... ' && \
	\
	./scripts/strip-binaries '${INITRD_PATH}' && \
	cd "${INITRD_PATH}" && \
	\
	find . -name '*.la' \
	  -type f -exec rm -f {} \; && \
	\
	find . \( -name '*.so' -o -name '*.so.*' \) \
	  -type f -exec chmod +x {} \; && \
	\
	rm -rf include share lib/*.py && \
	echo 'done.'

build-initrd: copy-compiler-libraries prepare-initrd
	@echo -n 'Creating initrd image... ' && \
	\
	chmod 0440 initrd/common/etc/sudoers && \
	cp -a initrd/common/* '${INITRD_PATH}/' && \
	\
	echo '${TIMESTAMP}' \
	  > '${INITRD_PATH}/etc/build-timestamp' && \
	\
	(cd '${INITRD_PATH}' && \
	  find * | cpio -o -H newc 2>/dev/null \
	    | sh ../../source/core/source/linux/scripts/xz_wrap.sh \
	      > '../../images/${PLATFORM}/iso/boot/image.xz') && \
	\
	(cd 'images/${PLATFORM}' && \
	  if [ -d xen ]; then cp -a iso/boot/image.xz xen/boot/; fi) && \
	\
	echo 'done.'

build-x86-image-nopae:
	@if [ '${PLATFORM}' = 'x86' ]; then \
	  echo 'Building x86 kernel with PAE disabled...' && \
	  (cd source && \
	    ../scripts/prepend-path '${COMPILER_ROOT}' \
	      ${QMAKE} rebuild-kernel KERNEL_EXTRA='-nopae' && \
	    ../scripts/prepend-path '${COMPILER_ROOT}' \
	      ${QMAKE} rebuild-vm-tools-modules) && \
	  \
	  ${QMAKE} build-iso-image ISO_EXTRA='-nopae'; \
	fi

rebuild-kernel:
	@echo 'Rebuilding kernel...' && \
	(cd source && \
	  ../scripts/prepend-path '${COMPILER_ROOT}' \
	    ${QMAKE} rebuild-kernel rebuild-vm-tools-modules)

strip-binaries:
	@echo -n 'Removing unnecessary symbols... ' && \
	./scripts/strip-binaries packages && \
	echo 'done.'

concierge-pkg:
	@echo -n 'Compressing package 'concierge'... ' && \
	scripts/build-package 'concierge' 10110 '${PLATFORM}' && \
	echo 'done.'

java-pkg:
	@echo -n 'Compressing package 'java'... ' && \
	scripts/build-package 'java' 17760 '${PLATFORM}' && \
	echo 'done.'

medic-core-pkg:
	@echo -n 'Compressing package 'medic-core'... ' && \
	scripts/build-package 'medic-core' 16120 '${PLATFORM}' && \
	echo 'done.'

system-services-pkg:
	@echo -n 'Compressing package 'system-services'... ' && \
	scripts/build-package 'system-services' 10200 '${PLATFORM}' && \
	echo 'done.'

vm-tools-pkg:
	@echo -n 'Compressing package 'vm-tools'... ' && \
	scripts/build-package 'vm-tools' 94611 '${PLATFORM}' && \
	echo 'done.'

shrink-gardener:
	@./scripts/shrink-gardener

gardener-pkg: shrink-gardener
	@echo -n 'Compressing package 'gardener'... ' && \
	scripts/build-package 'gardener' 11001 '${PLATFORM}' && \
	echo 'done.'

medic-transport-pkg:
	@echo -n 'Compressing package 'medic-transport'... ' && \
	scripts/build-package 'medic-transport' 10101 '${PLATFORM}' && \
	echo 'done.'

convert-boot-logo:
	for file in logo-medic logo-medic-gray; do \
	  pngtopnm "config/kernel/common/boot-logo/$$file.png" \
	    | ppmquant 224 2>/dev/null | pnmtoplainpnm \
	      > "config/kernel/common/boot-logo/$$file.ppm" || exit "$$?"; \
	done

download:
	@rm -f status/download.finished && \
	for manifest in source/manifests/*; do \
	  pkg="`basename "$$manifest"`" && \
	  if [ "$$pkg" != '*' ]; then \
	    ./scripts/retrieve-sources \
	      "$$pkg" "source/$$pkg/incoming" '${URL}' && \
	    (cd "source/$$pkg" && ./scripts/rearrange-sources) || exit "$$?"; \
	  fi; \
	done && \
	touch status/download.finished;

move-downloaded:
	@if ! [ -f status/move.finished ]; then \
	  for manifest in source/manifests/*; do \
	    pkg="`basename "$$manifest"`" && \
	    if [ "$$pkg" != '*' ]; then \
	      (cd "source/$$pkg" && find incoming -mindepth 1 \
	        -maxdepth 1 -exec mv {} source/ \;) || exit "$$?"; \
	    fi; \
	  done && \
	  touch status/move.finished; \
	fi

delete-downloaded:
	@rm -f status/* && \
	for type in incoming source; do \
	  for manifest in source/manifests/*; do \
	    pkg="`basename "$$manifest"`" && \
	    if [ "$$pkg" != '*' ]; then \
	      (cd "source/$$pkg" && rm -rf "$$type"/*) || exit "$$?"; \
	    fi; \
	  done; \
	done

