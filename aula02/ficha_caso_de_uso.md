# Ficha do caso de uso — LLM no MVP

**Grandes Modelos de Linguagem · Residência em Engenharia de Software Inteligente (FACOM/UFMS)**
Aula 2 · Parte B · em grupo · entrega até 18/11/2026

**Grupo:** Grupo do MVP Inventário (SGM/TJMS)
**Integrantes:** Arthur Bueno, Sueli Joboji, Henrique Chinaglia, André Kley
**MVP:** Módulo de Inventário Patrimonial do SGM (`sgm-php`): coleta de patrimônio por celular ou leitor de código de barras, com aprovação das digitações manuais.

## 1. Funcionalidade inteligente e problema

**Funcionalidade:** classificar a *observação* em texto livre que o coletor escreve ao digitar à mão o número de um patrimônio, indicando qual dos motivos da lista (`motivo_digitacao`) ela descreve.

**Problema:** a digitação manual é a exceção da coleta e fica auditada. O coletor escolhe um motivo numa lista fechada e pode escrever uma observação, mas escolhe "Outro" quando não sabe e a observação fica sem estrutura. Sem classificar o texto, a coordenadoria não enxerga por que se digita (plaqueta danificada, apagada, gravada, impressa, leitor falhando) e não detecta quando o motivo escolhido contradiz o que foi escrito.

## 2. Situação atual: o modelo construído no bloco

O grupo **não tinha um modelo próprio desta funcionalidade** do bloco anterior. Como referência, usamos o baseline do Exercício 2 da Aula 2 (BERTimbau + regressão logística), treinado agora.

| Item | Descrição |
|---|---|
| Tarefa | Classificação de texto em 5 classes: `etiqueta_danificada`, `etiqueta_ilegivel`, `plaqueta_impressa`, `plaqueta_gravada_material`, `outro` (as opções atuais do enum do sistema) |
| Dados de treino | 60 observações **sintéticas** (12 por classe), redigidas no padrão do fluxo; rótulo = motivo correspondente |
| Métrica usada | Acurácia e F1 por classe, no conjunto de teste do grupo (25 exemplos, fora do treino) |
| Desempenho obtido | **72%** de acurácia, F1 macro 0,72. Por dificuldade: fácil 90%, média 86%, difícil **38%**. Com apenas 40 exemplos de treino (Ex. 2): 55% |
| Pontos fracos observados | Confunde `etiqueta_danificada` com `etiqueta_ilegivel`; erra textos ambíguos, informais ou com erro de grafia; a probabilidade que ele devolve quase não separa acertos (média 0,33) de erros (0,27); dados sintéticos, ainda sem validação com observações reais |

## 3. Hipótese de uso do LLM

- [ ] **Substituir**
- [ ] **Complementar**
- [x] **Híbrido** — cada um atua onde é melhor.

**Justificativa:** nos 25 exemplos os erros dos dois são complementares: só o classificador acertou 7, só o LLM (Qwen2.5-1.5B, zero-shot, 60% de acurácia) acertou 4, e escolher sempre o certo daria 88%. O LLM não saiu do formato em nenhuma resposta (0/25) e dispensa dados rotulados; o classificador responde em ~7 ms e o LLM local (Qwen 1,5B) em ~0,23 s na GPU T4, ambos gratuitos por rodarem localmente. Substituir não se sustenta (o LLM pequeno ficou abaixo do classificador). Ainda **não sabemos como rotear**: a confiança da regressão logística não discrimina bem, e o desenho do roteamento será o foco da oficina (inclusive testar um LLM maior).

## 4. Entrada e saída

| | Descrição | Formato | Exemplo |
|---|---|---|---|
| **Entrada** | Observação digitada na coleta manual (opcionalmente, o motivo já escolhido) | Texto livre, 1 a 3 frases | `Número gravado a punção na estrutura metálica da cadeira, não tem adesivo.` |
| **Saída** | Motivo previsto e justificativa curta | JSON `{"motivo": <uma das 5 classes>, "justificativa": <texto>}` | `{"motivo": "plaqueta_gravada_material", "justificativa": "Número gravado no metal, sem adesivo."}` |

