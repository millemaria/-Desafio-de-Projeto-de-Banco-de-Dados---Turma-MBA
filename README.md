## 🧠 Entendimento do Projeto

**🧩 1. Relacionamento entre FORNECEDOR e ENQUADRAMENTO_FORNECEDOR**

A tabela FORNECEDOR representa a entidade central que contém todos os dados cadastrais dos fornecedores que participam da política de antecipação. Cada fornecedor é identificado de forma única por um campo técnico, id_fornecedor, que funciona como chave primária dessa tabela. Embora o radical_cnpj represente uma forma de identificação empresarial, o uso de uma chave técnica garante a integridade interna e evita problemas com formatação, duplicidade ou alterações legais do CNPJ.

Já a tabela ENQUADRAMENTO_FORNECEDOR funciona como uma entidade de ligação (tabela de fato) entre o fornecedor e sua classificação dentro de uma faixa de performance em um dado momento no tempo. O campo id_fornecedor dentro da tabela de enquadramento é uma **chave estrangeira** (foreign key) que referencia a chave primária id_fornecedor da tabela FORNECEDOR. Esse relacionamento é do tipo **um-para-muitos**, ou seja: um único fornecedor pode aparecer diversas vezes na tabela de enquadramento ao longo do tempo, cada uma representando um momento diferente de avaliação de seu desempenho.

Esse relacionamento permite, por exemplo, consultar todo o histórico de enquadramentos de um fornecedor, acompanhar alterações no seu share percentual e nas taxas aplicadas em função da sua performance e da política vigente em diferentes períodos.

**🧩 2. Relacionamento entre FAIXA_PERFORMANCE e ENQUADRAMENTO_FORNECEDOR**

A tabela FAIXA_PERFORMANCE representa os critérios paramétricos utilizados para classificar os fornecedores com base no seu percentual de participação no volume de compras (*share_percentual*). Cada faixa de performance define uma combinação de taxa_min_cdi_percent, share_min_percent e share_max_percent, representando um range em que um fornecedor pode se enquadrar.

Essa tabela possui uma chave primária chamada id_faixa, que é usada como referência na tabela ENQUADRAMENTO_FORNECEDOR através do campo id_faixa (foreign key). Isso configura outro relacionamento do tipo **um-para-muitos**, no qual uma única faixa de performance pode estar associada a múltiplos enquadramentos de fornecedores, ou seja, a vários registros na tabela de ENQUADRAMENTO_FORNECEDOR.

Esse relacionamento é essencial para permitir que a política de risco seja aplicada de forma parametrizada e padronizada, de modo que o cálculo da taxa de antecipação possa ser derivado diretamente da classificação do fornecedor. Dessa forma, caso a política mude (ex: alteração de faixa de corte de share), é possível manter o histórico das condições anteriores e aplicar as novas regras apenas a partir de novas datas de enquadramento.

**🧩 3. Relacionamento entre COMITE_TAXAS e ENQUADRAMENTO_FORNECEDOR**

A tabela COMITE_TAXAS registra os eventos de reuniões do comitê de crédito e risco, onde são estabelecidas as metas de remuneração financeira — notadamente, a taxa Selic anual (meta_selic_aa) e sua correspondente taxa mensal (meta_selic_am). Cada reunião possui uma vigência definida por um intervalo de datas (vigencia_inicio e vigencia_fim), durante o qual suas definições devem ser consideradas válidas.

A tabela ENQUADRAMENTO_FORNECEDOR utiliza o campo id_reuniao como chave estrangeira que referencia a chave primária id_reuniao da tabela COMITE_TAXAS. Mais uma vez, temos um relacionamento do tipo **um-para-muitos**, onde uma única reunião pode ter impacto sobre o enquadramento de diversos fornecedores no período de sua vigência.

Esse relacionamento temporal é crucial porque permite vincular o enquadramento de um fornecedor não apenas à sua performance em termos de share, mas também ao cenário econômico vigente, refletido na meta Selic do período. Assim, o campo taxa_aplicada_mes na tabela de enquadramento pode ser calculado com base na taxa mínima da faixa multiplicada pelo valor da Selic mensal definida naquela reunião específica.

**🧠 Considerações Técnicas**

A modelagem proposta privilegia a **normalização dos dados**, o que significa evitar redundâncias e permitir controle rigoroso de integridade referencial. A separação entre fornecedor, faixa de performance e reuniões do comitê permite:

- Registrar o histórico de decisões de forma auditável;
- Alterar regras de negócio futuras sem comprometer registros passados;
- Realizar simulações e análises preditivas por período, por faixa, ou por fornecedor;
- Implementar lógicas complexas como: qual foi a taxa média aplicada aos fornecedores da faixa “MÉDIA-ALTA” em função das Selics praticadas no último semestre?

