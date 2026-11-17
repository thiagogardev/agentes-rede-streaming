# StreamTech Solutions: sistema multiagente para previsão de demanda e alocação dinâmica de rede

Projeto Prático Final da Pós-Graduação em Sistemas Cognitivos com Agentes de IA, Data Science Academy.

Autor: Thiago Garofalo. Documento complementar com a descrição completa da solução que fundamenta as respostas do projeto.

## Resumo

- **Problema:** em picos de demanda (eventos ao vivo, lançamentos) a rede congestiona e o assinante vê buffering. Hoje a empresa reage depois.
- **Solução:** um sistema de seis agentes de IA, coordenados por um supervisor, que prevê o pico, prepara a rede antes dele e ajusta a entrega durante ele.
- **Ideia central:** cada decisão no relógio certo. Modelos preditivos e otimização decidem em segundos; LLM cuida de linguagem, contexto, coordenação e explicação, fora do caminho crítico.
- **Conhecimento:** os agentes consultam uma base própria da empresa (Agentic RAG sobre relatórios e manuais, GraphRAG sobre a topologia da rede).
- **Modelos de linguagem:** um modelo pequeno e aberto (SLM), com ajuste fino, classifica o feedback dentro da empresa; um LLM mais forte orquestra e explica.
- **Infraestrutura:** reaproveita NetFlow/sFlow, Spark, BigQuery, Splunk e a nuvem híbrida. Nada é substituído. Databricks entra como plataforma do Spark para pipelines, publicação e operação dos agentes.
- **Segurança:** guardrails, human-in-the-loop, autonomia progressiva, reversão automática e retorno às regras atuais se algo falhar.
- **Entrega:** agentes publicados como serviços (API), painel para a operação e relatório pós-evento.

## 1. Entendimento do problema

"Tudo começa com o objetivo, não com os dados."

### Palavras-chave do enunciado

Prever picos; alocar recursos dinamicamente; tempo real; regiões e tipos de conteúdo; ajuste automático da entrega; condições da rede; feedback do usuário; eventos ao vivo e lançamentos.

### Perguntas de negócio (reduzidas a duas)

1. **Quanta demanda de rede teremos, onde e quando?** (pergunta sobre o futuro: exige modelo preditivo)
2. **Dada essa demanda e o estado atual da rede, qual é a melhor ação agora para manter o vídeo fluido ao menor custo?** (pergunta de decisão: exige agentes)

O enunciado lista três capacidades (prever, alocar, ajustar a entrega). Alocar e ajustar são duas formas de responder à pergunta 2, por isso ficam no mesmo projeto.

### Precisamos mesmo de Machine Learning? E de agentes?

| Parte do problema | Resolve com regra ou agregação? | Decisão |
|---|---|---|
| Saber o estado atual da rede | Sim, agregação em fluxo | Sem ML |
| Prever demanda futura | Não. Depende de sazonalidade, eventos e campanhas | Machine Learning (regressão em série temporal) |
| Decidir alocação | Parcialmente. Há restrições e custo a equilibrar | Otimização com restrições |
| Interpretar comentários de usuários | Não. Texto livre e ambíguo | LLM |
| Coordenar tudo, em ciclo contínuo, com autonomia | Não. Um modelo isolado só prevê, não age | Agentes de IA |

Agentes se justificam porque o problema pede um ciclo contínuo de perceber, decidir e agir, com várias especialidades e relógios diferentes. Um modelo sozinho entregaria apenas uma previsão em um painel.

### Métricas de sucesso

O enunciado não traz metas. Proposta, a validar com a área de negócio:

| Camada | Métrica | Por quê |
|---|---|---|
| Experiência | Taxa de rebuffering (tempo travado / tempo assistido) | É o sintoma do cenário |
| Experiência | Tempo até o primeiro quadro, falhas de início, bitrate médio | Completam o que o assinante sente |
| Rede | Utilização por enlace e ponto de presença, latência, perda de pacotes, acerto de cache | Causas que os agentes conseguem mover |
| Previsão | WAPE (erro percentual ponderado), perda quantílica, recall de picos | Avalia o modelo isoladamente |
| Negócio | Cancelamentos após incidentes, nota das avaliações, custo por GB entregue | Liga a solução ao resultado da empresa |

