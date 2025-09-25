# Case for Procurement with SQL
- Um case real para a área de Compras, o qual o desafio era desenvolver um cálculo de Saving, resolvendo um problema clássico de SQL : "Gaps and Inslands".

- Irei apresentar a lógica criada para resolver e desenvolver o Insight para a área de negócio.

### 1) Entender o conceito de "Gaps and Inslands"

Quando analisamos um conjunto de dados e desejamos identificar intervalos de valores ausentes e/ou valores existentes estamos enfrentando um caso de Gaps and Inslands. Sendo que o objetivo final é agrupar os valores existentes (ilhas) que possuem a mesma caracteristica e os "espaços" (lacunas) que estão entre esses valores.

Recentemente, tive a oportunidade de resolver um case o qual possuia esse conjunto de problemas e necessitava de um algoritmo de agrupamento para resolve-lo. Para esse caso, era necessário encontrar o valor do saving de cada mes para um produto específico. o cálculo para isso nada mais é do que (Valor Original – Valor Negociado). Contudo o desafio estava em identificar qual seria o "valor original" e o "valor negociado", pois na base de dados exisitia apenas o valor de cada mes para cada produto. 

Tabela de Exemplo:

<img width="739" height="120" alt="image" src="https://github.com/user-attachments/assets/c6dc8b49-9dbd-403d-bbf8-b30c2eac3edb" />

Sendo assim, a partir disso, o desafio de resolver a problématica clássica de SQL começa

### 2) A Solução Técnica: Lógica em Etapas

Para construir uma base de preço robusta, a solução foi implementada utilizando Common Table Expressions (CTEs) em SQL, seguindo as etapas:
#### **Etapa 1: Identificar Novos Preços**

- Primeiro, precisamos identificar a mudança de valor de compras onde ou quando o preço se manteve constante. Criamos uma flag (`flag_nova_ilha`) sempre que o `valor` muda em relação à linha anterior ou se mantem igaul. 
- Nessa etapa, para que possamos ver quando ocorre mudanças ou nao de preços é necessário usar uma funçao que compare as janelas das linhas. Nesse caso, utiliei a funçao "LAG()" 

Lógica principal da CTE "flag_nova_ilha"

(sql\nconsole.log('CASE
      WHEN valor <> LAG(valor, 1) OVER (PARTITION BY produto,fornecedor ORDER BY data)
           OR LAG(vlr_uni, 1) OVER (PARTITION BY produto, fornecedor ORDER BY data) IS NULL
      THEN 1
      ELSE 0');/n )




