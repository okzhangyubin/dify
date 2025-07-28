# base image
FROM crpi-732ggfcslctj33s4.cn-guangzhou.personal.cr.aliyuncs.com/moredoc/dify-official-node:22-alpine3.21 AS base
LABEL maintainer="takatost@gmail.com"

# if you located in China, you can use aliyun mirror to speed up
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

RUN apk add --no-cache tzdata
RUN npm install -g pnpm@10.13.1
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"


# install packages
FROM base AS packages

WORKDIR /app/web

# 新增：清理缓存和旧依赖，确保安装环境干净
RUN pnpm cache clean --all  
RUN rm -rf node_modules

COPY package.json .
COPY pnpm-lock.yaml .

# if you located in China, you can use taobao registry to speed up
# 新增：--verbose 输出依赖安装详细日志，便于排查依赖问题
RUN pnpm install --frozen-lockfile --registry https://registry.npmmirror.com/ --verbose

# 保留原有安装命令（确保依赖安装完整性）
RUN pnpm install --frozen-lockfile


# build resources
FROM base AS builder
WORKDIR /app/web
COPY --from=packages /app/web/ .
COPY . .

# 调整：增加内存分配至6GB，解决大型项目构建内存不足问题
ENV NODE_OPTIONS="--max-old-space-size=6144"
# 新增：输出Next.js构建详细日志，定位具体失败步骤
ENV NEXT_DEBUG=true  
ENV DEBUG=next:*  

# 新增：--verbose 输出构建详细日志
RUN pnpm build --verbose


# production stage
FROM base AS production

ENV NODE_ENV=production
ENV EDITION=SELF_HOSTED
ENV DEPLOY_ENV=PRODUCTION
ENV CONSOLE_API_URL=http://127.0.0.1:5001
ENV APP_API_URL=http://127.0.0.1:5001
ENV MARKETPLACE_API_URL=https://marketplace.dify.ai
ENV MARKETPLACE_URL=https://marketplace.dify.ai
ENV PORT=3000
ENV NEXT_TELEMETRY_DISABLED=1
ENV PM2_INSTANCES=2

# set timezone
ENV TZ=UTC
RUN ln -s /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo ${TZ} > /etc/timezone


WORKDIR /app/web
COPY --from=builder /app/web/public ./public
COPY --from=builder /app/web/.next/standalone ./
COPY --from=builder /app/web/.next/static ./.next/static

COPY docker/entrypoint.sh ./entrypoint.sh


# global runtime packages
RUN pnpm add -g pm2 \
    && mkdir /.pm2 \
    && chown -R 1001:0 /.pm2 /app/web \
    && chmod -R g=u /.pm2 /app/web

ARG COMMIT_SHA
ENV COMMIT_SHA=${COMMIT_SHA}

USER 1001
EXPOSE 3000
ENTRYPOINT ["/bin/sh", "./entrypoint.sh"]
