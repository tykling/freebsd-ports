# Created by: Jyun-Yan You <jyyou@cs.nctu.edu.tw>
# $FreeBSD$

PORTNAME=	rust
PORTVERSION?=	1.44.0
PORTREVISION?=	0
CATEGORIES=	lang
MASTER_SITES=	https://static.rust-lang.org/dist/:src \
		LOCAL/tobik/rust:bootstrap \
		https://static.rust-lang.org/dist/:bootstrap
DISTNAME?=	${PORTNAME}c-${PORTVERSION}-src
DISTFILES?=	${NIGHTLY_DATE:D${NIGHTLY_DATE}/}${DISTNAME}${EXTRACT_SUFX}:src \
		${_RUSTC_BOOTSTRAP}${BOOTSTRAPS_SUFFIX}${EXTRACT_SUFX}:bootstrap \
		${_RUST_STD_BOOTSTRAP}${BOOTSTRAPS_SUFFIX}${EXTRACT_SUFX}:bootstrap \
		${_CARGO_BOOTSTRAP}${BOOTSTRAPS_SUFFIX}${EXTRACT_SUFX}:bootstrap
DIST_SUBDIR?=	rust

MAINTAINER=	rust@FreeBSD.org
COMMENT=	Language with a focus on memory safety and concurrency

LICENSE=	APACHE20 MIT
LICENSE_COMB=	dual
LICENSE_FILE_APACHE20=	${WRKSRC}/LICENSE-APACHE
LICENSE_FILE_MIT=	${WRKSRC}/LICENSE-MIT

IGNORE_FreeBSD_11_powerpc64=	is missing a bootstrap for FreeBSD 11.x powerpc64
ONLY_FOR_ARCHS?=	aarch64 amd64 armv6 armv7 i386 powerpc64
ONLY_FOR_ARCHS_REASON?=	requires prebuilt bootstrap compiler

BUILD_DEPENDS=	cmake:devel/cmake \
		libgit2>=1.0.0:devel/libgit2
.if defined(NIGHTLY_DATE)
BUILD_DEPENDS+=	ninja:devel/ninja
.endif
LIB_DEPENDS=	libcurl.so:ftp/curl \
		libgit2.so:devel/libgit2 \
		libssh2.so:security/libssh2

USES=		pkgconfig python:3.3+,build ssl tar:xz

MAKE_ENV=	LIBGIT2_SYS_USE_PKG_CONFIG=1 \
		LIBSSH2_SYS_USE_PKG_CONFIG=1 \
		OPENSSL_DIR="${OPENSSLBASE}" \
		RUST_BACKTRACE=1
TEST_ENV=	${MAKE_ENV} \
		ALLOW_NONZERO_RLIMIT_CORE=1

CONFLICTS_INSTALL?=	rust-nightly

OPTIONS_DEFINE=		DOCS GDB SOURCES WASM
OPTIONS_DEFAULT=	SOURCES WASM

GDB_DESC=	Install ports gdb (necessary for debugging rust programs)
SOURCES_DESC=	Install source files
WASM_DESC=	Build the WebAssembly target (wasm32-unknown-unknown)

DOCS_VARS=		_RUST_BUILD_DOCS=true
DOCS_VARS_OFF=		_RUST_BUILD_DOCS=false
GDB_RUN_DEPENDS=	${LOCALBASE}/bin/gdb:devel/gdb
WASM_VARS=		_RUST_BUILD_WASM=true \
			_RUST_TARGETS+=wasm32-unknown-unknown
WASM_VARS_OFF=		_RUST_BUILD_WASM=false

# See WRKSRC/src/stage0.txt for the date and version values.
BOOTSTRAPS_DATE?=		2020-05-07
RUST_BOOTSTRAP_VERSION?=	1.43.1
CARGO_BOOTSTRAP_VERSION?=	0.44.0

