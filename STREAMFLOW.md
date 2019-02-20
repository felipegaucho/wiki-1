# Livepeer Streamflow Paper

**Escalabilidade da Livepeer através de Orquestragem, Micropagamentos Probabilísticos, e Negociação de Jobs Offchain**

**Autores**    
Doug Petkanics <doug@livepeer.org>    
Yondon Fu <yondon@livepeer.org>

**Pesquisadores**    
Eric Tang <eric@livepeer.org>    
Philipp Angele <philipp@livepeer.org>    
Josh Allmann <josh@livepeer.org>

**STATUS: PROPOSTA - Feedback e _peer review_ são bem-vindos.**

## Abstract #####################################

A proposta Streamflow introduz mudanças ao protocolo Livepeer, assim como implementações offchain que permitirão à rede escalar além das limitações atuais da versão _alpha_, funcional na blockchain da Ethereum. A proposta sugere updates que endereçam a acessibilidade, confiabilidade, performance e escala da rede. Elementos-chave dela são: um registro de serviços, um mecanismo de negociação de jobs offchain e micropagamentos, uma divisão entre nós Orquestradores e nós Transcodificadores, a eliminação do problema da disponibilidade de dados na verificação independente de confiança, e o aumento no número de nós que podem competir para performar serviço à rede, valor que era arbitrariamente limitado durante o _alpha_. A arquitetura resultante permitirá a usuários ter acesso a serviços de transcodificação em escala, através de provedores concorrentes, enquanto tornará a viabilidade econômica da rede mais resistente a variações de preço e demanda na blockchain subjacente (Ethereum).

## Índice ###########################################

