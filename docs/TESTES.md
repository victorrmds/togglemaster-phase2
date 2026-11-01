# Evidências de validação

Fonte: saídas de PowerShell e logs fornecidos durante a execução do projeto.
Este registro não representa auditoria independente nem execução de testes automatizados.

## Local - 01/11/2026

- Nove contêineres ativos: cinco aplicações, dois PostgreSQL, Redis e DynamoDB Local.
- Cinco endpoints /health responderam status ok.
- Criação autenticada da flag local-test-93e2608b e regra de 100%.
- Avaliação retornou True para local-integration-user.
- Evaluation publicou no SQS dedicado ao ambiente local.
- Analytics consumiu e salvou o evento ff73552d-d373-4d3c-a5b3-344d2bbc044c.
- Consulta ao DynamoDB Local confirmou esse event_id, flag, usuário e resultado True.

## EKS e Ingress

- Cinco microsserviços implantados, imagens ECR e serviços ClusterIP.
- Regra de 0% retornou False; alteração para 100% retornou True após 35s de espera.
- Ingress /evaluate retornou True pelo hostname do Load Balancer.
- RDS inicializados: api_keys, flags e targeting_rules.

## Carga de evaluation - 29/10/2026

Fortio: 100 req/s por 180s, oito conexões, timeout 3s, Service interno.
18.000 requisições medidas e oito warm-ups; 18.000 respostas HTTP 200.
Latência média 3,254ms; mediana aproximada 2,887ms; p99 aproximado 5,985ms.
HPA CPU 60%: ciclo 1 -> 2 -> 3 -> 1. Job Complete em 3m6s.

Limites: flag repetida e cache favorável; conexões persistentes; não demonstra
distribuição uniforme entre réplicas nem teste de carga do Ingress. HTTP 200 não
comprova entrega assíncrona individual de cada evento.

DynamoDB estava provisionado com 1 WCU. Após mudança para PAY_PER_REQUEST,
o backlog diminuiu e todas as contagens aproximadas de SQS chegaram a zero.
Isso sustenta capacidade de escrita como gargalo, sem medição de throttling CloudWatch.

## KEDA - 01/11/2026

Job analytics-keda-test-tmjsg enviou 1.000 mensagens com delay de 30s.
Identificador do teste: keda-test-5c099db1.
Ciclo observado: 0 -> 1 -> 3 -> 1 -> 0 réplicas de analytics.
Os três pods registraram gravações no DynamoDB. A busca nos logs disponíveis
não encontrou ERROR, WARNING, AccessDenied, Throttl, ReceiptHandle ou Traceback.
Contagens finais de mensagens visíveis, in-flight e atrasadas: todas zero.

Isso valida ativação, expansão e redução automática. Não comprova exatamente
1.000 registros únicos: as contagens SQS são aproximadas e não houve reconciliação
por evento. O consumidor não garante processamento exatamente uma vez.

## Manifestos - 01/11/2026

Server dry-run aceitou todos os recursos de k8s/eks. ConfigMaps: unchanged.
Os recursos com configured no dry-run não foram alterados no cluster.
Replicas foi omitido dos Deployments controlados por autoscalers. O ARN de analytics
foi corrigido com base na anotação efetiva, não na anotação last-applied antiga.
