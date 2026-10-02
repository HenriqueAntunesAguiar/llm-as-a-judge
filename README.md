# Avaliação de um sistema multiagente com RAG usando LLM as a Judge

## Escopo

Construi um benchmark para avaliar um sistema multiagente que utiliza RAG por meio de chamadas de ferramenta (*tool calling*), com busca híbrida em PostgreSQL e pgvector.

A base de conhecimento é um manual de uso de um dashboard técnico no Power BI. O material reúne orientações de navegação, como acesso às páginas e uso das setas, além de explicações sobre gráficos, indicadores e o contexto das visualizações.

O objetivo da avaliação foi identificar falhas nas respostas e orientar melhorias na base de conhecimento e nos prompts. Para isso, utilizamos uma LLM como avaliadora - abordagem conhecida como **LLM as a Judge** - em conjunto com verificações de recuperação e revisão humana.

Neste relato, compartilho como construi o benchmark, coletei os resultados, registrei as configurações das execuções e utilizei as avaliações para orientar decisões.

## Sumário

- [Escopo](#escopo)
- [1 Construção do benchmark](#1-construção-do-benchmark)
- [2 Coleta das respostas do sistema](#2-coleta-das-respostas-do-sistema)
- [3 Versionamento e rastreabilidade](#3-versionamento-e-rastreabilidade)
- [4 Avaliação com LLM as a Judge](#4-avaliação-com-llm-as-a-judge)
  - [Exemplo de avaliação com nota 0](#exemplo-de-avaliação-com-nota-0)
  - [Exemplo de avaliação com nota 8](#exemplo-de-avaliação-com-nota-8)
  - [Exemplo de avaliação com nota 9,5](#exemplo-de-avaliação-com-nota-95)
- [5 Verificações complementares](#5-verificações-complementares)
- [6 Avaliação do atendimento](#6-avaliação-do-atendimento)
- [7 Aprendizados e oportunidades de melhoria](#7-aprendizados-e-oportunidades-de-melhoria)

## 1 Construção do benchmark

Para avaliar o sistema, começamos pela definição das perguntas e das respostas de referência. A base utilizada no RAG foi estruturada em Markdown, o que facilitou a organização do conteúdo e a definição dos limites dos chunks, buscando preservar o contexto das informações.

Criamos cinco perguntas por tópico presente nos chunks de contexto, totalizando 200 perguntas. Cada caso de teste contém:

- **Pergunta:** solicitação enviada ao sistema.
- **ID do documento de referência:** identificação do documento utilizado na elaboração da pergunta.
- **Resposta esperada:** gabarito com as informações necessárias para responder à pergunta.

As perguntas e as respostas esperadas foram geradas com auxílio de outra LLM e validadas por um técnico responsável pelo processo. Essa revisão verificou a pertinência das perguntas e a correção das respostas de referência.

Após cada execução, registramos também a resposta produzida pelo sistema e a nota atribuída pelo Judge, de 0 a 10. Assim, distinguimos o conjunto de referência dos resultados obtidos em cada avaliação.

Uma das principais decisões foi complementar a avaliação textual com a comparação entre o ID do documento de referência e os documentos recuperados pelo agente de RAG. Essa comparação ajuda a investigar se a recuperação alcançou a fonte esperada, mas precisa ser analisada em conjunto com a resposta: recuperar o documento correto, por si só, não demonstra que o sistema utilizou adequadamente seu conteúdo.

## 2 Coleta das respostas do sistema

Com o material de teste validado, desenvolvemos um script para enviar as perguntas à API do sistema multiagente e registrar as respostas.

Cada pergunta foi enviada em uma nova conversa, evitando a influência do histórico das perguntas anteriores. Dessa forma, avaliamos o comportamento do sistema em solicitações isoladas.

As respostas obtidas foram associadas aos respectivos casos de teste e salvas para comparação com os gabaritos e posterior avaliação pelo Judge.

Esse recorte é relevante: o benchmark descrito avalia perguntas independentes. A avaliação de conversas com múltiplas interações é tratada em outro fluxo, apresentado ao final deste documento.

## 3 Versionamento e rastreabilidade

Para comparar execuções e entender o efeito das alterações, registramos as configurações utilizadas em cada avaliação.

Antes de cada execução do benchmark, salvamos os system prompts em pastas versionadas e registramos os modelos de LLM utilizados. Essas informações foram resumidas em um arquivo JSON.

Também geramos hashes dos prompts para associar seus conteúdos aos registros armazenados no banco de dados. O hash funciona como uma referência para identificar a versão utilizada; o conteúdo do prompt permanece preservado nos arquivos versionados.

Esse registro permite rastrear as configurações de cada execução e interpretar os resultados considerando as mudanças realizadas.

## 4 Avaliação com LLM as a Judge

Neste projeto, o Judge foi especialmente útil para identificar casos que exigiam revisão e ajudar a priorizar a análise humana.

Nos casos avaliados com nota 0, encontramos respostas que não atendiam à pergunta. Na revisão, identificamos situações em que o sistema havia recuperado um contexto semelhante, mas diferente daquele solicitado.

Muitas respostas avaliadas com nota 10 foram consideradas corretas, claras e alinhadas ao gabarito durante a revisão. Essa concordância foi útil para a análise, sem transformar a nota do Judge em uma garantia de qualidade.

As notas intermediárias, como 7 e 8, exigiram uma análise mais cuidadosa. As justificativas apontaram problemas de clareza ou completude que, após revisão, orientaram ajustes na organização dos chunks, na redação da base de conhecimento e nos prompts.

O prompt utilizado foi o seguinte, aqui apresentado com ajustes de redação:

```text
# Papel

Você é um avaliador imparcial de respostas de uma IA sobre um
dashboard de {ÁREA DE NEGÓCIO}.

Compare a resposta obtida com a resposta esperada, considerando
a pergunta apresentada.

# Critérios

- Correção: a resposta não deve contradizer as informações
  do gabarito.
- Completude: deve conter os pontos essenciais para responder
  à pergunta, sem exigir reprodução literal do gabarito.
- Objetividade: deve responder diretamente, sem conteúdo
  irrelevante.

Atribua uma nota geral de 0 a 10.
A resposta não precisa ser idêntica ao gabarito, mas deve
conter as informações essenciais para responder à pergunta.

Explique a razão da nota e indique quais informações estão
faltando ou o que poderia ser melhorado na resposta da IA.
```

### Exemplo de avaliação com nota 0

> A resposta da IA não conseguiu fornecer a informação solicitada. Ela afirmou não ter encontrado a informação no manual, enquanto a resposta esperada indica claramente a localização: “Na página Resumo, na área [INFORMAÇÃO OMITIDA]”. A resposta obtida é incorreta e incompleta, não atendendo a nenhum dos critérios.

Após revisar esse caso, reformulamos o trecho correspondente na base de conhecimento em Markdown, acrescentando informações que ajudaram a diferenciar cenários semelhantes e a tornar a resposta mais clara.

### Exemplo de avaliação com nota 8

> A resposta da IA aborda corretamente os dois principais métodos de acesso a páginas: menu lateral e barra inferior. No entanto, inclui informações irrelevantes sobre filtros e uma pergunta final que não contribuem para a objetividade da resposta à pergunta original. Além disso, faltou um dos exemplos de página, “[INFORMAÇÃO OMITIDA]”, mencionado no gabarito, embora os exemplos fornecidos sejam suficientes.

Esse exemplo merece atenção na revisão do próprio avaliador. A justificativa menciona a ausência de um exemplo, mas reconhece que os exemplos fornecidos eram suficientes. Isso levanta a necessidade de esclarecer no prompt quais elementos são essenciais e quais são apenas ilustrativos.

### Exemplo de avaliação com nota 9,5

> A resposta da IA é correta e completa, indicando tanto a página, “[INFORMAÇÃO OMITIDA]”, quanto o gráfico específico, “[INFORMAÇÃO OMITIDA]”, onde a informação pode ser encontrada. Ela adiciona informações úteis sobre como navegar até a página e como consultar outros períodos, o que agrega valor. A única pequena melhoria seria remover a pergunta de acompanhamento, “Você tem mais alguma dúvida...”, para manter a objetividade estrita, mas isso não compromete a qualidade da resposta principal.

Nesse caso, a observação se concentra no estilo da resposta. Ao interpretar os resultados, é útil distinguir falhas de conteúdo de ajustes de objetividade e considerar o comportamento desejado para o chatbot.

## 5 Verificações complementares

O benchmark ajudou a orientar revisões dos documentos e ajustes nos prompts. Além da nota do Judge, utilizamos ou identificamos como relevantes as seguintes verificações:

| Verificação | Objetivo |
| --- | --- |
| Comparar os documentos recuperados com o documento de referência | Investigar se a recuperação alcançou a fonte esperada |
| Verificar palavras-chave na resposta | Buscar sinais da presença de informações esperadas, sem substituir a análise do significado |
| Analisar as notas do Judge | Observar o conjunto de resultados e priorizar casos para revisão |
| Revisar as justificativas do Judge | Entender os motivos das notas e verificar se fazem sentido |
| Comparar a ferramenta esperada com a ferramenta chamada | Verificar se o sistema selecionou a ferramenta adequada para a pergunta |

A avaliação das chamadas de ferramenta foi realizada separadamente. Por se tratar de um sistema multiagente, essa verificação ajuda a investigar falhas na seleção das ferramentas antes de analisar a recuperação e a resposta final.

Essas evidências devem ser interpretadas em conjunto. Uma resposta inadequada pode exigir revisão da seleção de ferramentas, do conteúdo recuperado, da forma como a resposta foi produzida ou de algum system prompt.

## 6 Avaliação do atendimento

Além do benchmark de perguntas isoladas, desenvolvemos um fluxo de avaliação da conversa ao final do atendimento ao usuário.

Nesse fluxo, outra avaliação com LLM as a Judge analisa aspectos da experiência, como a dificuldade do usuário para chegar à resposta e a presença de comportamentos inadequados, incluindo respostas percebidas como hostis.

As observações são armazenadas para análise posterior e podem acionar alertas quando são identificadas avaliações críticas.

Esse fluxo complementa o benchmark: enquanto as perguntas isoladas ajudam a avaliar respostas específicas, a análise da conversa permite observar como o atendimento se desenvolveu ao longo das interações. As avaliações também fornecem indicadores para acompanhar a qualidade do atendimento em produção.

## 7 Aprendizados e oportunidades de melhoria

O principal aprendizado deste projeto foi a utilidade de combinar a nota e a justificativa do Judge com evidências de recuperação, verificações de ferramentas e revisão humana. Esse conjunto orientou melhorias na base de conhecimento e nos prompts.

Como próximos aprimoramentos, recomendamos:

- **Definir uma rubrica de pontuação mais explícita:** descrever os critérios para diferentes faixas de nota e distinguir falhas de conteúdo de questões de estilo.
- **Refinar o prompt do Judge:** explicitar que exemplos opcionais não precisam ser reproduzidos e que informações adicionais relevantes devem ser avaliadas conforme sua contribuição à resposta.
- **Ampliar a variedade de perguntas:** incluir perguntas sem resposta na base, ambiguidades e solicitações que dependam de mais de um trecho.
- **Detalhar a referência de recuperação:** registrar IDs de chunks quando a identificação apenas do documento for ampla demais e considerar múltiplas fontes válidas quando aplicável.
- **Ampliar os registros das execuções:** versionar também o benchmark, a base de conhecimento e a configuração do Judge, além dos prompts e modelos já registrados.

Esses itens representam oportunidades de evolução da metodologia. Alguns já foram implementados em testes separados, sem terem sido reunidos em um único benchmark.

