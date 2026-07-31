# Controle Supervisório de Sistema Multi-Robô com Gestão de Energia

## Contexto do Sistema

O ambiente modelado neste projeto baseia-se em um armazém de produção automatizado. Neste ambiente, operam Robôs Humanóides que trabalham de maneira independentemente e 100% autônoma.

A infraestrutura física desse ambiente é dividida em regiões:

1. **Setores:** São as áreas de produção sequenciais onde os robôs aguardam e executam seus serviços. 
2. **Baia de Carregamento (CS - *Charging Station*):** A estação de energia onde os robôs precisam ir periodicamente para restaurar suas baterias.

O dia a dia dos robôs é ditado pelo surgimento de tarefas nos setores. Esses pedidos de trabalho surgem de forma assíncrona e incontrolável (ditados pela demanda da fábrica). Quando um trabalho aparece, o robô entra em estado de alerta (pendente) e precisa decidir se tem condições de aceitar e executar aquela ordem.

O grande desafio da operação é a gestão da bateria de modo que nenhum robô descarregue. A execução de uma tarefa não tem um custo de energia fixo. Dependendo do peso da carga ou das condições do trajeto (fatores incontroláveis pelo sistema), a tarefa pode ser concluída sem grande impacto na bateria, ou pode exigir um esforço extra que degrada o nível de energia do robô.

#### As Restrições Críticas

Para que o ecossistema da fábrica funcione sem acidentes e sem paralisações, a operação está submetida a três requisitos:

- Cada baia de carregamento possui apenas um espaço de recarga. Fisicamente, é impossível que os dois robôs ocupem um carregador ao mesmo tempo.
- Um robô nunca pode permitir que sua bateria chegue a zero. Se isso acontecer, ele "morrerá" no meio de um corredor, travando a fábrica e exigindo resgate manual.
- Os setores são sequenciais. Para ir da estação de carga até o Setor N, é necessário passar pelo Setor N-1 obrigatoriamente.

#### O Papel do Supervisor

Os robôs seriam gananciosos: continuariam aceitando tarefas até a energia acabar repentinamente, ou correriam todos para o carregador no mesmo instante. Essa inteligência precisa orquestrar o trânsito, impedir que um robô aceite um trabalho se a sua bateria estiver no limite, e organizar a fila do carregador para que todos sobrevivam e a fábrica opere em harmonia.

---

## Estrutura do Repositório

O sistema foi modelado por duas abordagens complementares:

| Diretório | Abordagem | Ferramenta |
|---|---|---|
| [`supremica/`](supremica/) | Autômatos e síntese de supervisor pela Teoria de Controle Supervisório | Supremica |
| [`cpn/`](cpn/) | Rede de Petri Colorida Hierárquica, parametrizada em número de robôs, setores, vagas de recarga e níveis de bateria | CPN Tools |

O modelo em Supremica fixa 2 robôs, 2 setores e 1 baia para manter o autômato
tratável. Na Rede de Petri Colorida esses quatro tamanhos passam a ser
**parâmetros**: a identidade do robô vira a cor da ficha e o setor vira um índice
inteiro, de modo que a rede tem 7 lugares e 9 transições para qualquer
configuração. As duas modelagens coincidem em 968 estados alcançáveis na
configuração comum, e a guarda sintetizada pelo supervisor é preservada.

---