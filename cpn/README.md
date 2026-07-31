# Modelagem em Rede de Petri Colorida (CPN Tools)

Este diretório contém `humanoides_in_warehouse.cpn` — uma **Rede de Petri Colorida Hierárquica** para o **CPN Tools 4.x** que modela o armazém de robôs humanoides descrito na raiz do projeto.

É a re-modelagem do SED construído no Supremica (ver [`../supremica/`](../supremica/)), generalizada: onde o autômato fixava 2 robôs, 2 setores e 1 baia, aqui os quatro tamanhos do sistema são **parâmetros** do modelo.

## Os quatro parâmetros

```sml
val N_ROBOTS       = 2;   (* Número de Robôs    *)
val N_SECTORS      = 2;   (* Número de Setores  *)
val N_CHARGERS     = 1;   (* Vagas de recarga   *)
val BATTERY_LEVELS = 3;   (* Níveis de bateria  *)
```

Para mudar: abra `Declarations` → bloco `Parametros` no índice e edite os quatro `val`.

---

## 1. Remodelagem

### 1.1 Robô

Os autômatos de cada robô são o mesmo autômato com eventos renomeados. É
exatamente para isso que servem as CPN: declara-se

```sml
colset ROBOT = index rb with 1..N_ROBOTS;
```

e a identidade do robô passa a ser o dado carregado pela ficha. Os autômatos
de localização colapsam em um lugar `Loc : ROBOT × LOCATION` com `N_ROBOTS` fichas, e os
eventos `M_S1_1`/`M_S1_2` colapsam em uma transição cuja variável `R` se liga
a um robô ou a outro.

### 1.2 Setor

O ganho maior vem de tratar o setor como índice inteiro:

```sml
colset LOCATION = int with 0..N_SECTORS;    (* 0 = baia de recarga, 1..N_SECTORS = setores *)
```

O corredor passa a ser aritmética (`L+1` / `L-1`), e o ponto central "estar no setor onde está o trabalho" 
passa a ser a **ligação de uma variável** em vez de um caso escrito à mão por setor. Os eventos de chegada de trabalho
colapsam numa transição `J_ARRIVE`, e os de aceite numa `J_ACCEPT`.

### 1.3 Baias de Recarga

A especificação `SPEC_CS` do Supremica (3 estados, 6 arestas) vira um lugar com
fichas: `Charger : UNIT` com `N_CHARGERS` fichas. A vaga *é* o recurso.

---

## 2. Declarações

```sml
(* --- parametros --- *)
val N_ROBOTS       = 2;
val N_SECTORS      = 2;
val N_CHARGERS     = 1;
val BATTERY_LEVELS = 3;

(* --- conjuntos de cores --- *)
colset UNIT     = unit;
colset ROBOT    = index rb with 1..N_ROBOTS;   (* rb(1) .. rb(N_ROBOTS)     *)
colset SECTOR   = int with 1..N_SECTORS;       (* setores                   *)
colset LOCATION = int with 0..N_SECTORS;       (* 0 = baia, 1..N = setores  *)
colset BATTERY  = int with 0..BATTERY_LEVELS;

colset ROBOT_LOCATION = product ROBOT * LOCATION;
colset ROBOT_BATTERY  = product ROBOT * BATTERY;
colset ROBOT_SECTOR   = product ROBOT * SECTOR;

(* --- constantes --- *)
val B_F = 0;                   (* descarregada -- nunca deve ser alcancada *)
val B_C = 1;                   (* nivel critico *)
val B_O = BATTERY_LEVELS;    (* cheia *)
val CS  = 0;                   (* apelido legivel para a baia *)

(* --- variaveis --- *)
var R : ROBOT;   var S : SECTOR;   var L : LOCATION;   var B : BATTERY;

(* --- marcacoes iniciais --- *)
fun allRobots () = List.tabulate (N_ROBOTS, fn k => rb(k+1));

fun allRobotsAt (p : LOCATION) =
      List.foldl (fn (x,acc) => acc ++ 1`(x,p)) empty (allRobots());

fun allBatteriesAt (v : BATTERY) =
      List.foldl (fn (x,acc) => acc ++ 1`(x,v)) empty (allRobots());

fun allChargers () =
      List.foldl (fn (_,acc) => acc ++ 1`()) empty
                 (List.tabulate (N_CHARGERS, fn k => k));

