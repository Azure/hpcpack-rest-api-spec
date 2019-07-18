FROM ubuntu:16.04
RUN \
  apt-get update && \
  apt-get install -y \
    curl \
    git \
    openjdk-9-jre-headless
WORKDIR /opt
RUN \
  curl https://jcenter.bintray.com/io/github/swagger2markup/swagger2markup-cli/1.3.3/swagger2markup-cli-1.3.3.jar -LO && \
  curl https://raw.githubusercontent.com/ekalinin/github-markdown-toc/master/gh-md-toc -LO && chmod +x gh-md-toc
ENV swagger2markup_cli=/opt/swagger2markup-cli-1.3.3.jar \
  PATH="/opt:$PATH"
WORKDIR /app
