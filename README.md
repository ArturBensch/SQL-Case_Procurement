# Case for Procurement with SQL
- Um case real para a área de Compras, focado no desenvolvimento de um cálculo de Saving preciso, solucionando um desafio clássico de SQL: **"Gaps and Islands"**.

- Este documento apresenta a lógica em SQL criada para resolver o problema e gerar um indicador de performance confiável para a área de negócio.
  
### 1) O Desafio: Entendendo "Gaps and Islands"

O problema de "Gaps and Islands" ocorre quando precisamos analisar um conjunto de dados para agrupar sequências contínuas de valores idênticos (as "Ilhas") e identificar os pontos de transição entre esses grupos (os "Gaps").

Recentemente, tive a oportunidade de resolver um case o qual possuia esse conjunto de problemas e necessitava de um algoritmo de agrupamento para resolve-lo. Para esse caso, era necessário encontrar o valor do saving de cada mes para um produto específico. o cálculo para isso nada mais é do que (Valor Original – Valor Negociado). Contudo o desafio estava em identificar qual seria o "valor original" e o "valor negociado", pois na base de dados exisitia apenas o valor de cada mes para cada produto. 

Tabela de Exemplo:

<img width="739" height="120" alt="image" src="https://github.com/user-attachments/assets/c6dc8b49-9dbd-403d-bbf8-b30c2eac3edb" />

Sendo assim, a partir disso, o desafio de resolver a problématica clássica de SQL começa

### 2) A Solução Técnica: Lógica em Etapas

Para construir uma base de preço robusta, a solução foi implementada utilizando Common Table Expressions (CTEs) em SQL, seguindo as etapas:

### **Etapa 1: Identificar Novos Preços**

- Primeiro, precisamos identificar a mudança de valor de compras onde ou quando o preço se manteve constante. Criamos uma flag (`flag_nova_ilha`) sempre que o `valor` muda em relação à linha anterior ou se mantem igaul. 
-A função de janela **LAG()** é utilizada para comparar o valor da linha atual com o da linha anterior. A condição LAG(...) IS NULL garante que a primeira compra de um produto também seja marcada como o início de uma ilha. 

Lógica principal da CTE "FLAG_NOVA_ILHA"

```SQL
      FLAG_NOVA_ILHA AS (
    SELECT
        *,
        CASE
            WHEN valor <> LAG(valor, 1) OVER (PARTITION BY produto, fornecedor ORDER BY data)
                OR LAG(valor, 1) OVER (PARTITION BY produto, fornecedor ORDER BY data) IS NULL
            THEN 1
            ELSE 0
        END AS inicio_nova_ilha
    FROM
        sua_tabela )
```

### **Etapa 2: Atribuir um ID Único para Cada "Ilha"** 

- Agora vem o "truque" para resolver o problema das ilhas. Usamos uma soma cumulativa (SUM() OVER...) na flag que criamos.
- Como a flag é 0 para todas as linhas no meio de uma ilha, a soma não muda, mantendo o mesmo ID para todo o grupo. Quando a flag é 1 (início de uma nova ilha), a soma é incrementada, gerando um novo ID.

```SQL
 ID_ILHA AS (
    SELECT
        *,
        -- A soma cumulativa da flag cria um ID de grupo para cada ilha
        SUM(inicio_nova_ilha) OVER (
            PARTITION BY produtocod, fornecedor 
            ORDER BY data) AS id_ilha_de_preco
    FROM
        flag_nova_ilha )  
```

### **Etapa 3: Agrupar cada Ilha existente**
```SQL
ILHA_GRUPO AS (
    SELECT
        produto,
        fornecedor,
        id_ilha_de_preco,
        -- Pega o primeiro valor de vlr_uni, já que é o mesmo para toda a ilha
        MIN(valor) AS vlr_da_ilha
    FROM
        ID_ILHA
    GROUP BY
        produto,
        fornecedor,
        id_ilha_de_preco
)
```

### **Etapa 4: Anexando o Preço da Ilha Anterior**

- Aqui esta o segredo chave para olhar o valor anterior de cada ilha :  id_ilha_de_preco = preco_anterior.id_ilha_de_preco + 1
-  Para cada linha do grupo atual (ex: id_ilha_de_preco = 3), ela procura uma correspondência na tabela de resumo onde o grupo seja o anterior (id_ilha_de_preco = 2).
```SQL
PRECO_ANTERIOR AS (
    SELECT
        idp.*, 
        COALESCE(
            preco_anterior.vlr_da_ilha,          
            idp.valor          
        ) AS valor_anterior
    FROM
        ID_ILHA idp
    LEFT JOIN
        ILHA_GRUPO AS preco_anterior ON idp.produto = preco_anterior.produto
        AND idp.fornecedor = preco_anterior.fornecedor
        AND idp.id_ilha_de_preco = preco_anterior.id_ilha_de_preco + 1
)
```

### **Etapa 5: Cálculo Final**
- Agora so juntamos tudo e calculamos o saving final com o valor de referÊncia (valor_anterior) e o valor original (valor)

```SQL
CalculoSavingDetalhado AS (
    SELECT
       pa.fornecedor,
       pa.produto,
       pa.data,
       pa.id_linha_recebimento,
       pa.valor,
       pa.valor_anterior
        CASE 
          WHEN (pa.valor_anterior - pa.valor) < 0 THEN 0
          ELSE (pa.valor_anterior - pa.valor)
        END AS saving
    FROM
        PRECO_ANTERIOR pa
)
```

## **Resultado e Impacto**
A implementação desta lógica transformou um histórico de transações complexo em um indicador de performance claro e auditável, permitindo que a área de Procurement tome decisões mais estratégicas com base em dados confiáveis.


### Fluxo de Transformação dos dados

1. FLAG_NOVA_ILHA
<img width="488" height="138" alt="image" src="https://github.com/user-attachments/assets/05781659-3bcc-46e9-a434-709b9eef895e" />

--------------------------------------------------------------------------------------------------------------------------------------

2. ID_ILHA
<img width="488" height="140" alt="image" src="https://github.com/user-attachments/assets/403b88fc-7adc-456e-92a4-1315cbc6bc19" />

--------------------------------------------------------------------------------------------------------------------------------------

3. PRECO_ANTERIOR
<img width="340" height="99" alt="image" src="https://github.com/user-attachments/assets/91dcdfce-4bea-48df-ab74-e4b5dc4e3cd8" />

--------------------------------------------------------------------------------------------------------------------------------------

4. Final 
<img width="599" height="128" alt="image" src="https://github.com/user-attachments/assets/464b25b9-13d6-471a-84f1-ae232c4bf0e8" />




