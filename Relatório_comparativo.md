Criar página html com relatório comparativo de duas tabelas (vigente e proposta)

O relatório deve:

1. Receber tabela.txt vigente e tabela.txt da proposta
2. Verificar se ambos são do mesmo itinerário e Autos de linha.
3. Analise comparativa das seções
4. Análise comparativa dos trechos disponibilizados
5. Das distâncias para distância e tarifa
6. Número de viagens ofertadas
7. Número de opções de viagem ofertadas

Detalhamento da análise:

3. Apresentar diferença entre ambas tabelas horárias (seções de ida e de volta são sempre as mesmas).
4. Verificar qual a diferença entre as tabelas horarias em trechos disponibilizados, exemplo:
   vigente: A>B>C>D, OPERA AB, AC, AD, BC E BD.
   Proposta: A>B>C, OPERA AB, AC e BD.
   Redução no número de trechos ofertados de x%. Deixará de ofertar os trechos ...
   Porém, supondo que C e D ficam na mesma cidade.
   Não há redução no número de cidades ofertadas, número de conexões intermunicipais não se reduz.
   Faça o relatório responder e apresentar essas informações de forma objetiva e aplicável as diversas possibilidades. (deixar de operar uma cidade, ou uma seção, ou inserir cidade nova ou outra seção em outra cidade ou na mesma, deixar de poder vender passagem entre determinaado trecho, ou pedir para vender passagem em determinado trecho).
5. Fazer comparação das distâncias e distancias para tarifa, dos trechos que se mantem. E citar e apresentar as distancias dos novos trechos que houverem na proposta. Se for o caso, apresentar tudo junto na analise do item 4.
6. 7. Calcular o numero de viagens ofertadas geral, mas também por tipo de viagem (existem viagens inteiras, parciais, semidiretas, diretas), devendo apresentar tbm tudo por faixa horária (manha, entrepico, tarde, noite, madrugada).
      Calcular em conjunto e apresentar como informações complementares, o número de opções de viagens (oportunidades de viagens ofertadas).
      A ideia destes itens é demonstrar a capacidade de oferta de horarios, tanto geral tanto por faixa de horário.

ex de calculo:
Os trechos, além de serem atendidos quando tem um horário do onibus passando, também só são atendidos quando possuem seccionamento tarifario (é permitido vender passagem no trecho) A informação está constando no arquivo txt (que é um json), tem duas listas a distancia e a lista de tarifa, quando a lista de tarifa tem um número é porque tem venda de passagem no trecho. Regra importatne para saber qual distancia bate como trecho: Para N seções, são sempre montados e calculados **todos os C(N,2) pares (i, k) com i < k**.

vigente ida
A 8h00 9h00 11h00
B 9h00 10h00 12h00
C 10h00 - 13h00
D 11h00 - -

1 viagem direta (ABCD)
1 viagem parcial (AB)
1 viagem parcial (ABC)
AB = 3 opcoes de horario
AC = 2 opcoes de horario
AD = 1 opcoes de horario
BC = 2 opcoes
CD = 1 opcao

proposta ida
A 8h00 9h00 11h00
B 10h00 10h00 12h00
C 11h00 11h00 13h00
3 viagens completas (ABC)
AB=3
AC=3
AD=0
BC=3
BD=0
CD=0

Comparativo viagens:
ABCD 1 -> 0
AB -> 0
ABC 1 -> 3

Comparativo opcoes de viagem:
apresentar comparativo total, geral e por faixa de horario.
apresentar tbm trechos variacoes dos trechos, de alguma forma. Proponha uma forma para apresentar esses calculos.
