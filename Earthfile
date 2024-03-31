VERSION 0.7


COPY_CI_DATA:
    COMMAND
    COPY "./ci" "./ci"
    COPY ".github" ".github"
    COPY ".goreleaser.yaml" ".goreleaser.yaml"


COPY_METADATA:
    COMMAND
    DO +COPY_CI_DATA
    COPY ".git" ".git"


rust-base:
    FROM rust:1.70.0


check-clean-git-history:
    FROM +rust-base
    RUN cargo install clean_git_history --version 0.1.2 --locked
    DO +COPY_METADATA
    ARG from_reference="origin/HEAD"
    RUN ./ci/check-clean-git-history.sh --from-reference "${from_reference}"


check-conventional-commits-linting:
    FROM +rust-base
    RUN cargo install conventional_commits_linter --version 0.12.3 --locked
    DO +COPY_METADATA
    ARG from_reference="origin/HEAD"
    RUN ./ci/check-conventional-commits-linting.sh --from-reference "${from_reference}"


INSTALL_DEPENDENCIES:
    COMMAND
    COPY "go.mod" "go.mod"
    COPY "go.sum" "go.sum"
    RUN go mod download


COPY_SOURCECODE:
    COMMAND
    DO +COPY_CI_DATA
    COPY "./cmd" "./cmd"
    COPY "./api" "./api"


golang-base:
    FROM golang:1.20.4
    WORKDIR /tmp/starling-bank-technical-challenge
    ENV GOPROXY=direct
    ENV CGO_ENABLED=0
    ENV GOOS=linux
    ENV GOARCH=amd64


check-go-formatting:
    FROM +golang-base
    DO +INSTALL_DEPENDENCIES
    DO +COPY_SOURCECODE
    RUN ./ci/check-go-formatting.sh


sh-formatting-base:
    FROM golang
    RUN go install mvdan.cc/sh/v3/cmd/shfmt@v3.6.0
    DO +COPY_CI_DATA


check-shell-formatting:
    FROM +sh-formatting-base
    RUN ./ci/check-shell-formatting.sh


yaml-formatting-base:
    FROM golang
    RUN go install github.com/google/yamlfmt/cmd/yamlfmt@v0.9.0
    COPY ".yamlfmt" ".yamlfmt"
    DO +COPY_CI_DATA


check-yaml-formatting:
    FROM +yaml-formatting-base
    RUN ./ci/check-yaml-formatting.sh


check-formatting:
    BUILD +check-go-formatting
    BUILD +check-shell-formatting
    BUILD +check-yaml-formatting


fix-go-formatting:
    FROM +golang-base
    DO +INSTALL_DEPENDENCIES
    DO +COPY_SOURCECODE
    RUN ./ci/fix-go-formatting.sh
    SAVE ARTIFACT "./src" AS LOCAL "./src"


fix-shell-formatting:
    FROM +sh-formatting-base
    RUN ./ci/fix-shell-formatting.sh
    SAVE ARTIFACT "./ci" AS LOCAL "./ci"


fix-yaml-formatting:
    FROM +yaml-formatting-base
    RUN ./ci/fix-yaml-formatting.sh
    SAVE ARTIFACT "./.github" AS LOCAL "./.github"


fix-formatting:
    BUILD +fix-go-formatting
    BUILD +fix-shell-formatting
    BUILD +fix-yaml-formatting


check-go-linting:
    FROM +golang-base
    RUN go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.53.0
    DO +INSTALL_DEPENDENCIES
    DO +COPY_SOURCECODE
    RUN ./ci/check-go-linting.sh


ubuntu-base:
    FROM ubuntu:22.04
    # https://askubuntu.com/questions/462690/what-does-apt-get-fix-missing-do-and-when-is-it-useful
    RUN apt-get update --fix-missing


check-shell-linting:
    FROM +ubuntu-base
    RUN apt-get install shellcheck -y
    DO +COPY_CI_DATA
    RUN ./ci/check-shell-linting.sh


check-github-actions-workflows-linting:
    FROM +golang-base
    RUN go install github.com/rhysd/actionlint/cmd/actionlint@v1.6.26
    DO +COPY_CI_DATA
    RUN ./ci/check-github-actions-workflows-linting.sh


check-linting:
    BUILD +check-go-linting
    BUILD +check-shell-linting
    BUILD +check-github-actions-workflows-linting


check-modules:
    FROM +golang-base
    DO +INSTALL_DEPENDENCIES
    DO +COPY_SOURCECODE
    RUN ./ci/check-modules.sh


fix-modules:
    FROM +golang-base
    DO +INSTALL_DEPENDENCIES
    DO +COPY_SOURCECODE
    RUN ./ci/fix-modules.sh
    SAVE ARTIFACT "go.mod" AS LOCAL "go.mod"
    SAVE ARTIFACT "go.sum" AS LOCAL "go.sum"


INSTALL_GORELEASER:
    COMMAND
    ENV GORELEASER_VERSION=1.22.1
    RUN wget "https://github.com/goreleaser/goreleaser/releases/download/v${GORELEASER_VERSION}/goreleaser_Linux_x86_64.tar.gz"
    RUN tar -xzvf "goreleaser_Linux_x86_64.tar.gz"
    RUN cp "goreleaser" /bin/goreleaser


compile:
    FROM +golang-base
    DO +INSTALL_GORELEASER
    DO +INSTALL_DEPENDENCIES
    DO +COPY_SOURCECODE
    RUN ./ci/compile.sh
    SAVE ARTIFACT "dist" AS LOCAL "dist"
    SAVE ARTIFACT "go.sum" AS LOCAL "go.sum"


unit-test:
    FROM +golang-base
    DO +INSTALL_DEPENDENCIES
    DO +COPY_SOURCECODE
    RUN ./ci/unit-test.sh


INSTALL_GITHUB_CLI:
    COMMAND
    ENV GITHUB_CLI_VERSION=2.30.0
    RUN wget "https://github.com/cli/cli/releases/download/v${GITHUB_CLI_VERSION}/gh_${GITHUB_CLI_VERSION}_linux_amd64.tar.gz"
    RUN tar -xzvf "gh_${GITHUB_CLI_VERSION}_linux_amd64.tar.gz"
    RUN cp "./gh_${GITHUB_CLI_VERSION}_linux_amd64/bin/gh" /bin/gh


release-artifacts:
    FROM +ubuntu-base
    RUN apt-get install git wget jq -y
    DO +INSTALL_GITHUB_CLI
    COPY +compile/dist ./dist
    DO +COPY_METADATA
    ARG release
    RUN --secret GH_TOKEN ./ci/release-artifacts.sh --release "${release}"
