# syntax=docker/dockerfile:1
#
# newMSM, built from THIS repository (the jrussell9000 fork, which carries the multithreading
# race fixes 1d0f44a and ce2c6d0). Build from the repository root:
#
#     docker build -f docker/Dockerfile -t newmsm .
#
# The final stage holds ONLY the stripped newmsm binary, the shared libraries it loads, and the
# licence texts that cover them, on a glibc-only distroless base: no shell, no Python, no conda.
# The FSL toolchain lives in the build stage and is discarded. The image's entrypoint is the
# binary; pass newmsm's arguments directly.
#
# LICENCE. newMSM itself is MIT, but this image is not: it links ~8 MB of FSL libraries
# (miscmaths, newimage, newmesh, utils; FSL licence, non-commercial use only), and newMSM bundles
# FastPD (research/non-commercial, patent-pending) and ELC (non-commercial, redistributable as
# part of MSM). The texts are in /opt/newmsm/licenses. Build it for your own non-commercial use;
# do not push it to a public registry, and do not use it commercially (contact Oxford University
# Innovation for FSL and the FastPD authors).
#
# Size (2026-11-08): 37.6 MB uncompressed / 14.7 MB gzipped, of which /opt/newmsm is 21 MB.
# BLAS/LAPACK is netlib, not OpenBLAS: newMSM calls ~15 routines on small matrices, and on a
# pinned c6i.2xlarge netlib ran exactly as fast (1 thread 11.2-11.4 min, 4 threads 3.8-4.0 min,
# 8 ABCD hemispheres) while OpenBLAS adds ~30 MB.
#
# Threads: newMSM reads its thread count from the CONFIG file (--numthreads=N), not the command
# line. Set OMP_NUM_THREADS to match.

FROM ghcr.io/prefix-dev/pixi:0.83.0 AS build
WORKDIR /build
COPY docker/pixi.toml docker/pixi.lock ./
RUN pixi install --locked
COPY LICENSE /src/LICENSE
COPY libraries /src/libraries
COPY src /src/src

# Build with FSL's make system, then assemble /out = bin/newmsm + lib/<only the libraries it
# loads, under their SONAMEs>, stripped, RUNPATH=$ORIGIN. glibc comes from the runtime base.
RUN <<'EOF'
#!/bin/bash
set -euo pipefail
# An empty /out/lib on the first DT_NEEDED pass must expand to nothing, not to the literal
# '/out/lib/*' (which readelf then reports as a missing file).
shopt -s nullglob
eval "$(pixi shell-hook)"
P="$CONDA_PREFIX"
export FSLDIR="$P" FSLCONFDIR="$P/config" FSLDEVDIR=/stage
export CXX="$P/bin/x86_64-conda-linux-gnu-g++" CC="$P/bin/x86_64-conda-linux-gnu-gcc"
# FSL's makefiles take optimisation ONLY from the environment's CFLAGS/CXXFLAGS, which conda's
# compiler activation scripts normally set. pixi's shell-hook does not run them here, and without
# these exports the build is -O0: ~2.8x slower (found 2026-11-08). These are conda-forge's
# linux-64 defaults, the flags fsl-newmsm itself is built with: portable (nocona baseline) and
# comparable with the conda package.
COMMON="-march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem $P/include"
export CFLAGS="$COMMON" CXXFLAGS="-fvisibility-inlines-hidden -fmessage-length=0 $COMMON"
export CPPFLAGS="-DNDEBUG -D_FORTIFY_SOURCE=2 -O2 -isystem $P/include"
export LDFLAGS="-Wl,-O2 -Wl,--sort-common -Wl,--as-needed -Wl,-z,relro -Wl,-z,now -Wl,--disable-new-dtags -Wl,--gc-sections -Wl,--allow-shlib-undefined -Wl,-rpath,$P/lib -Wl,-rpath-link,$P/lib -L$P/lib"
mkdir -p /stage /out/bin /out/lib /out/licenses
for d in libraries/msm-newresampler/src libraries/msm-newmeshreg/src src; do
  make -C "/src/$d" -j"$(nproc)"
  make -C "/src/$d" install
done
cp /stage/bin/newmsm /out/bin/
# Walk DT_NEEDED rather than reading ldd: ldd lists each loaded FILE once, so a library known by
# several names (OpenBLAS is libopenblas.so.0, libblas.so.3 and liblapack.so.3) loses names.
# Each real file is copied once; any further name becomes a symlink to it.
READELF="$P/bin/x86_64-conda-linux-gnu-readelf"
while :; do
  added=0
  for obj in /out/bin/newmsm /out/lib/*; do
    for name in $("$READELF" -d "$obj" | sed -n 's/.*(NEEDED).*\[\(.*\)\]/\1/p'); do
      case "$name" in libc.so.*|libm.so.*|libpthread.so.*|libdl.so.*|librt.so.*|ld-linux*) continue ;; esac
      [ -e "/out/lib/$name" ] && continue
      src=""; for d in /stage/lib "$P/lib"; do [ -e "$d/$name" ] && { src="$d/$name"; break; }; done
      [ -n "$src" ] || { echo "cannot find $name (needed by $obj)" >&2; exit 1; }
      real=$(readlink -f "$src"); base=$(basename "$real")
      [ -e "/out/lib/$base" ] || cp "$real" "/out/lib/$base"
      [ "$base" = "$name" ] || ln -s "$base" "/out/lib/$name"
      added=1
    done
  done
  [ "$added" = 1 ] || break
done
chmod u+w /out/bin/newmsm
find /out/lib -type f -exec chmod u+w {} +
"$P/bin/x86_64-conda-linux-gnu-strip" --strip-unneeded /out/bin/newmsm
find /out/lib -type f -exec "$P/bin/x86_64-conda-linux-gnu-strip" --strip-unneeded {} +
patchelf --set-rpath '$ORIGIN/../lib' /out/bin/newmsm
find /out/lib -type f -exec patchelf --set-rpath '$ORIGIN' {} \;
# Fail the build if anything is unresolved from /out alone (+ system glibc).
if env -u LD_LIBRARY_PATH ldd /out/bin/newmsm | grep -q 'not found'; then
  env -u LD_LIBRARY_PATH ldd /out/bin/newmsm; echo "unresolved libraries" >&2; exit 1
fi
# The licence texts for what ships: newMSM (MIT), its bundled FastPD and ELC, and FSL's.
cp /src/LICENSE /out/licenses/newMSM.LICENSE
cp /src/libraries/msm-newmeshreg/licenses/* /out/licenses/
[ -f "$P/LICENCE.FSL" ] || { echo "fsl-base ships no LICENCE.FSL" >&2; exit 1; }
cp "$P/LICENCE.FSL" /out/licenses/
EOF

# glibc only (no shell): conda's libstdc++/libgomp ship in /opt/newmsm/lib, because conda's GCC
# needs a newer GLIBCXX than Debian's. Digest is the `nonroot` index, as resolved 2026-11-08.
# tag: nonroot
FROM gcr.io/distroless/base-nossl-debian12:nonroot@sha256:be40c00dfabd86576d92666e87e406714d5618342de1a0c213ad232de255172e
COPY --from=build /out/ /opt/newmsm/
ENTRYPOINT ["/opt/newmsm/bin/newmsm"]
