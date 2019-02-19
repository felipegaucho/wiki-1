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
    * [Negociação de Jobs Offchain](#negociação-de-jobs-offchain)
    * [Micropagamentos Probabilísticos (PM)](#micropagamentos-probabilísticos-(PM))
    * [Verificação On Chain Baseada em Faltas](#verificação-on-chain-baseada-em-faltas)
* [Economic Analysis](#economic-analysis)
    * [Livepeer Token](#livepeer-token)
    * [Delegation as Security and Reputation Signal](#delegation-as-security-and-reputational-signal)
    * [Inflation Into Bonded State and Apathetic Delegators](#infaltion-into-bonded-state-and-apathetic-delegators)
    * [Offchain Engineering Considerations](#offchain-engineering-considerations)
* [Attacks](#attacks)
    * [Delegator Squeezing](#delegator-squeezing)
    * [Delegator Fee Theft](#delegator-fee-theft)
* [Open Research Areas](#open-research-areas)
    * [Non Deterministic Verification](#non-deterministic-verification)
    * [Public Transcoder Pool Protocols](#public-transcoder-pool-protocols)
    * [Broadcaster Doublespend Mitigation](#broadcaster-doublespend-mitigation)
    * [VOD Payments](#vod-payments)
* [Migration Path](#migration-path)
* [Apêndice](#apêndice)
    * [Appendix A: Probabilistic Micropayments Workflow](#appendix-a-probabilistic-micropayments-workflow)
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
* Um mecanismo offchain de negociação de preços e designação de jobs entre _Broadcasters_ e Orquestradores.
* Micropagamentos probabilísticos offchain, com liquidação e depósitos de segurança onchain.
* Update no esquema de verificação, em que a verificação onchain só precisa ocorrer no caso da observação de uma falta.

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
2. **Determinar um stake mínimo para se tornar um Orquestrador**: Isso estabeleceria um N máximo possível, ao passo que permitiria a qualquer um saber exatamente o quanto é preciso para se atingir o "piso de segurança" para se manter no grupo ativo. Também permitiria a expansão orgânica do limite ao passo que LPT inflácionários são gerados, encorajando delegadores a buscarem novos Orquestradores que potencialmente estejam oferecendo condições mais atrativas para entrar ou permanecer no grupo ativo e competir por trabalho.
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

Parte do 

🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯 
🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯🎯

Part of the argument against this method is that the Broadcaster doesn’t have significant compute resources to re-encode video to check whether the job was done correctly or not. Using the same randomized approach as the original protocol however, the Broadcaster can check 1 out of `VerificationRate` segments should it choose to. It could check more if it requires more reliability, or it could outsource the checking to another node on the network and pay that node to check efficiently on its behalf - the equivalent of hiring a second Orchestrator just for one out of `VerificationRate` segments. They could be using a cheap Orchestrator for the main work, but rely on the high reputation high cost Orchestrator as a more trusted verifier. There are also far cheaper checks that can be done by analyzing frames of the output video rather than fully re-encoding, such as metrics-based verification. These cheap checks can be used to test whether there is a likely fault, and only in that case then re-encode before bringing the challenge to Truebit.

The key point however, is that the Orchestrator doesn’t know which segments will be challenged, and should any segments fail verification, it stands to lose a tremendous amount of stake. The benefits of cheating would have to exceed the value of a fully slashed fixed stake deposit, which is unlikely for many use cases. Additionally, as Broadcasters may use redundancies, should it detect an inconsistency or have suspicion of cheating from a cheap check-without-re-encoding operation, it could simply choose to work with another Orchestrator on that segment in order to get a proper encoding and insert it into its playlist. 

One impact of this is that the cost of Truebit doesn’t need to be incurred, except in the case of obvious cheating - and hence almost never, since it should never be worth it for an orchestrator to intentionally cheat. This makes the network far cheaper to use, than the cost of invoking Truebit on every `verificationRate` segments of video. 


## Economic Analysis #################################

The changes proposed by Streamflow lead to slightly different incentives and behaviors for both Orchestrators and Delegators, resulting in what will be a more scalable, reliable, cost effective network. This section begins an economic impact analysis of these proposed changes, including a look at the role of the Livepeer Token, the role of delegation, how inflation effects the network, and some offchain economic considerations.

### Livepeer Token

The Livepeer Token (LPT) could always be described as a work token. Those who staked it had the opportunity to perform work on the network, and therefore earn the future fees (in ETH) for doing said work. Work was routed in direct proportion to stake, if prices offered by all nodes were constant. There were conceived mechanisms from the beginning for a “work requirement”, in that if a node did not perform enough work within some threshold proportion of their stake, then they could be slashed. This was an attempt at ensuring that nodes would actually contribute value (or incur overhead tax for not doing so or faking it), rather than just sit idly on stake and accrue inflation. In addition, there was no requirement that work be done cost effectively or in a performant manner. Competition could be socially encouraged, but not enforced at a protocol level.

The updates to the protocol to relax the artificially constrained number of Orchestrators, and the offchain job negotiation appear to change this direct connection between token and the right to do work on the surface, but upon further analysis, the same value accrues in an equilibrium state. Let’s look at the function that a token holder is attempting to maximize:

`Value accrued in a single round = inflationary LPT earned + fees earned.`

The inflationary LPT is predictable, based upon the rewardCut of an orchestrator. A Delegator can choose exactly how much LPT they would like to earn in exchange for the QA work they are doing. 

The fees earned on the other hand are less in control of the token holder. This is because it depends on: 

1. How much work their Orchestrator performs
2. The Orchestrator’s `feeShare`
3. How much total stake is delegated towards the Orchestrator, and therefore what percent of the fee pool they are entitled to

At the completion of a round, a Delegator will be able to calculate the earning power of their staked LPT. It’s this fee ratio:

`ETH in fees / unit of staked LPT`

Which will be the visible statistic that Delegators can use to compare Orchestrators to one another, and predictably, delegation should shift from round to round towards nodes where there is opportunity to maximize this ratio. In short, why stick with a node who’s sharing out 1gwei /  LPT staked when there’s another node you could switch to that is sharing out 2 gwei / LPT staked?

<img src="https://livepeer-dev.s3.amazonaws.com/docs/feeratio.jpg" alt="Fee Ratio">

But then it is worth noting that the act of switching more stake onto this opportunistic node, means that the fees will be split amongst more stake, and the fee ratio will decrease. The equilibrium state is that nodes who are performing more work (earning more) have more stake, and nodes performing less work with same fee share have less stake. Essentially all competitive nodes should end up with the same equilibrium fee ratios, with intelligently delegated stake earning a Delegator the equilibrium return - and hence staked LPT intelligently applied yields access to do work to earn fees on the network independently of how jobs are assigned.

### Delegation as Security and Reputational Signal

One negative outcome people could foresee is that nodes who are winning a lot of work could provide 0% fee share, and hence not attract any delegation. This is ok - they are running hardware and incurring costs, and providing great service to the network - they may not need delegation. But delegation on the other hand provides additional security - it is more stake that can be slashed if the node cheats - more reputational signal. Clients use this signal to select nodes to work with, and so a competitive node advertising a > 0% fee share would be more likely to attract stake, and hence work - as long as they can perform it competitively or better or cheaper than the 0% fee share node. Again, this contributes to the flexible setups and use cases of the network. It increases the opportunity for competition, decentralization, diversity, and resilience of the network.

As new nodes are looking to compete to do work on the network, they may need to attract enough stake to offer the security required by Broadcasters. In these cases, it is likely that these nodes would set a greater fee share. Active delegators will have the opportunity to search for and stake towards nodes that are winning outsized portions of works, with greater fee shares, resulting in higher fee ratios. In short, delegated stake can provide security and route work, in exchange for fees shared back when the work is performed well. Active delegation can lead to giving more opportunistic nodes the ability to expand the footprint and capabilities of the network in a competitive way.


### Inflation into Bonded State and Apathetic Delegators
One of the criticisms of the uncapped stake model with no minimum stakes is that it enables lazy behavior on behalf of the delegators. Inflationary LPT continues to accrue into the bonded state, continues to compound, and allows a delegator to set-it-and-forget-it while collecting LPT without adding significant value to the network. 

This may be the case in the very early days of the network, before fees serve as an additional incentive for delegators to take action, but is unlikely to yield a maximal result when Orchestrators are competing to do work, earn, and distribute fees. At this point, autopilot behavior may still lead to accruing LPT, but would be forgoing the potential fees that could be earned by switching to Orchestrators who are yielding a higher ETH/staked LPT ratio. 

As the inflation rate is likely to decrease under scaled usage, when token holders are staking to compete to earn the fees, the portion of the reward function that is accounted for by inflationary LPT also continues to decrease, with a great portion coming from fees. And as outlined above in the LPT section, the necessity to constantly QA the network and route work towards nodes who are outcompeting other nodes is financially motivated by the opportunistic returns. In short, an apathetic delegator is rewarded less than an active delegator.

Additionally, as Orchestrators who once needed to attract outside delegation in order to achieve the minimum stake, accrue enough stake themselves to secure their own node, they may decrease their fee share. At this stage, an optimizing delegator would be best served by seeking out a new up-and-coming node - essentially one who can expand the footprint of the network - who may be offering a higher fee share in order to attract stake. It's this constant QA performed by the optimizing Delegator, and stake-for-fee tradeoff which will create constant competition and further the decentralization of the network.

### Offchain Engineering Considerations
As previously mentioned, one of the core philosophies within Streamflow is to move many of the opinions about valid parameter values and p2p interactions out of the core protocol and into client implementations and configurations. Multiple implementations and configurations of these parameters will lead to a robust network that is resillient to attacks and malicious actors. However since the protocol itself is less opinionated, a lot is left up to client implementation. Here are some of the major considerations that need to be undertaken from an engineering perspective to make Streamflow work really effectively out of the box:

* PM risk management policies - when an Orchestrator should work with or not work with a Broadcaster based upon reputation and history, and vice versa.
* Secure random number generation for PM protocol.
* DDoS resistance for Orchestrators.
* Redundancy and failover algorithms for Broadcasters under different scenarios and use cases.
* Price discovery strategy for Broadcasters.
* Low latency streaming protocols and signature/payment verification when final segment isn't available before work needs to begin.

Each of the above can effect the efficiency of the network from the perspective of a Broadcaster - and hence the necessary redundancies, and eventually costs. The good news is that much of the above can be handled via off chain strategies, and can be constantly experimented with across different competing implementations or configurations. A network that has agents acting in different and unpredictable ways is harder to optimize for an attacker who would otherwise be looking to game a single implementation. 


## Attacks ###############################
Some of the specific sub-protocols, such as PM’s and Truebit based verification are subject to their own attacks, which we leave for analysis within those areas of research. Here is some brief discussion of potential attacks and countermeasures within the economics of the Streamflow changes to the protocol.

### Delegator Squeezing

When a candidate Orchestrator would like to operate a node and express their candidacy, they may need to attract delegation in order to reach the client specified minimum deposit amount to attract mainstream work. To do so, they may represent an attractive `RewardCut` and `FeeShare`. However, as their node begins to perform work, and they start to earn inflationary LPT, they may wish to use this LPT to stake more towards their single node in order to reduce the amount of inflation and fees they need to share with their delegators. To do so, they may drive off current delegators by manipulating their `RewardCut` and `FeeShare` to an unattractive point, and then filling the gap with their own stake.

This is theoretically ok, as delegators can move on to more attractive nodes and adjust in their best interest. Unfortunately, it creates an annoying UX, in that the delegators need to be constantly vigilant and active to operate in their best interest. Each round, shares may shift from under them.

One belief is that Orchestrators who would like to run additional nodes, maintain a positive reputation to attract significant delegation, and compete for fees, will have their reputation harmed by this approach and will not attract future delegation.

### Delegator Fee Theft

As mentioned in the Delegator Squeezing Attack above, it is possible for the Orchestrator to drive off its delegates. This could become a particularly malicious technique if the Orchestrator also holds onto its winning PM tickets until the point when the delegators leave, and then cashes them when it contains all the stake for its node. Essentially the fees and rewards that the delegates are entitled to would be delivered to the Orchestrator instead.

This can be counteracted by having expiration dates on the PM tickets, which occur prior to the withdrawal date on the Broadcasters time-locked deposits. As such, the tickets would need to be cashed in short order, and would potentially contain the committed fee share of the Orchestrator at the time, such that when cashing a winning ticket the appropriate splits could be made amongst delegators and the Orchestrator.


## Open Research Areas ############################

As with all work in the early field of blockchain based crypto economic protocols, there are still many research problems which need to be persued before the systems can achieve full decentralization, trustlessness, and economic efficiency. Here are a couple areas that the project is actively conducting research in. Community participation is welcome in pressing forward on these areas as well.

### Non Deterministic Verification

Work continues on the research to verify the likelihood that a GPU encoded segment represents the same content as the pre-encoded segment. This probabilistic and metrics driven approach has been shown in experiments and early research to yield accurate scores, however the suitability for actually slashing deposits based upon probabilistic outcomes is certainly debatable and requires further research.

Deterministic encoding on the other hand can continue to be checked by a variety of verification schemes including Truebit, SGX based hardware verification, Oracles, or even trusted verifiers.

### Public Transcoder Pool Protocols

The split between orchestration responsibilities and transcoder responsibilities should help to dramatically scale the operations of nodes on Livepeer, by leveraging idle hardware to transcode video, without necessarily requiring all those machines to be Livepeer aware 24/7 operating, staked nodes. It is believed that private pools, where the Orchestrator also contains this transcoding hardware, will be the most cost effective, because the Orchestrator can trust that the result that comes out of the transcoders is correct and not malicious.

Public pools, where the orchestrator doesn't trust the transcoders, but can allow anyone to opt in to race to transcode segments, could be very powerful at leveraging any idle compute, without having to have dedicated infrastructure oneself. However, since these remote transcoding nodes aren't trusted, the Orchestrator would have to check their work, or else risk being slashed. This incurs additional costs, and therefore may be unlikely to compete with private pools - unless economic protocols can be created to secure these public pools in the form of staking deposits. If it can be shown that a particular unknown transcoder was the result of a slashing condition being invoked, and they have enough of a deposit/stake to cover the cost of the slash, then public pools could be viable.

Further research and design here is an open topic.

### Broadcaster Doublespend Mitigation

In a probabilistic micropayments scheme, there is always a chance with some probability that a Broadcaster has issued more winning tickets than they have balance to pay (accidentally). And since Orchestrators may not notify the Broadcaster of a winning ticket immediately, it is hard to get an accurate accounting of a Broadcaster's balance. We're continuing research on the required parameters and deposit management to avoid an accidental double spend under various usage patterns in the network. See further analysis in the Probabilistic Micropayments appendix.

### VOD Payments

One of the nice properties that Broadcasters may look for when it comes to video-on-demand transcoding is the notion that they can make the content available, request a job, and disappear - such that the Orchestrator can perform the job asynchronously, distribute it across many nodes, or schedule it when they have idle resources available.

However, in the PM scheme described in Streamflow, the Broadcaster needs to be online in order to continuously send payments as the content streams. Part of the security is in the recognition that if an Orchestrator doesn't continue doing the work, it's ok, as the Broadcaster will simply stop sending future payments.

For VOD jobs though, if a Broadcaster pays up front for all segments of video and then disappears offline, there is no security to guarantee that the Orchestrator will perform the transcoding or make the transcoded segments available back to the Broadcaster. For now, VOD transcoding is possible, but upload-and-disappear is not. Research will continue on better mechanisms to enable VOD payments.

## Migration Path ############################

The Streamflow proposal is early on in its research, design, feedback, and implementation cycle. It certainly deserves a thorough community critique, testing, audits, and acceptance prior to going live on the Ethereum main net as the next iteration on top of Livepeer's alpha protocol. This section aims to list out a couple early considerations with regards to how a protocol migration could occur:

* New smart contract logic would be deployed to Ethereum, however it is anticipated that very little to no data migration would be necessary. Livepeer's existing proxy-delegatecall update mechanism could be utilized.
* Existing state in the Livepeer protocol including staking balances, fees, rewards, delegation, etc would be maintained.
* Transcoders, Broadcasters, Orchestrators, and Delegators would update their client software, which would contain logic for job negotiation, redundancy, payments, and updated verification.
* Orchestrators would register any new required parameters to the service registry, including supported services and possibly locations.
* Broadcasters would establish PM contracts and deposits. Existing deposits could migrate from the Minter to the PM contract via user driven action whenever requested.
* It is anticipated this could be accomplished with little-to-no downtime to the protocol.
* 3rd party clients such as protocol explorers and analytics tools would likely need updates in order to reflect the new protocol interactions.

A formal migration path, checklist, and multiple observed testnet runs will be made available over time as the candidate Streamflow release date nears.

## Summary #################################

In conclusion, the proposals contained within this document aim to shine a light on a scalable path for Livepeer's video infrastructure network - one that decouples the cost of using the Ethereum blockchain from the cost of using the network itself, and that provides existing scaled video developers with the reliability and performance that they require from their infrastructure.

All feedback, ideas, and input are welcomed, so please do not hesitate to drop into [The Livepeer Forum](https://forum.livepeer.org) or [Discord Chat](https://discord.gg/RR4kFAh) to participate.


## Apêndice ################################

### Appendix A: Probabilistic Micropayments Workflow

* An orchestrators security deposit is their stake. This can get slashed if they cheat and fail a verification.
* A broadcaster (using this term for general user, which may be more of a developer than broadcaster) places a time-locked deposit to cover the future work that they’ll pay for on the network.
* A broadcaster wants video transcoded. They look at the on chain registry of orchestrators advertising their services, and negotiate off chain with the ones that fit their needs:
    * Orchestrators provide them a price quote.
    * Orchestrators provide probabilistic micropayment (PM) parameters - these can vary depending on Ethereum network conditions. For example they can set the winning ticket amount such that the cost of cashing in is less than 1% of the value received.
* Broadcaster sends segments of video to the orchestrator(s) they want to work with along with PM ticket.
    * PM ticket is an interactive protocol in order to prevent either party from biasing the source of randomness used to determine whether a ticket wins or not - after every winning ticket, the orchestrator needs to generate a new random # and send the commitment to the broadcaster. This is probably ok since the broadcaster and orchestrator will already be sending data back and forth already - this would just entail an additional message sent by the orchestrator every time a new commitment is required. A later optimization that might be possible is the use of a verifiable random function (VRF) implemented in a smart contract - the orchestrator would give the broadcaster a pub key and the orchestrator signs received tickets with the corresponding priv key - the VRF contract would verify that the sig is correct which is then used as the source of pseudorandomness. As a result, the protocol becomes non-interactive. Would need to evaluate feasibility of implementing the VRF in a smart contract and the cost of verifying that type of sig - ok not to focus on this right now, but a possibility for the future.
* Broadcaster receives transcoded output back from orchestrator.
    * Broadcaster can verify any segments it wants to check.
    * If the work doesn’t verify, they can provide this proof to Truebit on chain to slash the orchestrator, and earn massive reward.
* If broadcaster doesn’t receive work back from orchestrator, simply stop sending them future segments and work with different orchestrator.
* If the orchestrator doesn’t receive a valid PM ticket, just don’t do the work and don’t send any output back.
    * The protocol will contain messages for certain error conditions, such as `LowPMBalance` or `SegmentFormatDidntMatchJobInputParams` so that the broadcasters can receive some useful information to debug, or so that their node can make decisions like refilling their balance.
* Orchestrator monitors broadcaster’s deposit and assesses risk of default.
    * Simple algorithm to begin with. If their balance is too low, just stop doing work.
    * Orchestrator cashes winning tickets as they’re received (or waits until gas is cheaper, assessing risk of default).
    
For a full analysis and specification of the ticket data structures, double spend prevention, and other design considerations, see this [external document](https://hackmd.io/uHMFeNSyS_GyzwnO3Ld74A?view).

## Referências ###########################################

1. Livepeer Whitepaper - Doug Petkanics, Eric Tang - <https://github.com/livepeer/wiki/blob/master/WHITEPAPER.md>
2. The Video Miner, A Path To Scaling Video Transcoding -Philipp Angele  - <https://medium.com/livepeer-blog/the-video-miner-a-path-to-scaling-video-transcoding-a3487d232a1>
3. Ethereum Probabilistic Micropayments - Gustav Simonson - <https://medium.com/@gustav.simonsson/ethereum-probabilistic-micropayments-ae6e6cd85a06>
4. Electronic Lottery Tickets as Micropayments - Ron Rivest - MIT Lab for Computer Science - <https://people.csail.mit.edu/rivest/pubs/Riv97b.pdf>
5. Decentralized Anonymous Micropayments - A. Chiesa, M. Green, J. Liu, P. Miao, I. Miers and P. Mishra - <https://eprint.iacr.org/2016/1033.pdf>