### Forma de entrega (definida no início, não no fim)

1. **Deploy via API:** os agentes rodam como serviços em produção.
2. **Painel (dashboard)** para a equipe de operação: previsão, estado da rede, ações tomadas e por quê.
3. **Relatório pós-evento** gerado automaticamente, em linguagem clara, para gestores.

## 2. Lacunas do enunciado e premissas adotadas

O enunciado avisa que faltam informações de propósito. Boa prática: se o problema não pode ser respondido com os dados disponíveis, ou adquirimos novos dados ou mudamos a definição do problema. Estas perguntas iriam para a empresa; cada uma tem premissa explícita para o projeto não travar.

| # | O que falta | Premissa adotada | Se a premissa cair |
|---|---|---|---|
| 1 | **Calendário de conteúdo e marketing.** Os picos vêm de eventos e lançamentos, mas essa fonte não está entre os dados | Adquirir como quinta fonte (grade de eventos, datas de lançamento, campanhas) | Só dá para prever sazonalidade; evento vira surpresa |
| 2 | **Telemetria do player** (rebuffering e bitrate por sessão). Feedback de usuário é subjetivo, esparso e atrasado | Adquirir como sexta fonte. Até lá, o feedback valida e a qualidade é inferida da rede | Ajuste automático mais lento e menos preciso |
| 3 | **Quais ações a rede aceita por API** | A nuvem híbrida permite escalar e redirecionar tráfego por API | O sistema só recomenda, não executa |
| 4 | **Metas (SLOs)** | Métricas da seção 1, calibradas pela linha de base atual | Sem alvo, não há como provar ganho |
| 5 | **Volume, granularidade e profundidade do histórico** | 12 a 24 meses no BigQuery, por região, por hora ou menos | Histórico curto pede modelos mais simples |
| 6 | **Formato do feedback** | Nota mais texto livre | Se for só nota, dispensa LLM nesse agente |
| 7 | **Teto de custo de nuvem** | Existe; custo é restrição, não só qualidade | Sem teto, a resposta trivial é superdimensionar |
| 8 | **Autonomia aceita pela operação** | Progressiva, começando com humano aprovando | Muda o cronograma |
| 9 | **Restrições legais** | LGPD sobre feedback e dispositivos; priorização só dentro do próprio serviço | Mais anonimização |
| 10 | **Documentação de processos e incidentes passados** | Existe parcialmente; o restante se levanta conversando com a operação | A base de conhecimento começa menor |

## 3. Os dados

| Fonte | Tipo de variável | Chegada | Problemas de qualidade esperados | Uso |
|---|---|---|---|---|
| Tráfego em tempo real (NetFlow/sFlow) | Quantitativa, série temporal | Tempo real (fluxo) | sFlow é amostrado; coletores podem cair e gerar lacunas | Estado atual, detecção de anomalia, previsão de curto prazo |
| Histórico de uso (BigQuery) | Quantitativa, série temporal | Lote | Mudanças de medição ao longo do tempo; eventos raros (poucos exemplos de pico) | Treino da previsão, eventos análogos |
| Feedback do usuário | Qualitativa (texto) e quantitativa (nota) | Quase tempo real, com atraso | Viés (quem reclama é quem teve problema), volume baixo, texto ruidoso | Sinal de qualidade percebida, validação |
| Dispositivos | Qualitativa (tipo) e quantitativa (resolução, consumo) | Lote | Categorias inconsistentes entre fontes; precisa padronizar | Política de entrega por classe de dispositivo |
| Calendário de eventos (a adquirir) | Qualitativa e datas | Lote | Mudanças de última hora | Variável explicativa principal dos picos |
| Telemetria do player (a adquirir) | Quantitativa | Tempo real | Volume alto; agregar antes de guardar | Medida direta da experiência |

