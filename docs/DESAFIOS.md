# Desafios, ajustes técnicos e aprendizados

ToggleMaster — Tech Challenge Fase 2  
Victor Leonardo · RM rm375744 · Discord: _vimedina_18014

## Visão geral

A implantação exigiu adaptar a configuração local para serviços gerenciados na AWS, alinhar permissões e verificar o comportamento sob carga. Os casos abaixo descrevem problemas observados e decisões verificáveis nos arquivos e registros do projeto. Não pressupõem que as dificuldades tenham sido introduzidas intencionalmente no enunciado.

## Problemas encontrados e soluções

| Desafio | Diagnóstico e ajuste | Evidência e aprendizado |
|---|---|---|
| Autenticação AWS do Analytics | O pod tentava assumir `ToggleMasterAnalyticsRole`, mas a role existente chamava-se `ToggleMasterAnalyticsPolicy`. A anotação da ServiceAccount foi corrigida e o Deployment reiniciado. | O erro `AccessDenied` em `AssumeRoleWithWebIdentity` deixou de aparecer nos registros apresentados, e o consumidor passou a salvar eventos. O nome de uma role pode terminar em “Policy”; o ARN e a relação de confiança determinam sua identidade. |
| Contexto Kubernetes incorreto | O contexto ativo estava em `docker-desktop`, onde já existia uma instalação de KEDA. No EKS, KEDA ainda não estava instalado. Os comandos passaram a indicar explicitamente o contexto EKS. | A consulta com `--context` confirmou a diferença entre os clusters. Ter o recurso no computador não significa tê-lo na nuvem. |
| Banco acessível, mas sem tabelas | As consultas conectavam ao RDS, porém `to_regclass` não encontrava as tabelas necessárias. Os SQLs foram executados por Jobs de inicialização. | Os Jobs concluíram e confirmaram `api_keys`, `flags` e `targeting_rules`. Conectividade e existência do schema são verificações diferentes. |
| Acúmulo de eventos durante a carga | DynamoDB estava provisionado com 1 unidade de escrita. A tabela foi alterada para `PAY_PER_REQUEST`; após ficar ativa, o backlog foi consumido. | A fila chegou a ter mais de 16 mil mensagens disponíveis e depois apresentou todas as contagens aproximadas em zero. A evolução sugere um gargalo de escrita, mas não houve medição de throttling no CloudWatch para quantificá-lo. |
| Critério de escala do consumidor | Para o Analytics foi configurado KEDA com base no tamanho da fila, mínimo zero e máximo três réplicas. O Evaluation manteve HPA por CPU. | Foi observado Analytics sair de zero, chegar a três réplicas e retornar a zero após a drenagem. CPU do serviço HTTP e trabalho pendente na fila representam necessidades distintas. |
| Mensagens disponíveis em zero, mas processamento ainda ativo | Além das mensagens disponíveis, foram acompanhadas as mensagens em processamento e atrasadas. | Durante os testes, a contagem disponível zerou antes da contagem em processamento. Não se declarou a fila drenada com base em apenas um indicador. |
| Alteração da regra não refletida imediatamente | O Evaluation mantém informações de flag e targeting em cache por 30 segundos. No teste de 0% para 100%, houve espera de 35 segundos. | A avaliação retornou `False` antes e `True` depois. A consistência observada depende também do tempo de cache. |
| Repetição dos testes | Um Job concluído continuava existindo e uma nova criação com o mesmo nome retornava `AlreadyExists`. Nas gravações foram usados nomes novos para as execuções. | Nomes únicos permitiram repetir os testes sem confundir resultados anteriores com a execução atual. |
| DynamoDB Local sem tabela após reinício | O Compose usa `-inMemory`. A tabela precisa ser criada quando essa instância local é reiniciada. | `list-tables` retornou uma lista vazia; a criação de `ToggleMasterAnalytics`, com chave string `event_id`, resolveu a ausência da tabela. |
| Repositórios Git dentro dos serviços | As pastas dos microsserviços continham repositórios próprios e foram inicialmente adicionadas como referências, sem os arquivos completos. Seus históricos foram preservados fora do projeto e os serviços foram adicionados como diretórios comuns. | O commit passou a conter o código e os Dockerfiles. Uma referência a repositório embutido não equivale à inclusão de seus arquivos. |

