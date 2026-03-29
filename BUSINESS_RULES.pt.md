# BUSINESS RULES: LIMBOAI - GOVERNANCE CONTRACT

> [!TIP]
> **Read this in other languages / Leia isto em outros idiomas:**
> [**English**](LIMBOAI_BUSINESS_RULES.md) | [**Português**](LIMBOAI_BUSINESS_RULES.pt.md)

Este documento exaustivo estabelece as fronteiras arquiteturais, dogmas de engenharia e regras de negócio inegociáveis para o framework **LimboAI** (Behavior Trees e Hierarchical State Machines). Devido à magnitude conceitual e o impacto extremo na performance do jogo, este contrato atua como a documentação técnica definitiva para os desenvolvedores da engine (C++/GDScript). Qualquer implementação que viole estes limites é considerada lixo arquitetural e deve ser refatorada sumariamente.

---

## 1. FILOSOFIA E ENGENHARIA RIGOROSA

O projeto rejeita a cultura de "Vibe-Coding" (programação guiada por tentativa e erro ou otimismo). Cada linha de inteligência artificial em Árvores de Comportamento (BT) e Máquinas de Estado (HSM) é tratada como um compromisso de engenharia de software de alta performance, determinística e Thread-safe.

### 1.1 Pair Programming e A Matriz de Controle

- **Desapego Radical ao Código:** A Árvore ou HSM é a **Fonte Única de Verdade (Single Source of Truth)**. Se o ator se move para a posição errada, a falha reside exclusivamente na abstração da árvore, do `LimboState`, ou do formulário do `BlackboardPlan`. A correção é feita reestruturando o nó visual ou o `_tick()` lógico dele.
- **Soberania Absoluta (Anti-Padrão Proibido):** Nunca aplique vetores de velocidade (`velocity = ...`), reproduza animações (`AnimationPlayer.play()`) ou dispare lógicas cruciais em scripts soltos atrelados ao `_physics_process` do ator em paralelo à I.A. A árvore não divide o controle do corpo. Ela _é_ o controle do corpo.
- **Imutabilidade Base:** Arquivos `.tres` (BehaviorTree, BlackboardPlan) são "Plantas Baixas" read-only em runtime (Resources nativos da Godot). O que opera e muta no jogo é a cópia em RAM (`BTInstance`).

### 1.2 O Protocolo TDD para IA

Inteligência Artificial deve ser determinística perante falhas.

1. **RED:** Descreva o contrato visual ("Se a vida cair, a árvore ataca ou foge?"). Caso não cumpra, o status falho deve ser rastreado.
2. **GREEN:** Aplique o mínimo de Tasks folha possíveis via Nodes Custom ou Nativos C++ paramétricos.
3. **REFACTOR:** Remova condicionais `_tick()` espaguete em favor de `BTDynamicSelectors` ou purgue loops C++ repetitivos.

---

## 2. A MENTE DO AGENTE: BLACKBOARD SYSTEM E PLANOS (API COMPLETA)

A Behavior Tree, por definição canônica, é "Stateless" (sem estado). Ela esquece o que fez assim que finaliza. O **Blackboard** é sua "Memória RAM", um dicionário otimizado `StringName -> Variant` gerenciado via RefCounted.

### 2.1 A API Básica do Blackboard

Os desenvolvedores devem utilizar a API unificada do quadro (Blackboard). O acesso direto a propriedades cruas é contido pelas funções C++ nativas providas pela classe `Blackboard`:

```cpp
class Blackboard : public RefCounted {
public:
	 // Define ou sobrescreve uma variável. Operação O(1).
	 void set_var(const StringName &p_key, const Variant &p_value);

	 // Recupera uma variável. O default_value é OBRIGATÓRIO por governança
	 // para evitar crashes quando a árvore tentar acessar chaves nulas no C++.
	 Variant get_var(const StringName &p_key, const Variant &p_default = Variant()) const;

	 // Atesta a existência da chave formal no dicionário (mesmo que nula).
	 bool has_var(const StringName &p_key) const;

	 // Destrói ativamente a entrada. Use com extrema cautela.
	 void erase_var(const StringName &p_key);

	 // Conecta uma propriedade do Actor base dinamicamente (Ver Sec 3.2).
	 void bind_property(const StringName &p_key, Object *p_object, const StringName &p_property);

	 // Referencia herança vertical de memória (Ver Sec 2.2).
	 void set_parent_scope(const Ref<Blackboard> &p_board);
	 Ref<Blackboard> get_parent_scope() const;
};
```

