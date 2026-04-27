# Solução - Desafio Técnico Cientista de Dados (Squad WhatsApp)

Este repositório contém minha solução para o case técnico com foco em:

- análise de qualidade de fontes de telefone;
- modelagem de ranking de sistemas;
- algoritmo de priorização dos 2 melhores telefones por CPF;
- proposta de validação via experimento A/B.

O principal artefato é o notebook:

- `case-ds-pleno.ipynb`

---

## 1) Abordagem

### 1.1 Preparação dos dados

Foram usadas duas bases mascaradas:

- `whatsapp_base_disparo_mascarado` (histórico de disparos);
- `whatsapp_dim_telefone_mascarado` (dimensão de telefones e aparições por sistema).

Passos principais de tratamento:

1. Explosão de `telefone_aparicoes` para recuperar `id_sistema_mask` e `registro_data_atualizacao`;
2. Agrupamento por `telefone_numero` e `id_sistema_mask` para evitar inflar volumes, selecionando `registro_data_atualizacao` mais recente;
3. Junção da base de disparo com a dimensão por telefone;
4. Construção de flags de sucesso para cálculo de taxas;
5. Cálculo de recência (`dias_desde_atualizacao`) para análise de decaimento.

### 1.2 Parte 1 - Qualidade de fontes

Foi realizada a comparação de desempenho por sistema de origem e o estudo de decaimento por janela de atualização.

Para reduzir viés por baixo volume, a taxa de sucesso por sistema foi suavizada via ajuste bayesiano:

$$
\text{taxaajustada} = \frac{\text{sucessos} + \alpha \cdot \bar{p}}{\text{tentativas} + \alpha}
$$

onde:

- $\bar{p}$ = taxa global;
- $\alpha$ = força do prior.

### 1.3 Parte 2.1 - Ranking de sistemas

O ranking foi definido com dois componentes:

- `score_origem`: taxa suavizada por Bayes;
- `score_atualidade`: fator de recência por sistema.

Regra adotada para atualidade:

- até 180 dias: score 1.0;
- acima de 180 dias: decaimento exponencial com meia-vida anual.

Score final do sistema:

$$
\text{scoresistema} = 100 \cdot \text{scoreorigem} \cdot \text{scoreatualidade}
$$

### 1.4 Parte 2.2 - Algoritmo de escolha de telefones

No nível telefone, o algoritmo usa:

- `score_sistema` (já contendo origem + atualidade);
- fator de qualidade (`telefone_qualidade`: VALIDO, SUSPEITO, INVALIDO);
- `score_ddd` com suavização por volume.

Probabilidade por aparição:

$$
\text{paparicao} =
\left(\frac{\text{scoresistema}}{100}\right)
\cdot \text{fatorqualidade}
\cdot \text{scoreddd}
$$

Depois, há consolidação por `cpf + contato_telefone`, bônus leve por multissistema e seleção dos 2 melhores por CPF.

### 1.5 Parte 3 - Desenho do experimento A/B

A validação proposta compara:

- controle: estratégia atual;
- teste: priorização por score.

Com definição de:

- hipótese nula e alternativa
  - H0: taxa de sucesso da nova estratégica é igual
- métrica primária (taxa de sucesso/entrega);
- métricas secundárias (ex.: custo por entrega);
- estimativa de tamanho de amostra e duração.

---

## 2) Premissas

As principais premissas adotadas no notebook:

1. A taxa de sucesso observada no histórico é um bom proxy de “telefone quente”;
2. Bases com pouco volume exigem suavização estatística para evitar conclusões instáveis;
3. Atualidade da informação é relevante para ranking de fonte;
4. Qualidade declarada do telefone e sinal regional (DDD) agregam poder de priorização;
5. Para operação, priorizar top-2 por CPF é um bom compromisso entre cobertura e custo.

---

## 3) Como reproduzir

### 3.1 Requisitos

- Python 3.10+ (ou Anaconda equivalente)
- Jupyter Notebook / JupyterLab
- Dependências:
  - `pandas`
  - `numpy`
  - `matplotlib`

Instalação sugerida:

```bash
pip install pandas numpy matplotlib jupyter
```

### 3.2 Dados

Os arquivos parquet devem estar no diretório raiz do projeto com os nomes:

- `whatsapp_base_disparo_mascarado`
- `whatsapp_dim_telefone_mascarado`

### 3.3 Execução

1. Abra o notebook:
  - `case-ds-pleno.ipynb`
2. Execute as células em ordem (`Run All`).
3. Verifique as saídas de:
  - ranking de sistemas (`ranking_sistemas`);
  - tabela de candidatos e seleção `top2_por_cpf`;
  - seção do experimento A/B.

---

## 4) Estrutura de entrega

- `case-ds-pleno.ipynb`: solução principal.
- `README.md`: abordagem, premissas e reprodução.

---

## 5) Observações finais

- A solução prioriza transparência de regra de negócio e interpretabilidade.
- Os pesos e parâmetros (prior bayesiano, meia-vida, bônus multissistema) podem ser calibrados com validação online (A/B) e monitoramento contínuo.

