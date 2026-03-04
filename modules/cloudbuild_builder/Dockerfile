# Copyright 2021 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM ubuntu:24.04

# Use ARG so that values can be overriden by user/cloudbuild
ARG TERRAFORM_VERSION=1.14.0

ENV ENV_TERRAFORM_VERSION=$TERRAFORM_VERSION

RUN apt-get update -y && apt-get install wget curl jq unzip git ca-certificates software-properties-common apt-transport-https gnupg python3 -y
RUN echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
RUN apt-get update -y && apt-get install google-cloud-cli google-cloud-cli-terraform-tools -y
RUN curl -O https://releases.hashicorp.com/terraform/${ENV_TERRAFORM_VERSION}/terraform_${ENV_TERRAFORM_VERSION}_linux_amd64.zip
RUN mkdir -p /builder/terraform
RUN unzip terraform_${ENV_TERRAFORM_VERSION}_linux_amd64.zip -d /builder/terraform
RUN rm -f terraform_${ENV_TERRAFORM_VERSION}_linux_amd64.zip

ENV PATH=/builder/terraform:$PATH
ENTRYPOINT ["/bin/bash"]