BOOTSTRAPS_SUFFIX?=		${BOOTSTRAPS_SUFFIX_${ARCH}}
BOOTSTRAPS_SUFFIX_powerpc64?=	-${PPC_ABI:tl}

CARGO_VENDOR_DIR?=		${WRKSRC}/vendor

# Rust's target arch string might be different from *BSD arch strings
_RUST_ARCH_amd64=	x86_64
_RUST_ARCH_i386=	i686
_RUST_TARGET=		${_RUST_ARCH_${ARCH}:U${ARCH}}-unknown-${OPSYS:tl}
_RUST_TARGETS=		${_RUST_TARGET}

_RUSTC_BOOTSTRAP=	${BOOTSTRAPS_DATE_${ARCH}:U${BOOTSTRAPS_DATE}}/rustc-${RUST_BOOTSTRAP_VERSION_${ARCH}:U${RUST_BOOTSTRAP_VERSION}}-${_RUST_TARGET}
_RUST_STD_BOOTSTRAP=	${BOOTSTRAPS_DATE_${ARCH}:U${BOOTSTRAPS_DATE}}/rust-std-${RUST_BOOTSTRAP_VERSION_${ARCH}:U${RUST_BOOTSTRAP_VERSION}}-${_RUST_TARGET}
_CARGO_BOOTSTRAP=	${BOOTSTRAPS_DATE_${ARCH}:U${BOOTSTRAPS_DATE}}/cargo-${CARGO_BOOTSTRAP_VERSION_${ARCH}:U${CARGO_BOOTSTRAP_VERSION}}-${_RUST_TARGET}

.include <bsd.port.pre.mk>

.if exists(${PATCHDIR}/${ARCH}${BOOTSTRAPS_SUFFIX})
EXTRA_PATCHES+=	${PATCHDIR}/${ARCH}${BOOTSTRAPS_SUFFIX}
.endif

.if ${ARCH} == powerpc64
# The bootstrap is hardcoded to use gcc9
# but we can build with a newer or older compiler as provided by USE_GCC=yes
BUILD_DEPENDS+=	gcc9:lang/gcc9
USE_GCC=	yes
.endif

.if ${ARCH} == aarch64 && ${OSVERSION} < 1200502
IGNORE=	fails to run due to a bug in rtld, update to 12-STABLE r342847 or 13-CURRENT r342113
.endif

.ifdef QEMU_EMULATING
IGNORE=	fails to build with qemu-user-static
.endif

post-patch:
	@${REINPLACE_CMD} 's,gdb,${LOCALBASE}/bin/gdb,' ${WRKSRC}/src/etc/rust-gdb
.if defined(NIGHTLY_DATE)
	@${REINPLACE_CMD} '/^rustfmt:/d' ${WRKSRC}/src/stage0.txt
.endif
.if ${ARCH} == powerpc64
# Starting with 13.0 powerpc64 is ELFv2 while 12.0 is ELFv1.  Pass the correct LLVM target triple.
	@${REINPLACE_CMD} 's,powerpc64-unknown-freebsd,powerpc64-unknown-freebsd${OSREL},' \
		${WRKSRC}/src/librustc_target/spec/powerpc64_unknown_freebsd.rs