Além disso, o uso da tabela intermediária ENQUADRAMENTO_FORNECEDOR permite criar lógicas dinâmicas e temporais, como *slowly changing dimensions* (SCD tipo 2), o que é uma prática recomendada em bancos que possuem carga de dados analíticos e demandas por histórico.

---

## 🛠️ Scripts de Criação das Tabelas

Abaixo estão os scripts SQL utilizados para criar as tabelas do modelo relacional descrito acima.

### 📄 Tabela `FORNECEDOR`

```sql
CREATE TABLE FORNECEDOR (
    id_fornecedor INT PRIMARY KEY,
    radical_cnpj VARCHAR(255),
    razao_social VARCHAR(255),
    cod_depto VARCHAR(255),
    nome_depto VARCHAR(255),
    share_percentual DECIMAL(10, 4),
    data_base_share DATE
);

CREATE TABLE FAIXA_PERFORMANCE (
    id_faixa INT PRIMARY KEY,
    nome_faixa VARCHAR(255),
    taxa_min_cdi_percent DECIMAL(10, 4),
    share_min_percent DECIMAL(10, 4),
    share_max_percent DECIMAL(10, 4)
);

CREATE TABLE COMITE_TAXAS (
    id_reuniao INT PRIMARY KEY,
    numero_reuniao VARCHAR(255),
    data_reuniao DATE,
    vigencia_inicio DATE,
    vigencia_fim DATE,
    meta_selic_aa DECIMAL(10, 4),
    meta_selic_am DECIMAL(10, 4)
);

CREATE TABLE ENQUADRAMENTO_FORNECEDOR (
    id_enquadramento INT PRIMARY KEY,
    id_fornecedor INT,
    id_faixa INT,
    id_reuniao INT,
    data_enquadramento DATE,
    taxa_aplicada_mes DECIMAL(10, 4),
    FOREIGN KEY (id_fornecedor) REFERENCES FORNECEDOR(id_fornecedor),
    FOREIGN KEY (id_faixa) REFERENCES FAIXA_PERFORMANCE(id_faixa),
    FOREIGN KEY (id_reuniao) REFERENCES COMITE_TAXAS(id_reuniao)
);
```
 