### 2.2 Parent Scopes e a "Hive-Mind" (Escopo Ancestral)

Provavelmente o paradigma arquitetural mais poderoso sob tutela do LimboAI.

- **Conceito de Resolução (Fallback):** O Blackboard é uma Linked List direcional. Quando a tarefa clama por `get_var("target_pos")`, a engine revira o Blackboad local. Se a chave não for achada e o board apontar para um `parent_scope`, a busca sobe na hierarquia, interrogando proceduralmente até o Master Root.
- **Implementação "Esquadrão/Hive-Mind":** Em jogos RTS ou combate massivo (ex: 50 lobos atacando o jogador), não duplique 50 vezes o rastreamento do Player.
  1. Aloque **UM** `Blackboard` mestre global num Node de "Diretor" (O `PackManager`).
  2. O `PackManager` atualiza a variável `target_pos` em seu próprio Master Board.
  3. No spawn, os 50 lobos rodam: `wolf_board.set_parent_scope(pack_manager_board)`.
  4. Mágica do Desacoplamento Absoluto: Nenhum lobo sabe quem é o líder ou quem é o alvo de forma codificada (hardcoded). Suas árvores rodam perfeitamente limpas extraindo sua cognição atrelada, economizando até 80% do RAM e acessos C++ desnecessários.

---

## 3. O ESQUELETO DECLARATIVO: PLANOS E BINDINGS

O `BlackboardPlan` não é um dicionário; é a **Assinatura de Tipagem Forte** da IA.

### 3.1 O Dogma das Estruturas de Tipagem

- **Regra de Ouro:** Nenhuma tarefa (`BTTask`) tem autorização para "deduzir" ou "injetar" chaves fantasmas no quadro em runtime. Toda variável deve ser pré-registrada e exportada através do plano correspondente.
- **BBParams:** A API baseia-se na restrição rigorosa. Usamos os pacotes:
  - `BBNode`: Requer injeção de `NodePath` ou alocação direta.
  - `BBInt`, `BBFloat`, `BBString`, `BBVector2`, `BBVector3`.
  - `BBVariant`: O coringa. Evite-o sempre que possível para não corromper tipagens estritas em C++.

### 3.2 Property Bindings (Ponte Ativa Otimizada)

**O Problema (Anti-Padrão):** O desenvolvedor "junior" escreve uma Action chamada `BTCondition_HealthLow` que, a cada tick, executa `blackboard->set_var("hp", agent->get_health())`. Isso gasta 1 frame de processamento só para transporte inútil de escopo.

**A Solução Híbrida (Bindings):** O Blackboard não guarda o float. O Blackboard se torna um _Ponteiro C++_ para a variável do Ator.

- **Implementação Obrigatória:** Se uma variável reflete uma propriedade externa mutável, registre no momento da instanciação (`_ready` do Player):
  ```gdscript
  var board: Blackboard = bt_player.get_blackboard()
  board.bind_property(&"current_health", self, &"health")
  ```
- **Consequência Analítica:** Sempre que a árvore invocar `get_var(&"current_health")`, o C++ desce pela ponte do Object nativamente e retorna o número vivo. Zero delay, zero sync-steps inúteis.

---

## 4. O CORAÇÃO DO MOTOR: BEHAVIOR TREES E LIFECYCLE VIRTUAL

O LimboAI opera avaliando heurísticas reativas de fluxo `Parent -> Child`. A classe atômica, o núcleo que estendemos, é a `BTTask`.

### 4.1 A Matriz de Status Numérica (A Linguagem Primitiva)

Todo e qualquer retorno de métodos avaliativos na Árvore de Comportamento Godot é expresso neste dicionário canônico (enum no C++). Estes Status são a engrenagem e ignorá-los quebra o motor:

1. **`FRESH` (Intocado):** Estado primordial. Uma Task é `FRESH` quando a árvore foi instanciada, OU quando o ramo foi abortado/resetado por uma transação superior de Composer.
   - **Proibição Exata:** Se você escrever C++ e seu loop `_tick()` retornar `FRESH`, a engine vai travar num loop silencioso fatal. Tasks puras da folha NUNCA retornam `FRESH` no pipeline vivo.
