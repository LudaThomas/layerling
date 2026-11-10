# syntax=docker/dockerfile:1

FROM node:24-bookworm-slim AS base
WORKDIR /app

FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci

FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production
ENV LAYERLING_STANDALONE=true
RUN npm run build

# Only what the running server needs: the traced server, the browser files and an empty /tmp. The webpack cache
# (about 300 MB) stays behind because the standalone output leaves it out. Taken out of what the tracer picked up:
# sharp, which only serves Next's image optimizer (off in next.config.ts) and is built for glibc while the runner
# uses musl; typescript and the source folder, which only the build reads; and the bundler's copy of the CAD kernel,
# since every loader asks for /occt/<version>/ instead (scripts/generate-service-worker.mjs skips it for that reason).
RUN mkdir -p /runtime/app /runtime/tmp \
  && chmod 1777 /runtime/tmp \
  && cp -a apps/web/.next/standalone/. /runtime/app/ \
  && cp -a apps/web/.next/static /runtime/app/apps/web/.next/static \
  && cp -a apps/web/public /runtime/app/apps/web/public \
  && rm -rf /runtime/app/node_modules/sharp /runtime/app/node_modules/@img \
    /runtime/app/node_modules/typescript /runtime/app/apps/web/src \
    /runtime/app/apps/web/.next/static/media/occt-wasm.*.wasm \
  && find /runtime/app/node_modules \( -name '*.map' -o -name '*.d.ts' \) -type f -delete \
  && ln -s ../../.codex /runtime/app/apps/web/.codex \
  && mkdir -p /thumbnails/project-thumbnails

# The Node binary and the few libraries it links against, without npm, yarn or a package manager.
FROM node:24-alpine AS node-runtime
RUN apk add --no-cache binutils \
  && mkdir -p /out/lib /out/usr/lib /out/usr/local/bin \
  && cp -L /lib/ld-musl-*.so.1 /out/lib/ \
  && cp -a /lib/libc.musl-*.so.1 /out/lib/ \
  && cp -L /usr/lib/libstdc++.so.6 /usr/lib/libgcc_s.so.1 /out/usr/lib/ \
  && cp /usr/local/bin/node /out/usr/local/bin/node \
  && strip --strip-all /out/usr/local/bin/node /out/usr/lib/libstdc++.so.6 \
  && /out/usr/local/bin/node --version

# There is no shell in this image: "docker exec ... sh" does not work, on purpose.
FROM scratch AS runner
ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME=0.0.0.0
WORKDIR /app

COPY --from=node-runtime /out/ /
COPY --from=builder /runtime/ /

# The start page keeps the preview pictures of the designs here; the folder must belong to the user the
# server runs as, or every picture is refused and the cards stay on "No preview yet". The standalone server
# changes into apps/web, and the link it finds there leads back to this folder, where the volume is mounted.
COPY --from=builder --chown=999:999 /thumbnails/ /app/.codex/

USER 999:999
EXPOSE 3000

CMD ["/usr/local/bin/node", "apps/web/server.js"]
