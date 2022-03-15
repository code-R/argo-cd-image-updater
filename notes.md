https://gist.github.com/vfarcic/acf300f415b6fc9f699222bfe5b9e14f

# Referenced videos:
# - Argo CD - Applying GitOps Principles To Manage Production Environment In Kubernetes: https://youtu.be/vpWQeoaiRM4
# - kind - How to run local multi-node Kubernetes clusters: https://youtu.be/C0v5gJSWuSo
# - GitHub CLI - How to manage repositories more efficiently: https://youtu.be/BII6ZY2Rnlc
# - Argo Workflows and Pipelines - CI/CD, Machine Learning, and Other Kubernetes Workflows: https://youtu.be/UMaivwrAyTA
# - Running Jenkins In Kubernetes - Tutorial And Review: https://youtu.be/2Kc3fUJANAc
# - Github Actions Review And Tutorial: https://youtu.be/eZcAvTb0rbA
# - Tekton - Kubernetes Cloud-Native CI/CD Pipelines And Workflows: https://youtu.be/7mvrpxz_BfE
# - Environments Based On Pull Requests (PRs): Using Argo CD To Apply GitOps Principles On Previews: https://youtu.be/cpAaI8p4R60
# - How To Apply GitOps To Everything - Combining Argo CD And Crossplane: https://youtu.be/yrj4lmScKHQ
# - Koncrete - GitOps As A Service With Argo CD: https://youtu.be/F2EdxLMQsCw

#################
# Setup Cluster #
#################

# Watch https://youtu.be/BII6ZY2Rnlc if you are not familiar with GitHub CLI
gh repo fork \
    https://github.com/vfarcic/argo-cd-image-updater \
    --clone

cd argo-cd-image-updater

# Feel free to use any other Kubernetes cluster
# If using `kind`, please note that the commands were tested with Docker Desktop using 2 CPUs and 6 GB RAM.
# It might work with fewer resources though.
kind create cluster --mconfig kind.yaml

# NGINX Ingress installation might differ for your k8s provider
kubectl apply \
    --filename https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml

# If not using kind, replace `127.0.0.1` with the base host accessible through NGINX Ingress
export INGRESS_HOST=127.0.0.1

kubectl create namespace production

#################
# Setup Argo CD #
#################

# Replace `[...]` with the GitHub organization or user
export GITHUB_ORG=code-R

# Replace `[...]` with the GitHub user (it might be the same as the value of `GITHUB_ORG`)
export GITHUB_USER=code-R

# Replace `[...]` with the GitHub access token
export GITHUB_TOKEN=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

export REPO_URL=https://github.com/$GITHUB_ORG/argo-cd-image-updater

# Replace `[...]` with your Docker Hub user
export DH_USER=mven

cat argocd-app.yaml \
    | sed -e "s@repoURL: .*@repoURL: $REPO_URL@g" \
    | tee argocd-app.yaml

cat orig/devops-toolkit.yaml \
    | sed -e "s@repoURL: .*@repoURL: $REPO_URL@g" \
    | tee production/devops-toolkit.yaml

cat kustomize/overlays/production/ingress.yaml \
    | sed -e "s@host: .*@host: devops-toolkit.$INGRESS_HOST.nip.io@g" \
    | tee kustomize/overlays/production/ingress.yaml

git add .

git commit -m "Customizations"

git push

docker image build \
    --tag $DH_USER/devops-toolkit \
    .

docker image push \
    $DH_USER/devops-toolkit

docker image tag \
    $DH_USER/devops-toolkit \
    $DH_USER/devops-toolkit:99999999

docker image push \
    $DH_USER/devops-toolkit:99999999

helm repo add argo \
    https://argoproj.github.io/argo-helm

helm repo update

helm upgrade --install \
    argocd argo/argo-cd \
    --namespace argocd \
    --create-namespace \
    --set server.ingress.hosts="{argo-cd.$INGRESS_HOST.nip.io}" \
    --set server.ingress.enabled=true \
    --set server.extraArgs="{--insecure}" \
    --set controller.args.appResyncPeriod=30 \
    --wait

kubectl apply --filename argocd-app.yaml

export PASS=$(kubectl \
    --namespace argocd \
    get secret argocd-initial-admin-secret \
    --output jsonpath="{.data.password}" \
    | base64 --decode)

argocd login \
    --insecure \
    --username admin \
    --password $PASS \
    --grpc-web \
    argo-cd.$INGRESS_HOST.nip.io

argocd account update-password \
    --current-password $PASS \
    --new-password admin123

echo http://argo-cd.$INGRESS_HOST.nip.io

# Open it in a browser

# Use `admin` as the username and `admin123` as the password

##########################
# Using Pipelines For CD #
##########################

# Open https://github.com/vfarcic/argo-combined-demo/blob/master/argo-workflows/base/templates.yaml

####################################
# Installing Argo CD Image Updater #
####################################

# Show Argo CD UI

# It can be installed in the same cluster/Namespace or in a different cluster

kubectl --namespace argocd apply \
    --filename https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

kubectl --namespace argocd logs \
    --selector app.kubernetes.io/name=argocd-image-updater

####################################
# Configuring ArgoCD Image Updater #
####################################

# Add the following `metadata.annotations` to `production/devops-toolkit.yaml`:
# `argocd-image-updater.argoproj.io/image-list: vfarcic/devops-toolkit`
# `argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds`
# `argocd-image-updater.argoproj.io/git-branch: main`

kubectl --namespace argocd \
    create secret generic git-creds \
    --from-literal=username=$GITHUB_USER \
    --from-literal=password=$GITHUB_TOKEN

git add .

git commit -m "Image updater"

git push

kubectl --namespace argocd logs \
    --selector app.kubernetes.io/name=argocd-image-updater \
    --follow

##################################
# Updating To The Latest Release #
##################################

gh repo view --web

kubectl --namespace production \
    describe deployment devops-toolkit

# Update strategies: `semver`, `latest`, `digest`, and `name`

##########################
# Defining Version Range #
##########################

# Update the `image-list` annotation in `production/devops-toolkit.yaml` to the following:
# `argocd-image-updater.argoproj.io/image-list: vfarcic/devops-toolkit:~1`

# Open https://github.com/Masterminds/semver

git pull

git add .

git commit -m "Only version 1"

git push

docker image build \
    --tag $DH_USER/devops-toolkit:1.2.3 \
    .

docker image push \
    $DH_USER/devops-toolkit:1.2.3

kubectl --namespace argocd logs \
    --selector app.kubernetes.io/name=argocd-image-updater \
    --follow

kubectl --namespace production \
    describe deployment devops-toolkit

###################################
# Going Outside The Version Range #
###################################

docker image build \
    --tag $DH_USER/devops-toolkit:2.0.0 \
    .

docker image push \
    $DH_USER/devops-toolkit:2.0.0

kubectl --namespace argocd logs \
    --selector app.kubernetes.io/name=argocd-image-updater \
    --follow

gh repo view --web

###########
# Destroy #
###########

kind delete cluster

rm kustomize/overlays/production/.argocd-source-devops-toolkit.yaml

git add .

git commit -m "Cleanup"

git push