Tamanho típico da entrada, em tokens (Parte A): **cerca de 13 tokens** (média do conjunto de teste: GPT-4o 13,0 · BERTimbau 13,3 · Qwen 15,6; máximo de 19 a 23). Com o prompt completo, a chamada tem ~135 tokens de entrada e 6 de saída. Siglas e números (ex.: `PJ 0098765`) fragmentam mais: o Qwen gasta um token por dígito.

## 5. Critério de sucesso

| Critério | Meta |
|---|---|
| Métrica principal e limiar | Acurácia **≥ 85%** no `conjunto_teste.csv` e **≥ 60%** nos 8 exemplos difíceis (referência atual: 72% e 38%; teto observado de 88% com o híbrido ideal) |
| Latência máxima por requisição | **≤ 2 s** (p95) quando o LLM é acionado; classificador próprio **< 100 ms** (medido na T4: classificador 4 a 7 ms; LLM Qwen 1,5B local ~0,23 s por chamada, 7,5 s em CPU; um LLM maior ou via API ainda será medido na oficina) |
| Custo máximo aceitável | **≤ R$ 0,01 por chamada** ao LLM e **≤ R$ 50/mês**, com no máximo ~30% das classificações indo ao LLM (volume mensal ainda a estimar) |
| Outros | **100%** das saídas em JSON válido e com `motivo` dentre as 5 classes (referência atual em texto simples: 0 respostas fora do formato em 25) |

## 6. Conjunto de teste

- Arquivo: `conjunto_teste.csv`, **25 exemplos rotulados** (5 danificada, 6 ilegível, 5 impressa, 4 gravada, 5 outro).
- Os mesmos exemplos comparam o classificador e o LLM; nenhum aparece no treino (verificado por código).

**Como foi montado:** as observações são **sintéticas**, redigidas a partir do padrão real do fluxo da coleta (sem nomes, CPFs ou matrículas), com apoio de IA generativa. Variedade: as 5 classes, em 3 níveis de dificuldade (10 fáceis, 7 médias, 8 difíceis). Os difíceis incluem textos ambíguos entre duas classes (ex.: "rasurada e amassada"), informais e sem pontuação ("digitei pq o leitor nao pega"), com erro de grafia e que misturam motivos; a coluna `observacao` do CSV registra a ambiguidade. Rótulos definidos por um integrante (Arthur Bueno) segundo o significado de cada opção no sistema; **a revisão cruzada por um segundo integrante ainda está pendente.**

## 7. Restrições de dados

| Pergunta | Resposta |
|---|---|
| Dados pessoais ou sensíveis (LGPD)? | Observações reais podem citar nome de servidor, sala ou número de patrimônio (dado pessoal indireto); não se espera dado sensível (art. 5º, II). O teste e o treino atuais são sintéticos. |
| Pode enviar para API externa? | **A validar** com a preceptora e a política de TI do TJMS. Enquanto não houver autorização, nenhum dado real sai do ambiente. |
| Dá para anonimizar antes do envio? | Sim: a classificação depende só da descrição da plaqueta, então dá para enviar apenas o texto da observação, sem identificar coletor, espaço ou processo, mascarando nomes e números antes de montar o prompt. |
| Restrição de licença ou uso comercial? | BERTimbau (MIT) e Qwen2.5-1.5B-Instruct (Apache 2.0) permitem uso comercial (conferir model cards). O PTT5 de sumarização (CC BY-NC-SA 4.0) **não** pode ser usado no MVP, e ainda alucinou nos testes. |
| Alternativa sem API externa | Modelo aberto local (Qwen2.5 via Ollama/servidor próprio), já testado no Colab com 60% zero-shot; modelos maiores exigem GPU no ambiente do tribunal. |

---

### Uso de IA generativa na elaboração da ficha

Foi usada IA generativa (Claude Code, da Anthropic) para: ler o material da disciplina; propor a funcionalidade a partir do código do `sgm-php`; redigir os textos sintéticos de treino e de teste; adaptar o notebook guiado; executar o notebook e calcular as métricas; e redigir a primeira versão desta ficha. As decisões (funcionalidade, classes, hipótese, metas) foram do integrante Arthur Bueno, que executou o notebook no Colab (GPU T4); os números desta ficha foram conferidos contra essa execução. Os dados são sintéticos; nenhum dado real do sistema foi enviado a serviços externos.