2. **`RUNNING` (Trabalho Contínuo Transitório):** Significa "Me invoque novamente no próximo tick". Ancorado num Action ("Atacando", "Correndo", "Castando Magia"). A árvore pausa a varredura linear do mesmo nível de profundidade e repousa o foco computacional em você.
3. **`SUCCESS` (Solução/Verdadeiro):** "Cheguei no ponto A", "Acabei o golpe", "Tinha HP suficiente". Informa ao Composite Pai (Sequences/Selectors) para dar andamento ou bypass para o próximo candidato estrutural.
4. **`FAILURE` (Abstrusão/Falso/Impossível):** É um desvio condicional, não um erro. "Alvo fugiu", "Fora de Raio", "Fui interrompido". Sinaliza ao Composite Pai que a via quebrou, acionando planos de contigência (Selector) ou abortando a missão global (Sequence).

### 4.2 O Ciclo de Vida do BTTask (Override Obrigatórios API)

Se um desenvolvedor (Engenheiro C++ ou GDScripter) criar um Nó Novo (`BTActionCustom`), a API do Limbo expõe os 4 métodos virtuais canônicos. Eles compõem o **Ciclo da Ferramenta**:

```cpp
// --- A INTERFACE VIRTUAL ESTENDIDA DA CLASSES BTTask ---

// 1. O momento do Spawn no Heap
virtual void _setup() override;
/* Disparado exata 1 vez após a cópia Resource->Instance (No _ready).
 * Use APENAS para buscar referencias base via SceneTree (ex: RayCasts, AnimationPlayers)
 * que custariam recursos inestimáveis para buscar a cada frame se ficassem no _tick. */

// 2. O Despertar da Task
virtual void _enter() override;
/* Disparado no exato instante na qual a variável Status sai de FRESH e evolui.
 * É onde o evento começa no tempo! Aqui você aplica Animações, inicializa a Data
 * de um Timer ou formata os Raycasts para a direção do alvo num frame zero. */

// 3. A Matemática Ciclo-a-Ciclo
virtual int _tick(double p_delta) override;
/* O Motor. Invocado pela taxa de atualização da Engine (PhysicsProcess).
 * OBRIGAÇÕES: Operar matemática contínua e invocar RETORNO NUMÉRICO OBRIGATÓRIO
 * (SUCCESS, FAILURE, RUNNING). Processamento brutal vive aqui. */

// 4. O Expurgo (The Cleanup Mandate) - A Regra de Ouro
virtual void _exit() override;
/* Disparado assim que a task despacha SUCCESS/FAILURE de volta à árvore,
 * OU, de maneira drástica, quando um Composite Pai (ex: DynamicSelector) ABORTA
 * VOCÊ enquanto você ainda estava num loop de RUNNING contínuo. */

```

### 4.3 A Regra de Ouro da Preempção (Por que \_Exit salva o jogo?)

Imagine: Uma task de `BTAction` inicia um Timer interno para soltar uma explosão ligada ao Callable `_explode()` que ocorreria em 3.0s (e reporta `RUNNING`). Aos 2.0s, o inimigo toma Root e a árvore inteira superior (`BTSelector`) cancela o combate pra ir pro ramo de defesa. Sua Action é abortada a fórceps!
Se o Timer interno _NÃO_ for resetado/destruído na sua função virtual `_exit()`, no segundo 3 ele chamará o `_explode()` vazando memória, crashando a engine Godot C++ por chamar callbacks zombies fora de contexto.
**O Mantra do LimboAI:** Tudo estruturado no `_enter()` **TEM QUE SER DESCONSTRUIDO/LIBERADO** no `_exit()`.

---

## 5. O ROTEAMENTO FLUXO E LÓGICA (COMPOSITES)

Os Composites controlam a passagem do Cérebro. Eles aceitam `N` filhos na SceneHierarchy visual, iterando de Cima para Baixo / Esquerda p/ Direita. A avaliação de "se ele tranca ou continua" dita a semântica da Mente.

### 5.1 O "OR LÓGICO": BTSelector

- **Implementação C++:** Varre seus filhos através de um Loop iterador. No momento que o filho de Index `I` disparar `SUCCESS` ou `RUNNING`, o Loop aborta. O Selector copia pra si esse retorno e cospe para cima na árvore.
- **Efeito:** Se os primeiros falharam, ele "tenta o próximo" (`FAILURE` -> Próximo i++). Ele cimenta **Lista de Prioridades**. Só reportará `FAILURE` se a malha inteira falhar miseravelmente abaixo dele.
- **Uso Estrito:** Determinar o grande bloco Master ("Ataca, ou Foge, ou Patrulha").