## Compatibilidade de versões e dependências

### Flask e Werkzeug

Os três serviços Python fixam `Flask==2.2.2` e `Werkzeug==2.2.2`. Isso registra uma combinação específica de dependências para a entrega. O código do Flask 2.2.2 importa `url_quote` de Werkzeug; mudanças posteriores nessa biblioteca podem quebrar aplicações antigas. Portanto, atualizar apenas uma dependência sem verificar sua compatibilidade não é um procedimento seguro de atualização.

A revisão confirma as versões fixadas, mas não recuperou um log de falha original que permita afirmar que esse erro ocorreu durante esta implantação. Esse ponto é uma consideração de compatibilidade, não um incidente comprovado. Manter versões antigas também não substitui uma avaliação de segurança e um plano de atualização conjunta.

Referências: [código do Flask 2.2.2](https://raw.githubusercontent.com/pallets/flask/2.2.2/src/flask/app.py) e [histórico do Werkzeug](https://werkzeug.palletsprojects.com/en/stable/changes/).

### Campo `version` do Compose

O `docker-compose.yml` entregue começa diretamente com `services`. O campo superior `version` é obsoleto no Compose atual e não seleciona a versão do esquema. Ele não é um pacote e sua ausência não é um erro.

Não foi recuperada uma versão anterior desse arquivo que comprove quando o campo foi removido. A observação descreve o formato final, sem atribuir um erro ao material inicial.

Referência: [documentação do Docker Compose](https://docs.docker.com/reference/compose-file/version-and-name/).

### Comando do Helm

No ambiente apresentado, Helm 4.3.1 rejeitou `helm list --all` com `unknown flag`. A consulta foi feita com `helm list --kube-context ... -n keda`. A ocorrência demonstra a necessidade de conferir as opções disponíveis na versão instalada; não foi necessário reinstalar KEDA por causa desse erro de comando.

### Pacotes que não foram considerados dispensáveis

`python-dotenv` e `godotenv` continuam importados e usados pelos serviços para carregar arquivos `.env` no desenvolvimento local. Embora Compose e Kubernetes possam fornecer variáveis diretamente, remover esses pacotes sem ajustar o código pode impedir a inicialização ou compilação. Não foi identificada nesta revisão evidência suficiente para declarar um pacote como desnecessário e removê-lo.

## Diferenças importantes entre o ambiente local e a nuvem

- No ambiente local, PostgreSQL, Redis e DynamoDB Local são contêineres. O SQS continua sendo um serviço AWS, em uma fila dedicada ao teste local.
- No EKS, os dados ficam em RDS, ElastiCache e DynamoDB gerenciado. O Analytics não deve usar o endereço do DynamoDB Local nesse ambiente.
- No ambiente local, os SDKs acessam as credenciais montadas em modo somente leitura. No EKS, os pods usam IRSA e permissões associadas às ServiceAccounts.
- A `MASTER_KEY` autoriza a emissão de uma chave no Auth. Ela não substitui a `SERVICE_API_KEY` emitida e usada pelo Evaluation.

## Limites da entrega e próximos passos

Os testes demonstraram integração funcional, acesso pelo Ingress e escala automática. Eles não provam processamento exatamente uma vez nem correspondência individual entre todos os eventos enviados e gravados. O Analytics gera um novo identificador em cada processamento, portanto a deduplicação é uma evolução necessária.

Também permanecem como evoluções: HTTPS no Ingress, verificações de saúde das dependências, imagens fixadas por versão ou digest e atualização coordenada das dependências. Reduzir o Analytics a zero diminui os pods ativos, mas não elimina os custos dos nós EKS e dos demais serviços AWS.
