# Roteiro de demonstração - até 20 minutos

Meta: 17 a 19 minutos. Ensaiar antes de gravar. Não mostrar .env, Secrets,
credenciais AWS, tokens nem comandos que imprimam esses valores.
Preparar terminais local e EKS separados, com contexto explícito.

| Tempo | Mostrar e explicar |
|---|---|
| 0:00-1:00 | Nome, Fase 2, objetivo: evolução para cinco microsserviços |
| 1:00-3:00 | Diagrama do README; função dos serviços; RDS, Redis e DynamoDB |
| 3:00-6:00 | docker compose up -d --build, nove contêineres, health checks e integração local |
| 6:00-8:00 | Console EKS, nós, pods, ECR e recursos AWS; Ingress respondendo |
| 8:00-12:00 | Teste Fortio, observar HPA evaluation e réplicas; explicar CPU relativa ao request |
| 12:00-16:00 | Envio manual de eventos de teste; acompanhar KEDA e pods analytics; mostrar dados no DynamoDB |
| 16:00-18:00 | Fila drenada e redução; desafios de IRSA, contexto Kubernetes e capacidade DynamoDB |
| 18:00-19:00 | Repositório, evidências, limitações e encerramento |

## Preparação

1. Confirmar permissões do vídeo e do repositório para o avaliador.
2. Abrir os comandos de carga já testados. Para nova execução, usar nome novo
   de Job; não executar novamente create com o mesmo nome de Job concluído.
3. Manter observação de HPA e pods aberta ANTES de enviar a carga.
4. Analytics pode estar em zero quando ocioso. Durante o teste de fila, mostrar
   seus pods ativos para demonstrar os cinco microsserviços em execução.
5. Separar demonstração local (fila local e DynamoDB Local) da demonstração EKS
   (fila de nuvem e DynamoDB gerenciado).
6. Usar chave de API em variável privada para o teste funcional, sem imprimi-la.
7. Fazer um ensaio e ajustar o ritmo; se o cooldown demorar, explicar a arquitetura
   enquanto aguarda, sem afirmar que a escala terminou antes de observar o resultado.

## Comandos de observação

```powershell
docker compose ps
$eksContext = 'arn:aws:eks:sa-east-1:302795277290:cluster/ToggleMasterEKS'
kubectl --context $eksContext get nodes
kubectl --context $eksContext get pods -n togglemaster
kubectl --context $eksContext get hpa -n togglemaster --watch
```

Em outro terminal, acompanhar o Deployment analytics durante o envio de mensagens:

```powershell
kubectl --context $eksContext get deployment analytics-service -n togglemaster --watch
```

Ctrl+C interrompe apenas a observação. Mostrar a tabela DynamoDB no console
usando um evento do teste atual; não confundir com a tabela local.

## Falas sugeridas

"RDS mantém os dados relacionais. Redis reduz a latência das avaliações.
DynamoDB recebe os eventos de análise, enquanto SQS desacopla esse processamento."

"Evaluation escala por CPU. Para analytics escolhi KEDA porque o tamanho da fila
representa o trabalho pendente mesmo quando o uso de CPU é baixo. Quando não há
trabalho, os pods de analytics podem voltar a zero. Os nós EKS continuam existindo."

"Os resultados apresentados são de testes controlados. O Ingress demonstrado
usa HTTP e os health checks são simples. Idempotência, HTTPS e observabilidade
mais completa são pontos de evolução."
