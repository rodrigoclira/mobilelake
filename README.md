# RandomLake com Objetivo Móvel

> Pré-requisito: Utilize sempre como base, os códigos disponibilizados em sala de aula.


---

**Instruções gerais**

Esta avaliação parte das classes e conceitos dos notebooks em sala de aula. Você implementará um novo ambiente derivado do `RandomLake`, resolverá os problemas de observação e recompensa, e discutirá os resultados.

Para esta versão da atividade, considerando as entregas já enviadas, a correção valerá **até 5,0 pontos**, contabilizando as **Seções A, B e C**. Os questionamentos analíticos foram distribuídos dentro dessas próprias seções.

- Você pode resolver a atividade em **um ou mais notebooks**. Caso use mais de um, organize por seções (A, B e C) e indique claramente no topo de cada notebook o que foi implementado ali.
- Copie as classes `FrozenPond`, `RandomLake`, `RandomLakeObs`, `MyCallbacks` e `ppo_config_base` dos notebooks de aula para o início do notebook (ou notebooks) de entrega, sem modificações.
- Execute o notebook completo (`Kernel > Restart & Run All`) antes de entregar. Células com erros não serão avaliadas.
- Respostas discursivas devem ser escritas em células Markdown imediatamente após a célula de código correspondente.

---

## Contextualização

Nos notebooks de aula, o `RandomLake` sorteia os buracos a cada episódio, mas o **objetivo permanece sempre fixo** em (3,3). O agente nunca precisa descobrir para onde ir, basta se deslocar para o canto inferior direito evitando buracos. Nesta avaliação, você vai remover essa vantagem: o **objetivo será sorteado aleatoriamente** a cada episódio, e o agente precisará inferi-lo a partir do que consegue observar.

**Regras do novo ambiente:**

- Grid 4×4. O agente sempre começa em (0,0).
- A cada episódio, o **objetivo é sorteado uniformemente** entre todas as células do grid, exceto (0,0).
- Os **buracos são sorteados** com probabilidade 20% por célula, exceto nas células de início e objetivo.
- O episódio termina ao chegar ao objetivo, ao cair em buraco, ou após 100 passos.

---

## Seção A: Implementação do ambiente *(2,0 pontos)*

Implemente a classe `MobileLake`, subclasse de `RandomLake`, que realiza as modificações descritas acima. Siga os passos na ordem, cada passo corresponde a uma célula de código no notebook.

### A1: Método reset *(1,0 ponto)*

Sobrescreva `reset` para sortear o objetivo aleatoriamente. O objetivo deve ser qualquer célula exceto (0,0), e a célula sorteada não deve ser um buraco. Os buracos continuam sendo gerados com probabilidade 20%, exceto nas células de início e objetivo.

O `info` retornado por `reset` deve conter a chave `"goal"` com a posição sorteada, os callbacks da Seção C dependem disso.

Valide rodando 5 episódios com sementes 0 a 4: imprima o mapa (via `render()`) e a posição do objetivo sorteado em cada um.

### A2: Métodos reward e done *(0,5 ponto)*

Sobrescreva `reward` e `done` para refletir o objetivo móvel. O agente recebe +1 ao chegar à célula sorteada, e o episódio termina nesse caso ou quando cai em buraco.

Confirme executando manualmente uma sequência de ações que leve o agente ao objetivo no episódio com semente 0.

### A3: Treinamento baseline *(0,5 pontos)*

Treine um agente PPO no `MobileLake` por **10 iterações** usando `ppo_config_base` com `MyCallbacks`. Registre `goal_reached_mean` a cada iteração.

Treine também um agente no `RandomLake` original (objetivo fixo, observação de posição) pelo mesmo número de iterações. Sobreponha as duas curvas em um único gráfico com eixos rotulados, título e legenda.

### Questionamento da simulação A

Com base no gráfico de A3, compare o comportamento de aprendizado entre `RandomLake` e `MobileLake`. O objetivo móvel dificultou o aprendizado já no baseline? Justifique com as tendências de `goal_reached_mean`.

---

## Seção B: Engenharia de observações *(2,0 pontos)*

A observação de vizinhança de 4 bits do `RandomLakeObs` foi suficiente para o objetivo fixo porque o agente sempre sabia para onde se dirigir. Com o objetivo móvel, essa observação não fornece qualquer informação sobre onde o objetivo está.

### B1: Sinalização do objetivo *(1,0 ponto)*

Implemente `MobileLakeObs`, subclasse de `MobileLake`, que estende a observação com **dois componentes adicionais**: a **linha** e a **coluna do objetivo** (inteiros em {0,1,2,3}). O `observation_space` resultante deve ser `MultiDiscrete([2, 2, 2, 2, 4, 4])`: quatro bits de vizinhança seguidos de linha e coluna do objetivo.

Valide rodando dois episódios com sementes diferentes: imprima o mapa e a observação inicial, e confirme manualmente que os últimos dois componentes correspondem à posição do objetivo exibida pelo render.

### B2: Treinamento e comparação *(1,0 ponto)*

Treine um agente PPO no `MobileLakeObs` por **10 iterações**. Adicione a curva ao gráfico de A3, que deve conter ao final três curvas: `RandomLake`, `MobileLake` e `MobileLakeObs`.

Produza uma tabela comparativa (Markdown ou DataFrame) com `goal_reached_mean` e `episode_len_mean` dos três agentes, obtidas via `.evaluate()`.

### Questionamentos da simulação B

1. Compare as curvas de `MobileLake` e `MobileLakeObs`. A diferença de desempenho entre os dois agentes é maior, menor ou similar à diferença observada em aula entre `RandomLake` e `RandomLakeObs`? Explique por que o objetivo móvel amplifica ou atenua o efeito da mudança de observação.
2. O `MobileLakeObs` inclui a posição do objetivo na observação. Em aula, discutimos que incluir informação demais pode aumentar a dimensionalidade e prejudicar o aprendizado. Neste caso, a inclusão das coordenadas do objetivo ajudou ou prejudicou? Justifique com base nas métricas da tabela de B2.
3. Suponha que você precisasse escalar o ambiente para um grid 8×8 com objetivo móvel. Identifique **um problema** que surgiria na representação de observação atual (`MultiDiscrete([2,2,2,2,4,4])`) e proponha **uma modificação concreta** na observação para mitigá-lo, sem revelar a posição absoluta do objetivo diretamente. Não é necessário implementar — descreva e justifique.

---

## Seção C: Engenharia de recompensa *(1,0 ponto)*

Com a observação de B1, o agente tem informação suficiente para navegar até o objetivo. Investigue agora se uma função de recompensa mais densa acelera o aprendizado.

### C1: Recompensa com penalidades diferenciadas *(1,0 ponto)*

Implemente `MobileLakeObsRew`, subclasse de `MobileLakeObs`, com a seguinte função de recompensa:

| Evento | Recompensa |
|---|:---:|
| Passo sem evento especial | −0,01 |
| Tentar sair pela borda (acumula com passo) | −0,05 adicional |
| Cair em buraco (substitui penalidade de passo) | −0,5 |
| Chegar ao objetivo | +1,0 |

Treine `MobileLakeObsRew` por **10 iterações** e adicione sua curva ao gráfico. Atualize a tabela comparativa com as métricas deste agente.

### Questionamento da simulação C

Em aula, a recompensa de distância de Manhattan causou reward hacking: o agente evitava o objetivo porque terminar o episódio cortava a recompensa futura. A recompensa de C1 foi projetada para evitar esse problema. Seus resultados confirmam que o reward hacking foi evitado? Se sim, o que na estrutura da recompensa de C1 impede esse comportamento?

