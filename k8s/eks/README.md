# Manifestos EKS

Reconstruídos a partir das saídas fornecidas do cluster em 01/11/2026.
JSON é sintaxe válida em arquivos YAML. Metadados de execução e status foram removidos.

## Pré-requisitos

EKS, imagens ECR, três RDS com schemas inicializados, Redis, SQS e DynamoDB;
Metrics Server, NGINX (classe nginx) e KEDA 2.21.0 instalados.
Roles IRSA e provedor OIDC devem existir. KEDA usa ToggleMasterKedaRole,
associada à ServiceAccount keda-operator no namespace keda.
Os arquivos contêm identificadores do ambiente demonstrado; adapte-os para outra conta.

## Credenciais

Crie privadamente auth-secret, flag-secret, targeting-secret e evaluation-secret
no namespace togglemaster. Use o modelo de Secrets fora desta pasta.
Evaluation usa uma chave emitida pelo auth deste ambiente, não a MASTER_KEY.
Não publique arquivos preenchidos. Nenhuma Secret real foi exportada.

## Aplicação

Primeiro aplique namespace.yaml. Configure os Secrets, inicialize os schemas
usando os SQLs de cada serviço e emita a chave de evaluation após iniciar auth.
Depois aplique os demais recursos conforme essas dependências.
Para este cluster já configurado, valide antes de aplicar:

```powershell
kubectl --context $eksContext apply --dry-run=server -R -f k8s/eks
```

O comando de validação não persiste alterações. O pacote não foi reaplicado ao cluster.
Não aplique a pasta antiga kubernetes junto com esta pasta.
Não crie um HPA adicional para analytics: KEDA gera o HPA correspondente.
Replicas foi omitido em evaluation e analytics para deixar o controle aos autoscalers.
A migração de gerenciamento de replicas em ambientes existentes deve ser revisada antes de reaplicar.

## Decisões verificadas

- HPA evaluation: CPU 60%, mínimo 1, máximo 3; estabilização de redução 300s.
- KEDA analytics: 0 a 3; alvo 20 mensagens incluindo in-flight; polling 10s; cooldown 120s.
- Role efetiva de analytics: ToggleMasterAnalyticsPolicy (é uma role apesar do nome).
  A anotação last-applied antiga continha ToggleMasterAnalyticsRole e não foi reutilizada.
- Ingress HTTP: /evaluate Exact, porta 8004. Não há TLS configurado neste manifesto.
- Probes /health são simples e não validam a saúde de todas as dependências.
- Imagens usam latest como no ambiente testado; fixação por digest é evolução futura.

ConfigMaps foram recuperados da configuração anteriormente aplicada e ainda precisam
ser comparados com os valores atuais. Não há garantia de reprodução integral da
infraestrutura AWS apenas com estes arquivos Kubernetes.
