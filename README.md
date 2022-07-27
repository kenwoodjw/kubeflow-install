最近又开始折腾kubeflow，发现以前用的kfctl 安装方式，官网github已经两年没更新，官方也推出了新的安装方式，但有些镜像是国外的，所以需要解决国外谷歌镜像拉取问题

#### 官方安装脚本仓库
```
https://github.com/kubeflow/manifests
```
#### 安装前准备
```
# 需要安装kustomize，kubectl，注意k8s版本兼容,
https://github.com/kubeflow/manifests#prerequisites
# StorageClass 可使用
https://github.com/rancher/local-path-provisioner
```
#### 镜像同步至dockerhub方式
```
### 获取gcr镜像，因为我的网络只无法获取gcr.io, quay.io正常，可以根据需求修改
kustomize build example |grep 'image: gcr.io'|awk '$2 != "" { print $2}' |sort -u 
###   使用github-ci同步至个人dockerhub仓库
https://github.com/kenwoodjw/sync_gcr
修改https://github.com/kenwoodjw/sync_gcr/blob/master/images.txt 提交会触发ci同步镜像至dockerhub
可根据需求修改https://github.com/kenwoodjw/sync_gcr/blob/master/sync_image.py
```
#### 修改安装脚本拉取镜像
```
#https://github.com/kubeflow/manifests/blob/master/example/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
# Cert-Manager
- ../common/cert-manager/cert-manager/base
- ../common/cert-manager/kubeflow-issuer/base
# Istio
- ../common/istio-1-14/istio-crds/base
- ../common/istio-1-14/istio-namespace/base
- ../common/istio-1-14/istio-install/base
# OIDC Authservice
- ../common/oidc-authservice/base
# Dex
- ../common/dex/overlays/istio
# KNative
- ../common/knative/knative-serving/overlays/gateways
- ../common/knative/knative-eventing/base
- ../common/istio-1-14/cluster-local-gateway/base
# Kubeflow namespace
- ../common/kubeflow-namespace/base
# Kubeflow Roles
- ../common/kubeflow-roles/base
# Kubeflow Istio Resources
- ../common/istio-1-14/kubeflow-istio-resources/base


# Kubeflow Pipelines
- ../apps/pipeline/upstream/env/cert-manager/platform-agnostic-multi-user
# Katib
- ../apps/katib/upstream/installs/katib-with-kubeflow
# Central Dashboard
- ../apps/centraldashboard/upstream/overlays/kserve
# Admission Webhook
- ../apps/admission-webhook/upstream/overlays/cert-manager
# Jupyter Web App
- ../apps/jupyter/jupyter-web-app/upstream/overlays/istio
# Notebook Controller
- ../apps/jupyter/notebook-controller/upstream/overlays/kubeflow
# Profiles + KFAM
- ../apps/profiles/upstream/overlays/kubeflow
# Volumes Web App
- ../apps/volumes-web-app/upstream/overlays/istio
# Tensorboards Controller
-  ../apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow
# Tensorboard Web App
-  ../apps/tensorboard/tensorboards-web-app/upstream/overlays/istio
# Training Operator
- ../apps/training-operator/upstream/overlays/kubeflow
# User namespace
- ../common/user-namespace/base

# KServe
- ../contrib/kserve/kserve
- ../contrib/kserve/models-web-app/overlays/kubeflow
images:
 - name: gcr.io/arrikto/istio/pilot:1.14.1-1-g19df463bb
   newName: kenwood/pilot
   newTag: "1.14.1-1-g19df463bb"
 - name: gcr.io/arrikto/kubeflow/oidc-authservice:28c59ef
   newName: kenwood/oidc-authservice
   newTag: "28c59ef"
 - name: gcr.io/knative-releases/knative.dev/eventing/cmd/controller@sha256:dc0ac2d8f235edb04ec1290721f389d2bc719ab8b6222ee86f17af8d7d2a160f
   newName: kenwood/cmd/controller
   newTag: "dc0ac2"
 - name: gcr.io/knative-releases/knative.dev/eventing/cmd/mtping@sha256:632d9d710d070efed2563f6125a87993e825e8e36562ec3da0366e2a897406c0
   newName: kenwood/cmd/mtping
   newTag: "632d9d"
 - name: gcr.io/knative-releases/knative.dev/serving/cmd/domain-mapping-webhook@sha256:847bb97e38440c71cb4bcc3e430743e18b328ad1e168b6fca35b10353b9a2c22
   newName: kenwood/domain-mapping-webhook
   newTag: "847bb9"
 - name: gcr.io/knative-releases/knative.dev/eventing/cmd/webhook@sha256:b7faf7d253bd256dbe08f1cac084469128989cf39abbe256ecb4e1d4eb085a31
   newName: kenwood/webhook
   newTag: "b7faf7"
 - name: gcr.io/knative-releases/knative.dev/net-istio/cmd/controller@sha256:f253b82941c2220181cee80d7488fe1cefce9d49ab30bdb54bcb8c76515f7a26
   newName: kenwood/controller
   newTag: "f253b8"
 - name: gcr.io/knative-releases/knative.dev/net-istio/cmd/webhook@sha256:a705c1ea8e9e556f860314fe055082fbe3cde6a924c29291955f98d979f8185e
   newName: kenwood/webhook
   newTag: "a705c1"
 - name: gcr.io/knative-releases/knative.dev/serving/cmd/activator@sha256:93ff6e69357785ff97806945b284cbd1d37e50402b876a320645be8877c0d7b7
   newName: kenwood/activator
   newTag: "93ff6e"
   .....
```
#### 开始安装
```
while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done

```

