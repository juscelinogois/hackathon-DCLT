# Pipeline CI/CD — SolidaryTech

## O que este pipeline faz
Para cada serviço que mudou no push/PR (`ngo-service`, `donation-service`,
`volunteer-service`):

1. Roda testes automatizados (se existirem)
2. **Trivy - scan de dependências (SCA)**: verifica vulnerabilidades no
   `requirements.txt` / `go.mod` *antes* de buildar a imagem
3. Builda a imagem Docker
4. **Trivy - scan da imagem**: verifica vulnerabilidades na imagem final
   (SO base + dependências + camadas)
5. Publica os dois resultados na aba **Security > Code scanning** do
   repositório no GitHub (formato SARIF)
6. Faz push da imagem para o ECR, taggeada com o SHA do commit (e uma
   tag curta de 7 caracteres)

**O que este pipeline NÃO faz**: deploy no cluster. Isso fica pro
ArgoCD (GitOps), que é a próxima etapa do desafio.

## Configuração necessária no repositório GitHub

### 1. Secrets (Settings → Secrets and variables → Actions → New repository secret)

| Nome                    | Valor                                              |
|-------------------------|-----------------------------------------------------|
| `AWS_ACCESS_KEY_ID`     | Da sessão atual do AWS Academy ("AWS Details")      |
| `AWS_SECRET_ACCESS_KEY` | Da sessão atual do AWS Academy                      |
| `AWS_SESSION_TOKEN`     | Da sessão atual do AWS Academy                      |

### ⚠️ Atenção: credenciais temporárias do AWS Academy
O Learner Lab não permite criar um usuário IAM fixo nem usar OIDC
federation (GitHub → AWS sem senha, que é o jeito recomendado hoje em
dia) - já vimos essa mesma restrição de IAM ao configurar o EKS. Isso
quer dizer que os 3 Secrets acima usam as credenciais **temporárias**
da sessão do Lab, que expiram a cada ~4h.

**Na prática**: sempre que for rodar o pipeline (ou gravar a demo em
vídeo), confirme que a sessão do Lab está ativa e, se necessário,
atualize os 3 Secrets com as credenciais mais recentes antes de dar
push. Se o pipeline falhar no step "Configurar credenciais AWS" com
erro de autenticação/token expirado, é exatamente isso.

Vale mencionar essa limitação no relatório - é uma restrição real do
ambiente educacional, não uma falha de design do pipeline.

## Onde colocar este arquivo
Copie a pasta `.github/workflows/ci.yml` inteira para a raiz do
repositório `hackathon-DCLT` (mantendo a estrutura de pastas
`.github/workflows/`).

## Testando localmente antes de confiar no CI
Para não gastar minutos do GitHub Actions testando, você pode rodar o
Trivy localmente (se instalar o binário) apontando pra pasta de um
serviço:
```powershell
trivy fs ngo-service/
```