Tratamento previsto: lacunas curtas de tráfego são interpoladas e marcadas; lacunas longas ficam fora do treino. Antes de tratar um valor ausente, entende-se por que ele ocorreu (coletor fora do ar é diferente de tráfego zero). Fontes diferentes são padronizadas para as mesmas chaves de região e dispositivo. Toda alteração é documentada.

## 4. Visão geral da solução

### A linha de produção

```
Coleta -> Limpeza e agregação -> Previsão -> Decisão -> Ação na rede -> Medição do resultado -> Aprendizado
(NetFlow)      (Spark)          (Agente 2)  (Ag. 4,5,6)   (APIs)        (Agentes 1 e 3)       (BigQuery)
```

### Cada decisão no relógio certo

Um LLM leva segundos para responder, custa por chamada e não dá sempre a mesma resposta. Uma rede de streaming decide em milissegundos. Por isso:

| Camada | Relógio | Quem decide | Exemplos |
|---|---|---|---|
| **Plano de dados** | milissegundos a segundos | Mecanismos determinísticos já existentes, sem agentes | Streaming adaptativo no player, balanceadores |
| **Controle tático** | segundos a minutos | Agentes com modelos preditivos e otimização, dentro de limites | Escalar a borda de uma região, redirecionar tráfego, ajustar teto de qualidade |
| **Controle estratégico** | minutos a dias | Agentes com LLM | Planejar um evento, interpretar feedback, resolver conflitos, explicar decisões |

O desenho segue o ciclo Monitorar, Analisar, Planejar, Executar sobre uma base de Conhecimento compartilhada (MAPE-K), referência clássica de sistemas que se autogerenciam.

## 5. Os agentes

Seis agentes, uma responsabilidade cada. Só existe agente onde há uma decisão distinta a tomar; menos agentes significa menos falha e menos coordenação.

```
            Calendário de eventos        Feedback dos usuários
                     |                            |
                     v                            v
 NetFlow/sFlow -> [1 Monitoramento] -> [2 Previsão] <- [3 Feedback e QoE]
                     |                    |                   |
                     +---------+----------+-------------------+
                               v
        Base de        [6 Orquestrador] <-> Operação humana (aprovação, veto)
     conhecimento <---->   |           |
   (RAG e GraphRAG)        v           v
              [4 Alocação]     [5 Entrega de conteúdo]
                        |           |
                        v           v
               Nuvem híbrida, servidores de borda, roteamento, player
```

| # | Agente | O que faz | Como | Tipo |
|---|---|---|---|---|
| 1 | **Monitoramento** | Mantém o estado atual por região; detecta tráfego fora do previsto | Spark em fluxo; anomalia estatística; alertas no Splunk | Reativo baseado em modelo |
| 2 | **Previsão de demanda** | Prevê tráfego por região, tipo de conteúdo e dispositivo, em curto prazo (5 a 60 min) e planejado (dias) | Modelos da seção 7 | Agente que aprende |
| 3 | **Feedback e QoE** | Transforma avaliações em sinal estruturado (tema, gravidade, região, dispositivo) e cruza com incidentes | SLM aberto com ajuste fino (LoRA), saída validada por esquema | Baseado em modelo de linguagem |
| 4 | **Alocação de recursos** | Decide capacidade por região: escalar a borda, expandir para a nuvem pública, redirecionar tráfego, pré-aquecer cache | Otimização com restrições (menor custo mantendo folga no percentil 95 previsto) | Baseado em utilidade |
| 5 | **Entrega de conteúdo** | Ajusta a entrega quando capacidade não basta: teto de qualidade por dispositivo e região, pré-posicionamento, prioridade ao ao vivo | Políticas com limites, usando os dados de dispositivos | Baseado em objetivos |
| 6 | **Orquestrador** | Coordena, resolve conflitos (qualidade contra custo), aplica travas, aciona humanos, registra a justificativa | LLM em LangGraph: grafo com estado compartilhado, roteamento entre agentes, human-in-the-loop e memória | Supervisor hierárquico |

