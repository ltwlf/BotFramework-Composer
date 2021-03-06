#################
#
# Because Composer is organized as a monorepo with multiple packages
# managed by yarn workspaces, our Dockerfile may not look like other
# node / react projects. Specifically, we have to add all source files
# before doing yarn install due to yarn workspace symlinking.
#
################

FROM node:12-alpine as build

ARG YARN_ARGS

WORKDIR /src/Composer
COPY . .
# run yarn install as a distinct layer
RUN yarn install --frozen-lock-file $YARN_ARGS
ENV NODE_OPTIONS "--max-old-space-size=4096"
ENV NODE_ENV "production"
RUN yarn build:prod $YARN_ARGS



FROM node:12-alpine as composerbasic
ARG YARN_ARGS

WORKDIR /app/Composer
COPY --from=build /src/Composer/yarn.lock .
COPY --from=build /src/Composer/package.json .
COPY --from=build /src/Composer/packages/client ./packages/client
COPY --from=build /src/Composer/packages/adaptive-flow ./packages/adaptive-flow
COPY --from=build /src/Composer/packages/adaptive-form ./packages/adaptive-form
COPY --from=build /src/Composer/packages/extension ./packages/extension
COPY --from=build /src/Composer/packages/extension-client ./packages/extension-client
COPY --from=build /src/Composer/packages/intellisense ./packages/intellisense
COPY --from=build /src/Composer/packages/server ./packages/server
COPY --from=build /src/Composer/packages/lib ./packages/lib
COPY --from=build /src/Composer/packages/tools ./packages/tools
COPY --from=build /src/Composer/packages/ui-plugins ./packages/ui-plugins
COPY --from=build /src/extensions ./extensions

ENV NODE_ENV "production"
RUN yarn --production --frozen-lockfile --force $YARN_ARGS && yarn cache clean
WORKDIR /app/Composer

FROM composerbasic

RUN apk add --no-cache \
  ca-certificates \
  \
  # .NET Core dependencies
  krb5-libs \
  libgcc \
  libintl \
  libssl1.1 \
  libstdc++ \
  zlib

# Install .Net Core SDK
ENV \
  # Unset the value from the base image
  ASPNETCORE_URLS= \
  # Disable the invariant mode (set in base image)
  DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
  # Enable correct mode for dotnet watch (only mode supported in a container)
  DOTNET_USE_POLLING_FILE_WATCHER=true \
  LC_ALL=en_US.UTF-8 \
  LANG=en_US.UTF-8 \
  # Skip extraction of XML docs - generally not useful within an image/container - helps performance
  NUGET_XMLDOC_MODE=skip \
  # PowerShell telemetry for docker image usage
  POWERSHELL_DISTRIBUTION_CHANNEL=PSDocker-DotnetCoreSDK-Alpine-3.10

# Add dependencies for disabling invariant mode (set in base image)
RUN apk add --no-cache icu-libs

# Install .NET Core SDK 3.1
ENV DOTNET_SDK_VERSION 3.1.101

RUN wget -O dotnet.tar.gz https://dotnetcli.azureedge.net/dotnet/Sdk/$DOTNET_SDK_VERSION/dotnet-sdk-$DOTNET_SDK_VERSION-linux-musl-x64.tar.gz \
  && dotnet_sha512='ce386da8bc07033957fd404909fc230e8ab9e29929675478b90f400a1838223379595a4459056c6c2251ab5c722f80858b9ca536db1a2f6d1670a97094d0fe55' \
  && echo "$dotnet_sha512  dotnet.tar.gz" | sha512sum -c - \
  && mkdir -p /usr/share/dotnet \
  && tar -C /usr/share/dotnet -oxzf dotnet.tar.gz \
  && ln -s /usr/share/dotnet/dotnet /usr/bin/dotnet \
  && rm dotnet.tar.gz

# Enable detection of running in a container
ENV DOTNET_RUNNING_IN_CONTAINER=true \
  # Set the invariant mode since icu_libs isn't included (see https://github.com/dotnet/announcements/issues/20)
  DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true

CMD ["yarn","start:server"]
