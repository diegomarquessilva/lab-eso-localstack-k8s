# lab-eso-localstack-k8s

Nesse caso, vamos adaptar o laboratório para rodar **o LocalStack dentro do Kubernetes**, mantendo tudo **self-contained** no cluster local (Kind, Minikube, etc.). Isso permite testar o **External Secrets Operator (ESO)** com o **Secrets Manager do LocalStack** sem sair do cluster.

---

# 🧪 Lab: Testando o External Secrets Operator com LocalStack (Secrets Manager) e Kubernetes local — sem conta AWS

## ✅ Objetivo

Executar um laboratório que roda **External Secrets Operator (ESO)** e **LocalStack** dentro de um cluster Kubernetes local (Kind ou Minikube), simulando a AWS Secrets Manager **sem usar conta AWS real** e sem nenhum custo.

---

## 📦 Pré-requisitos

* [Docker](https://docs.docker.com/engine/install/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [Helm](https://helm.sh/docs/intro/install/)
* Cluster Kubernetes local ([Kind](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), Minikube, etc.)
* [aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) (opcional)

---

## 🧱 Arquitetura do Lab

```text
[Kubernetes local]
  ├─ LocalStack (Secrets Manager simulado)
  ├─ External Secrets Operator
  └─ Aplicações/testes consumindo Secrets via ESO
```

---

## 🔧 Etapas

### 1. Criar o cluster local (com Kind como exemplo)

```bash
kind create cluster --name eso-lab
```

---

### 2. Subir o LocalStack no Kubernetes

#### `localstack-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: localstack
  labels:
    app: localstack
spec:
  replicas: 1
  selector:
    matchLabels:
      app: localstack
  template:
    metadata:
      labels:
        app: localstack
    spec:
      containers:
        - name: localstack
          image: localstack/localstack:latest
          ports:
            - containerPort: 4566
          env:
            - name: SERVICES
              value: secretsmanager
            - name: DEFAULT_REGION
              value: us-east-1
            - name: AWS_ACCESS_KEY_ID
              value: fake
            - name: AWS_SECRET_ACCESS_KEY
              value: fake
```

#### `localstack-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: localstack
spec:
  selector:
    app: localstack
  ports:
    - protocol: TCP
      port: 4566
      targetPort: 4566
```

```bash
kubectl apply -f localstack-deployment.yaml
kubectl apply -f localstack-service.yaml
```

---

### 3. Instalar o External Secrets Operator via Helm

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```


#### Configure o ESO para usar o LocalStack com AWS_ENDPOINT_URL e a AWS_SECRETSMANAGER_ENDPOINT

Rode isso:
```bash
kubectl -n external-secrets set env deployment/external-secrets AWS_ENDPOINT_URL=http://localstack.default.svc.cluster.local:4566
```
```bash
kubectl -n external-secrets set env deployment/external-secrets AWS_SECRETSMANAGER_ENDPOINT=http://localstack.default.svc.cluster.local:4566
```
##### Isso só funciona se o LocalStack estiver rodando como Service no Kubernetes com o nome localstack no namespace default.

---

### 4. Criar o Secret com as credenciais falsas

```bash
kubectl create secret generic aws-creds \
  --from-literal=access-key=fake \
  --from-literal=secret-access-key=fake \
  --namespace default
```

---

### 5. Criar o SecretStore apontando para o LocalStack

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: localstack-secrets
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-creds
            key: access-key
          secretAccessKeySecretRef:
            name: aws-creds
            key: secret-access-key
```

---

### 6. Criar um Secret no Secrets Manager do LocalStack

Use o pod do LocalStack para executar o comando:

```bash
kubectl exec -it deploy/localstack -- sh
```

Dentro do pod:

```bash
awslocal secretsmanager create-secret \
  --name /dev/app/password \
  --secret-string "meu-segredo"
```

---

### 7. Criar o recurso `ExternalSecret`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-secret
spec:
  refreshInterval: 10s
  secretStoreRef:
    name: localstack-secrets
    kind: SecretStore
  target:
    name: app-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: /dev/app/password
```

---

### 8. Validar resultado

```bash
kubectl get secret app-secret -o yaml
```

Você verá um Secret do tipo `Opaque` com a chave `password` e o valor convertido (base64).

```bash
echo "bWV1LXNlZ3JlZG8=" | base64 -d
```

---

## 🧼 Cleanup

```bash
kind delete cluster --name eso-lab
```

---

## 📝 Conclusão

Com esse lab, você consegue simular o uso do **External Secrets Operator** e **AWS Secrets Manager** com **infraestrutura 100% local**, ideal para testes e pipelines offline. Nenhuma conta AWS foi necessária, e todos os recursos rodaram dentro do cluster Kubernetes.