### Justificativas principais

- **Agente 3 usa modelo de linguagem** porque o insumo é linguagem natural variada, onde regras falham. Usa um SLM (modelo pequeno) aberto, com ajuste fino, e não um LLM grande por API: a tarefa é repetitiva e bem definida, o volume é alto, o custo por chamada pesa e o texto do assinante não precisa sair da empresa (LGPD). Ele só classifica: o texto do usuário é dado não confiável (pode conter instrução maliciosa) e nunca chega a quem executa ações.
- **Agente 4 usa otimização, não aprendizado por reforço**, porque otimização é previsível, auditável e respeita restrições rígidas. Reforço aprende explorando, e explorar em rede de produção derruba cliente. Fica como evolução, treinado em simulador com replay de eventos passados.
- **Agente 5 existe** porque degradar onde não se percebe (limitar 4K em tela pequena) é melhor do que travar. É o amortecedor quando a previsão erra.
- **Supervisor em vez de agentes negociando livremente:** em operação crítica, um responsável único, fluxo rastreável e um só ponto de travas e auditoria.

### Comunicação e ferramentas

- Agentes trocam mensagens em formato fixo por um barramento de eventos (Pub/Sub ou Kafka).
- Memória de curto prazo: estado do evento em andamento. Memória de longo prazo: previsões, decisões e resultados no BigQuery, mais a base de conhecimento.
- **Gerenciamento de contexto:** o LLM recebe um resumo do estado (região, desvio, previsão, opções), nunca o dado bruto. Contexto menor custa menos, responde mais rápido e erra menos.
- **Ferramentas:** cada agente usa ferramentas com o mínimo de permissão. O acesso ao BigQuery é por ferramenta de consulta SQL somente leitura. Só os agentes 4 e 5 escrevem na rede. As ferramentas são expostas por MCP (Model Context Protocol), o que desacopla agentes de sistemas: trocar o Splunk ou o orquestrador da nuvem não exige reescrever agente.
- **Saídas tipadas:** toda saída de modelo de linguagem é validada contra um esquema (Pydantic) antes de seguir adiante. Saída fora do esquema é rejeitada.
- **Framework: LangGraph.** O fluxo é um grafo explícito: cada agente é um nó, o estado é compartilhado e persistido, o roteamento entre nós é condicional e há pontos de human-in-the-loop antes de qualquer ação acima do limite. LangChain fornece a base (prompts, modelos, ferramentas).
- **Por que não CrewAI:** organiza agentes por papéis e é excelente para protótipo rápido, mas dá menos controle sobre o fluxo. Em operação crítica preciso saber exatamente por qual caminho uma decisão passou, poder pausar para aprovação e retomar do ponto em que parou. Usaria CrewAI para validar a ideia na fase de descoberta.

## 6. Base de conhecimento dos agentes (RAG, GraphRAG, Agentic RAG)

Um LLM não conhece a rede da StreamTech nem o que aconteceu nos eventos passados. Sem isso, ele inventa.

| Técnica | Conteúdo | Pergunta que responde |
|---|---|---|
| **RAG** (busca vetorial) | Relatórios pós-evento, manuais de operação, registros de incidentes, políticas | "O que fizemos na última final e o que deu errado?" |
| **GraphRAG** (grafo de conhecimento) | Topologia: regiões, pontos de presença, enlaces, servidores, conteúdos em cache e suas dependências | "Se o ponto de presença X saturar, quais vizinhos absorvem o tráfego e com que folga?" |
| **Agentic RAG** | O Orquestrador escolhe a fonte por pergunta: vetorial, grafo ou SQL no BigQuery; avalia se a resposta basta e busca de novo se não | Perguntas que combinam histórico, topologia e números |

Como a busca funciona:

- **Divisão em trechos:** relatórios e manuais são divididos por seção, preservando o título, para o trecho fazer sentido sozinho.
- **Modelo de embeddings:** escolhido por lidar bem com português e com vocabulário técnico de rede; testado com perguntas reais da operação antes de adotar.
- **Re-ranking:** a busca traz muitos trechos candidatos e um segundo modelo os reordena pelos mais relevantes. Só os melhores vão para o LLM. Melhora a precisão e reduz o contexto.
- **Roteamento:** pergunta sobre números vira consulta SQL; sobre vizinhança e dependência vai ao grafo; sobre o que foi feito antes vai à busca vetorial.

Justificativas:

- **RAG para conhecimento que muda, ajuste fino para tarefa que se repete.** A rede e os incidentes mudam toda semana; atualizar um índice é barato, retreinar um modelo não. E o RAG permite citar a fonte, o que ajuda a auditoria. Já a classificação de feedback é sempre a mesma tarefa, e aí o ajuste fino compensa (seção 7).
- **Grafo para a topologia:** rede é um grafo por natureza. Busca por similaridade de texto não responde perguntas de vizinhança e dependência.
- **Ciclo de aprendizado:** cada relatório pós-evento gerado pelo sistema entra na base. O sistema melhora a cada pico.

## 7. Modelos e técnicas

### Previsão de demanda (regressão em série temporal)

| Papel | Técnica | Por quê |
|---|---|---|
| Linha de base | Sazonal ingênuo e ARIMA_PLUS no BigQuery ML | Barata, já na infraestrutura; é a régua a superar |
| Modelo principal | Gradient boosting (LightGBM) com variáveis de calendário, evento, campanha e defasagens | Forte em dado tabular, rápido de treinar, explicável |
| Incerteza | Previsão por quantis (percentis 50, 90, 95) | A rede se dimensiona pelo percentil alto, não pela média |
| Lançamento sem histórico | Analogia com eventos parecidos (gênero, alcance da campanha) | Não há série para aprender |
| Evolução | Redes neurais temporais (TFT, DeepAR) | Só se vencerem o boosting no teste |

**Por que previsão probabilística:** o custo do erro é assimétrico. Errar para menos causa buffering e perda de assinante; errar para mais custa só capacidade ociosa por algumas horas.

**Avaliação:** validação com janela deslizante no tempo (nunca embaralhar série temporal), com atenção separada aos períodos de evento. Otimização de hiperparâmetros só depois que o modelo simples estiver validado.

### LLMs

| Uso | Perfil de modelo | Por quê |
|---|---|---|
| Agente 3 (classificar feedback) | SLM aberto (Hugging Face), com ajuste fino por instrução usando LoRA e quantização, hospedado na empresa | Volume alto, tarefa repetitiva, custo baixo, dado não sai da empresa |
| Agente 6 (orquestrar e explicar) | LLM com raciocínio mais forte, via API | Poucas chamadas, decisões de maior consequência |

- **LoRA e quantização:** LoRA treina só uma pequena parte dos pesos, o que barateia o ajuste; a quantização reduz o tamanho do modelo para rodar em hardware modesto. Os exemplos de treino vêm de feedbacks antigos rotulados por pessoas da operação.
- **LLM routing:** perguntas simples vão ao SLM, as complexas ao LLM forte. Paga-se o modelo caro só quando ele faz diferença.
- **Raciocínio do Orquestrador:** prompts encadeados e raciocínio passo a passo (Chain-of-Thought): situação, opções, restrições, decisão. O raciocínio fica registrado e é a base da explicação entregue à operação.
- **Engenharia de prompt:** papel, contexto, formato de saída e exemplos. Geração conservadora (temperatura baixa, limite de tokens), porque decisão operacional precisa ser repetível. Prompts versionados como código.
- **Critérios de escolha de modelo:** qualidade na tarefa, latência, custo, saída estruturada confiável e privacidade.
- **Avaliação:** o Agente 3 é medido por precisão, recall e F1 contra uma amostra rotulada por pessoas. O Orquestrador é medido em um conjunto fixo de cenários passados: escolheu a ação certa, usou a fonte certa, a explicação confere com os dados.

## 8. Infraestrutura e deploy

Princípio: aproveitar o que já existe.

