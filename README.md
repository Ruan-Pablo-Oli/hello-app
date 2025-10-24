# Projeto: CI/CD com GitHub Actions e ArgoCD 🚀✨

---

## 🛠️ Tecnologias Utilizadas 🛠️

![GitHub](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-blue?logo=githubactions)
![Docker](https://img.shields.io/badge/Docker-Containerization-blue?logo=docker)
![FastAPI](https://img.shields.io/badge/FastAPI-Framework-green?logo=fastapi)
![Rancher Desktop](https://img.shields.io/badge/Rancher-Kubernetes-blue?logo=rancher)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-blue?logo=argo)

---

## 🎯 Objetivo 🎯

🌟 Automatizar o ciclo completo de desenvolvimento, build, deploy e execução de uma aplicação FastAPI simples. Utilizamos GitHub Actions para CI/CD, Docker Hub como registry e ArgoCD para entrega contínua (GitOps) em um cluster Kubernetes local gerenciado pelo Rancher Desktop.

---


## 📋 Pré-requisitos 📋

✅ Conta no GitHub (repo público)  
✅ Conta no Docker Hub com token de acesso  
✅ Rancher Desktop com Kubernetes habilitado  
✅ **kubectl** configurado corretamente  
✅ **Git** instalado  
✅ **Python 3** e **Docker** instalados  
✅ ArgoCD instalado no cluster (ver Etapa 3)  

---

## 🚀 Passo a Passo da Implementação 🚀

### 🌟 Etapa 1: Configuração dos Repositórios e Aplicação FastAPI 🌟

1. 📝 Crie dois repositórios públicos no GitHub:
    * **hello-app**: Para o código da aplicação FastAPI, Dockerfile e GitHub Actions.
    * **hello-manifests** (ou **manifest**): Para os manifestos do Kubernetes (**deployment.yaml**, **service.yaml**).

2.  No repositório **hello-app**, crie os seguintes arquivos:

    **main.py**
    ```python
    from fastapi import FastAPI

    app = FastAPI()

    @app.get("/")
    async def root():
        return {"message": "Hello World"}
    ```

    **requirements.txt**
    ```
    fastapi
    uvicorn[standard]
    ```

    **Dockerfile**
    ```dockerfile
    FROM python:3.9-slim
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt
    COPY . .
    EXPOSE 8000
    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```

---

### ☸️ Etapa 2: Criação dos Manifestos Kubernetes ☸️

1.  No repositório **hello-manifests**, crie os arquivos de deploy e service.

    **deployment.yaml**
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hello-app
      template:
        metadata:
          labels:
            app: hello-app
        spec:
          containers:
          - name: hello-app
            # Substitua 'SEU_USUARIO_DOCKERHUB' pelo seu usuário
            image: SEU_USUARIO_DOCKERHUB/hello-app:latest 
            ports:
            - containerPort: 8000
    ```

    **service.yaml**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: hello-app-service
    spec:
      selector:
        app: hello-app
      ports:
        - protocol: TCP
          port: 8080       # Porta que o serviço expõe
          targetPort: 8000 # Porta do contêiner (definida no Dockerfile)
    ```

---

### 🤖 Etapa 3: Instalação e Acesso ao ArgoCD 🤖

1.  Instale o ArgoCD no seu cluster:
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
    ```

2.  Acesse a interface do ArgoCD (deixe este terminal rodando):
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```

3.  Obtenha a senha inicial (username: **admin**):
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```

4.  Acesse a UI em **https://localhost:8080/**.

---

### 🔄 Etapa 4: Configuração do Pipeline CI/CD (GitHub Actions) 🔄

1. **Gere chaves SSH:** Para permitir que o **hello-app** atualize o **hello-manifests**.
    ```bash
    ssh-keygen -t rsa -b 4096 -C "github-actions" -f github-actions-key -N ""
    ```

2. **Configure os Segredos no GitHub**:
    * Vá em **hello-app** > **Settings** > **Secrets** > **Actions**:
        * **DOCKER_USERNAME**: Seu usuário do Docker Hub.
        * **DOCKER_PASSWORD**: Seu token de acesso do Docker Hub.
        * **SSH_PRIVATE_KEY**: O conteúdo do arquivo **github-actions-key** (a chave privada).
    * Vá em **hello-manifests** > **Settings** > **Deploy keys**:
        * Adicione a chave pública (**github-actions-key.pub**) e **marque "Allow write access"**.

3. **Crie o Workflow** no **hello-app**, no arquivo **.github/workflows/cicd.yaml**:
    ```yaml
    name: CI/CD Pipeline

    on:
      push:
        tags:
          - 'v*.*.*'

    jobs:
      build-and-push:
        runs-on: ubuntu-latest
        outputs:
          image_tag: ${{ steps.meta.outputs.version }}
        steps:
        - name: Checkout code
          uses: actions/checkout@v3

        - name: Log in to Docker Hub
          uses: docker/login-action@v2
          with:
            username: ${{ secrets.DOCKER_USERNAME }}
            password: ${{ secrets.DOCKER_PASSWORD }}

        - name: Extract metadata (tags, labels) for Docker
          id: meta
          uses: docker/metadata-action@v4
          with:
            # Substitua 'SEU_USUARIO_DOCKERHUB'
            images: SEU_USUARIO_DOCKERHUB/hello-app
            tags: |
              type=semver,pattern={{version}},enable=true,latest=false

        - name: Build and push Docker image
          uses: docker/build-push-action@v4
          with:
            context: .
            push: true
            tags: ${{ steps.meta.outputs.tags }}
            labels: ${{ steps.meta.outputs.labels }}

      update-manifest:
        needs: build-and-push
        runs-on: ubuntu-latest
        steps:
        - name: Checkout manifests repository
          uses: actions/checkout@v3
          with:
            # Substitua 'SEU_USUARIO_GITHUB/SEU_REPO_MANIFESTS'
            repository: SEU_USUARIO_GITHUB/SEU_REPO_MANIFESTS
            ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}

        - name: Update image tag in deployment.yaml
          run: |
            IMAGE_TAG=${{ needs.build-and-push.outputs.image_tag }}
            # Substitua 'SEU_USUARIO_DOCKERHUB'
            sed -i "s|image:.*|image: SEU_USUARIO_DOCKERHUB/hello-app:${IMAGE_TAG}|" deployment.yaml

        - name: Commit and push changes
          run: |
            git config --global user.name "GitHub Actions"
            git config --global user.email "actions@github.com"
            git add deployment.yaml
            git commit -m "ci: update image tag to ${{ needs.build-and-push.outputs.image_tag }}"
            git push
    ```

---

### 🌟 Etapa 5: Criação do App no ArgoCD 🌟

1.  Na interface do ArgoCD, clique em **NEW APP**.
2.  Preencha os campos:
    * **Application Name**: **hello-app**
    * **Project Name**: **default**
    * **Sync Policy**: **Automatic** (com **Prune Resources** e **Self Heal**)
    * **Repository URL**: O URL **https** do seu repositório **hello-manifests**.
    * **Revision**: **HEAD**
    * **Path**: **.**
    * **Cluster URL**: **https://kubernetes.default.svc**
    * **Namespace**: **default**
3.  Clique em **CREATE**. O ArgoCD irá sincronizar e fazer o deploy da aplicação.

### ✅ Etapa 6: Teste e Validação do Fluxo ✅

1.  Após o ArgoCD sincronizar o app, acesse a aplicação localmente com **port-forward**:
    ```bash
    kubectl port-forward svc/hello-app-service 8080:8080
    ```
2.  Abra **http://localhost:8080/** no seu navegador. Você deve ver **{"message":"Hello World"}**.
3.  **Teste o fluxo:** Altere a mensagem no **main.py** (ex: "Hello CI/CD"), faça **commit** e **push**.
4.  Aguarde o pipeline do GitHub Actions rodar e o ArgoCD sincronizar (pode levar alguns minutos, ou clique em **Sync** no ArgoCD).
5.  Atualize o navegador em **http://localhost:8080/** e veja a mensagem atualizada.

---

## 📸 Evidências (Entregas Esperadas)

Esta seção contém todas as evidências solicitadas para a conclusão do projeto.


### 1. Evidência de Build e Push da imagem no Docker Hub



![Evidência Docker Hub](/assets/docker.png)
![Evidência Docker Hub](/assets/build_image_1.png)
![Evidência Docker Hub](/assets/build_image_2.png)


### 2. Evidência de atualização automática dos manifests


![Evidência Commit Automático](/assets/hello_manifest.png)

### 3. Captura de tela do ArgoCD com a aplicação sincronizada


![Evidência ArgoCD](/assets/argo_cd.png)

### 4. Print do **kubectl get pods** com a aplicação rodando 

![Evidência get pods](/assets/get_pods.png)


### 5. Print do retorno do curl na aplicação

![Retorno curl](/assets/aplicacao.png)