### 5.2 O "AND LÓGICO": BTSequence

- **Implementação C++:** Varre seus filhos em Loop. Se o filho Index `I` disser `SUCCESS`, o loop avança para Index `I+1`. A pegadinha é: Se UM ÚNICO FILHO abortar cuspir `FAILURE`, o Loop sangra na hora. A Sequence aborta e ela despacha um `FAILURE` maciço pra cima, matando o plano.
- **Efeito:** Ela só cospe `SUCCESS` se **TODOS** os filhos conseguiram ser 100% vitoriosos um por vez.
- **Uso Estrito:** Sequenciamentos rituais que não operam pela metade (1. "Gira a Maçaneta" -> 2. "Abre Porta" -> 3. "Entra e Anda").

### 5.3 Os Modificadores Reavaliativos (Dinâmicos)

Sistemas em Compositores regulares ancoram a verificação. Se o 3º filho deu `RUNNING`, no frame seguinte a Sequence chama ele direto (sem reavaliar filho 1 e 2). Essa ancora bloqueia reação de reflexo tática.

- **`BTDynamicSelector`:** Este é agressivo. Mesmo que o 3º filho ("Atacar a Distância") ainda esteja emitindo `RUNNING`, toda vez que o Engine chamar o `_tick` neste Selector, ele reavalia forçadamente o primeiro e o segundo filho! Se o primeiro filho ("Bater de perto" devido inimigo aproximar) der `SUCCESS/RUNNING`, o Seleto aborta covardemente e ativamente o filho 3, atirando a chamada Virtual `_exit()` matando a antiga execução e roteando na nova base. A API C++ pura garante "Reflexo O(1) Instântaneo".
- **`BTDynamicSequence`:** Força toda a malha antes do nó Running reportar SUCCESS todo maldito tick, garantindo que "A condição vital de continuar fazendo X ainda existe". Extremamente custoso como malha de processamento e CPU over-head. Cuidado.

### 5.4 O Abismo Concorrente: BTParallel

- **A Abstração:** Ele simula Multithreading sem a thread secundária. A `BTParallel` varre toda a lista empurrando um `_tick` em **TODOS** os filhos no EXATO MESMO FRAME simultâneamente (Não importa que alguns derem Running).
- **As Políticas de Retorno Canônicas:** A classe do C++ possui flag de aceitação base. `SUCCESS_IF_ALL` (A parallel só cospe success quando todo mundo acabar junto) ou `SUCCESS_IF_ONE` (Basta uma microtask brilhar e o paralelo aborta cruelmente todo o resto dos Runners que estavam processando com ele).
- **Dogma Anti-Padrão Categórico:** NUNCA orquestre lógicas mutativas base excludentes em Paralelos. Inserir tarefas `WalkToA` paralela a `WalkToB` fará a engine reescrever a Matrix do vetor `Velocity` do Actor C++ 2 vezes no instante, destruindo o delta physics paramétrico.

---

## 6. OS FILTROS DE INTERCEPÇÃO: DECORATORS API

Decoradores limitam-se **absolutamente** a ancorar 1 e somente UM filho direto. Eles aglutinam um "anel interceptador" entre nós Composites e Leaves, e atuam subvertendo respostas.

### 6.1 O Grupo Temporizador e Watchdogs (Cronômetros Lógicos)

A API do Custom Node encapsula um mecanismo reativo que tranca processamento a todo custo.

- **`BTCooldown`:** Parâmetro: `BBFloat CooldownTime`. Após o sub-nó atrelado devolver `SUCCESS`, este parasita instila uma barreira de tempo O(1) pura. A barreira tranca o acesso. Se a árvore descer nele enquanto conta, ele mente respondendo `FAILURE` direto e desviando o fluxo do pai sem invocar o C++ do sub-nó. Ideal para: "Lançar BolaDeFogo" (Aguarde 6segs para pular essa árvore e tente bater).
- **`BTTimeLimit`:** A coleira antietanol. Parâmetro: `BBFloat Time_Limit`. Se o child prender o galho travando resposta em `RUNNING` iterativamente superior ao TimeMax limit... O TimeLimit aplica a sentença: Aborta a root emitindo `FAILURE` à força e forçando a rotina de `_exit()` transitar no filho e matá-lo. (Proteção final contra PathFindings inatingiveis de A\* ou NavigationServer corrompido que prenderiam a IA tentando "correr" num buraco infinitamente na Godot Engine).
- **`BTDelay`:** Suspensão forçada. Intercepta o start, joga uma variável `RUNNING` inútil pra engine por N segundos (fingindo carregamento pesado) até enfim delegar a chamada base pra valer.

