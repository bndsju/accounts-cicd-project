# Accounts Service — CI/CD Pipeline Project

> **Nome do Projeto:** Accounts Service CI/CD Pipeline
> Projeto final do módulo de CI/CD: pipeline completo que faz lint,
> testes unitários, build de imagem e deploy automatizado, combinando
> GitHub Actions (CI) com Tekton/OpenShift Pipelines (CD).

## Estrutura
```
.github/workflows/workflow.yml   -> CI no GitHub Actions (lint + testes)
.tekton/tasks.yml                -> Tasks Tekton: lint (flake8) + testes (nose)
.tekton/pipeline.yml             -> Pipeline: clone -> lint/test -> build -> deploy
.tekton/pipelinerun.yml          -> Dispara uma execução da pipeline
.tekton/pvc.yaml                 -> Workspace persistente usado na PipelineRun
.tekton/deployment.yaml          -> Deployment base da aplicação no OpenShift
.tekton/service.yaml             -> Service da aplicação
.tekton/route.yaml               -> Route (exposição externa) da aplicação
```

## 1. GitHub Actions
Basta o arquivo `.github/workflows/workflow.yml` estar na raiz do repositório
(dentro de `.github/workflows/`). A cada push/PR na `main`, ele roda:
1. `flake8` (lint)
2. `nosetests` (testes unitários), com um serviço Postgres disponível.

Ajuste `requirements.txt`, o nome do pacote (`service`) e o comando de teste
(`nosetests`/`pytest`) conforme sua stack real.

## 2. Aplicar as Tasks e a Pipeline no OpenShift
Pré-requisito: o **OpenShift Pipelines Operator** instalado no cluster (ele
já traz as ClusterTasks `git-clone`, `buildah` e `openshift-client`).

```bash
# login no cluster do lab
oc login <api-url> -u <usuario> -p <senha>

# criar/selecionar o namespace/projeto
oc project <seu-namespace>

# aplicar as Tasks customizadas (lint + testes, no mesmo arquivo)
oc apply -f .tekton/tasks.yml

# aplicar o PVC do workspace
oc apply -f .tekton/pvc.yaml

# aplicar a Pipeline
oc apply -f .tekton/pipeline.yml
```

## 3. Deploy inicial da aplicação (para o passo `deploy` ter o que atualizar)
Aplique o Deployment/Service base uma vez, manualmente:

```bash
oc apply -f .tekton/deployment.yaml
oc apply -f .tekton/service.yaml
oc apply -f .tekton/route.yaml
```

## 4. Disparar a Pipeline
Edite `.tekton/pipelinerun.yml` com a URL do seu repositório e rode:

```bash
oc create -f .tekton/pipelinerun.yml
```

Ou via CLI `tkn` (mais prático para passar parâmetros):

```bash
tkn pipeline start cd-pipeline \
  -w name=pipeline-workspace,claimName=pipelinerun-pvc \
  -p repo-url="https://github.com/<seu-usuario>/<seu-repo>.git" \
  -p branch="main" \
  -p build-image="image-registry.openshift-image-registry.svc:5000/$(oc project -q)/accounts:latest" \
  --showlog
```

## 5. Acompanhar
```bash
tkn pipelinerun list
tkn pipelinerun logs --last -f
```

Se tudo passar (lint -> testes -> build -> deploy), o pod da aplicação será
atualizado com a nova imagem e o `oc rollout status` confirma o sucesso do
deploy.