| Já existe | Papel na solução |
|---|---|
| Monitoramento com NetFlow/sFlow | Sentidos do Agente 1 |
| Apache Spark | Agregação em fluxo e preparo de variáveis |
| Google BigQuery | Histórico, linha de base (BigQuery ML), memória de longo prazo, trilha de auditoria |
| Splunk | Observabilidade dos próprios agentes, alertas, painel |
| Servidores distribuídos e nuvem híbrida | Atuadores: capacidade local para a base, nuvem pública para o pico |

O que se acrescenta:

**Databricks**, como plataforma gerenciada do Spark que a empresa já usa. Lê o BigQuery por conector, sem migrar o data warehouse. Papéis:

| Recurso | Uso na solução |
|---|---|
| Pipelines com Delta Live Tables | Consolidação, limpeza e enriquecimento dos dados em camadas (bruto, limpo, pronto para consumo), com regras de qualidade declaradas no próprio pipeline |
| Serving endpoints | Publicação do modelo de previsão e do SLM como API, com escala automática |
| Pipeline de RAG com busca vetorial | Índice dos relatórios e manuais, atualizado quando entra documento novo |
| MLflow | Registro de experimentos, versões de modelo e rastreamento de cada passo dos agentes |
| Avaliação de agentes | Conjunto fixo de cenários rodado antes de cada nova versão |
| Infraestrutura como código (IaC) | Ambientes de desenvolvimento, teste e produção idênticos e reproduzíveis |

Alternativa, se a empresa não quiser plataforma nova: MLflow aberto sobre o Spark atual, ou Vertex AI por já estar no Google Cloud. A escolha do Databricks se justifica pela continuidade com o Spark e por reunir dados, modelos e agentes em um lugar só.

Também se acrescenta:

- **Barramento de mensagens** (Kafka ou Pub/Sub) para a comunicação entre agentes.
- **Banco de grafos** (Neo4j) para a topologia (GraphRAG).
- **Agentes em contêineres** (Docker e Kubernetes), expostos por API (FastAPI), com infraestrutura como código (Terraform) e implantação automatizada (CI/CD).
- **Painel da operação** (Looker Studio, conectado ao BigQuery).

Os produtos específicos podem ser trocados por equivalentes já homologados pela empresa sem alterar a arquitetura.

**Operação contínua (LLMOps e AgentOps):**

- Rastreamento de cada passo dos agentes: qual nó, qual ferramenta, qual fonte, quanto tempo, quanto custou.
- Monitoramento de custo, latência e qualidade das respostas dos modelos de linguagem.
- Monitoramento de desvio do modelo de previsão (os padrões de consumo mudam), com re-treino programado e após cada grande evento.
- Nenhuma versão nova de prompt, modelo ou agente vai para produção sem passar pela avaliação.

## 9. Dois cenários de funcionamento

**A. Final de campeonato ao vivo (pico previsível)**

1. Sete dias antes: o calendário informa o evento. O Agente 2 prevê por região, com intervalo, por analogia com finais anteriores. O Orquestrador consulta a base: o que aconteceu da última vez.
2. Dois dias antes: plano de capacidade enviado à operação, com custo e justificativa, para aprovação.
3. Uma hora antes: o Agente 4 pré-escala a borda e reserva nuvem; o Agente 5 pré-posiciona o conteúdo.
4. Durante: o Agente 1 compara real e previsto a cada minuto. Desvio fora do intervalo dispara nova otimização. Se uma região satura, o Agente 5 aplica teto de qualidade seletivo enquanto o Agente 4 traz capacidade.
5. Depois: capacidade devolvida, relatório pós-evento gerado e guardado na base, modelos re-treinados.

**B. Pico inesperado (conteúdo viraliza)**

1. O Agente 1 detecta tráfego fora do intervalo em uma região.
2. O Agente 2 refaz a previsão de curto prazo.
3. O Agente 4 age dentro dos limites pré-aprovados (ações reversíveis, sem esperar humano); o Agente 5 protege a experiência até a capacidade chegar. O Orquestrador consulta o grafo para saber quais vizinhos podem absorver tráfego.
4. A operação é notificada e aprova apenas o que ultrapassa os limites.