### 6.2 O Grupo de Manipulação Pura Estática

- **`BTInvert` (NOT logical gate):** Em C++, a interceptação pura O(1) que troca `SUCCESS` por `FAILURE` e `FAILURE` por `SUCCESS`. (Usado comumente associado à lógicas Condition nativas. Se a `Condition(TaVivo)` tá associado, com invert ele se torna O `Enquanto(Morto)`).
- **`BTAlwaysSucceed` / `BTAlwaysFail`:** Blinda a árvore base. Use num nó do qual o Compositor Mestre (Selector/Seq) não deve sofrer interrupções base, fingindo que a tática suja de debug "sempre funcionou" forçadamente sem quebrar a cascata contígua original da árvore visual no Editor da Godot.
- **`BTRepeat`:** Laço for/while puro. Força o filho a ficar restritivamente respondendo repetitivas re-entradas com SUCCESS `Times_X` contadas (Ou iterativamente para Repeat While Failure infinite loop).

---

## 7. O ENDPOINT TERMINAL: ACTIONS E CONDITIONS (LEAVES)

Todo o maquinário visual abstrato da BT desponta em Folhas ("Leaves"), onde interações de mundo C++ com o Environment Scene de Godot são operadas factualmente. O Dogma Separa em 2 classes estritas.

### 7.1 BTCondition (O Princípio da Avaliação Instantânea)

- **Natureza Inegociável:** A classe C++/GDScript da Condição **NUNCA pode adotar fluxos de rotinas demoradas**. Operações `O(1)` computacional ou O(N) levasse-mas lidas em vetores diretos. Elas olham o BBPlan, atestam fatos numéricos e devem reportar a devolução final IMEDIATAMENTE.
- **Proibição Crítica ABSOLUTA:** É **EXPRESSAMENTE PROIBIDO E PUNIDO** codificar no motor nativo `_tick()` de um `BTCondition` a constância instável de status numérico `RUNNING`. Eles respondem apressados: Fato VERDADEIRO (`SUCCESS`) ou Fato FALSO (`FAILURE`).
- **Casos Base Licitos:** Checar booleanos `bool can_jump`, checar colisão base `is_on_floor()`, raycast linear estático instantâneo (Sem Iterativo varrimento frame). Atrasos de Coroutines do `await` do GDScript num Condition corrompem a performance determinística da árvore paralela Limbo.

### 7.2 BTAction (O Paradigma Operacional Ocupacional)

O Modificador absoluto do Ator. Move vetores de Força, dita caminhos de colisão base pro NavMesh Server, altera status de AnimTree BlendSpaces base.

- **Ato Permitido Integral:** Somente uma Action tem domínio jurisdicional autorizado de declarar Estado temporais no motor retendo a Árvore através de um `return RUNNING`.
- Se a mecânica gasta Tempo Contínuo do Gameplay... É um BTAction. Se gasta Zero Tempo (Apenas um IF)... É Pura Condition adaptada visualmente. (Falsas Actions são Actions inúteis que resolvem em 1_tick = SUCESS sem carregar status. Nesses casos escreva em formato de condition com nomenclatura de checker passivo).

---

## 8. HIERARCHICAL STATE MACHINES (HSM): O MODO CONTÍNUO

Ao confrontarmos inimigos gigantes complexos e Chefes que mudam Formas e invulnerabilidades temporais engessadas, Behavior Trees puros paramétricos perdem força num mar de Ifs. Para esses Estados Engessados Contínuos de Transição Abrupta usamos `LimboHSM`.

### 8.1 Arquitetura Física e Estática

Em brutal contraste às Árvores Clones em Matrix RAM no \_ready (`BTInstance`), o **HSM** deforma a Godot baseando-se em `Nodes Hierárquicos Cenas Vivas`. Todo componente HSM, a base de Root a Branch final, é composta de Nós Estaticamente visualizáveis na SceneTree Nativa (`LimboState`).

