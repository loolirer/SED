# Controle Supervisório de Sistema Multi-Robô com Gestão de Energia

Este diretório contém a modelagem, análise e síntese de um Sistema de Eventos Discretos (SED) para o controle de locomoção e execução de tarefas de dois humanoides autônomos que realizam tarefas em um armazém. O projeto foi desenvolvido utilizando a Teoria de Controle Supervisório através do software **Supremica**.

Veja o vídeo gravado explicando o sistema [aqui](https://drive.google.com/drive/folders/1eUcm5swgOXOCwMUgZOkpGm4N9fe7BDnC?usp=sharing).

## Contexto do Sistema

Para fins de simplificação do autômato, o sistema foi modelado com apenas 2 robôs humanoides, 2 setores e 1 baia de carregamento.

## Arquitetura do Modelo

O sistema foi particionado em diferentes plantas (comportamento físico) e especificações, nomeados para os Robôs 1 e 2 (`R1` e `R2`):

### Plantas de Localização (`R1_L` e `R2_L`)
Define a movimentação física dos robôs pelos setores da fábrica.
- **S1 e S2:** Setores de trabalho.
- **CS (Charging Station):** Baia de recarga.
- **Eventos de movimento:** `M_S1`, `M_S2`, `M_CS` (Controláveis).

### Plantas de Bateria (`R1_B` e `R2_B`)
Modela o estado das baterias.
- **Estados:** `B_O` (Bateria Operacional), `B_H` (Bateria pela metade), `B_C` (Bateria em nível crítico), `B_F` (Bateria descarregada).
- **Degradação do nível de energia:** `D_NOK_X` identifica que houve um consumo de energia a ponto de cair de um nível de bateria par outro.

### Plantas de Tarefas (`R1_J` e `R2_J`)
Modela o fluxo de trabalho e o desgaste aleatório das baterias.
- **Estados:** `IDLE` (Ocioso), `P_S1 / P_S2` (Trabalho Pendente nos setores) e `EXEC` (Executando um trabalho).
- **Desgaste associado à ação:** Ao aceitar uma tarefa, o ambiente (incontrolável) decide se a execução não exauriu a bateria do robô para outro nível (`D_OK_X`) ou se consumiu bateria a ponto de cair de um nível de bateria par outro (`D_NOK_X`).

### Especificação da Baia (`SPEC_CS`)
Autômato fiscal que restringe o uso do carregador.
- Permite movimento livre nos corredores quando a baia está vazia (`CS_F`).
- Bloqueia colisões impedindo que um robô entre na baia se o outro já estiver em estado de recarga (`CS_1` ou `CS_2`).

---

## Síntese do Supervisor

O sistema puro, quando analisado isoladamente, entra em estado de **Blocking** (bloqueio), pois a física permite que os robôs continuem aceitando tarefas até a bateria zerar.

Para resolver isso, foi aplicado o Sintetizador Simbólico (BDD - Binary Decision Diagram) do Supremica:
- O algoritmo gerou um supervisor Non-Blocking e Controllable.
- A solução final utiliza a funcionalidade de Guards: as regras do supervisor foram injetadas diretamente nas plantas originais.
- Quando a bateria atinge um nível crítico de operação, o supervisor desabilita as opções de aceite de trabalho (`D_S1_X`) em tempo real, forçando a prioridade do evento de recarga (`M_CS_X`).

---