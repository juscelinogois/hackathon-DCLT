# Manifestos Kubernetes — SolidaryTech

## Ordem de aplicação

### 0. Pré-requisito: Metrics Server
O EKS não vem com o Metrics Server instalado por padrão, e sem ele o HPA
(HorizontalPodAutoscaler) não consegue ler métricas de CPU dos pods —
ele fica criado, mas nunca escala. Instale antes de tudo:

```powershell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Confirme que subiu (pode levar ~1 min):
```powershell
kubectl get deployment metrics-server -n kube-system
```

### 1. Namespace
```powershell
kubectl apply -f 00-namespace.yaml
```

### 2. Secrets (NÃO use o arquivo .example.yaml diretamente)
Crie os Secrets via linha de comando, para a senha nunca ficar salva
em nenhum arquivo rastreado pelo Git:

```powershell
kubectl create secret generic ngo-db-secret `
  --namespace solidarytech `
  --from-literal=DATABASE_URL="postgres://solidarytech_admin:SUA_SENHA@solidarytech-postgres.cqae8ivsbl2h.us-east-1.rds.amazonaws.com:5432/ngo_db"

kubectl create secret generic donation-db-secret `
  --namespace solidarytech `
  --from-literal=DATABASE_URL="postgres://solidarytech_admin:SUA_SENHA@solidarytech-postgres.cqae8ivsbl2h.us-east-1.rds.amazonaws.com:5432/donation_db"
```

(troque `SUA_SENHA` pela senha real do RDS - o crase `` ` `` no fim da
linha é o "quebra-linha" do PowerShell, equivalente ao `\` do bash)

### 3. ConfigMap
```powershell
kubectl apply -f 02-configmap.yaml
```

### 4. Deployments + Services + HPA
```powershell
kubectl apply -f 03-ngo-service.yaml
kubectl apply -f 04-donation-service.yaml
kubectl apply -f 05-volunteer-service.yaml
```

## Verificando

```powershell
kubectl get pods -n solidarytech
kubectl get svc -n solidarytech
kubectl get hpa -n solidarytech
```

Os pods devem ficar `Running` com `READY 1/1`. Se ficarem em
`CrashLoopBackOff`, o motivo mais provável é a `DATABASE_URL` errada
no Secret (senha ou endpoint) — confira com:

```powershell
kubectl logs -n solidarytech deployment/ngo-service
```

## Por que o volunteer-service e o donation-service não têm
## AWS_ACCESS_KEY_ID no manifesto
Os nodes do EKS rodam com a `LabRole` (definida no Terraform), então
os pods herdam essas permissões automaticamente através do IMDS da
instância EC2 - o AWS SDK (boto3 no Python, aws-sdk-go no Go) já
resolve isso sozinho, sem precisar de credenciais explícitas. Isso
substitui o IRSA, que não pudemos usar por causa do bloqueio de IAM
do AWS Academy (ver histórico do `eks.tf`).

## Testando um serviço rapidamente (sem Ingress/LoadBalancer ainda)
```powershell
kubectl port-forward -n solidarytech svc/ngo-service 8081:80
```
E em outro terminal:
```powershell
curl http://localhost:8081/health
```

## Próximos passos
- Ingress (ou API Gateway) para expor os serviços publicamente
- Pipeline CI/CD para automatizar o build + push + apply
- ArgoCD para GitOps (aplicar estes manifestos automaticamente a
  partir do Git, em vez de `kubectl apply` manual)