.endif
# Disable vendor checksums
	@${REINPLACE_CMD} 's,"files":{[^}]*},"files":{},' \
		${CARGO_VENDOR_DIR}/*/.cargo-checksum.json

post-patch-SOURCES-off:
# Mimic tools in config.toml with just src excluded
	@${REINPLACE_CMD} 's,config.tools.*"src".*,false;,' \
		${WRKSRC}/src/bootstrap/install.rs

do-configure:
# Check that the running kernel has COMPAT_FREEBSD11 required by lang/rust post-ino64
	@${SETENV} CC="${CC}" OPSYS="${OPSYS}" OSVERSION="${OSVERSION}" WRKDIR="${WRKDIR}" \
		${SH} ${SCRIPTSDIR}/rust-compat11-canary.sh
.for _component in cargo rust-std rustc
	@cd ${WRKDIR}/${_component}-*-${OPSYS:tl} && \
		${SH} install.sh --disable-ldconfig --prefix=${WRKDIR}/bootstrap \
		--verbose
.endfor
	@${ECHO_CMD} '[build]' > ${WRKSRC}/config.toml
	@${ECHO_CMD} 'vendor=true' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'extended=true' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'python="${PYTHON_CMD}"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'docs=${_RUST_BUILD_DOCS}' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'verbose=2' >> ${WRKSRC}/config.toml
.if defined(NIGHTLY_DATE)
	@${ECHO_CMD} 'profiler=true' >> ${WRKSRC}/config.toml
.endif
	@${ECHO_CMD} 'target=[${_RUST_TARGETS:@.target.@"${.target.}"@:ts,}]' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'cargo="${WRKDIR}/bootstrap/bin/cargo"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'rustc="${WRKDIR}/bootstrap/bin/rustc"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} '[install]' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'prefix="${PREFIX}"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'sysconfdir="${PREFIX}/etc"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} '[rust]' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'channel="${PKGNAMESUFFIX:Ustable:S/^-//}"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'default-linker="${CC}"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'verbose-tests=true' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'lld=${_RUST_BUILD_WASM}' >> ${WRKSRC}/config.toml
.if ${ARCH} == powerpc64
# XXX: crashes without this?
	@${ECHO_CMD} 'debuginfo-level-std=1' >> ${WRKSRC}/config.toml
.endif
	@${ECHO_CMD} '[llvm]' >> ${WRKSRC}/config.toml
.if defined(WITH_CCACHE_BUILD) && !defined(NO_CCACHE)
	@${ECHO_CMD} 'ccache="${CCACHE_BIN}"' >> ${WRKSRC}/config.toml
.else
	@${ECHO_CMD} 'ccache=false' >> ${WRKSRC}/config.toml
.endif
.if defined(NIGHTLY_DATE)
	@${ECHO_CMD} 'ninja=true' >> ${WRKSRC}/config.toml
.endif
.if ${ARCH} == armv6
# fails to link with base ld.bfd: rustc_codegen_llvm.e2557spx-cgu.11:(.text._ZN89_$LT$rustc_target..abi..call..CastTarget$u20$as$u20$rustc_codegen_llvm..abi..LlvmType$GT$9llvm_type17h1296210ab461fc57E+0x54): relocation truncated to fit: R_ARM_CALL against symbol `__aeabi_uldivmod' defined in .text.__aeabi_uldivmod section in /tmp/rustcdnGbao/libcompiler_builtins-ee65b414e4115a8f.rlib(compiler_builtins-ee65b414e4115a8f.compiler_builtins.ay8p39ey-cgu.13.rcgu.o)
	@${PRINTF} '#!/bin/sh\nexec ${CC} -fuse-ld=lld "$$@"' > ${WRKDIR}/cc-wrapper
	@${CHMOD} +x ${WRKDIR}/cc-wrapper
.endif
.for _target in ${_RUST_TARGETS}
	@${ECHO_CMD} '[target.${_target}]' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'ar="${AR}"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'cc="${CC}"' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'cxx="${CXX}"' >> ${WRKSRC}/config.toml
.if ${ARCH} == armv6
	@${ECHO_CMD} 'linker="${WRKDIR}/cc-wrapper"' >> ${WRKSRC}/config.toml
.else
	@${ECHO_CMD} 'linker="${CC}"' >> ${WRKSRC}/config.toml
.endif
.endfor
	@${ECHO_CMD} '[dist]' >> ${WRKSRC}/config.toml
	@${ECHO_CMD} 'src-tarball=false' >> ${WRKSRC}/config.toml
.if defined(NIGHTLY_DATE)
# Don't abort if optional tools fail to build
	@${ECHO_CMD} 'missing-tools=true' >> ${WRKSRC}/config.toml
.endif
	@${REINPLACE_CMD} 's,%CC%,${CC},g' \
		${WRKSRC}/src/librustc_llvm/build.rs \
		${WRKSRC}/src/bootstrap/native.rs

do-build:
	@cd ${WRKSRC} && \
		${SETENV} ${MAKE_ENV} ${PYTHON_CMD} x.py build --jobs=${MAKE_JOBS_NUMBER}

do-install:
# DESTDIR not in MAKE_ENV as it would cause the bundled LLVM to be
# staged into it during do-build.
	@cd ${WRKSRC} && \
		${SETENV} ${MAKE_ENV} DESTDIR=${STAGEDIR} ${PYTHON_CMD} \
		x.py install --jobs=${MAKE_JOBS_NUMBER}
# We autogenerate the plist file.  We do that, instead of the
# regular pkg-plist, because several libraries have a computed
# filename based on the absolute path of the source files.  As it
# is user-specific, we cannot know their filename in advance.
	@${RM} -r ${STAGEDIR}${DOCSDIR}/*.old \
		${STAGEDIR}${DOCSDIR}/html/.lock \
		${STAGEDIR}${DOCSDIR}/html/.stamp \
		${STAGEDIR}${PREFIX}/lib/rustlib/install.log \
		${STAGEDIR}${PREFIX}/lib/rustlib/manifest-* \
		${STAGEDIR}${PREFIX}/lib/rustlib/uninstall.sh
	@${FIND} ${STAGEDIR}${PREFIX}/bin ${STAGEDIR}${PREFIX}/lib -exec ${FILE} -i {} + | \
		${AWK} -F: '/executable|sharedlib/ { print $$1 }' | ${XARGS} ${STRIP_CMD}
	@${FIND} ${STAGEDIR}${PREFIX} -not -type d | \
		${SED} -E -e 's,^${STAGEDIR}${PREFIX}/,,' \
			-e 's,(share/man/man[1-9]/.*\.[0-9]),\1.gz,' >> ${TMPPLIST}

post-install-SOURCES-on:
# Silence stage-qa warnings by sanitizing permissions on sources
	@${FIND} ${STAGEDIR}${PREFIX}/lib/rustlib/src -type f -exec ${CHMOD} \
		${SHAREMODE} {} +

# Note that make test does not work when rust is already installed.
do-test:
	@cd ${WRKSRC} && \
		${SETENV} ${TEST_ENV} ${PYTHON_CMD} x.py test --jobs=${MAKE_JOBS_NUMBER}

.if !defined(_RUST_MAKESUM_GUARD)
makesum:
	${MAKE} -D_RUST_MAKESUM_GUARD makesum ARCH=${ONLY_FOR_ARCHS:O:[1]} DISTINFO_FILE=${DISTINFO_FILE}.tmp
.for arch in ${ONLY_FOR_ARCHS:O:[2..-1]}
	${MAKE} -D_RUST_MAKESUM_GUARD makesum PPC_ABI=ELFv1 ARCH=${arch} DISTINFO_FILE=${DISTINFO_FILE}.${arch}
	${SED} 1d ${DISTINFO_FILE}.${arch} >> ${DISTINFO_FILE}.tmp
	${RM} ${DISTINFO_FILE}.${arch}
.endfor
.if ${ONLY_FOR_ARCHS:Mpowerpc64}
	${MAKE} -D_RUST_MAKESUM_GUARD makesum PPC_ABI=ELFv2 ARCH=powerpc64 DISTINFO_FILE=${DISTINFO_FILE}.powerpc64-elfv2
	${SED} 1d ${DISTINFO_FILE}.powerpc64-elfv2 >> ${DISTINFO_FILE}.tmp
	${RM} ${DISTINFO_FILE}.powerpc64-elfv2
.endif
	${AWK} '!seen[$$0]++' ${DISTINFO_FILE}.tmp > ${DISTINFO_FILE}
	${RM} ${DISTINFO_FILE}.tmp
.endif

.include <bsd.port.post.mk>