## 10. Guardrails, explicabilidade, observabilidade e monitoramento

**Guardrails (o que o sistema não pode fazer)**

- **Na entrada:** feedback de usuário é tratado como dado não confiável e passa por filtro antes do modelo.
- **Na saída:** toda saída de modelo de linguagem é validada por esquema; texto livre nunca vira comando.
- **Na ação:** limites por ciclo de variação de capacidade e de custo; acima disso, human-in-the-loop.
- **Reversão automática** se a métrica piorar após uma ação.
- **Falha segura:** se os agentes caírem, valem as regras estáticas atuais. O sistema nunca fica pior do que hoje.
- **Botão de parada** com a operação.

**Autonomia progressiva:** modo sombra (registra o que faria), depois recomendação com aprovação humana, depois automação de ações reversíveis dentro de limites.

**Explicabilidade:** cada decisão vem com o motivo em linguagem clara, os dados que a sustentam, a alternativa considerada e a fonte consultada. No modelo de previsão, a importância das variáveis mostra o que puxou a previsão para cima (por exemplo, o evento no calendário).

**Observabilidade:** rastreamento completo de cada execução dos agentes, disponível no Splunk junto com os dados de rede, para a operação ver causa e efeito no mesmo lugar.

**Monitoramento:** métricas de rede, de experiência, de previsão e de custo dos modelos acompanhadas em painel, com alerta quando saem da faixa.

**LGPD:** dados agregados por região e classe de dispositivo; feedback anonimizado e processado por modelo hospedado na própria empresa.

## 11. Como provar que funciona

1. **Previsão, offline:** validação temporal no histórico; precisa vencer a linha de base, principalmente nos eventos.
2. **Modelos de linguagem e agentes, offline:** F1 do Agente 3 em amostra rotulada; Orquestrador avaliado em cenários fixos (seção 7).
3. **Sistema, em simulação:** replay de picos passados para ver o que os agentes teriam feito.
4. **Produção, modo sombra:** decisões registradas sem executar, comparadas com as da operação.
5. **Canário:** uma região de menor risco primeiro, com grupo de controle (teste A/B).
6. **Validação de negócio:** rebuffering, custo e cancelamentos contra a operação atual. O resultado responde às duas perguntas de negócio?

## 12. Implantação em fases

Prazo estimado: cerca de 9 meses (aproximadamente 40 semanas), com entrega de valor a partir do terceiro mês.

| Fase | Duração | Entrega | Critério para avançar |
|---|---|---|---|
| 0. Descoberta | 4 semanas | Respostas às lacunas, linha de base das métricas, aprovação do problema pelos gestores | Dados e atuadores confirmados |
| 1. Previsão | 8 semanas | Agentes 1 e 2 em modo sombra; painel | Previsão vence a linha de base |
| 2. Recomendação | 8 semanas | Agentes 3, 4 e 6 sugerindo; base de conhecimento no ar; humano aprova | Operação aceita a maioria das sugestões |
| 3. Automação limitada | 10 semanas | Ações reversíveis automáticas em uma região piloto | Rebuffering cai sem estourar custo |
| 4. Expansão | 10 semanas | Todas as regiões, Agente 5 ativo, estudo de reforço em simulador | Resultado sustentado em eventos reais |

Começa pela previsão porque ela já gera valor sozinha (a operação se prepara manualmente) e porque tudo depende dela.

Premissas do prazo: equipe dedicada de 6 a 8 pessoas (engenharia de dados, ciência de dados, engenharia de IA, especialista de redes e gestão do projeto); histórico disponível e com qualidade razoável no BigQuery; APIs existentes para atuar na rede. As fases 3 e 4 dependem do calendário da empresa, pois a automação só se valida em eventos reais de alta demanda.

## 13. Riscos principais