## Modelagem ER
![Diagrama de Relacionamento](https://raw.githubusercontent.com/irodrigosouza/-Desafio-de-Projeto-de-Banco-de-Dados---Turma-MBA/main/Diagrama%20de%20Relacionamento.drawio.svg)


## 🛠️ Scripts de Inserção de dados nas tabelas

```sql
INSERT INTO FORNECEDOR (id_fornecedor, radical_cnpj, razao_social, cod_depto, nome_depto, share_percentual, data_base_share) VALUES
(1, '99.999.999', 'RODRIGO''S SNACKS', '1001', 'BOMBONIERE', 68.70, '2025-07-01'),
(2, '88.800.000', 'LINCOLN&CO DESIGN', '1002', 'PAPELARIA', 48.50, '2025-07-01'),
(3, '77.000.001', 'MENINO''S TOYS', '1019', 'BRINQUEDOS', 33.00, '2025-07-01'),
(4, '50.500.005', 'MILLE''S TECH', '1004', 'INFORMÁTICA', 91.30, '2025-07-01'),
(5, '10.100.111', 'LARI EDITORA', '1033', 'LIVROS', 10.00, '2025-07-01'),
(6, '22.111.333', 'MEGA DISTRIBUIDORA', '1001', 'BOMBONIERE', 76.20, '2025-07-01'),
(7, '33.444.555', 'PAPEL BRASIL', '1002', 'PAPELARIA', 35.00, '2025-07-01'),
(8, '44.666.777', 'TECHLANDIA', '1004', 'INFORMÁTICA', 82.40, '2025-07-01'),
(9, '55.888.999', 'BRINQUEDOS YEAH', '1019', 'BRINQUEDOS', 26.00, '2025-07-01'),
(10, '66.123.456', 'ARTE & CIA', '1033', 'LIVROS', 22.10, '2025-07-01'),
(11, '11.111.111', 'FORNECEDOR A', '1002', 'PAPELARIA', 15.00, '2025-07-01'),
(12, '22.222.222', 'FORNECEDOR B', '1004', 'INFORMÁTICA', 92.00, '2025-07-01'),
(13, '33.333.333', 'FORNECEDOR C', '1019', 'BRINQUEDOS', 51.00, '2025-07-01'),
(14, '44.444.444', 'FORNECEDOR D', '1001', 'BOMBONIERE', 44.00, '2025-07-01'),
(15, '55.555.555', 'FORNECEDOR E', '1033', 'LIVROS', 12.00, '2025-07-01'),
(16, '66.666.666', 'FORNECEDOR F', '1002', 'PAPELARIA', 38.00, '2025-07-01'),
(17, '77.777.777', 'FORNECEDOR G', '1001', 'BOMBONIERE', 28.00, '2025-07-01'),
(18, '88.888.888', 'FORNECEDOR H', '1004', 'INFORMÁTICA', 87.00, '2025-07-01'),
(19, '99.999.998', 'FORNECEDOR I', '1019', 'BRINQUEDOS', 19.90, '2025-07-01'),
(20, '10.101.102', 'FORNECEDOR J', '1033', 'LIVROS', 55.00, '2025-07-01');


INSERT INTO FAIXA_PERFORMANCE (id_faixa, nome_faixa, taxa_min_cdi_percent, share_min_percent, share_max_percent) VALUES
(1, 'FORN. ALTA PERFORMANCE', 100.00, 75.00, 100.00),
(2, 'FORN. MÉDIA-ALTA PERFORMANCE', 110.00, 50.00, 74.99),
(3, 'FORN. MÉDIA-BAIXA PERFORMANCE', 120.00, 25.00, 49.99),
(4, 'FORN. BAIXA PERFORMANCE', 130.00, 0.00, 24.99);

INSERT INTO COMITE_TAXAS (id_reuniao, numero_reuniao, data_reuniao, vigencia_inicio, vigencia_fim, meta_selic_aa, meta_selic_am) VALUES
(1, '266ª', '2024-11-07', '2024-11-08', '2024-12-11', 13.50, 1.13),
(2, '267ª', '2024-12-12', '2024-12-13', '2025-01-25', 14.00, 1.17),
(3, '268ª', '2025-01-29', '2025-01-30', '2025-03-19', 17.00, 1.42),
(4, '269ª', '2025-03-19', '2025-03-20', '2025-05-09', 18.00, 1.50),
(5, '270ª', '2025-05-08', '2025-05-10', '2025-07-19', 16.50, 1.38),
(6, '271ª', '2025-07-18', '2025-07-20', '2025-08-19', 15.00, 1.25);

INSERT INTO ENQUADRAMENTO_FORNECEDOR (id_enquadramento, id_fornecedor, id_faixa, id_reuniao, data_enquadramento, taxa_aplicada_mes) VALUES
(1, 1, 2, 6, '2025-07-21', 1.38),
(2, 2, 2, 6, '2025-07-21', 1.38),
(3, 3, 3, 6, '2025-07-21', 1.50),
(4, 4, 1, 6, '2025-07-21', 1.25),
(5, 5, 4, 6, '2025-07-21', 1.63),
(6, 6, 1, 6, '2025-07-22', 1.25),
(7, 7, 3, 6, '2025-07-22', 1.50),
(8, 8, 1, 6, '2025-07-22', 1.25),
(9, 9, 3, 6, '2025-07-22', 1.50),
(10, 10, 4, 6, '2025-07-22', 1.63),
(11, 11, 4, 5, '2025-06-15', 1.63),
(12, 12, 1, 5, '2025-06-15', 1.25),
(13, 13, 2, 5, '2025-06-15', 1.38),
(14, 14, 3, 5, '2025-06-15', 1.50),
(15, 15, 4, 5, '2025-06-15', 1.63),
(16, 16, 3, 5, '2025-06-15', 1.50),
(17, 17, 3, 5, '2025-06-15', 1.50),
(18, 18, 1, 5, '2025-06-15', 1.25),
(19, 19, 4, 5, '2025-06-15', 1.63),
(20, 20, 2, 5, '2025-06-15', 1.38),
(21, 1, 2, 4, '2025-04-10', 1.65),
(22, 2, 2, 4, '2025-04-10', 1.65),
(23, 3, 3, 4, '2025-04-10', 1.80),
(24, 4, 1, 4, '2025-04-10', 1.50),
(25, 5, 4, 4, '2025-04-10', 1.95),
(26, 6, 1, 4, '2025-04-10', 1.50),
(27, 7, 3, 4, '2025-04-10', 1.80),
(28, 8, 1, 4, '2025-04-10', 1.50),
(29, 9, 3, 4, '2025-04-10', 1.80),
(30, 10, 4, 4, '2025-04-10', 1.95),
(31, 11, 4, 3, '2025-02-10', 1.85),
(32, 12, 1, 3, '2025-02-10', 1.42),
(33, 13, 2, 3, '2025-02-10', 1.56),
(34, 14, 3, 3, '2025-02-10', 1.70),
(35, 15, 4, 3, '2025-02-10', 1.85),
(36, 16, 3, 3, '2025-02-10', 1.70),
(37, 17, 3, 3, '2025-02-10', 1.70),
(38, 18, 1, 3, '2025-02-10', 1.42),
(39, 19, 4, 3, '2025-02-10', 1.85),
(40, 20, 2, 3, '2025-02-10', 1.56);
```
## Regras de Negócio

### **Classificação por Faixa de Performance**

**Regra:** Cada fornecedor deve ser classificado em uma faixa de performance com base no seu `share_percentual` atual.
```
SELECT 
    f.id_fornecedor,
    f.razao_social,
    f.share_percentual,
    fp.nome_faixa
FROM 
    FORNECEDOR f
JOIN 
    FAIXA_PERFORMANCE fp 
ON 
    f.share_percentual BETWEEN fp.share_min_percent AND fp.share_max_percent;
```    
### Aplicação de Taxa Dinâmica

**Regra:** A taxa aplicada para antecipação deve ser superior ou igual à `taxa_min_cdi_percent` da faixa de performance e **não pode ser menor que a meta Selic mensal vigente** da última reunião de comitê.
```
SELECT 
    ef.id_enquadramento,
    f.razao_social,
    fp.nome_faixa,
    ef.taxa_aplicada_mes,
    fp.taxa_min_cdi_percent,
    ct.meta_selic_am,
    CASE 
        WHEN ef.taxa_aplicada_mes < ct.meta_selic_am THEN 'TAXA ABAIXO DA SELIC'
        WHEN ef.taxa_aplicada_mes < fp.taxa_min_cdi_percent THEN 'TAXA ABAIXO DO MÍNIMO DA FAIXA'
        ELSE 'TAXA OK'
    END AS status_taxa
FROM 
    ENQUADRAMENTO_FORNECEDOR ef
JOIN 
    FORNECEDOR f ON ef.id_fornecedor = f.id_fornecedor
JOIN 
    FAIXA_PERFORMANCE fp ON ef.id_faixa = fp.id_faixa
JOIN 
    COMITE_TAXAS ct ON ef.id_reuniao = ct.id_reuniao;
```

### **Alerta para Reenquadramento**

**Regra:** Fornecedores cujo `share_percentual` atual esteja fora da faixa associada no último enquadramento devem ser sinalizados para reenquadramento.
```
SELECT 
    ef.id_fornecedor,
    COUNT(*) AS qtd_enquadramentos,
    MIN(ct.vigencia_inicio) AS inicio,
    MAX(ct.vigencia_fim) AS fim
FROM 
    ENQUADRAMENTO_FORNECEDOR ef
JOIN 
    COMITE_TAXAS ct ON ef.id_reuniao = ct.id_reuniao
GROUP BY 
    ef.id_fornecedor
HAVING 
    COUNT(*) > 1 AND MAX(ct.vigencia_inicio) < MIN(ct.vigencia_fim);
```

### **Histórico e Validade de Enquadramentos**

**Regra:** Um fornecedor não pode ter múltiplos enquadramentos válidos simultaneamente. Avalie se existem sobreposições de vigência com reuniões distintas.
```
SELECT 
    ef.id_fornecedor,
    COUNT(*) AS qtd_enquadramentos,
    MIN(ct.vigencia_inicio) AS inicio,
    MAX(ct.vigencia_fim) AS fim
FROM 
    ENQUADRAMENTO_FORNECEDOR ef
JOIN 
    COMITE_TAXAS ct ON ef.id_reuniao = ct.id_reuniao
GROUP BY 
    ef.id_fornecedor
HAVING 
    COUNT(*) > 1 AND MAX(ct.vigencia_inicio) < MIN(ct.vigencia_fim);

```

### Fornecedores elegíveis para antecipação com taxa reduzida

**Regra:** Fornecedores na faixa mais alta de performance e taxa aplicada igual ou menor à taxa mínima da faixa têm acesso preferencial a linhas de antecipação.
```
SELECT 
    f.id_fornecedor,
    f.razao_social,
    ef.taxa_aplicada_mes,
    fp.nome_faixa,
    fp.taxa_min_cdi_percent
FROM 
    ENQUADRAMENTO_FORNECEDOR ef
JOIN 
    FORNECEDOR f ON ef.id_fornecedor = f.id_fornecedor
JOIN 
    FAIXA_PERFORMANCE fp ON ef.id_faixa = fp.id_faixa
WHERE 
    fp.nome_faixa = 'FAIXA A'
    AND ef.taxa_aplicada_mes <= fp.taxa_min_cdi_percent;
```

Resumo das Regras Criadas
## 📋 Regras de Negócio Aplicadas

| Número | Regra                              | Finalidade                                                    |
|--------|------------------------------------|---------------------------------------------------------------|
| 1      | Classificação automática           | Atribuir faixa correta ao fornecedor com base no share       |
| 2      | Validação de taxa aplicada         | Garantir conformidade com meta Selic e faixa                 |
| 3      | Reenquadramento necessário         | Monitorar desvios de performance                             |
| 4      | Exclusividade de vigência          | Evitar sobreposição de decisões de comitês                  |
| 5      | Identificar elegíveis para taxa especial | Selecionar fornecedores aptos a condições diferenciadas |