- **Base Nodes:** `LimboHSM` é o pai de Scene Nodes. Se ele hospedar `State1` e `State2` abaixo dele... Ele assume Controle Máximo gerenciando o Cérebro Ativo num processo Unicolor (Ou seja 1 state só em execução Master).
- **Aninho de Super-Estados (O Poder Hierárquico):** Se botarmos o HSM basear num node derivado sub-pai `MovableState` que tem abaixo dele as leaves filhas Node-scene `WalkState` e `SprintState`. Toda a injeção mecânica do Pai se sobrepoem no filho sem código extra ("If Moving Base = True and Player inputs then..."). Um State encobre perfeitamente o seu submundo.

### 8.2 O Ciclo Ouro de HSM (The LimboState Virtual Engine)

Todo script ou C++ herdado de `LimboState` se pauta num virtual engine rigoroso O(1):

1. **`_setup()`:** Node \_ready de inicialização C++. Extrair variaves nulas ("GetNode", "FindChildRaycast_Path"). Execução de "Init Isolados" fixados.
2. **`_enter()`:** A Exata Célula temporal onde "Você ganhou as chaves do império". A Troca do Limbo ocorreu e Focou em Ti! Começar timers e anims base.
3. **`_update(delta)`:** Update-Fix. Física procedurais e interpolações lineares frame_rate locked rolam aqui. Diferente das BTs que reportam status, states ficam travados nesse Loop eternamente até forças extremamentes externas agirem ou os Guarda-Costas atuarem via despachos.
4. **`_exit()`:** Dogma Mestre de Limpeza C++ de RAM (`Queue Free`). Tudo encadeado em temporários de `_enter` some instantaneo e sem piedade engatatilhada na engine.

---

## 9. TRANSITIONS API: A DOUTRINA DE DESPACHOS

Mecanismo crítico arquitetural. É expressamente PROIBIDO o Polling-Espaguete C++. (Fazer `if (dead) change_state("Morto_Cena")` num loop de 60 FPS dentro de TODO state vivo consumindo CPU imensa processual pra todos os Atores de um mapa MMO at the same time).

### 9.1 Event-Driven Network Dispatcher

Transições não são caçadas; Elas reagem. Elas vivem no Setup estático parametrizado formal:

```cpp
// 1. Estabelecendo e criando pontes fixas intransponíveis fora da lógica no root do C++ setup
meu_hsm->add_transition("State_Combat_Boss", "State_Cinematic_Stun", "boss_poise_broken_event");
```

**O Milagre Dispatcher:** Quando o dano quebrar a postura do Chefão, o Cérebro num local aleatório remoto emite apressadamente na rede de AI um evento Limbo de choque:

```cpp
meu_hsm->dispatch("boss_poise_broken_event");
```

O próprio núcleo Limbo abstrai todo o `_exit` do Boss_Combat e ativa nativamente procedural as chaves engine Limbo States para o Stun de Cena fechada. ZERO loops C++ desperdiçados passivos varrendo arrays inteiros de flags de inimigos estáticos O(N) perdidos nas engrenagens ativas `_update` por Jogo Adentro.

### 9.2 As Chaves de Conditional Guards (Limbo Catracas C++)

Transições do Engine Suportam Interceptar Despachos Baseados puramente mediante `Callable` paramétricos de checagem.

```cpp
// Adicionando transição baseada puramente numa chave se uma Func Callable de Node retornar TRUE
meu_hsm->add_transition("A", "B", "ir_para_b_evt", Callable(this, "tem_hp_pra_usar_skill"));
```

- **A Regra De Ouro dos Guards:** Guards (O Callback Callable) são Validadores "ReadOnly" de fatos baseados. Se um Func Guard aplicar um dano a um inimigo nativo... Ou dropar itens base de array do player em inventários só ao tentar validar o bool... O Vibe-Coding transbordou e você criou as Piores Consequências Técnicas Side-Effects Base Engine Godot de Difícil Debug. Um Guard LÊ e COSPE BOOL sem alterar NADA de dados.

---

## 10. O PARADIGMA HÍBRIDO (BTSTATE O ENVELOPE-MESTRE)

A glória arquitetural: Envelopar Behaviors Trees em Leaves Nativos Cenas de Limbo States Máquinas!
A limitação engessada da HSM, aliada com o fluxo visual Reativo da Árvore de Comportamento.

**Como e Porquê Operar (The Master Hybrid Loop):**
Node nativo `BTState` roda abrigado numa hierarquia Cenas HSM base.