* [Introdução e Background](#introdução-e-background)
* [Proposta do Protocolo Streamflow](#proposta-do-protocolo-streamflow)
    * [Orquestradores e Transcodificadores](#orquestradores-e-transcodificadores)
    * [Relaxamento no Limite de Transcodificadores e Segurança Garantida por _Stakes_](#relaxamento-no-limite-de-transcodificadores-e-segurança-garantida-por-stakes)
    * [Registro de Serviços](#registro-de-serviços)
    * [Negociação de Jobs Off Chain](#negociação-de-jobs-offchain)
    * [Micropagamentos Probabilísticos (PM)](#micropagamentos-probabilísticos-(PM))
    * [Verificação On Chain Baseada em Faltas](#verificação-on-chain-baseada-em-faltas)
* [Análise Econômica](#análise-econômica)
    * [Livepeer Token](#livepeer-token)
    * [Delegação como um Sinal de Segurança e Reputação](#delegação-como-um-sinal-de-segurança-e-reputação)
    * [Inflação que Vira _stake_ Aplicado e Delegadores Apáticos](#inflação-que-vira-stake-aplicado-e-delegadores-apáticos)
    * [Considerações de Engenharia Off Chain](#considerações-de-engenharia-off-chain)
* [Ataques](#ataques)
    * [_Squeezing_ de Delegadores](#squeezing-de-delegadores)
    * [Roubo da Taxa do Delegador](#roubo-da-taxa-do-delegador)
* [Áreas Abertas de Pesquisa](#áreas-abertas-de-pesquisa)
    * [Verificação Não-Determinística](#verificação-não-determinística)
    * [Protocolos para Pools Públicas de Transcodificadores](#protocolos-para-pools-públicas-de-transcodificadores)
    * [Mitigando Gastos Duplos de _Broadcasters_](#mitigando-gastos-duplos-de-broadcasters)
    * [Pagamentos VOD](#pagamentos-vod)
* [Caminho de Migração](#caminho-de-migração)
* [Apêndice](#apêndice)
    * [Apêndice A: Esquema de Micropagamentos Probabilísticos](#apêndice-a-esquema-de-micropagamentos-probabilísticos)
* [Referências](#referências)

## Introdução e Background ###########################################

O protocolo Livepeer incentiva e assegura uma rede decentralizada de nós transcodificadores de vídeos. Usuários que desejam transcodificar um vídeo podem submeter um job à rede, ao preço que determinarem; ter um transcodificador designado para si; e ter o vídeo transcodificado com garantias econômicas de acurácia. O protocolo usa um mecanismo baseado na delegação de _stakes_ para eleger os nós mais adequados para encodificar e decodificar transmissões ao vivo de modo rápido e performante.

A versão _alpha_, funcional na blockchain da Ethereum desde maio de 2018, implementou muitos dos designs originalmente especificados no [Whitepaper da Livepeer](https://github.com/livepeer/wiki/blob/master/WHITEPAPER.md). O sistema de delegação de _stakes_, com seus incentivos inflacionários, se provou eficiente em motivar a participação, e em criar uma rede emergente de transcodificadores e delegadores para garantir a qualidade do trabalho sendo performado. A rede é usável, e para certos casos de uso, como transmissões ao vivo de longa duração, ou prototipagem de apps descentralizados, provou-se uma opção viável, ainda que em estágio incipiente. No entanto, a versão _alpha_ não comporta demanda em escala por infraestrutura para processamento de vídeos, devido às seguintes fraquezas:


1. O custo de se usar a rede é correlacionado demais com flutuações nos preços de gas da Ethereun, fazendo com que, em momentos de preços altos, ou cenários que requerem muitas transações, a rede se torna cara demais para ser uma alternativa viável.
2. A delegação de jobs baseada em _stakes_ e as negociações onchain criam cenários imprevisíveis para o _broadcaster_ - se o transcodificador que lhe foi designado fics offline, incorrerão custos extras e atrasos durante a negociação por um novo transcodificador, o que pode ser proibitivo em um contexto de transmissão ao vivo.
3. O problema da disponibilidade de dados permanece não resolvido (em produção), e a verificação de trabalhos performados ainda não pode ser completamente independente de confiança e não-interativa.
4. Transcodificadores não tem como gerir sua disponibilidade em função de sua capacidade e trabalhos em curso, somente atravé de _stake_.
5. Enquanto a rede encoraja a competição por preços, ela não motiva competição a nível de performance e confiabilidade diretamente.
6. Limitações atuais da Ethereum (limites de gas) e aspectos práticos da implementação do protocolo restringem o número de transcodificadores que podem estar ativos a qualquer momento, criando uma barreira de entrada para quem quer competir por trabalho na rede.

O resto desse paper propõe soluções que endereçam cada uma dessas fraquezas. Começa com uma descrição das mudanças arquiteturais propostas ao protocolo. Então analisa os impactos econômicos dessas mudanças, antes de endereças possíveis vetores de ataque. Enfim, debruça-se sobre áreas de pesquisa aberta cujo desenvolvimento pode contribuir para elevar essa proposta de um modelo de segurança social/reputacional para um que resida em garantias puramente criptográficas. Finalmente, conclui com ideias para um caminho de migração que leve o protocolo _alpha_ à versão Streamflow, em ambiente de produção, caso a comunidade aceite e abrace tais mudanças.

Esse paper descreve alterações conceituai e analisa seus impactos, mas deixa implementações específicas para um documento de SPECs a ser provido como parte do processo de LIPs (Livepeer Improvement Proposals).

_Nota: Para absorver propriamente os updates propostos, é importante ter um entendimento de como o protocolo atual da Livepeer funciona, como descrito no whitepaper [[1](#references)]._


## Proposta do Protocolo Streamflow #################################

Os updates e novos conceitos dessa proposta impactam uma ou mais das seguintes áreas: acessibilidade, performance, confiabilidade e escalabilidade. Eles incluem:

* A introdução do novo papel de Orquestrador, somando-se aos papeis já existentes dos _Broadcasters_ e Transcodificadores.
* O relaxamento no limite do número de transcodificadores, permitingo o acesso aberto à competição por trabalho entre quaisquer aspirantes a Orquestradores que possuam tokens e superem os requisitos mínimos de _stake_ e segurança.
* Um registro de serviços no qual Orquestradores propagandeiam sua disponibilidade e serviços, onchain.
* Um mecanismo off chain de negociação de preços e designação de jobs entre _Broadcasters_ e Orquestradores.
* Micropagamentos probabilísticos off chain, com liquidação e depósitos de segurança onchain.
* Update no esquema de verificação, em que a verificação on chain só precisa ocorrer no caso da observação de uma falta.

### Orquestradores e Transcodificadores

Na versão _alpha_, um Transcodificador na rede Livepeer é um nó ciente do protocolo que tanto assiste quanto interage com a blockchain, enquanto performa serviços de processamento de vídeo. Em suma, ele tanto contribui para orquestrar o trabalho na rede, quanto transcodifica vídeo de fato. A ausência de distinção entre as funções pode causar problemas de performance e confiabilidade, dificultando a nós escalarem suas operações. Streamflow propõe uma "arquitetura em dois níveis", separando entre:

* Orquestradores, que são cientes do protocolo, negociam com _Broadcasters_, são responsáveis pela entrega de segmentos verificadamente transcodificados de volta a eles, e coordenam a execução de trabalho entre um grupo potencialmente grande de Transcodificadores.
* Transcodificadores, que não necessariamente são cientes do protocolo, do mecanismo de _staking_, ou da blockchain, mas, por outro lado, têm hardware competitivo em custo-benefício, que se dedica somente ao trabalho de transcodificar vídeos da maneira mais rápida e barata possível, conforme coordenado por Orquestradores.

<img src="https://livepeer-dev.s3.amazonaws.com/docs/otsplit.jpg" alt="Orchestrator Transcoder Split" style="width:750px">

**O primeiro nível** dessa arquitetura é similar ao protocolo _alpha_ da Livepeer, mas Transcodificadores são renomeados como Orquestradores. Orquestradores aplicam (_stake_) seus tokens LPT como depósitos de segurança contra o trabalho que performam, ao passo que, caso façam mal à rede, incorrem penalidade econômica. _Broadcasters_ são cientes dos Orquestradores, negociam jobs com eles, e recebem segmentos transcodificados de volta, com a habilidade de induzir penalidades aos Orquestradores caso estes ajam desonestamente.

**O segundo nível** dessa arquitetura engloba um conceito novo, nomeado Pool de Transcodificadores. O job de se correr para processar vídeos o mais barato e rápido possível pode ser performado por quaisquer GPUs com capacidade disponível, como aquelas descritas na Proposta do Mineração de Vídeo (The Video Miner Proposal[[2](#references)]), ou seja, placas que carregam ASICs NVENC ociosos. Este hardware é capaz de competir pelo trabalho de encodificação e decodificação, e forças de mercado devem contribuir para que os preços sejam significantemente menores do que seriam se Orquestradores, por conta própria, tivessem de performá-lo paralelamente ao trabalho (que exige bastante de CPUs) de se assistir à blockchain e coordenar jobs na rede.

Essa arquiteura remete à de pools de mineração de criptomoedas, que podem ser mais ou menos centralizadas. Podem ser coordenadas por um operador central ou podem ser abertas para que qualquer computador ao redor do mundo possa se conectar e competir para minerar o próximo bloco.

Um Orquestrador pode prover serviços de transcodificação para sua própria pool, o que resultaria no mesmo setup que se tem na versão _alpha_ do protocolo, mas, para fazê-lo, tem de se especializar em dois trabalhos distintos - além de competir com outros Orquestradores otimizando preços por outros caminhos. Por outro lado, se um Orquestrador operar sua própria pool, e somente ela, pode ganhar confiança nos serviços que provê sem incorrer o fardo de se verificar frequentemente o trabalho de Transcodificadores "públicos".

Alguns benefícios do setup em dois níveis são:

* Qualquer um que quiser competir por _fees_ pode fazê-lo, ligando seu hardware e correndo para performar jobs disponíveis. Não é preciso conhecimento sobre o funcionamenot da blockchain, criptomoedas, _staking_, ou depósitos. É como se conectar a uma pool de mineração de bitcoin para ganhar alguns trocados. No entanto, enquanto o acesso é aberto para qualquer um que queira competir, operadores com vantagens relativas a eletricidade, banda larga e localização devem performar melhor que aqueles em cenários menos competitivos.
* Broadcasters desfrutam da segurança combinada dos _stakes_ de seus Orquestradores onchain, mas implementações para otimização de escalabilidade e segurança, tanto de pools públicas quanto privadas, podem ser experimentadas livremente, fora do escopo do protocolo Livepeer.
* Orquestradores podem focar somente em operação de interações e garantias da segurança, em vez de se comprometer também a escalar seu hardware. Um Orquestrador pode coordenar centenas de streams concorrentemente, sem ter de transcodificar um vídeo sequer.
* Implementações alternativas para pools de transcodificação podem ser testadas, fazendo uso de GPUs em setups distribuídos para melhor servir jobs em diferentes regiões, e encorajando competição capaz de resultar em preços menores para Broadcasters.

Enquanto este paper descreve o protocolo para as comunicações e segurança entre Broadcaster/Orquestradores, ele deixa o segundo nível (Orquestrador/Transcodificadores) da arquitetura livre para experimentação por parte de implementações concorrentes. No caso simples em que um Orquestrador é seu próprio pool de transcodificação, a segurança do protocolo se mantém, enquanto outros tradeoffs entre confiabilidade e performance podem ser explorados por pools, abrangendo desde as mais centralizadas até as mais descentralizadas. Teorizamos que, desde que a verificação de trabalho por agentes randômicos em uma pool pública incorra custo adicional a um Orquestrador, então pools privadas devem performar melhor que pools públicas, no entanto, isso pode ser sobrepujado com um esquema funcional de depósitos, punições e protocolos criptoeconômicos de verificação.

### Relaxamento no Limite de Transcodificadores e Segurança Garantida por _Stakes_

A segunda grande mudança em Streamflow é o relaxamento do limite artificial no número de Transcodificadores (Oequestradores, na versão Streamflow) que podem estar ativos a qualquer momento. Na gênese, esse parâmetro foi cravado em 10, e desde então foi expandido para 15, mas a capacidade baixa ainda cria barreira de entrada significante, uma vez que um nó precisa de cada vez mais LPT aplicados (_staked_) para entrar e permanecer no grupo de Transcodificadores ativos. Em Streamflow, o objetivo é permitir a qualquer nó capaz de prover segurança suficiente (na forma de _stake_ ou _stake_ delegado) o direito à competição.

As razões por trás do limite eram:

* Com o trabalho sendo designado onchain a Transcodificadores ativos, era crítico que eles estivessem online, disponíveis para trabalhar. Poucos Transcodificadores significam uma disponibilidade média maior, além da visualização fácil de suas estatísticas de performance, contribuindo para a saúde da rede. 
* Limitações impostas por requerimentos de gas nos cálculos para a contabilidade do trabalho de Transcodificadores criaram um teto artificial. Este ainda poderia ser "esticado", mas não por uma ordem de magnitude ou mais.
* Durante o _alpha_, era importante que o grupo de Transcodificadores ativos fosse acessível e facilmente comunicável para coordenar upgrades, responder a bugs e garantir a qualidade da rede.
* Transcodificadores ativos precisavam de _stake_ suficiente em risco, de modo que sofressem penalidade economicamente relevante em caso de comportamento malicioso.

Os efeitos do relaxamento no limite de 15 Transcodificadores ativos nos itens acima serão realizado por meio de:

* Negociação de jobs offchain e _failover_, de modo que Orquestradores indisponíveis ou que não performarem direito somente percam trabalho futuro, mas não prejudiquem a experiência do _Broadcaster_.
* O grupo ativo não precisa ser calculado a cada _round_, em vez disso, mantendo-se no lugar enquanto Orquestradores aplicam, desaplicam _stakes_ ou são penalizados.
* Orquestradores em competição ainda precisarão prestar atenção a upgrades, bugs e o desenvolvimento na rede - Orquestradores inativos que não o fizerem simplesmente falharão em atrair trabalho (mas sem prejudicar _Broadcasters_).
* Agora que uma boa parcela do montante circulante de LPT já participa ativamente da rede, os requerimentos mínimos podem ser setados de modo a garantir _stake_ suficiente assegurando a competição entre um grande número de nós competindo por trabalho na rede.

O número exato de Transcodificadores e Orquestradores ainda é um problema aberto de pesquisa, assim como o método de implementação. Inicialmente, deve-se ter um aumento de uma ordem de magnitude - como uma centena de Orquestradores em vez de 15 -, com o objetivo de se esticar o aumento até três ordens de grandeza (milhares de Orquestrdores) para atender demanda em qualquer região do mundo com redundância. Abaixo estão alguns mecanismos considerados, assim como descrições curtas de alguns de seus tradeoffs:

1. **Expandir N (# de vagas de Orquestradores) de 15 para algo muito maior, como 200**: As coisas funcionariam essencialmente como na versão _alpha_, com uma barreira de entrada consideravelmente menor para se tornar um nó ativo. Mas isso tornaria ações relativas a aplicação/desaplicação mais caras. Problemas de escalabilidade na Ethereum e custos de gas podem ser fatores relevantes.
2. **Determinar um _stake_ mínimo para se tornar um Orquestrador**: Isso estabeleceria um N máximo possível, ao passo que permitiria a qualquer um saber exatamente o quanto é preciso para se atingir o "piso de segurança" para se manter no grupo ativo. Também permitiria a expansão orgânica do limite ao passo que LPT inflácionários são gerados, encorajando delegadores a buscarem novos Orquestradores que potencialmente estejam oferecendo condições mais atrativas para entrar ou permanecer no grupo ativo e competir por trabalho.
3. **Determinar uma quantidade de _stake_ fixa para qualquer Orquestrador**: Isso forçaria Orquestradores a operarem nós adicionais, e delegadores a constantemente reaplicarem seus _stakes_, de modo a pôr LPT inflacionários para render. Emergem algumas fraquezas em torno da experiência de usuário resultante tanto para Orquestradores quanto delegadores, assim como detalhes complexos referentes à implementação.
4. **Eliminar qualquer requerimento mínimo de _stake_ do protocolo, e deixar que cada client configure o _stake_ mínimo que requer para garantir a segurança de um job**: Isso cria o acesso mais aberto possível e aparenta maior grau de descentralização, mas oferece o menor nível de coordenação entre delegadores e Orquestradores - essencialmente, a reputação ganha importância, o que pode levar à centralização no longo prazo conforme delegadores perdem a habilidade coletiva de rotear trabalho.

Enquanto os benefícios e malefícios de cada _approach_ são considerados, é importante notar que o resultado da implementação de qualquer um será uma rede expandida de Orquestradores, mais redundância e competição a serviço de _Broadcasters_, e incentivos continuados para se rotear _stake_ rumo a nós que possam performar serviços adicionais confiavelmente e com eficiência de custo em troca de _fees_.

Um dos pontos positivos dos modelos baseados em um _stake_ mínimo é o fato de que enquanto as _fees_ correm pela rede, há pouca razão para se operar um nó a não ser que seja para competir por trabalho. O número de vagas é limitado, e, para qualquer agente que tenha LPT, é melhor que o _stake_ esteja aplicado em um nó que esteja ativo, redistribuindo parte dos _fees_ que ganha por trabalho, em vez de aplicado a um nó que esteja passivamente recolhendo LPT inflacionários.

### Registro de Serviços

Streamflow expande o papel do Registro de Serviços no protocolo onchain. Orquestradores continuarão a propagandear seu `rewardCut`, `feeShare`, e informações de conexão, mas também propagandearão os serviços que seus nós estão oferecendo, e as regiões que estão servindo. Isso levará a impactos de performance e facilitará a _Broadcasters_ buscarem pelos serviços que desejam, sendo servidos por um nó próximo. Orquestradores não mais anunciarão o preço que estão cobrando, uma vez que preço e disponibilidade serão negociados offchain. Quanto aos serviços que podem ser oferecidos, estas são duas abstrações:

1. **Serviço**
    1. Identificador de serviço - o ID que representa um serviço específico, como "CPUTranscoding", "GPUTranscoding", ou "SegmentVerification". Ainda há trabalho a ser feito na definição exata, e é possível que os serviços sejam identificados mais granularmente, como por pares de input/output, por exemplo "H264 1080p -> 720p".
    2. Função verificadora - o endereço que aponta para a função a ser invocada na necessidade de se verificar a correção na execução do serviço (pode ser nulo).

2. **Localização**
    1. A implementação está por ser definida, mas esta é provavelmente uma abstração que especifica uma matriz de regiões servidas pelo nó.
    
A presença desses atributos permitirá a _Broadcasters_ filtrarem o Registro de Serviços atrás de nós que se encaixam na localização a ser servida e tipo de job demandado. Localização foi ignorada previamente na versão _alpha_, mas pode ser crítico para a entrega de vídeo ao vivo que a audiência esteja geograficamente próxima à fonte da transmissão, devido a questões de comunicação de rede, e à potencial latência introduzida por múltiplos "pulos" em conexões de nó para nó.

Como a localização propagandeada por um nó pode ser falsa, como em muitos aspectos de Streamflow, a implementação do _client_ deve rapidamente descobrir e filtrar Orquestradores de baixa performance, custando-lhes o direito a trabalho e taxas futuras. Nós honestos, maximizando suas relações bem sucedidas com _Broadcasters_, taxas e estatísticas de reputação, devem propagandear informação útil de localização, para continuar tendo sucesso em negociações, designações e performance sustentável de jobs.

Conforme nós capturam LPT inflacionário, de modo a pô-los em uso, a coisa mais eficiente a se fazer é adicionar um nó novo ao Registro de Serviços, com a capacidade de servir uma zona ou localização para a qual há demanda, mas ainda não há oferta suficiente ou com bom custo-benefício - expandindo o alcance da rede e a habilidade de servir diferentes casos e clientes.

### Negociação de Jobs Offchain
A mudança de um esquema de designação de jobs on chain para um off chain é talvez a maior das propostas por Streamflow. Muda a assumpção de que jobs são roteados estritamente por _stakes_, e isso será análisado em uma seção subsequente, mas também traz tremendos benefícios. Nomeadamente:

* **Disponibilidade** - _Broadcasters_ poderão garantir que Orquestradores estão disponíveis para trabalhar antes de entrar num acordo com eles.
* **Redundância** - Se um Orquestrador estiver indisponível antes ou durante um job, é só mudar para outro Orquestrador. Ou começar a trabalhar com múltiplos Orquestradores em primeiro lugar.
* **Velocidade** - Comece o trabalho imediatamente. Não é preciso mais esperar por uma confirmação on chain.
* **Eficiência de custo** - Não há mais custos associados à requisição de trabalho pela rede ou gas.

De modo a conduzir uma negociação, um _Broadcaster_ deve interagir com o seguinte protocolo:

1. Ler o Registro de Serviços e escanear todos Orquestradores disponíveis, atrás daqueles que batem com o serviço requisitado e os parâmetros de localização, assim como possuintes do mínimo _stake_ imposto.
2. Usar a informação de conectividade provida para fazer um _ping_ em cada Orquestrador escolhido com um pedido de job.
2.1. O pedido de job contém o serviço requisitado e a localização determinada (opcional).
3. Orquestradores respondem o quão rápido puderem com um preço para performar o job, se quiserem competir por ele e estiverem disponíveis.
3.1. Orquestradores também incluem parâmetros de micropagamentos probabilísticos (PM) na sua resposta (descritos abaixo).
4. _Broadcasters_ coletam esses dados, assim como os tempos de resposta por parte de Orquestradores.
5. Rodar seus algoritmos internos levando em conta preferências quanto ao tempo de resposta, preço, histórico de trabalhos, parâmetros de PM, requerimentos de redundância e segurança na forma de _stake_, de modo a eleger Orquestrador(es) para se trabalhar com.
6. Começar a mandar segmentos de vídeo e tíquetes de PM ao(s) Orquestrador(es) selecionado(s).
7. Orquestrador(es) verifica o depósito onchain do _Broadcaster_ (ver esquema de PM, abaixo), e, se o depósito está acima do mínimo, performa o trabalho, enviando de volta o segmento transcodificado para o _Broadcaster_.

O passo 5 deste protocolo deixa bastante aberto para a implementação. O sumário, aqui, é basicamente que _Broadcasters_ podem escolher seus próprios Orquestradores, e não precisam interagir com a blockchain para anunciar o job que demandam ou tê-lo designado para alguém.

Podem trabalhar com um Orquestrador proprietário se quiserem, e começar a enviar segmentos para outros candidatos somente depois de atingirem sua capacidade máxima. Podem trabalhar com os mesmos nós com os quais mantém relacionamentos de longa data, e trocar para outros somente quando estes estiverem indisponíveis. Podem começar com alta redundância e performance de CPUs para um evento premium ao vivo importante, ou podem escolher a pool de GPUs mais barata ao redor do mundo para um serviço on-demand com requerimentos simples, de modo a cortar custos.

<img src="https://livepeer-dev.s3.amazonaws.com/docs/pricenegotiation.jpg" alt="Offchain Job Negotiation" style="width: 750px">

Adicionar ou subtrair redundância não introduz overhead na forma de custos de transação on chain para _Broadcasters_. Na versão _alpha_, mudança do gênero requeria uma transação e 15-30 segundos para confirmação on chain.

Note que os passos 1-4 podem opcionalmente acontecer no background, de modo frequente, em vez de pontualmente no começo de uma nova transmissão. Se um _Broadcaster_ mantém várias transmissões concorrentemente, ele pode achar que vale a pena manter um feed atualizado de serviços/preços para todos Orquestradores disponíveis na rede, de modo que pode começar a trabalhar com um que julgue atraente a qualquer momento, sem interferência em suas transmissões.

### Micropagamentos Probabilísticos (PM)
O maior impacto de economia de custos em Streamflow vem desta proposta de micropagamentos probabilísticos (PM). Anteriormente, o protocolo usava um flow de deposit() -> job() -> claim() -> verify() -> distributeFees() para liberar pagamentos por trabalho performado. As 3 últimas dessas transações precisavam ser executadas para cada 1000 segmentos de vídeo, na média (ou mais), e fazer 5 transações para um job curto era proibitivo para qualquer Transcodificador.

Para um contexto em PM, sugere-se revisar um artigo pelo time da Orchid Protocol, sobre o uso do esquema numa rede descentralizada de VPN, assim como pesquisa acadêmica prévia [[3, 4, 5](#referências)]. Em suma, o _Broadcaster_ emite tíquetes assinados junto a cada segmento de vídeo enviado ao Orquestrador. O tíquete tem um valor de face alto se ele "for vencedor", e permitir ao Orquestrador trocá-lo por dinheiro on chain. Contudo, a probabilidade de que um tíquete "seja vencedor" é baixa, fazendo com que o valor esperado de cada tíquete converja para o preço por segmento com que o _Broadcaster_ e o Orquestrador concordaram. Ao longo do tempo, _Broadcasters_ pagarão sempre quase o valor exato pelo trabalho solicitado, devido às probabilidades em jogo.

Usando PM, os custos de se coletar pagamentos podem ser aqueles de uma única transação leve, e o valor a ser coletado pode ser agrupado em qualquer quantia desejada pelo Orquestrador. Por exemplo, o Orquestrador pode decidir liquidar seus pagamentos sempre que atingir o equivalente a U$10 em ETH, o que, a um custo de transação da liquidação (derivado de gas) em torno de U$0.10, significa um overhead de 1%. Se os custos de gas aumentarem subitamente em 10x, o Orquestrador pode subir o seu limite mínimo de liquidação para U$100, mantendo o mesmo overhead de 1%, ou pode incorrer um overhead maior se quiser manter a frequência de pagamentos. Assim como a maioria das decisões de design em Streamflow, isso será decidido pelo mercado e configurável pelo client, em vez de imposto pelo protocolo.

A habilidade de um _Broadcaster_ pagar, no momento em que um Orquestrador efetua uma liquidação, é assegurada por um depósito onchain, "time-locked", com uma cláusula programada de penalidade.

Devido à negociação off chain e a potenciais redundâncias que um _Broadcaster_ pode requerir, eles podem mandar tíquetes de PM para vários Orquestradores de uma vez, começar ou parar trabalho com um Orquestrador a qualquer hora, e, do mesmo jeito, Orquestradores podem encerrar a qualquer momento um trabalho para um _Broadcaster_ se determinar que não está sendo pago corretamente ou quiser ficar offline. 

Isto muda o modelo mental prévio de que um job era uma transmissão inteira e contínua, fazendo-nos enxergá-lo agora somente como um segmento de vídeo junto a um tíquete de PM.

O esquema de PM completo está no [apêndice](#apêndice), uma vez que tangencia questões de verificação, detalhes da negociação off chain, e outras áreas como riscos de gastos duplos e mitigações.

### Verificação On Chain Baseada em Faltas
A última grande mudança proposta por Streamflow é o ajuste no protocolo de verificação para reduzir custos e contornar o problema da disponibilidade de dados. Previamente, Transcodificadores precisavam invocar verificação via Truebit para 1 de cada `VerificationRate` segmentos, o que foi setado originalmente a 1 de cada 1000. Isso se mostrou caro, e era necessário tivesse o Transcodificador performado o trabalho corretamente ou não. A proposta nova é a de que:

* _Broadcasters_ são responsáveis por verificar segmentos transcodificados recebidos, e só desafiarão eles via Truebit on chain se acreditarem que o segmento está comprometido.
* Se Truebit (ou outra função apropriada de verificação on chain) "concordar", então o _stake_ do Orquestrador é devidamente punido, sendo que o _Broadcaster_ recebe um bônus significante.

<img src="https://livepeer-dev.s3.amazonaws.com/docs/faultverification.jpg" alt="Fault Based Verificaiton">

Parte do argumento contra este método é o de que o _Broadcaster_ não tem recursos computacionais suficientes para re-codificar vídeos e checar se o job foi feito corretamente ou não. Usando o mesmo _approach_ randomizado do protocolo original, no entanto, o _Broadcaster_ pode checar 1 a cada `VerificationRate` segmentos, conforme escolher. Pode checar mais, se requer mais confiabilidade, ou pode terceirizar a checagem para qualquer outro nó na rede, pagando-lhe para conferir o trabalho em seu nome (mesmo que este não saiba distinguir uma re-codificação de um job original) - o equivalente a contratsar um segundo Orquestrador para 1 a cada `VerificationRate` segmentos. Pode usar um Orquestrador barato para o trabalho principal, mas depender de um Orquestrador mais caro, reputável, como um verificador confiável. Há também checagens mais baratas que podem ser feitas analisando quadros do vídeo resultante em vez de re-codificar ele por completo, assim como verificação baseada em métricas. Essas checagens mais baratas podem ser usadas preliminarmente, apontar prováveis faltas, e só em caso de sinais positivos, engatilhar uma re-codificação de verificação, e então um desafio via Truebit.

O ponto chave, no entanto, é que o Orquestrador nunca sabe qual segmento será desafiado, e deve esperar que qualquer um esteja sujeito a isso, estando suscetível à perda de uma quantidade relevante de _stake_. Os benefícios de um comportamento desonesto devem exceder o valor de uma punição sobre o depósito de segurança (_stake_) fixado, o que é improvável para a grande maioria dos casos de uso. Adicionalmente, conforme _Broadcasters_ usem redundância, qualquer detecção de inconsistência ou suspeita de falha pode engatilhar a troca imediata para outros Orquestradores.

Um impacto disso é que o custo da resolução de disputa via Truebit não incorre, exceto em casos de desonestidade óbvia - o que deve acontecer poucas vezes, uma vez que não valerá a pena para um Orquestrador intencionalmente "burlar o sistema". Isso torna a rede mais barata de usar que o custo de uma invocação via Truebit a cada `verificationRate` segmentos de vídeo

## Análise Econômica #################################

As mudanças propostas em Streamflow levam a incentivos e comportamentos sutilmente diferentes por parte de Orquestradores e delegadores na rede. Trazem, de forma geral, mais escala, confiabilidade e eficiência de custo. Essa seção inicia uma análise de impacto econômico das mudanças propostas, incluindo um olhar sobre o papel do token Livepeer (LPT), o papel da delegação, como a inflação molda a rede, e outras considerações off chain.

### Livepeer Token

O token Livepeer (LPT) pode ser descrito como um "token de trabalho" (_work token_). Aqueles que o aplicaram (_stake_) ganharam a oportunidade de performar trabalho na rede, e consequentemente ganhar taxas (_fees_) em ETH pelo trabalho em questão. O trabalho sempre foi roteado em proporção direta aos _stakes_, assumindo equivalência constante de preços entre nós (na prática, o trabalho não é distribuído uniformemente). 

Desde o início, há uma espécie de "requisição compulsória por trabalho" em prática, no sentido de que, se um nó não performasse trabalho suficiente em proporção a seu _stake_, poderia ser punido. Isto era uma tentativa de se garantir que nós contribuiriam valor para a rede de forma contínua, em vez de simplesmente sentar sobre uma porção inerte de _stake_ e receber proventos inflacionários. Vale notar que não havia requerimento de que o trabalho fosse eficiente em termos de custo, ou de alta performance. A competição tinha como ser socialmente encorajada, mas não imposta a nível do protocolo.

Os updates de Streamflow que relaxam o limite de Orquestradores ativos e que introduzem um mecanismo de negociação de jobs off chain aparentam alterar essa conexão direta entre o token e o direito de performar trabalho, mas uma análise minuciosa revela que o mesmo valor é capturado por possuintes do token, num estado de equilíbrio. Olhemos para a função que quem possui tokens LPT tende a tentar maximizar:

`Valor capturado em uma única rodada = recompensa inflacionária (LPT) + taxas ganhas.`

LPT inflacionários são previsíveis, baseados na `rewardCut`do Orquestrador. Um delegador pode escolher exatamente quanto LPT gostaria de ganhar em troca pelo serviço de garantia de qualidade (QA) que provém.

Por outro lado, as taxas ganhas estão menos sob controle do dono de tokens. Isso ocorre porque estas dependem de:

1. Quanto trabalho o Orquestrador performa;
2. A `feeShare` do Orquestrador;
3. Quanto do _stake_ total está delegado para o Orquestrador em questão, e, por consequência, quantos porcento da pool de taxas (_fee pool_) ele tem direito a.

Quando completa uma rodada, o delegador poderá calcular o potencial de renda dos LPT que tem aplicados (em _stake_). Deriva dessa razão:

`ETH em taxas / unidades de LPT aplicados (staked)`

O que será uma estatística visível na qual delegadores podem se basear para comparar Orquestradores e tomar decisões. Previsivelmente, delegações devem fluir, entre uma rodada e outra, para nós que apresentem oportunidades de se maximizar essa razão. Em suma, porque se ater a um nó que redistribui 1gwei por LPT aplicado (_staked_), quando há outros nós dispostos a redistribuir 2gwei por LPT aplicado?

<img src="https://livepeer-dev.s3.amazonaws.com/docs/feeratio.jpg" alt="Fee Ratio">

Vale notar que o ato de se realocar _stake_ para outro nó com melhor custo de oportunidade vai significar que as _fees_ serão divididas entre uma quantia maior de _stake_, o que fará a razão (acima) decair. O estado de equilíbrio é aquele em que nós que estão performando mais trabalho (recebendo mais) tem mais _stake_, e nós performando menos com a mesma `feeShare` tem menos _stake_. Essencialmente todos os nós competitivos devem convergir para a mesma razão, com delegadores inteligentes alocando _stake_ de modo a obter um retorno também em estado de equilíbrio. No fim, a delegação inteligente de LPT inteligentemente gera acesso a trabalho que dá direito ao ganho de taxas, independentemente de como jobs são designados ou roteados na rede.

### Delegação como um Sinal de Segurança e Reputação
Uma consequência negativa que se pode antecipar a partir das mudanças propostas é a de que nós que estão recebendo bastante trabalho podem prover 0% `feeShare`, não atrair delegação alguma, e mesmo assim continuar recolhendo boa parte do valor circulado na rede. Isto é OK - estes estariam operando hardware e incorrendo custos, além de provendo um serviço valoroso -, talvez nem precisem de delegadores.

Por outro lado, delegação provém segurança adicional - é mais _stake_ suscetível a punições no caso de comportamento desonesto por parte do nó, mais sinal reputacional. Clients usam esse sinal para selecionar nós com os quais se relacionar, então um nó competitivo, anunciando um `feeShare` > 0% deve atrair mais _stake_ delegado, e portanto trabalho - desde que consiga realizá-lo tão ou mais competitivamente que nós que estejam oferecendo 0% `feeShare`. De novo, isso contribui para a flexibilidade de stups e casos de uso da rede. Aumenta a oportunidade para competição, descentralização, diversidade e resiliência.

Conforme novos nós procurem competir por trabalho na rede, deverão atrair _stake_ suficiente para oferecer a segurança requerida por _Broadcasters_. No caso de novos entrantes, é provável que nós estreiem oferecendo `feeShares` mais generosos. Delegadores ativos poderão buscar e aplicar em nós que estejam sendo designados boa porção dos trabalhos correntes, oferecendo  uma `feeShare` atrativa, o que resulta em uma razão maior de retorno por unidade de capital aplicado. Em suma, _stake_ delegado pode prover segurança e rotear trabalho, em troca de taxas redistribuídas quando o trabalho é performado com sucesso. A delegação ativa pode levar nós oportunistas a expandirem o alcance e as capacidades da rede através de pura competição.

### Inflação que Vira _stake_ Aplicado e Delegadores Apáticos
Um dos criticismos ao modelo de _stake_ ilimitado sem barreira mínima é o de que ele dá espaço para comportamento preguiçoso por parte de delegadores. LPT inflacionários continuam a se acumular sobre _stake_ já aplicado, compondo exponencialmente, e facilitando ao delegador o hábito de configurar sua delegação e esquecer da rede, não necessariamente provendo valor subsequente a ela.

Este talvez seja o caso nos primórdios da rede, antes que _fees_ sirvam como incentivo adicional para que delegadores tomem ação, mas o comportamento não deve gerar retornos ótimos quando Orquestradores estiverem competindo por taxas e as redistribuindo. Neste ponto, o comportamento automático ainda deve levar ao acúmulo de LPT, mas abriria mão do potencial de se capturar parte das taxas fluindo pela rede, o que se conquistaria caso se tomasse a ação de realocar o _stake_ para Orquestradores que estiverem oferecendo retorno mais atrativo.

Conforme a inflação decai, uma porção maior das recompensas finais de um nó qualquer devem advir de taxas. E, conforme mencionado na seção anterior sobre o token LPT, o constante QA e re-roteamento de trabalho em direção a nós mais performantes é motivado financeiramente pela oportunidade de retorno. Em suma, um delegador apático é recompensado menos que um delegador ativo.

Adicionalmente, Orquestradores que outrora tinham de atrair _stake_ de delegadores terceiros de modo a atingir um _stake_ mínimo, caso tenham _stake_ suficiente para assegurar seu próprio nó, podem diminuir seu `feeShare`. Neste estágio, um delegador em busca de otimização seria melhor servido por um nó recém-entrante - que possa expandir o alcance da rede e ainda esteja oferecendo um `feeShare` atrativo de modo a receber _stakes_. E este QA constante no cerne da otimização a ser efetuada por cada delegador, e o tradeoff _stake_-por-_fee_ que criará competição constante e avançará a descentralização na rede.

### Considerações de Engenharia Off Chain
Como previamente mencionado, uma das filosofias centrais a guiar Streamflow é a de mover o máximo possível de opiniões sobre parâmetros válidos e interações p2p para fora do protocolo _core_ e para dentro das implementações de cada client e suas configurações. Múltiplas implementações e configurações desses parâmetros levarão a uma rede robusta e resiliente a ataques. No entanto, uma vez que o protocolo em si é menos opinionado, bastante fica em aberto para implementações específicas. Aqui estão algumas das principais considerações a ser ponderadas de uma perspectiva de engenharia, para tornar Streamflow efetivo "num estalar de dedos":

* Políticas de gestão de risco no esquema de PM - quando um Orquestrador deve ou não trabalhar com um _Broadcaster_, com base em reputação e histórico - e vice-versa.
* Geração segura de números randômicos para o esquema de PM.
* Resistência contra ataques DDoS, para Orquestradores.
* Redundância e algoritmos de _failover_ para _Broadcasters_ em diferentes cenários e casos de uso.
* Estratégias de descoberta de preço, para _Broadcasters_.
* Protocolos para transmissões de baixa latência ou assinatura/verificação de pagamento quando o segmento final não está disponível antes que o trabalho precise ser iniciado.

Cada um dos itens acima pode afetar a eficiência da rede na perspectiva de um _Broadcaster_ - daí as necessárias redundâncias, e custos relativos. A boa notícia é que grande parte dos itens pode ser atacado com estratégias off chain, constantemente iteradas através de diferentes implementações. Uma rede suportada por agentes configurando-se independentemente é mais imprevisível e difícil de se otimizar um ataque contra do que uma rede de implementação uniforme.

## Ataques ###############################
Alguns dos subprotocolos, como o esquema de PM e a verificação via Truebit, são sujeitos a seus próprios ataques, o que deixaremos para análise nas respectivas linhas de pesquisa, fora desse paper. Abaixo, segue uma breve descrição de alguns potenciais ataques e contramedidas dentro da economia de Streamflow.

### _Squeezing_ de Delegadores
Quando um candidato a Orquestrador quiser operar um nó e expressar candidatura, ele deve atrair delegações de modo a atingir o mínimo _stake_ especificado por diferentes clients, e receber trabalho. Para fazê-lo, deve apresentar um `RewardCut` e um `FeeShare` atrativos. No entanto, conforme o nó começa a trabalhar, e a receber LPT inflacionários, pode usar esse LPT para aplicar mais _stake_ ao seu próprio nó, indefinidamente, reduzindo a quantidade de inflação e taxas que devem ser redistribuídas para os delegadores. Assim, podem acabar "espantando" delegadores ao levarem suas `RewardCut` e `FeeShare` a níveis menos atrativos, para então preencher o vazio com seu próprio _stake_ e continuar operando.

Isso é teoricamente OK, uma vez que delegadores podem realocar seu _stake_ para nós mais atrativos como bem entenderem. Infelizmente, cria um problema de UX, já que delegadores precisam estar constantemente vigilantes para garantirem que estão operando para seu melhor interesse. A cada rodada, o cenário pode mudar.

Uma possibilidade é a de que Orquestradores que desejam operar nós adicionais, se seguirem esse caminho, tenham a reputação manchada e percam a capacidade de atrair _stake_ para continuar competitivos. 

### Roubo da Taxa do Delegador

Como mencionado no ataque acima, é possível para um Orquestrador "espantar" seus delegadores. Essa pode ser uma técnica particularmente maliciosa se o Orquestrador também se agarrar a tíquetes de PM vencedores até o ponto em que o delegadore o abandonam, então os liquida quando contém todo o _stake_ reservado para si. Essencialmente as _fees_ e recompensas que são do delegador, de direito, ficariam de posse do Orquestrador.

Isso pode ser prevenido com a inserção de datas de expiração nos tíquetes de PM, que ocorrem antes da data para retirada do depósito _time-locked_ do Broadcaster. Sendo assim, o tíquete precisaria ser liquidado quase imediatamente, e potencialmente conteria a _feeShare_ comprometida pelo Orquestrador no momento, de modo que a liquidação de um tíquete vencedor possa fazer a divisão correta entre delegadores e o Orquestrador para o qual delegam.


## Áreas Abertas de Pesquisa ############################

Assim como no caso de quaisquer outros protocolos manifestados em blockchains, há diversas frentes de pesquisa que devem ser exploradas antes que o sistema amadureça e atinja alto grau de descentralização, independência de confiança, e eficiência econômica. Abaixo, estão algumas áreas nas quais o projeto está ativamente conduzindo atividade de P&D. A participação da comunidade é encorajada.

### Verificação Não-Determinística

Nossos esforços de P&D seguem na direção de verificar a probabilidade de que um segmento encodado por GPU represente o mesmo conteúdo que o segmento pre-encodado. Esse _approach_ probabilístico tem se mostrado, em estágio incipiente de pesquisa, capaz de prover medidas acuradas, mas a adequação para determinar, de fato, punições econômicas no protocolo, é discutível e requer mais experimentação e testes.

Encodificação determinística, por outro lado, continua podendo ser checada por uma variedade de esquemas, incluindo Truebit, verificação baseada em hardware SGX, oráculos ou até mesmo verificadores terceiros confiados.

### Protocolos para Pools Públicas de Transcodificadores

A divisão entre as responsabildiades de Orquestradores e Transcodificadores deve escalar dramaticamente a operação de nós na rede Livepeer. Permite o uso de hardware ocioso, sem requerir que toda máquina esteja ciente do protocolo ou ligada 24/7, muito menos com _stake_ aplicado. Acreditamos que pools privadas, onde Orquestradores também operem seus próprios hardware e Transcodificadores, serão as com melhor eficiência de custo, dado que o Orquestrador pode confiar no resultado do trabalho e não incorre custo algum de verificação.

Pools públicas, onde o Orquestrador não necessariamente confia nos Transcodificadores, podem ser poderosas em aproveitar recursos ociosos, sem depender de infraestrutura dedicada. No entanto, uma vez que nós remotos não podem ser confiados cegamente, o Orquestrador incorre custos associados à verificação de trabalho, senão corre risco de punição sobre seu _stake_. Por isso, a eficiência de custo tende a ser menor que a de pools privadas - a menos que protocolos econômicos possam garantir a segurança dessas pools públicas na forma de _stakes_. Caso se possa provar que um Transcodificador em particular foi a causa de uma punição invocada, e este tem _stake_ suficiente para ser punido individualmente, então pools públicas tem um caminho viável para a competitividade.

O espaço de design, aqui, é uma área de P&B em aberto.

### Mitigando Gastos Duplos de _Broadcasters_

Em um esquema de pagamentos probabilísticos, há sempre a chance de que um _Broadcaster_ tenha emitido mais tíquetes vencedores do que seu balanço atual permite pagar de fato (acidentalmente). Uma vez que Orquestradores talvez não notifiquem _Broadcasters_ sobre tíquetes vencedores imediatamente após descobri-los, é difícil ter uma contabilidade precisa e em tempo real do balanço de um _Broadcaster_. Nosso P&D continua explorando parâmetros e mecanismos de gestão de depósitos para evitar gastos duplos acidentais no caso de certos padrões de uso da rede. Veja uma análise mais detalhada no Apêndice.

### Pagamentos VOD

Uma das propriedades que _Broadcasters_ podem buscar quando se trata de transcodificação de vídeo on-demand é a possibilidade de enviar conteúdo, requerir um job, e desaparecer - de modo que o Orquestrador possa performar o trabalho de forma assíncrona, distribui-lo por diferentes nós, ou agendá-lo para quando tiver recursos ociosos.

No entanto, com o esquema de PM descrito nesse paper, o _Broadcaster_ precisa estar online para, continuamente, enviar tíquetes conforme o conteúdo é enviado. Parte da segurança depende do reconhecimento de que, se um Orquestrador não continuar um certo trabalho, não há problema, já que o _Broadcaster_ pode simplesmente parar de enviar pagamentos.

No caso de jobs VOD, contudo, se um _Broadcaster_ pagar (enviar tíquetes) antecipadamente para todos os segmentos de um vídeo e então desaparecer (ficar offline), não há garantia de que o Orquestrador vai performar o job ou responder os segmentos transcodificados. Por ora, transcodificação de vídeos on demand é possível, mas a funcionalidade de se "subir conteúdo e desaparecer" não é. Nosso P&D continuará na direção de permitir pagamentos por serviços VOD em escala.

## Caminho de migração ############################

A proposta Streamflow está em fase incial no ciclo de pesquisa, design, _feedback_ e implementação. Merece certamente crítica da comunidade, testes, auditorias, e aceitação, antes de entrar em produção na main net da Ethereum como a próxima iteração do protocolo da Livepeer. Essa seção objetiva listar algumas considerações com relação ao modo pelo qual uma migração do protocolo pode ocorrer:

* Novas lógicas de contratos inteligentes seriam implementadas na Ethereum, mas se antecipa que muito pouco ou nenhuma migração de dados se faria necessária. O mecanismo de update (já existente) proxy-delegateCall pode ser utilizado.
* O estado atual no protocolo, incluindo balanços, _stakes_, taxas, recompensas e delegações, seria mantido.
* Transcodificadores, _Broadcasters_, Orquestradores e delegadores fariam update no client que estão rodando, o que lhe conferiria funcionalidades para negociação de jobs, configurações de redundância, pagamentos e o novo mecanismo de verificação.
* Orquestradores registrariam quaisquer novos parâmetros no Registro de Serviços, incluindo o suporte a novas demandas e, possivelmente, localizações. 
* _Broadcasters_ estabeleceriam contratos de PM e gerenciariam depósitos. Depósitos existentes poderiam migrar do `Minter` para o contrato de PM via ação do próprio usuário, mediante uma requisição. 
* Antecipa-se que isso pode ser realizado com pouco ou nenhum _downtime_ na rede.
* Clients mantidos independentemente, assim como exploradores do protocolo e ferramentas de _analytics_ terão de fazer updates para refletir as novas interações abrangidas pelo protocolo.

Um caminho de migração formal, listagem de passos, e observações derivadas de experimentação em test net serão disponibilizados com o tempo, conforme Streamflow amadurecer e sua data de lançamento se aproximar.

## Sumário #################################

À guisa de conclusão, as propostas contidas nesse documento miram jogar luz sobre um caminho escalável para a evolução da rede Livepeer. Um caminho que descase o custo de se usar a blockchain da Ethereum do custo de se usar a infraestrutura de vídeo descentralizada, e que ofereça a desenvolvedores de vídeo a confiabilidade e performance de que necessitam.

Todo _feedback_, ideias e inputs são bem-vindos, então não hesite em deixar uma mensagem no [Fórum da Livepeer](https://forum.livepeer.org) ou no [Chat no Discord](https://discord.gg/RR4kFAh) para participar da conversa.


## Apêndice ################################

### Apêndice A: Esquema de Micropagamentos Probabilísticos

* O depósito de segurança de um Orquestrador é o seu _stake_. Este pode ser tomado, na eventualidade de uma punição, se o Orquestrador for desonesto e falhar uma verificação (cometer uma "falta").
* Um _Broadcaster_ (usamos esse termo para o usuário comum, que pode ser mais um desenvolvedor que um Youtuber propriamente dito) efetua um depósito _time-locked_ para cobrir o trabalho futuro a ser demandado da rede pelo qual ele terá de pagar.
* O _Broadcaster_ quer seu vídeo transcodificado. Ele lê o Registro de Serviços on chain, e negocia off chain com Orquestradores que satisfazem seus requerimentos:
   * Orquestradores anunciam seus preços.
   * Orquestradores provém parâmetros de PM - estes podem variar de acordo com as condições atuais da Ethereum. Por exempplo, podem configurar o valor de face do tíquete vencedor de modo que o custo de se liquidar um pagamento seja menor que 1% do valor a ser recebido.
* O _Broadcaster_ envia segmentos de vídeo para o(s) Orquestrador(es) com quem decidiu trabalhar, junto a um tíquete de PM para cada segmento.
   * O tíquete de PM engloba um protocolo interativo desenhado para prevenir que cada um dos dois agentes interfira na fonte de randomicidade usada para determinar se um tíquete é vencedor ou não. Depois de todo tíquete vencedor, o Orquestrador precisa gerar um novo número randômico e mandar um "compromisso" assinado atrelado a ele para o _Broadcaster_. O _Broadcaster_ e o Orquestrador já deverão estar trocando dados entre si - isto só somaria uma mensagem adicional ao fluxo, a ser enviada pelo Orquestrador cada vez que um "compromisso" novo é preciso. Outra otimização possível é usar uma função de randomicidade verificável (VRF) implementada num contrato autônomo. O Orquestrador entregaria ao _Broadcaster_ uma _pub key_, e assinaria tíquetes recebidos com a chave privada correspondente. O contrato VRF verificaria a validade da assinatura, usando-a então como fonte para a pseudo-randomicidade. Como resultado, o protocolo se torna não-interativo. Seria preciso avaliar a viabilidade de se implementar um contrato para VRFs, assim como o custo de se verificar o tipo de assinatura supracitado - não é o foco, por ora, mas não deixa de ser uma possibilidade.
* O _Broadcaster_ recebe o resultado transcodificado do Orquestrador.
   * O _Broadcaster_ pode verificar qualquer segmento que quiser checar.
   * Se a checagem não for bem sucedida, pode-se ativar a resolução de disputa via Truebit e punir o Orquestrador, rendendo uma recompensa (derivada do _stake_ punido) para o _Broadcaster_.
* Se o _Broadcaster_ não receber o trabalho de volta do Orquestrador, ele simplesmente para de lhe enviar mais segmentos, e troca para outro Orquestrador.
* Se o Orquestrador não receber tíquetes de PM válidos, ele simplesmente não performa o trabalho e não responde nenhum resultado.
   * O protocolo conterá mensagens para certas condições de erro, tais como `LowPMBalance` ou `SegmentFormatDidntMatchJobInputParams`, de modo que _Broadcasters_ tenham informação útil com a qual contornar empecilhos, ou para suportar decisões como reabastecer um balanço.
* Orquestradores monitoram depósitos de _Broadcasters_ e atribuem um risco de inadimplência a cada perfil.
   * Algoritmo simples para começar: se o balanço é baixo demais, parar de performar trabalho para o cliente em questão.
   * Orquestradores liquidam tíquetes vencedores conforme os recebem (ou aguardam até que o gas seja mais barato).
    
Para uma análise e especificação completa das estruturas de dados dos tíquetes, prevenção a gastos duplos, e outras considerações de design, veja esse [documento externo](https://hackmd.io/uHMFeNSyS_GyzwnO3Ld74A?view).

## Referências ###########################################

1. Livepeer Whitepaper - Doug Petkanics, Eric Tang - <https://github.com/livepeer/wiki/blob/master/WHITEPAPER.md>
2. The Video Miner, A Path To Scaling Video Transcoding -Philipp Angele  - <https://medium.com/livepeer-blog/the-video-miner-a-path-to-scaling-video-transcoding-a3487d232a1>
3. Ethereum Probabilistic Micropayments - Gustav Simonson - <https://medium.com/@gustav.simonsson/ethereum-probabilistic-micropayments-ae6e6cd85a06>
4. Electronic Lottery Tickets as Micropayments - Ron Rivest - MIT Lab for Computer Science - <https://people.csail.mit.edu/rivest/pubs/Riv97b.pdf>
5. Decentralized Anonymous Micropayments - A. Chiesa, M. Green, J. Liu, P. Miao, I. Miers and P. Mishra - <https://eprint.iacr.org/2016/1033.pdf>
