# Credenciais e publicação

Os arquivos `.env` e os manifestos reais de Secret são privados. Os arquivos
`.env.example` e `kubernetes/secrets.example.yaml` contêm somente exemplos.
Os exemplos não são credenciais válidas e não devem ser aplicados diretamente.

## Ambiente local

Em uma instalação nova, copie cada `.env.example` para `.env`, preencha a URL
da fila local e gere uma MASTER_KEY privada. Não sobrescreva arquivos `.env`
existentes. Inicie auth e emita uma chave pela rota `/admin/keys`; use essa chave
como SERVICE_API_KEY de evaluation. Não use a MASTER_KEY como SERVICE_API_KEY.
Recrie o contêiner evaluation após alterar seu `.env`.

O Compose monta o diretório de credenciais AWS do usuário em modo somente
leitura. As credenciais ficam fora do projeto. Os valores `password` no Compose
são padrões do ambiente de demonstração local, não senhas RDS de produção.
A fila local deve ser diferente da fila EKS. Esta configuração requer acesso AWS.

## EKS

Use Kubernetes Secrets para DATABASE_URL, MASTER_KEY e SERVICE_API_KEY.
ConfigMaps recebem parâmetros não sensíveis. Roles IRSA fornecem acesso AWS
a evaluation, analytics e KEDA; não coloque access keys permanentes nesses pods.
O campo stringData do exemplo é convertido pelo Kubernetes para data em base64.
Essa codificação não equivale a criptografia.

## Antes de publicar

- Não envie a pasta inteira por upload manual: `.gitignore` só controla o Git.
- Revise `git status --short` e os arquivos preparados para commit.
- Não publique `.env`, manifests preenchidos, dumps, logs com tokens ou `.aws`.
- Verifique o histórico do repositório de destino. Um `.gitignore` novo não
  remove segredos previamente commitados. Se houve exposição, revogue/rotacione
  as credenciais afetadas e trate o histórico.
- Os manifestos antigos da pasta kubernetes ainda precisam ser separados da
  configuração EKS atual antes de serem usados como procedimento de implantação.

Auditoria local em 01/11/2026: não foi encontrado `.git` na pasta do projeto.
Portanto, o histórico remoto ainda não foi auditado. Os arquivos privados
existentes foram preservados para manter o ambiente de teste funcional.