1. Ao tomar Soberania ativando seu `_enter()`, o Node State desperta na Godot Base a "Árvore paramétrica visual atrelada Resource" atrelada a si. Toda fluidez `_tick()` entra na roda transiente.
2. Magia Negra da Transição: Ao fechar o arco do ciclo Root da árvore atrelada, soltando `SUCCESS` pra Fora... A Node State Capta a mensagem! Envolve-a no evento string predatório Limbo e Força ativamente o próprio despacho de emissão formal no HSM pai chamando os eventos string base `EVENT_SUCCESS` ou `EVENT_FAILURE` prefeitos na classe matriz macro LimboAI nativo.

- Usamos isso na engrenagem isolada inteira com perfeição paramétrica visual sem código extra na boss "Phase 3 Combat Form" isolados dos "Sttaters Phase 1". E tudo isso é governado pelas lógicas puras de dispatch nativas já implementadas no root do Limbo Engine Core Node.

---

## 11. DETERMINISMO C++ CORE E ESTRUTURAS DE REPLICAÇÃO

Para solidificar o LimboAI robusto para Networking (Onde "Jitters" na resposta Visual Base cagam Rollbacks):

- **Physics Update Engine C++ Core (The Lock Framerate):** Em configurações do `BTPlayer` e do Root Global Node `LimboHSM` a engrenagem pede a restritiva configuração da emissão. Setar para `_PHYSICS_PROCESS_` process modo! Trava-se a AI isolada para emitir e fazer CallBacks base atrelados à simulação Fixa Godot (Normal 60Hz param). Se colocar process visual atrelado normal da framerate Variante, a pipeline Netcode perde o Delta Time e crasha Client-side Server authoritative rollbacks!
- **Zero Iniciação RAM on-the-fly (`Queue_free` proibidos em Mid-Loop):** Quando o BTInstance (The Matrix memory) clona arrays de `Ref` da árvore Imutável `.tres` nas memória HEAPS... Godot sofre um impacto gigante Allocation OS paramétrico nos processadores de RAM O(N). Esse "Duplicate Matrix Clone" ocorre UMA_VEZ_NO_START. O pooling C++ deve reciclar Instâncias (Set Visiblity).

---

## 12. C++ CUSTOM TASK EXTENSION API

Extensões paramétricas para projetos recaem em Metodologias Exatas.

### 12.1 Binding Base

Ao herdar propriedades configuradas (BBFloat p_radius), sua injeção em Engine precisa de Property Info sólida de variant C++ e a extração necessita do quadro:

```cpp
// Exemplo Canonical Base Numérico C++ de Interceptação da Board:
// Se o "range" paramédio do BTAction depender de uma variável exportavel BBFloat:
void BTActionWalkTo::_bind_methods() {
	ClassDB::bind_method(D_METHOD("set_move_speed", "p_speed"), &BTActionWalkTo::set_move_speed);
	ClassDB::bind_method(D_METHOD("get_move_speed"), &BTActionWalkTo::get_move_speed);
	// ... (adicionar as definições de ADD_PROPERTY para Inspector do Editor C++)
}

// Uso do Call Paramédico Limpo no Tick_Routine
int BTActionWalkTo::_tick(double p_delta) {
	 // 1. Você DEVE passar a Blackboard pointer base pra extração limpa no FallBack do Board!
	 double speed_real_calc = move_speed->get_value(get_blackboard(), 5.0f); // default base nativo fallback
}
```

---

## 13. CONVENÇÕES DE NOMECLATURA

A API Visual exige Dogma nominal forte para gerar ícones nativos na Godot.

### 13.1 Prefixos de Registro C++

- **Node C++ de Tempo Físico / Modificador de Ator:** Obrigar prefixo `BTAction` Ex: `BTActionPlayAnim`.
- **Node C++ de Checker Numérico `O(1)`:** Obrigar prefixo `BTCondition` Ex: `BTConditionIsLowHealth`.

---

## 14. DEBUGGING AND PROFILING PATTERNS

- Erros visivos da BT NÃO se usam com `print()`. O `LimboDebugger` Godot Editor nativo inspeciona dezenas de instâncias conectadas sem pesar no RAM via Output Panel. A performance é blindada descartando outputs na build final.

---

## 15. PARADIGMAS DE TESTE DE IA

- Teste TDD AI focado no status (SUCCESS/FAIL/RUNNING). Mocar uma BT custom com Mock_Blackboard injetado no doctest header testa o loop perfeitamente isolado!