| Risco | Mitigação |
|---|---|
| Previsão erra um evento inédito | Intervalos, margem de segurança, reação de curto prazo, Agente 5 como amortecedor |
| Ação automática piora a rede | Limites, canário, reversão automática, falha segura |
| LLM inventa ou sofre injeção de prompt | LLM fora do caminho crítico, RAG com fonte, saída em formato fixo, feedback tratado como não confiável, LLM sem escrita na rede |
| Custo de nuvem dispara | Custo como restrição na otimização, teto por ciclo |
| Dados ruins ou atrasados | Checagem de qualidade no Agente 1; com dado suspeito, o sistema fica conservador |
| Padrão de consumo muda e o modelo envelhece | Monitoramento de desvio e re-treino |
| Resistência da operação | Modo sombra, explicação de cada decisão, humano com veto |

## 14. Resumo das decisões e justificativas

| Decisão | Justificativa | Alternativa descartada |
|---|---|---|
| Duas perguntas de negócio | Muitas perguntas indicam vários projetos em um | Tratar cada capacidade como projeto separado |
| Agentes, não só um modelo | O problema pede ciclo contínuo de perceber, decidir e agir | Modelo de previsão em um painel |
| Seis agentes com supervisor | Responsabilidades e relógios distintos, falha isolada, auditoria em um ponto | Agente único; agentes negociando livremente |
| LLM só na camada estratégica | Latência, custo e variabilidade não cabem em decisão de rede em tempo real | LLM decidindo alocação direto |
| Previsão probabilística | Custo do erro é assimétrico | Previsão pontual pela média |
| Boosting com linha de base ARIMA | Forte em dado tabular, explicável, barato | Começar por deep learning |
| Otimização antes de reforço | Respeita restrições e é auditável | Reforço aprendendo em produção |
| LangGraph como framework | Fluxo explícito, estado persistido, human-in-the-loop, rastreável | CrewAI (menos controle do fluxo; bom para protótipo) |
| RAG e GraphRAG para conhecimento | Conhecimento da empresa muda sempre; rede é um grafo | Ajuste fino para ensinar fatos ao LLM; só busca vetorial |
| SLM com ajuste fino para o feedback | Tarefa repetitiva, volume alto, custo baixo, dado fica na empresa | LLM grande por API para classificar |
| Re-ranking na busca | Mais precisão com menos contexto | Entregar ao LLM tudo o que a busca trouxe |
| MCP e saídas tipadas | Desacopla agentes de sistemas; impede saída malformada de virar ação | Integrações diretas e texto livre |
| Databricks sobre o Spark existente | Continuidade; dados, modelos e agentes em um lugar; sem migrar o BigQuery | Migrar tudo; ou montar cada peça em separado |
| Ajuste de entrega por dispositivo | Degrada onde não se percebe, em vez de travar | Só adicionar capacidade |
| Reaproveitar BigQuery, Spark, Splunk | Menor custo e risco, adoção rápida | Plataforma totalmente nova |
| Autonomia progressiva | Confiança se constrói com evidência | Automação total desde o início |
| Adquirir calendário e telemetria do player | O problema define os dados; são as causas e os efeitos reais | Trabalhar só com as quatro fontes listadas |

## 15. Onde cada disciplina da pós aparece

| Disciplina | Na solução |
|---|---|
| 1. IA Generativa e LLMs para PLN | SLM aberto com ajuste fino (LoRA, quantização) no Agente 3; engenharia de prompt; parâmetros de geração; métricas de avaliação; LangChain como base |
| 2. Infraestrutura com Databricks | Pipelines de limpeza e enriquecimento, serving endpoints, pipeline de RAG, avaliação e observabilidade de agentes, IaC, AgentOps |
| 3. RAG, GraphRAG e Agentic RAG | Base de conhecimento: busca vetorial com re-ranking, grafo da topologia, SQL sobre o BigQuery, roteamento entre fontes, LLMOps |
| 4. Construção e Deploy de Agentes | Seis agentes em LangGraph com estado, memória, ferramentas, roteamento e human-in-the-loop; MCP; guardrails, explicabilidade e observabilidade; deploy via API |
