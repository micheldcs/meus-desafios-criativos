# 🤖 Desafio DIO — Prompt para Automação no n8n

## Relatório Semanal de Mercado por Gerente Regional

Automação real em produção numa consolidadora aérea B2B: toda semana, cada gerente regional recebe por e-mail uma análise executiva, gerada por IA, sobre o desempenho de vendas **apenas das bases sob sua responsabilidade**.

---

## 🧱 Passo 1 — Definição da automação

```text
Quero criar uma automação no n8n para gerar e enviar semanalmente um relatório
executivo de vendas por gerente regional, com análise escrita por IA.

Público ou responsável:
Gerentes regionais da área comercial (cada um recebe só as suas bases)
e a área de BI, que recebe um resumo extra com os destaques de clientes.

Resultado esperado:
Toda terça-feira de manhã, cada gerente recebe um e-mail em HTML com:
- uma análise executiva (Doméstico e Internacional) escrita por IA;
- tabelas com venda e variação % em 7, 15, 30 e 90 dias por base;
- o impacto financeiro (R$) da semana em cada base.
```

---

## 🧱 Passo 2 — Contexto e regras

```text
Ferramentas envolvidas:
- Microsoft Entra ID (OAuth2 client credentials) para autenticação
- Power BI REST API (endpoint executeQueries) para consultar o modelo semântico via DAX
- Google Gemini (via nó AI Agent do n8n) para redigir a análise
- SMTP para envio dos e-mails

Fluxo desejado:
1. Schedule Trigger dispara toda terça-feira às 10h30.
2. Obter token de acesso no Entra ID.
3. Executar em paralelo duas consultas DAX no Power BI:
   uma para a empresa A (bases do Sul/Centro-Oeste) e outra para a
   empresa B (bases do Sudeste/Nordeste/Norte), agrupadas por BASE e
   tipo de viagem (Nacional/Internacional).
4. Dividir as linhas de cada resposta (Split Out) e juntar os dois
   resultados (Merge).
5. Nó Code: normalizar números (formato pt-BR), calcular impacto em R$
   (venda 7d − venda 7d anterior), ranking por tipo e sinal de tendência
   (Alta / Baixa / Estável a partir da evolução de 90 dias).
6. Nó Code: agrupar os registros por gerente usando um mapa
   BASE → e-mail(s) do gerente.
7. Loop (Split in Batches) por gerente:
   a. montar a tabela textual Doméstico / Internacional;
   b. enviar ao AI Agent (Gemini) com o prompt de analista;
   c. converter a resposta da IA em HTML com as tabelas coloridas;
   d. enviar o e-mail ao gerente;
   e. aguardar 7 minutos (Wait) antes do próximo, para respeitar
      limites de envio e de cota da API.
8. Fluxo paralelo: consultar vendas por cliente e enviar à área de BI
   o Top 10 de maior alta, maior queda e clientes reativados.

Regras importantes:
- Excluir vendas canceladas, clientes inativos, clientes internos e
  fornecedores que não são venda real (reembolsos, ADM/ACM, estornos).
- Cada gerente só pode ver as próprias bases: a IA nunca deve citar
  outras filiais, ranking nacional ou "share de mercado".
- Foco nos indicadores de 7 e 15 dias; 90 dias é só contexto.
- Se o gerente não tiver bases internacionais (ou domésticas), a seção
  correspondente é omitida inteira — nunca enviada vazia.
- Quando a tendência de 90 dias e o movimento da semana divergirem,
  a IA deve citar o contraste explicitamente.
- A IA não pode inventar causas (economia, concorrência, sazonalidade).
- Modo de teste (flag TEST_MODE) redireciona todos os e-mails para um
  único destinatário antes de ativar em produção.
- Gerente sem nenhum registro na semana não recebe e-mail.
- Segredos (client secret, chaves de API) ficam somente no gerenciador
  de credenciais do n8n, nunca no corpo dos nós.
```

---

## 🧱 Passo 3 — Prompt final

```text
Atue como um especialista em n8n, Power BI e automação de relatórios com IA.

Crie uma automação para gerar e enviar semanalmente um relatório executivo
de vendas por gerente regional, com análise escrita por IA.

Público:
Gerentes regionais da área comercial (cada um vê apenas as suas bases)
e a área de BI (recebe também o Top 10 de clientes da semana).

Ferramentas envolvidas:
Microsoft Entra ID (OAuth2 client credentials), Power BI REST API
(executeQueries com DAX), Google Gemini via AI Agent e SMTP.

Fluxo:
1. Schedule Trigger toda terça às 10h30.
2. HTTP Request para obter o token no Entra ID.
3. Duas HTTP Requests em paralelo ao Power BI (uma por empresa do grupo),
   com SUMMARIZECOLUMNS por BASE e tipo de viagem, trazendo venda em
   7/15/30/90 dias, períodos anteriores e evolução %.
4. Split Out + Merge para unificar os resultados.
5. Code: normalizar números pt-BR, calcular impacto em R$, ranking e
   sinal de tendência.
6. Code: agrupar por gerente com um mapa BASE → e-mails.
7. Split in Batches por gerente → Code (tabela textual) → AI Agent com
   Gemini → Code (HTML do e-mail) → Send Email → Wait de 7 minutos →
   volta ao loop.
8. Ramo paralelo: vendas por cliente → Top 10 alta, queda e reativação
   → AI Agent → e-mail para a área de BI.

Regras:
- Filtrar canceladas, inativos, internos e fornecedores que não são venda.
- A IA analisa somente as bases do gerente; nada de comparações externas,
  ranking nacional, share de mercado ou causas inventadas.
- Prioridade dos indicadores: Cresc 7d > Cresc 15d > Impacto R$ 7d >
  Cresc 30d > Tendência 90d.
- Omitir seções sem dados; citar quando tendência e semana divergirem.
- Flag de modo de teste redirecionando todos os e-mails.
- Não enviar e-mail para gerente sem dados.
- Credenciais apenas no cofre de credenciais do n8n.

Explique quais nós do n8n devem ser utilizados, como os dados trafegam
entre eles e a lógica de funcionamento do workflow, incluindo como
tratar falhas (token expirado, resposta vazia da API, erro da IA).
```

---

## 🧩 Nós do n8n utilizados

| Etapa | Nó |
|---|---|
| Agendamento | Schedule Trigger |
| Autenticação e consultas | HTTP Request |
| Quebra das linhas | Split Out |
| União das empresas | Merge |
| Cálculos e agrupamento | Code (JavaScript) |
| Iteração por gerente | Loop Over Items (Split in Batches) |
| Análise escrita | AI Agent + Google Gemini Chat Model |
| Envio | Send Email (SMTP) |
| Controle de ritmo | Wait |

## 💡 Aprendizados

- Separar **cálculo** (Code) de **interpretação** (IA) deixa os números confiáveis e a IA focada só em narrar.
- Um prompt de sistema com hierarquia de indicadores, proibições e checklist de validação reduz muito as "alucinações" da análise.
- Personalizar por destinatário (só as bases de cada gerente) torna o relatório lido de verdade, em vez de mais um e-mail genérico.