fun allSectors () =
      List.foldl (fn (k,acc) => acc ++ 1`k) empty
                 (List.tabulate (N_SECTORS, fn k => k+1));
```

> OBS.: **Ordem de declaração importa!**

---

## 3. Lugares

| Lugar | Cor | Marcação inicial | Representa |
|---|---|---|---|
| `Loc` | `ROBOT_LOCATION` | `allRobotsAt(1)` | posição de cada robô; todos começam no setor 1 |
| `Charger` | `UNIT` | `allChargers()` | vagas livres na baia |
| `Bat` | `ROBOT_BATTERY` | `allBatteriesAt(B_O)` | nível de bateria de cada robô |
| `Idle` | `ROBOT` | `ROBOT.all()` | robô ocioso |
| `Pend` | `ROBOT_SECTOR` | *(vazio)* | trabalho pendente, e em qual setor |
| `Exec` | `ROBOT` | *(vazio)* | robô executando |
| `Sectors` | `SECTOR` | `allSectors()` | reservatório constante com todos os setores |

`Idle + Pend + Exec` é invariante e vale sempre `ROBOT.all()` — cada robô está em
exatamente um dos três estados.

`Sectors` é lido por arco de teste e nunca consumido. Existe para resolver uma
armadilha real: a chegada de trabalho escolhe um setor de forma
não-determinística, então `S` apareceria **só** no arco de saída de `J_ARRIVE` — e
uma variável que nenhum arco de entrada liga faz o CPN Tools recusar a rede. Ler
`S` de um reservatório com todos os setores liga a variável e dá exatamente
`N_SECTORS` chegadas possíveis.

---

## 4. Transições e guardas

| Transição | Controlável? | Guarda | O que faz |
|---|---|---|---|
| `MOVE_FWD` | C | `[L >= 1 andalso L < N_SECTORS]` | anda um setor para longe da baia |
| `MOVE_BACK` | C | `[L >= 2]` | anda um setor na direção da baia |
| `ENTER_CS` | C | — | entra na baia, tomando uma vaga |
| `LEAVE_CS` | C | — | sai da baia, devolvendo a vaga |
| `J_ARRIVE` | **U** | — | surge trabalho num setor qualquer |
| `J_ACCEPT` | C | `[B > B_C]` ← **supervisor** | aceita o trabalho do próprio setor |
| `D_OK` | **U** | — | conclui sem perder nível de bateria |
| `D_NOK` | **U** | `[B > B_F]` | conclui perdendo um nível de bateria |
| `RECHARGE` | **U** | `[B >= B_C andalso B < B_O]` | recarrega até encher, dentro da baia |

C = controlável, U = incontrolável (o supervisor não pode desabilitar).

Origem das guardas:

- `[L >= 1 andalso L < N_SECTORS]` exclui `L = 0`, então sair da baia passa
  obrigatoriamente por `LEAVE_CS`, que devolve a ficha. Só as transições de baia
  mexem no recurso.
- `[B > B_C]` — **é a guarda sintetizada pelo supervisor.** No `.wmod` ela aparece
  como `R1_B_curr != B_C` no arco de aceite de tarefa.
- `[B > B_F]` — bateria descarregada não degrada mais. É o que torna o modelo
  **sem supervisor** bloqueante.
- `[B >= B_C andalso B < B_O]` — não se recarrega bateria cheia, e de descarregada
  não se recupera.

As três guardas estão escritas em termos de `B_F`, `B_C` e `B_O`, nunca de um
nível intermediário — é isso que as mantém corretas ao mudar `BATTERY_LEVELS`.

> **Onde está o supervisor.** Toda a síntese está na guarda `[B > B_C]` de
> `J_ACCEPT`. Apague-a e a rede volta a ser a planta pura, bloqueante.

---

## 5. Hierarquia

Quatro páginas. Os 7 lugares ficam na página topo (são os *sockets*); cada
subpágina declara os de que precisa como **portas `I/O`** ligadas ao socket de
mesmo nome.

```
Warehouse (pagina topo: 7 lugares + 3 transicoes de substituicao)
├── Locomotion  -> MOVE_FWD, MOVE_BACK, ENTER_CS, LEAVE_CS
├── Tasks       -> J_ARRIVE, J_ACCEPT, D_OK, D_NOK
└── Energy      -> RECHARGE
```

| Subpágina | Portas (todas `I/O`) |
|---|---|
| `Locomotion` | `Loc`, `Charger` |
| `Tasks` | `Idle`, `Pend`, `Exec`, `Loc`, `Bat`, `Sectors` |
| `Energy` | `Bat`, `Loc` |

`Loc` aparece nas três subpáginas e `Bat` em duas — é assim que a sincronização de
eventos do modelo original atravessa a hierarquia. `D_NOK`, por exemplo, vive em
`Tasks` mas escreve em `Bat`, que é porta ali e socket no topo: é a mesma
sincronização que no `.wmod` faz `D_NOK_1` pertencer ao alfabeto de `R1_J` **e**
de `R1_B`.

> Portas herdam a marcação do socket — por isso os lugares das subpáginas têm
> marcação inicial vazia; a marcação real está no topo.

---

## 6. Abrindo e simulando

### 6.1 CPN Tools no Linux

O CPN Tools é aplicação Windows, mas a instalação traz o simulador compilado
nativo para Linux. São dois processos: a GUI (`cpntools.exe`, sob Wine) e o
*daemon* do simulador (`cpnmld.x86-linux`, nativo), conversando por TCP 2098.
O script `run_cpntools.sh` da pasta de instalação sobe o daemon e só então a GUI:

```bash
cd ~/.wine/drive_c/Program*/CPN*
./run_cpntools.sh
```

O `cd` importa pois o script usa caminhos relativos. Abrir `cpntools.exe` direto
sobe só a GUI, sem daemon — e sem daemon a checagem de sintaxe nunca roda e a
rede fica inteira com aura laranja. Laranja significa "não verificado ainda",
não erro; erro é vermelho.

### 6.2 Carregando

O CPN Tools não tem barra de menus: tudo são *marking menus*. Clique com o botão
direito e segure no workspace, e solte sobre **Load Net**. A rede aparece no
índice à esquerda; expanda e dê duplo clique numa página para abri-la.

### 6.3 Simulando

Arraste a paleta `Simulation` do índice para o workspace. Clique numa ferramenta e
depois no alvo.

| Ferramenta | Efeito |
|---|---|
| **Rewind** | volta à marcação inicial (obrigatório para começar nova simulação) |
| **Single step** | dispara uma transição habilitada, ligação aleatória |
| **Bind manually** | abre o índice de ligações para você escolher |
| **Play** | roda o número de passos indicado na célula da ferramenta |
| **Stop** | interrompe |

Retorno visual: **aura verde** = transição habilitada (lugares nunca ficam verdes
— habilitação é propriedade de transição); **sublinhado verde** na aba ou no
índice = a página tem transição habilitada; **triângulo verde** no canto inferior
esquerdo de uma transição habilitada abre o índice de ligações.

Como quase toda transição tem a variável `R`, ela costuma ter `N_ROBOTS` ligações
possíveis. Use **Bind manually** para seguir uma história; `Single step` sorteia.

### 6.4 Um roteiro que mostra o supervisor

Com `N_ROBOTS=2, N_SECTORS=2, BATTERY_LEVELS=3`, na marcação inicial ficam habilitadas três transições:
`MOVE_FWD` e `ENTER_CS` (em `Locomotion`) e `J_ARRIVE` (em `Tasks`). `MOVE_BACK`
não, porque ninguém passou do setor 1; `RECHARGE` não, porque ninguém está na
baia; `J_ACCEPT`/`D_OK`/`D_NOK` não, porque `Pend` e `Exec` estão vazios.

Ligue sempre `R = rb(1)`:

1. **`J_ARRIVE`** com `S=1` → surge trabalho para o robô 1 no setor 1.
2. **`J_ACCEPT`** → aceita; a guarda `B > B_C` passa com `B=3`.
3. **`D_NOK`** → conclui gastando energia: `Bat` vira `(rb(1),2)`.
4. Repita 1–3 → `Bat` vira `(rb(1),1)`. Bateria em nível crítico.
5. **`J_ARRIVE`** de novo → há trabalho esperando.

Agora tente **`J_ACCEPT`**: **não liga** — `B > B_C` falha com `B=1`. Se o robô 2
também não tiver trabalho no setor onde está, `J_ACCEPT` perde a aura verde por
completo. É o supervisor sintetizado recusando a ordem de serviço, na tela.

A saída:

6. **`ENTER_CS`** → o robô 1 entra na baia e `Charger` esvazia. Repare que
   `ENTER_CS` fica desabilitada **para todos** os robôs: é a exclusão mútua,
   reduzida a uma ficha ausente.
7. **`RECHARGE`** → `Bat` volta a `(rb(1),3)`.
8. **`LEAVE_CS`** → devolve a vaga.

E aí `J_ACCEPT` habilita de novo e o trabalho pendente finalmente sai. Esse ciclo
— trabalho recusado, desvio obrigatório para recarregar, conclusão — é exatamente
o que a síntese comprou.

Para um teste solto: ponha `Play` em 200 passos e depois confira `Bat` — nenhuma
ficha deve mostrar componente `0`, por quanto tempo rodar. É a propriedade de
segurança, que o espaço de estados prova exaustivamente sobre todas as 968
marcações.

---