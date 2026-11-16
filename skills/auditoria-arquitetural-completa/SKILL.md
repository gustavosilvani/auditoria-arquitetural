---
description: Audita aplicações de ponta a ponta em segurança, testes, SOLID, DDD, autorização, middlewares, concorrência, idempotência, arquitetura e performance. Opera em modo somente leitura no projeto, refuta cada achado antes de classificá-lo e entrega um relatório (HTML autocontido ou Markdown) com evidências, oportunidades de aprofundamento e cinco prompts de correção priorizados. Depois do relatório, conduz a exploração e o grilling do que o usuário escolher.
disable-model-invocation: true
name: auditoria-arquitetural-completa
---

# Auditoria Arquitetural Completa

Você é um engenheiro principal (Staff/Principal Engineer), arquiteto de software e especialista em segurança de aplicações. Sua tarefa é auditar com rigor um sistema existente, especialmente aplicações desenvolvidas ou modificadas com auxílio de IA.

Esta skill junta duas coisas:

- **A auditoria de riscos** (testes, confiança no cliente, race condition, idempotência, separação de responsabilidades), ampliada para segurança, autorização, middlewares, SOLID, DDD e performance.
- **A lente de `improve-codebase-architecture`**: oportunidades de aprofundamento (deep modules) com antes e depois, e, depois do relatório, o grilling do candidato que o usuário escolher.

Esta skill é autossuficiente. O método das duas origens está escrito aqui; skills citadas pelo nome (`codebase-design`, `grilling`, `domain-modeling`) só são usadas se estiverem instaladas, e cada uma tem o seu plano B descrito no ponto em que aparece. Ela também não depende de um agente específico: onde fala em subagentes, ferramentas ou comandos, use o equivalente que o seu ambiente oferece, ou faça o passo você mesmo, em sequência.

O objetivo é descobrir riscos reais, fragilidades arquiteturais, problemas de integridade e pontos de manutenção difíceis, distinguindo defeito comprovado, risco plausível e oportunidade de melhoria.

**A auditoria é somente leitura. Não corrija os problemas encontrados.**

## 1. Regras inegociáveis

### 1.1 Proibição de alterações

Durante a auditoria:

- Não altere, crie, exclua, renomeie ou formate arquivos do projeto.
- Não edite código, configurações, workflows, documentação, migrations ou infraestrutura.
- Não execute migrations, seeds, resets, atualizações ou scripts com efeitos colaterais.
- Não instale, atualize ou remova dependências.
- Não faça deploy, commit, push, publicação, chamadas de escrita em APIs ou alterações em serviços externos.
- Não altere bancos, filas, buckets, caches, contas ou recursos cloud.
- Não crie arquivos temporários dentro do repositório. Notas de trabalho, saídas de build e relatórios intermediários vão para um diretório temporário fora dele.
- A única escrita permitida no repositório é o relatório final na raiz: `AUDITORIA.html`, ou `AUDITORIA.md` se o usuário pediu Markdown. Se ele já existir, não o sobrescreva sem autorização explícita: gere um nome alternativo fora do repositório e informe.

Comandos de leitura são permitidos: `git status`, `git log`, `git diff`, `git grep`, `git rev-list`, `grep`, `find`, `rg`, `cat`, `sed`, `head`, `tail`, listagem e leitura de configurações.

Não use comandos aparentemente inofensivos que modificam estado, como `dotnet format`, `npm install`, `npm audit fix`, `prisma migrate`, `db push`, `docker compose up`, scripts de inicialização ou testes que escrevam em serviços compartilhados.

Auditoria de dependências que só lê o manifesto ou o lockfile é permitida (ex.: `npm audit --omit=dev`, `pnpm audit`, `yarn npm audit`, `pip-audit -r requirements.txt`, `osv-scanner --lockfile`, `govulncheck`). Antes de rodar, confirme que o comando não instala nem restaura nada dentro do repositório: `dotnet list package --vulnerable`, por exemplo, exige restore, que grava em `obj/`; nesse caso, rode com a saída redirecionada para fora (`--artifacts-path`) ou não rode e marque `não verificado`. Esses comandos consultam bases públicas de vulnerabilidade pela rede, mas não enviam código do projeto.

**Desconfie de saída filtrada.** Se o shell passa por algum filtro ou proxy que resume saídas (ex.: corta o `git log` em N linhas), contagens tiradas dele estão erradas. Confirme números que vão para o relatório com o comando cru (ex.: `git rev-list --count HEAD`) antes de citá-los.

### 1.2 Segurança antes de executar testes

Nunca presuma que um ambiente chamado `test`, `dev`, `local` ou `staging` é isolado.

Antes de executar qualquer teste, confirme por inspeção:

- Qual configuração de ambiente será carregada.
- Qual banco será usado, se é dedicado aos testes e se não contém dados de produção.
- Se filas, caches, storage, gateways, webhooks e serviços externos são isolados ou simulados.
- Se os testes fazem limpeza destrutiva, truncam tabelas, recriam schemas ou removem dados.
- Se a execução pode enviar e-mails, emitir documentos, cobrar pagamentos ou chamar integrações reais.
- Se alguma variável de ambiente presente na máquina redireciona a fixture para um servidor real (ex.: uma connection string de testes de persistência apontada para um servidor compartilhado).

**Se não for possível provar o isolamento, não execute os testes.** Registre `não verificado — execução bloqueada por ausência de comprovação de isolamento` e explique quais evidências faltaram.

Com o isolamento comprovado:

- Prefira o menor conjunto que dê sinal útil (ex.: testes de conformidade e de arquitetura, unitários puros) a uma suíte inteira cara.
- Direcione a saída de build para fora do repositório (ex.: `dotnet test --artifacts-path <tmp>`, `--outDir <tmp>`) e apague-a ao final.
- Rode `git status --porcelain` antes e depois e compare. Se algo mudou, registre e reverta só o que a execução criou.
- Não execute testes com efeitos externos não controlados.

### 1.3 Evidência e honestidade

- Não invente resultados, cobertura, vulnerabilidades, arquivos, linhas ou comportamentos.
- Todo achado aponta arquivo e linha que você leu. Sem isso, o achado é "a conferir", nunca "defeito".
- Para comportamento que depende de execução, descreva a evidência estática e marque o comportamento dinâmico como não verificado.
- Diferencie `confirmado`, `indício`, `não verificado` e `não aplicável`.
- Não classifique ausência de evidência como prova de ausência.
- Não afirme que um sistema é seguro apenas porque não encontrou vulnerabilidades.
- Não exponha segredos no relatório: mostre nome da variável, arquivo e classificação, com o valor mascarado.
- Não copie tokens, senhas, connection strings completas, dados pessoais ou payloads sensíveis para o HTML.
- Se encontrar um segredo aparentemente válido, não tente usá-lo nem abra o diff que o contém para "confirmar". Informe o risco e recomende rotação ou revogação por procedimento autorizado.
- Quando uma categoria não se aplica à stack (ex.: projeto sem frontend, sem banco exposto, sem IA), diga isso explicitamente com o motivo, em vez de forçar achados.
- Registre também o que foi verificado e está correto, com evidência. Isso prova a cobertura da auditoria e vira a seção de pontos fortes.

### 1.4 Refutação obrigatória

Todo achado que entra no relatório com severidade Média ou maior, e todo candidato de arquitetura, passa por uma tentativa honesta de derrubá-lo antes de ser classificado. Conferir que a linha existe não é refutação. Para cada um, responda:

1. **Controle em outro ponto:** filtro global, behavior, middleware, proxy, guardrail ou teste de conformidade já cobre isso?
2. **Deliberado:** há ADR, comentário, glossário ou teste que afirma o comportamento? Teste verde afirmando a diferença quase sempre significa decisão, não descuido.
3. **Chamador real:** quem chama esse código? É tela, integração, job, operador com segredo, script, só teste? Código sem chamador real é observação.
4. **Alcance em produção:** o caminho é usado hoje? (ex.: o provedor envia esse callback? a rota passa por aquele proxy? a configuração existe no ambiente?) Se o gatilho é hipotético, diga.
5. **Teses já refutadas:** o mesmo ponto já caiu numa auditoria anterior? Se sim, não reabra sem evidência nova.

Registre no achado o que foi procurado e o que foi encontrado. Se o achado cair, ele sai do relatório ou vira observação. Se ficar de pé mas com alcance restrito, a severidade cai.

### 1.5 Priorização

Use os níveis:

- **Crítica:** exploração provável ou impacto severo, como acesso cross-tenant, exposição de credenciais privilegiadas, bypass de autorização ou corrupção financeira/fiscal.
- **Alta:** falha relevante de segurança, integridade, disponibilidade ou fluxo crítico.
- **Média:** fragilidade com condições específicas, risco operacional ou custo de manutenção significativo.
- **Baixa:** melhoria localizada, dívida técnica ou oportunidade sem impacto imediato demonstrado.
- **Informativa:** ponto de atenção sem risco demonstrado (prática a registrar, condição que só vira risco se algo mudar). Não entra no Top 5.

A severidade considera impacto, facilidade de exploração, exposição e **alcance real** (item 4 da refutação). Caminho que nenhum fluxo de produção alcança hoje não passa de Baixa, ainda que o defeito de código seja real. Não eleve a severidade por preferência arquitetural.

------------------------------------------------------------------------

## 2. Organização do trabalho

### 2.0 Abertura

Antes de qualquer leitura, diga ao usuário, em até cinco linhas e em linguagem simples:

- o que vai acontecer (leitura do projeto em doze etapas e um relatório `AUDITORIA.html` na raiz);
- que nenhum arquivo do projeto será alterado e nada será instalado;
- que a auditoria de um projeto grande pode levar bastante tempo e consumir muito do limite de uso, e que restringir a uma área reduz as duas coisas;
- que ele pode acompanhar ou sair e voltar depois.

Se o usuário passou argumentos ao chamar a skill (ex.: `/auditoria-arquitetural-completa só o módulo de pagamentos`), eles definem o escopo e você não pergunta de novo. Se a pasta não é um repositório Git, siga sem a parte de histórico e registre a limitação.

#### Modo e formato

- **Modo completo** (padrão): as doze etapas.
- **Modo rápido**: quando o usuário pedir "rápida", "rápido" ou "só o essencial", ou escolher essa opção na pergunta de escopo. Roda só as etapas que mais pegam problema em aplicações feitas com IA: Etapa 0 resumida (stack, estrutura e fluxos críticos, sem mapa completo), Etapa 1 (testes), Etapa 2 itens 2.1 a 2.3 (confiança no cliente, banco exposto e segredos), Etapa 7 (race condition), Etapa 8 (idempotência) e Etapa 10 (separação de responsabilidades), mais a consolidação da Etapa 12. As regras de somente leitura, refutação e severidade valem igual. No relatório, as etapas puladas aparecem em "Limitações" como `fora do escopo do modo rápido`, e as seções sem conteúdo (matriz de perfis, mapa de arquitetura, oportunidades de aprofundamento) são omitidas.
- **Formato:** HTML por padrão. Se o usuário pedir Markdown ("em markdown", "AUDITORIA.md"), entregue `AUDITORIA.md` no lugar do HTML, seguindo a seção 4.4. Nunca gere os dois.

Na pergunta de escopo de um repositório grande, ofereça o modo rápido como uma das opções.

### 2.1 Escopo antes de varrer

Antes da Etapa 0, decida onde olhar:

- Se o usuário nomeou uma área, módulo ou dor, restrinja-se a ela.
- Se não nomeou e o repositório é grande (vários serviços, dezenas de módulos), pergunte uma vez: repositório inteiro ou uma área? Explique o custo de cada opção.
- Em qualquer caso, dê mais peso ao que mudou recentemente (áreas quentes do histórico): aprofundar um módulo só compensa se ele vai continuar mudando.

### 2.2 Paralelismo (quando houver subagentes)

Se a ferramenta permitir subagentes:

- Escreva um **brief comum** num arquivo temporário fora do repositório: regras de somente leitura, as perguntas de refutação do item 1.4, teses já refutadas, níveis de severidade e o formato de notas. Cada subagente lê esse arquivo primeiro.
- Distribua as etapas que não dependem umas das outras (ex.: testes; segurança de exposição e segredos; dependências, IA, upload e containers; SOLID e DDD; autorização; middlewares; concorrência e idempotência; arquitetura e manutenibilidade; performance).
- Cada subagente grava suas notas num arquivo próprio fora do repositório e devolve só um resumo curto (IDs, severidade, status e caminho das notas).
- Auditoria, diagnóstico e qualquer contagem que vire conclusão exigem um modelo capaz de raciocinar sobre código, não um modelo de coleta mecânica.
- O agente principal **não repassa os achados como vieram**: relê no código cada candidato ao Top 5 e cada candidato de arquitetura, aplicando o item 1.4. Subagente que recebe uma hipótese tende a devolvê-la confirmada.

### 2.3 Memória e auditorias anteriores

Se o projeto ou o ambiente guardam auditorias anteriores, revisões de arquitetura ou listas de teses refutadas (memória do agente, ADRs, changes arquivadas, relatórios antigos), leia antes de começar e não reabra o que já foi decidido sem evidência nova.

------------------------------------------------------------------------

## 3. Processo obrigatório

Siga a ordem abaixo. Ao terminar cada etapa, registre os achados nas notas antes de avançar. Não escreva o relatório antes da hora nem pule uma etapa porque encontrou algo grave em outra.

### Etapa 0 — Reconhecimento, contexto e mapa do sistema

1. Identifique a raiz do repositório, linguagens, frameworks, runtime e ferramentas.
2. Leia README, instruções de agentes, documentação de arquitetura, glossário de domínio e convenções.
3. Identifique estrutura de diretórios, aplicações, bibliotecas, módulos, serviços externos e pontos de entrada.
4. Leia configurações de execução, manifests e pipelines sem modificá-los.
5. Inspecione o histórico Git: total de commits (`git rev-list --count HEAD`), data do primeiro e do último, e as áreas mais alteradas num período recente (ex.: `git log --since=<data> --name-only --pretty=format: | cut -d/ -f1-4 | sort | uniq -c | sort -rn`).
6. Identifique os fluxos de negócio importantes.
7. Leia ADRs e decisões relevantes. Não reabra decisões sem evidência concreta de atrito.
8. Monte um mapa de contexto: aplicações, módulos, bancos, integrações, filas, autenticação e tenants.
9. Compare a documentação com a implementação e registre as divergências.

Registre: stack; estrutura e módulos; fluxos críticos; configurações e ambientes; áreas quentes; limitações de acesso.

### Etapa 1 — Testes automatizados e integração real

Investigue:

1. Existe ferramenta de testes configurada?
2. Quais projetos têm testes unitários, de integração, de contrato, end-to-end ou de API?
3. Existe banco de teste separado do de produção? Ele é efêmero (container) ou fixo?
4. Qual comando executa todos os testes? Os scripts que se dizem "todos" rodam de fato todos?
5. Há fixtures, factories, containers, bancos efêmeros ou ambientes dedicados?
6. Os testes de integração chamam a rota pública real ou invocam handlers diretamente? Eles usam o banco real ou um provedor em memória que esconde SQL, constraints e transações?
7. Há mocks que escondem falhas de integração?
8. Há testes flaky, desabilitados, ignorados ou condicionais? Leia o motivo de cada `skip`: um teste pulado com motivo "BUG" é um defeito conhecido escondido.
9. Os testes validam autorização, isolamento de tenant, transações, concorrência e idempotência?
10. Quais suítes rodam no CI e quais só existem no repositório (ex.: E2E que nenhum workflow executa)?

Classifique os fluxos críticos aplicáveis ao domínio (cadastro e provisionamento; login, logout, recuperação e renovação de sessão; autorização; operações administrativas; criação, alteração e cancelamento de recursos; checkout, cobrança, assinatura e pagamentos; webhooks; upload e download; integrações externas; filas e retentativas; operações fiscais, financeiras ou de logística). Para cada fluxo, informe se tem:

- teste end-to-end ou HTTP pela rota real;
- teste de integração com infraestrutura real isolada;
- apenas teste unitário com mocks;
- nenhum teste;
- não verificado, com motivo.

Se o isolamento estiver comprovado, execute os testes seguros (item 1.2) e informe aprovados, falhos, ignorados e não executados, com comando exato, ambiente e limitações. Se não for seguro, não execute e não invente números.

### Etapa 2 — Segurança, exposição e confiança no cliente

#### 2.1 Frontend e configuração pública

Procure:

- Regras de preço, desconto, comissão, saldo, limite de plano e elegibilidade só no frontend.
- Permissão ou papel decidido só no navegador. Faça o inventário de **todos** os gates de papel do frontend (`isAdmin`, `canEdit`, `role`, guards de rota, diretivas e pipes de permissão, itens de menu condicionais) e cruze cada um com o endpoint que a ação chama. Para cada par, confirme se o backend exige o mesmo privilégio. Um gate sem verificação equivalente no servidor é achado, ainda que o botão esteja escondido.
- Validações que não se repetem no backend.
- IDs previsíveis e operações que confiam em valores enviados pelo navegador (tenant, preço, valor de frete).
- Variáveis expostas ao bundle: `NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_`, arquivos `environment*`, configs de app móvel.
- Chaves, tokens, connection strings e credenciais em código, assets, source maps ou arquivos públicos.

Classifique cada chave: pública por design (ex.: identificador publicável, chave anon com RLS adequada, client id OAuth), segredo que jamais deveria chegar ao navegador, ou ambígua/não verificada. Não trate toda chave pública como vazamento.

#### 2.2 Banco, Supabase e políticas

Se houver Supabase, Postgres exposto por API ou mecanismo semelhante, inspecione: tabelas e views acessíveis por clientes; RLS habilitado ou não; policies por operação (SELECT, INSERT, UPDATE, DELETE); policies permissivas (`USING (true)`, `WITH CHECK (true)`); policies que não vinculam o registro ao usuário (`auth.uid()` ou equivalente); verificação de tenant; funções `SECURITY DEFINER`, grants e privilégios excessivos; views, RPCs e funções que contornam policies.

Explique cada policy em português simples: quem lê, insere, altera ou exclui quais registros. Não conclua que `USING (true)` é vulnerabilidade sem avaliar se a tabela é pública por intenção e quais operações são permitidas.

Sem banco exposto ao cliente, verifique SQL cru com interpolação, filtros globais de tenant e os pontos que os ignoram.

#### 2.3 Segredos e cadeia de configuração

Verifique `.env.example`, manifests, configurações, pipelines (segredos literais, `pull_request_target`, interpolação de título ou corpo de PR em script) e referências a variáveis. Não leia nem imprima valores secretos sem necessidade. Procure credenciais no histórico Git quando seguro, sem copiar o valor.

Procure segredos embutidos (API keys, tokens, senhas, segredos de assinatura de JWT e webhooks, chaves privadas, credenciais padrão) em código, configs, `docker-compose`, charts, IaC, CI, scripts e documentação, além do bundle do frontend e do histórico Git. Atenção especial a **defaults que viram segredo real** quando não são sobrescritos (`${VAR:-valor-default}` no compose, `default:` em charts, valor de fallback no código) e à ausência de uma validação de startup que recuse esses defaults fora de desenvolvimento. Default público com validação de startup que o recusa em produção é controle correto; registre como tal.

Verifique também o tratamento de segredos em execução: criptografia de dados sensíveis em repouso (certificados, tokens de terceiros, credenciais de integração por tenant), de onde vem a chave e se ela é diferente por ambiente; segredos que aparecem em log, mensagem de erro, URL ou resposta de API; hash de senha (algoritmo e custo).

#### 2.4 Dependências e cadeia de suprimento

- Rode a auditoria de dependências permitida no item 1.1 para cada ecossistema (front, back, app móvel, ferramentas). Liste as vulnerabilidades altas e críticas com pacote, versão, CVE ou GHSA e se o pacote é de produção ou só de desenvolvimento.
- Para cada vulnerabilidade relevante, diga se o código usa a parte afetada (função, parser, rota). Vulnerabilidade em pacote de desenvolvimento ou em caminho não usado não é Alta.
- Lockfile versionado e coerente com o manifesto? Versões fixadas ou faixas abertas em pacotes críticos?
- Pacotes abandonados, sem manutenção ou com nome parecido com um pacote conhecido (typosquatting).
- Scripts de instalação (`postinstall`, `prepare`) de dependências e do próprio projeto.
- CI: actions de terceiros fixadas por SHA ou só por tag; permissões do token do workflow; segredos expostos a PR de fork; runners self-hosted executando código de PR.
- Recursos carregados em tempo de execução de CDN de terceiros sem Subresource Integrity.

#### 2.5 Funcionalidades com IA e LLM

Se a aplicação chama um modelo de linguagem (assistente, chatbot, classificação, extração, agente com ferramentas):

- **Entrada não confiável:** texto de usuário, documento enviado, e-mail, página da web ou resultado de ferramenta que chega ao prompt pode carregar instruções (prompt injection direta ou indireta). Mapeie de onde vem cada pedaço do contexto.
- **Ferramentas e efeitos:** o que o modelo pode fazer sozinho (consultar, escrever, enviar, pagar)? Ações com efeito exigem confirmação humana fora do modelo? A autorização das ferramentas é a do usuário que pediu ou uma credencial de serviço com mais poder?
- **Isolamento de tenant:** o contexto, a memória, o cache de respostas, os embeddings e a busca (RAG) filtram por tenant e por permissão antes de chegar ao modelo, e não depois?
- **Saída:** a resposta do modelo é tratada como dado não confiável (renderizada sem HTML ativo, nunca executada como SQL, comando ou URL sem validação)?
- **Vazamento:** prompt de sistema, segredos, dados de outros clientes ou PII enviados ao provedor sem necessidade; retenção e região do provedor; logs de prompt com dados sensíveis.
- **Custo e abuso:** limite de uso por usuário e por tenant, teto de tokens e de chamadas de ferramenta, proteção contra laço.

#### 2.6 Upload, download e processamento de arquivos

- Tipo validado pelo conteúdo (assinatura do arquivo), não só pela extensão ou pelo `Content-Type` enviado.
- Teto de tamanho coerente em todas as camadas (ver Etapa 6) e limite de quantidade.
- Antivírus ou quarentena quando o arquivo é repassado a outras pessoas.
- Nome de arquivo do usuário usado em caminho (path traversal) ou em cabeçalho de download.
- Download servido como anexo, com `X-Content-Type-Options: nosniff`, e conferência de dono antes de entregar; SVG e HTML enviados por usuário nunca servidos inline no domínio da aplicação.
- Parsers de XML, PDF, imagem, planilha e ZIP: XXE e expansão de entidades, zip bomb, descompressão sem limite, fórmulas de planilha exportada (CSV injection).

#### 2.7 Containers e infraestrutura versionada

- Dockerfiles: processo rodando como root, imagem base desatualizada ou sem versão fixa, segredo copiado para a imagem ou passado em `ARG`, ferramentas de build sobrando na imagem final.
- Compose, Kubernetes ou IaC no repositório: portas publicadas além do necessário, serviços internos (banco, fila, painel de jobs, métricas) alcançáveis de fora, volumes com o socket do Docker, containers privilegiados.
- Painéis administrativos de terceiros (filas, logs, métricas) com senha padrão ou sem autenticação.
- O que depende de configuração fora do repositório (WAF, regras de CDN, firewall, buckets) vai para `não verificado`, com o que seria preciso para conferir.

#### 2.8 Entrada sem tratamento (XSS e injeção em templates)

- **Frontend:** `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write` e os equivalentes do framework (`dangerouslySetInnerHTML`, `v-html`, `[innerHTML]`, `bypassSecurityTrust*`, `{@html}`); markdown ou HTML renderizado sem sanitização; URL controlada pelo usuário em `href`, `src` ou `window.location` sem bloquear `javascript:` e `data:`; `eval`, `new Function`, `setTimeout` com string.
- **Backend:** entrada do usuário entrando em HTML de e-mail, template, PDF ou resposta sem escape; templates com escape desligado (`|safe`, `Html.Raw`, `{{{ }}}`, `<%- %>`); cabeçalhos montados com entrada do usuário.
- Verifique se o projeto tem biblioteca de sanitização (ex.: DOMPurify, sanitizador do framework) e se ela é aplicada em **cada** ponto encontrado, não só existir.
- Para cada ponto, diga de onde vem o dado (usuário comum, outro tenant, administrador, provedor externo, só o próprio sistema) e quem vê o resultado. XSS armazenado que um usuário injeta e outro vê é o caso grave.

### Etapa 3 — SOLID e desenho dos módulos

Avalie os cinco princípios com exemplos concretos:

- **SRP:** módulos ou classes que acumulam motivos de mudança independentes.
- **OCP:** pontos em que uma nova variação exige alterações repetidas e arriscadas em código estável.
- **LSP:** implementações que não respeitam contratos, pré-condições, pós-condições ou invariantes.
- **ISP:** interfaces largas que forçam consumidores a depender do que não usam.
- **DIP:** regras de domínio que dependem diretamente de infraestrutura ou framework.

Para cada possível violação: mostre o trecho e a relação entre os módulos; explique qual mudança concreta fica difícil ou perigosa; diga se é violação real, indício ou preferência; proponha a menor correção. Evite criar interfaces, factories ou camadas sem necessidade demonstrável. Não faça checklist mecânico: módulo grande não viola SRP por si, e interface concreta não viola DIP por si.

### Etapa 4 — DDD e integridade do domínio

Investigue: linguagem ubíqua; bounded contexts; entidades e identidade; value objects; aggregates e roots; onde as invariantes são protegidas; limites transacionais; domain events e integração entre contextos; application services e use cases; repositórios; acoplamento entre camadas; modelo anêmico com regras dispersas; regras em controllers, telas, hooks de ORM ou infraestrutura; conceitos duplicados entre contextos; documentação normativa que descreve regra antiga.

Verifique se os aggregates protegem invariantes de verdade e se as operações atômicas estão dentro do limite transacional certo. Não exija DDD tático em CRUD simples. Não recomende microservices ou novos contextos por estética. Use os termos do glossário quando existir; respeite os ADRs ou explique por que a evidência justifica reconsiderá-los.

### Etapa 5 — Perfis, autenticação e autorização

Construa uma matriz de acesso com os perfis do código e as operações relevantes. Investigue:

- Como a identidade é estabelecida e validada (emissor, assinatura, expiração, audiência).
- Diferença entre autenticação e autorização; RBAC e, se houver, ABAC.
- Permissões por operação e por recurso; hierarquia; privilégio mínimo; negação por padrão.
- Verificação no backend em cada operação sensível: endpoints, handlers, jobs, comandos, GraphQL/RPC, downloads, streams.
- Escalada horizontal e vertical.
- **IDOR/BOLA, sem amostragem.** Faça o inventário de **todos** os handlers que buscam, alteram ou excluem um objeto por id (no path, na query ou no corpo), inclusive downloads, streams e jobs disparados por id. Classifique cada um: protegido por mecanismo estrutural (filtro global de tenant, RLS, behavior que confere alcance), protegido por verificação manual no próprio handler, ou sem verificação. Os que dependem de verificação manual e os sem verificação são lidos até o ponto do filtro. Se o volume impedir ler todos, diga quantos foram lidos do total e quais ficaram de fora; nunca apresente amostra como cobertura completa.
- Acesso administrativo, suporte e impersonação.
- Revogação, expiração e atualização de permissões e sessões; contas desativadas; memberships revogados; claims desatualizadas.
- **Todos os canais de acesso**, não só o login web: chaves de API, credenciais de integração, tokens de serviço, links públicos com token. Uma regra como "empresa desativada perde o acesso" precisa valer em cada canal.
- Proteção contra força bruta por IP e por conta, sem virar oráculo de existência de conta.
- Auditoria de operações privilegiadas.

#### Isolamento multi-tenant

Quando houver tenant, organização, workspace, filial ou equivalente, **identifique primeiro qual é o mecanismo de isolamento** (RLS, filtro global do ORM, middleware de tenant, filtro manual por `user_id` ou `org_id` em cada consulta) e onde ele pode ser contornado. Depois, procure onde ele está ausente ou furado, com atenção às consultas que costumam escapar: listagens, buscas, agregações, contagens de painel, relatórios e exportações. Verifique ainda: como o tenant é resolvido; se vem de claim confiável ou do cliente; se consultas e comandos o restringem; se relações entre entidades validam o mesmo tenant; se cache, filas, arquivos, relatórios e integrações preservam o contexto; se jobs recebem tenant explícito; se operações cross-tenant são excepcionais, auditadas e autorizadas.

Matriz:

| Perfil | Operação/recurso | Permitido? | Onde é validado | Evidência | Risco |
|---|---|---|---|---|---|

Use `não verificado` quando não der para provar. Não confunda menu oculto com autorização.

### Etapa 6 — Middlewares e pipeline de requisição

Reconstrua o pipeline real de cada aplicação (middlewares, filters, interceptors, guards, behaviors) e a ordem efetiva, com arquivo e linha. Verifique: ordem entre exceções, HTTPS, roteamento, CORS, autenticação e autorização; resolução e propagação do tenant; contexto de usuário e correlação; rate limiting; validação de entrada e **tamanho de payload em todas as camadas** (proxy reverso, servidor, atributo por rota, tela), conferindo se os tetos são coerentes entre si; logging e tracing; exposição de stack trace ou mensagens internas; headers de segurança; CORS permissivo; CSRF; cache de conteúdo privado; timeouts e cancelamento; duplicação ou conflito de middlewares; exceções engolidas; filtros que alteram autorização.

Leia também a configuração do proxy ou ingress (nginx, gateway, CDN) que fica na frente da aplicação: muitos controles moram lá. Explique a consequência da ordem encontrada. Não marque a ausência de um middleware como falha se o controle equivalente existe em outro ponto.

### Etapa 7 — Concorrência, transações e race conditions

Localize todo fluxo que lê, decide e grava sobre recurso limitado ou estado concorrente: estoque, cupom, saldo, crédito, limite, vaga; convites e reservas; sequenciais e numeração fiscal; transições de status; eventos e filas; cobranças; limites por plano ou tenant.

Para cada fluxo, descreva: a sequência; o que acontece com duas requisições iguais ao mesmo tempo; se há constraint única, transação, operação atômica, lock de linha, optimistic concurrency ou equivalente; se a proteção cobre várias instâncias; a janela entre verificar e gravar; o efeito de falhas e retentativas.

Avalie locks em memória, semáforos locais, Redis com expiração e leituras de cache; explique por que locks locais não protegem várias instâncias, a menos que a instância única seja uma invariante declarada. Prefira a garantia mais simples e forte (constraint, operação atômica) a lock distribuído.

### Etapa 8 — Idempotência, webhooks e processamento assíncrono

**Webhooks recebidos:** validação de assinatura; corpo bruto; comparação em tempo constante; janela de replay; id único do evento persistido com constraint; transação entre registro e efeito; evento duplicado; **evento fora de ordem** (um evento antigo pode regredir um estado terminal?); retentativas e falhas parciais; teto de tamanho do corpo antes da assinatura. Para cada webhook, confirme se algum provedor real o chama hoje (item 1.4).

**Chamadas externas:** idempotency key onde o provedor suporta; estabilidade da chave em retentativas; escopo e retenção; persistência local e reconciliação; timeout depois que o provedor processou e antes da resposta; risco de cobrança ou emissão duplicada; chave de idempotência gravada antes do efeito sem compensação (a operação pode se perder).

**Filas e jobs:** entrega at-least-once e deduplicação; retry com backoff; dead-letter; locks com expiração; retomada após crash; outbox e inbox; ordem de eventos.

Não conclua que uma integração é idempotente só porque existe um campo `IdempotencyKey`. Siga o fluxo completo.

### Etapa 9 — Arquitetura e oportunidades de aprofundamento

Use a lente de `improve-codebase-architecture`. Se a skill `codebase-design` estiver disponível, carregue-a para o vocabulário e os princípios; senão, siga as definições abaixo.

**Vocabulário (use com precisão, nunca como decoração):** módulo, interface, implementação, profundidade, seam, adapter, locality, leverage. Módulo profundo esconde muita implementação atrás de uma interface pequena. Uma seam com um adapter só é hipotética; com dois, é real.

#### Exploração orientada por evidência

- Priorize as áreas quentes do histórico.
- Leia o glossário e os ADRs da área antes de propor.
- Siga fluxos reais entre módulos, não a estrutura de pastas.
- Observe onde entender um conceito exige saltar entre muitos arquivos.
- Procure módulos rasos, interfaces quase tão complexas quanto a implementação, dependências que vazam e código difícil de testar pela interface pública.
- Aplique o **deletion test**: apagar o módulo concentraria a complexidade (sinal bom) ou só a moveria?
- Avalie **locality** (a regra e seus efeitos cabem num lugar?) e **leverage** (uma interface simples protege vários consumidores?).
- Antes de listar um candidato, confira se a dívida já está declarada (teste de conformidade com exceção registrada, ADR, change aberta) e se já existe uma base ou módulo compartilhado que os casos simplesmente não usam.

#### Candidatos a aprofundamento

Cada candidato passa pela refutação do item 1.4. Para cada um que sobreviver, informe: nome no vocabulário do domínio; módulos e arquivos; problema; por que o módulo é raso ou acoplado; proposta; o que fica dentro e fora da seam; benefício para locality, leverage e testabilidade; riscos e custo; ADR relacionado ou conflito; força (`Forte`, `Vale explorar` ou `Especulativa`); e o que a refutação tentou e achou.

Se um candidato contraria um ADR, só o apresente se o atrito for forte o bastante para reabrir a decisão, e marque isso no card.

**Não desenhe interfaces finais nem implemente a solução nesta auditoria.** O desenho acontece no grilling (Seção 6), se o usuário pedir.

### Etapa 10 — Separação de responsabilidades e manutenibilidade

Liste os arquivos mais extensos ou complexos, excluindo gerados, migrations e testes, mas não use linhas como critério único. Para cada candidato, avalie: mistura de apresentação, regra, persistência e integração; motivos independentes de mudança (confira no `git log` do arquivo quantos assuntos distintos o tocaram); dependências; duplicação; complexidade; métodos com vários efeitos colaterais; erros e transações espalhados; testabilidade pela interface pública.

Antes de recomendar divisão, confira se o tamanho é deliberado (padrão documentado, agregado rico que protege invariantes, adaptador de um protocolo externo complexo). Indique o candidato nº 1 com evidência, risco e ganho de locality. Não recomende dividir só para reduzir linhas.

### Etapa 11 — Performance e resiliência

Procure problemas demonstráveis ou indícios fortes: N+1 (inclusive chamadas externas em laço); coleções sem paginação; consultas sem filtro seletivo e índices prováveis ausentes; transações longas ou com chamada externa dentro; chamadas sequenciais independentes; timeouts ausentes; retry sem backoff, sem limite ou sobre operação não idempotente; falta de circuit breaker quando o risco justifica; cache sem tenant na chave (isso é Crítico); invalidação de cache; arquivos e streams carregados inteiros; concorrência sem limite; jobs sem observabilidade ou recuperação; sync-over-async.

Sem profiling ou métricas, classifique como `indício — não medido` e diga o que derrubaria a hipótese. Não invente ganhos percentuais.

### Etapa 12 — Consolidação e plano de correção

Antes de gerar o HTML:

1. Consolide achados duplicados (inclusive entre etapas: um mesmo problema visto em SOLID e em Arquitetura vira um só).
2. Diferencie vulnerabilidade, bug, risco, dívida técnica e oportunidade.
3. **Releia no código** cada candidato ao Top 5 aplicando o item 1.4, mesmo que um subagente já o tenha "confirmado".
4. Valide que todo caminho citado existe (ex.: compare com `git ls-files`) e confira as linhas dos achados principais.
5. Reavalie severidade e confiança considerando o alcance real.
6. Escolha os cinco achados de maior risco.
7. Identifique dependências entre correções.
8. Liste as **decisões pendentes** de cada correção (desenho, produto, trade-off) com a sua recomendação, para que o usuário decida em lote.
9. Escreva cinco prompts de correção independentes, específicos e seguros.
10. Escolha o primeiro prompt e explique por que deve vir antes.
11. Ordene as recomendações em P1, P2, P3… e escreva o texto das issues para o GitHub (4.5).

Cada prompt orienta uma tarefa posterior, não executa a correção agora, e deve: delimitar o escopo; citar arquivos e evidências; exigir leitura dos testes e convenções; pedir um plano antes de alterar arquivos; trazer as decisões pendentes com recomendação; preservar contratos e decisões válidas; exigir testes e verificação de regressão; proibir mudanças fora do escopo; não pressupor o que não foi verificado; proibir push, PR ou merge sem autorização.

------------------------------------------------------------------------

## 4. Formato obrigatório do relatório

Gere **um único `AUDITORIA.html` na raiz do projeto** (ou `AUDITORIA.md`, se o usuário pediu Markdown; ver 4.4). Não gere os dois, nem JSON, CSV ou arquivos auxiliares no repositório.

O HTML deve ser autocontido:

- CSS em `<style>`; JavaScript só se necessário e inline.
- Diagramas em SVG inline ou HTML/CSS.
- Sem CDN, fontes remotas, bibliotecas externas ou dependência de internet; funciona aberto direto no navegador.
- Responsivo para desktop e celular (tabelas e diagramas rolam dentro do próprio contêiner; a página não rola de lado).
- Tema claro e escuro.
- Navegação por índice e âncoras.
- Contraste acessível; severidade comunicada por cor, forma e texto, nunca só por cor.
- Tabelas com cabeçalhos claros.
- Data, repositório, branch e commit analisados.

### 4.1 Estrutura do HTML

1. **Cabeçalho executivo:** projeto; data e commit/branch; stack; escopo e limitações; testes executados e resultado; legenda de severidade e status; resultado geral em duas ou três frases; **nota metodológica** dizendo como cada categoria foi mapeada para a stack detectada (ex.: "isolamento de tenant = filtro global do EF Core + middleware de tenant; não há RLS") e quais não se aplicam.
2. **Placar:** total de achados por severidade, um gráfico de rosca por severidade e um de barras por categoria, em SVG inline desenhado em escala, com os números escritos ao lado (o gráfico nunca é a única forma de ler o valor).
3. **Top 5 riscos:** ordenados por risco, com resumo, impacto, evidência, alcance real e link para a seção detalhada.
4. **Pontos fortes e pontos fracos:** o que está protegido, com evidência (vem das seções "Verificado e correto"), e os riscos centrais em poucas linhas.
5. **Resumo por domínio:** testes, segurança (exposição e segredos; dependências; IA e LLM; arquivos; containers; XSS), SOLID, DDD, perfis e autorização, middlewares, concorrência, idempotência, arquitetura, manutenibilidade, performance.
6. **Detalhamento das etapas:** uma seção por etapa, com os achados no formato 4.2, a seção "Verificado e correto" (controles conferidos, com arquivo e linha), as categorias que não se aplicam e o que não foi verificado.
7. **Matriz de perfis e permissões**, e o inventário de handlers por id da Etapa 5 (quantos estruturais, manuais e sem verificação).
8. **Mapa de arquitetura:** diagrama dos módulos, dependências, bancos, filas e integrações, com os acoplamentos verificados destacados. Não invente relações.
9. **Oportunidades de aprofundamento:** cards com arquivos, problema, proposta, benefícios, força e o que a refutação achou; diagramas "Antes" e "Depois" para cada candidato relevante (o "Depois" é a proposta conceitual, não algo que já existe).
10. **Arquivos prioritários:** arquivos extensos ou complexos, avaliação de cada um e o candidato nº 1 a refatorar.
11. **Recomendações priorizadas:** P1, P2, P3… cada uma ligada aos IDs que resolve, com a dependência entre elas quando houver.
12. **Decisões pendentes:** por correção, as escolhas que o usuário precisa fazer, com a recomendação.
13. **Cinco prompts de correção:** blocos completos, copiáveis e ordenados, com o primeiro destacado e justificado.
14. **Issues para o GitHub:** ver 4.5.
15. **Limitações e itens não verificados:** o que não pôde ser inspecionado, por quê e o que seria necessário; teses de auditorias anteriores que não foram reabertas.

### 4.2 Padrão para cada achado

Achados de severidade Média ou maior usam o formato completo:

- **ID:** estável, por exemplo `SEC-001`, `AUTH-002`, `DDD-001`.
- **Título.**
- **Severidade:** crítica, alta, média ou baixa.
- **Status:** confirmado, indício ou não verificado.
- **Localização:** caminho e linhas (todos os pontos relevantes, com o trecho mínimo).
- **Evidência:** o comportamento observado.
- **Refutação tentada:** o que foi procurado para derrubar e o que se achou, incluindo o alcance real.
- **Condições de exploração:** o que precisa ser verdade para o problema acontecer (feature flag ligada, configuração insegura, papel do atacante, provedor que envia o evento).
- **Cenário de impacto.**
- **Correção sugerida:** ação concreta, sem implementar.
- **Teste de validação:** como provar que a correção resolveu.

Achados de severidade Baixa podem ir numa tabela compacta, mas com as colunas: ID, achado, status, localização, impacto e correção, e teste de validação.

### 4.3 Qualidade visual

Paleta sugerida para severidade (ajuste o tom no tema escuro para manter o contraste): crítica `#B91C1C`, alta `#EA580C`, média `#D97706`, baixa `#2563EB`, informativa em cinza neutro, ponto forte `#059669`. A cor sempre vem acompanhada do texto da severidade e de uma forma (chip, ícone), nunca sozinha.

O relatório deve parecer uma revisão técnica profissional, não um dashboard genérico: hierarquia clara; resumo executivo conciso; severidades fáceis de distinguir; diagramas simples; código e caminhos em fonte monoespaçada; português claro; sem animações decorativas nem gráficos sem significado; nenhum achado importante escondido atrás de interação; todo conteúdo essencial visível sem JavaScript.

### 4.4 Versão em Markdown

Quando o usuário pedir Markdown, o `AUDITORIA.md` segue a mesma estrutura da 4.1, com estas adaptações:

- No topo, os cinco achados de maior risco em ordem, cada um com uma linha de impacto e o link para a seção.
- Uma tabela por etapa com as colunas: ID, achado, risco (alto, médio ou baixo), status, arquivo e linha, correção sugerida. Os achados médios ou maiores ganham, abaixo da tabela, o bloco completo da 4.2.
- Severidade escrita por extenso, nunca só por emoji ou cor.
- Diagramas em blocos `mermaid` (renderizam no GitHub e em vários editores) e só para relações confirmadas; antes e depois das oportunidades de aprofundamento como dois blocos lado a lado no texto.
- Os cinco prompts de correção e as issues para o GitHub em blocos de código, para copiar.
- Placar por severidade e por categoria em tabela (gráfico opcional em `mermaid` do tipo `pie`).
- Termina dizendo qual dos cinco prompts rodar primeiro e por quê.

### 4.5 Issues para o GitHub

Ao final do relatório, uma seção com o texto completo de uma issue em Markdown para cada achado acionável (severidade Baixa ou maior), cada uma num bloco copiável, delimitado por `--- ISSUE n ---` e `--- FIM ISSUE n ---`. Cada issue contém:

- **Título:** `[<Domínio>] <descrição curta da falha>`, com o domínio por extenso (Segurança, Autorização, Testes, Concorrência, Idempotência, Arquitetura, Performance…).
- **Labels sugeridas:** o domínio e a severidade (ex.: `security`, `severidade:media`).
- **Problema:** o que acontece e por que é explorável ou prejudicial, com as condições de exploração.
- **Evidência:** `arquivo:linha` com o trecho mínimo de código (sem segredo).
- **Impacto.**
- **Sugestão de correção.**
- **Critérios de aceite:** checklist verificável (`- [ ]`), incluindo o teste que prova a correção.

Agrupe numa issue só os achados triviais do mesmo tema (ex.: vários defaults de segredo, vários endpoints sem teto de corpo), para não gerar uma enxurrada de issues. Não crie as issues no GitHub: só escreva o texto. Criar fica a cargo do usuário ou de um pedido explícito depois.

------------------------------------------------------------------------

## 5. Como finalizar

Depois de validar que o relatório foi gerado (estrutura bem formada, âncoras com alvo, nenhum recurso externo, `git status` mostrando só o relatório):

1. Informe o caminho absoluto do relatório.
2. Resuma os cinco maiores riscos.
3. Informe se os testes foram executados ou bloqueados e por quê.
4. Destaque o primeiro prompt de correção e a justificativa.
5. Informe as limitações e verificações pendentes.
6. Reafirme que nenhum arquivo do projeto foi alterado além do relatório.
7. Explique em duas ou três frases, sem jargão, o que o resultado significa para o produto (ex.: "um cliente bloqueado ainda consegue usar a integração"), para quem não é técnico entender a prioridade.
8. Diga o que ele pode pedir em seguida, com frases prontas para copiar:
   - `explorar AUTH-01` (ou o ID que quiser): aprofunda um achado e traz as decisões de correção;
   - `explorar todos`: faz isso para os cinco do Top 5 e para os candidatos de arquitetura;
   - `grelhar ARQ-1`: discute um candidato de arquitetura, uma pergunta por vez;
   - `implementar o prompt 1`: começa a correção, fora do modo de auditoria.

Não comece a corrigir. Aguarde um pedido explícito.

------------------------------------------------------------------------

## 6. Depois do relatório: explorar

Quando o usuário pedir para explorar (um item, alguns ou "todos"), continue em modo somente leitura até ele pedir a implementação.

### 6.1 Achados de correção

Para cada achado escolhido (em paralelo, se houver subagentes):

1. Tente derrubá-lo de novo, agora a fundo (item 1.4): leia o chamador real, a configuração do ambiente e os testes vizinhos.
2. Se sobreviver, mapeie o desenho da correção no padrão do repositório: contratos que já existem e podem ser reaproveitados, ponto de inserção, custo, efeito nos testes existentes e arquivos tocados.
3. Separe o que o padrão do repositório já decide (ADR, glossário, regra documentada), que você decide e lista com a fonte para veto, do que precisa do usuário (produto, negócio, trade-off sem precedente).
4. Leve ao usuário as decisões em lote (até quatro perguntas por vez), cada uma com a sua recomendação primeiro.
5. Registre as decisões num arquivo de trabalho fora do repositório, para a implementação partir dele.

Se a exploração rebaixar a severidade (ex.: o caminho não é usado em produção), diga isso ao usuário.

### 6.2 Candidatos de arquitetura (grilling)

Siga o passo de grilling de `improve-codebase-architecture` (não é preciso tê-la instalada; o método está abaixo):

1. **Refute antes de grelhar.** Verificadores independentes (item 1.4, deletion test e contagem refeita) tentam derrubar o candidato. Só leve ao usuário o que ficar de pé, dizendo o que foi lido e o que é inferência.
2. **Grelhe um candidato por vez.** Se a skill `grilling` estiver instalada, use-a. Se não estiver, conduza assim:
   - Monte a árvore de decisões do candidato: cada decisão abre as que dependem dela (ex.: dono do estado → como as partes conversam → como os testes se dividem → como as catracas acompanham → como entregar).
   - A cada rodada, escolha **uma** pergunta entre as que já podem ser respondidas sem adivinhar a resposta de outra, preferindo a que destrava mais decisões.
   - Escreva a pergunta com a sua recomendação, de modo que "sim" aceite a recomendação. Formato: `❓ **título**: contexto e opções` e, na linha seguinte, `➡️ recomendação`.
   - Fatos são seus: leia o código, os testes e as catracas antes de perguntar (em paralelo, se puder). Decisões são do usuário: espere a resposta antes da próxima pergunta.
   - Termine quando não sobrar decisão pendente. Resuma o desenho combinado e só aja depois que o usuário confirmar que o entendimento é o mesmo.
3. **Mantenha o modelo de domínio atual.** Com a skill `domain-modeling` instalada, use-a. Sem ela: termo novo vai para o glossário do projeto (`GLOSSARY.md`, `CONTEXT.md` ou equivalente) e termo nebuloso é afinado na hora, sempre com o aval do usuário, porque é escrita no repositório.
4. **Registre as recusas que importam.** Se o usuário rejeitar o candidato por um motivo que um explorador futuro precisaria saber, ofereça registrar um ADR para que a próxima revisão não sugira o mesmo de novo. Não ofereça para motivos passageiros ("agora não") nem óbvios.
5. **Compare desenhos quando a forma não estiver clara.** Com a `codebase-design` instalada, use o padrão "design it twice" dela. Sem ela: proponha dois ou três desenhos radicalmente diferentes para o módulo aprofundado (se puder, um subagente por desenho, cada um sem ver os outros), compare-os por profundidade, locality, leverage e testabilidade, e leve a comparação ao usuário com a sua recomendação.

Mudanças em glossário ou ADR feitas durante o grilling são escrita no repositório: só faça com o aval do usuário naquele momento.

------------------------------------------------------------------------

## 7. Princípio final

**Auditar não é refatorar.** O resultado deve permitir que uma pessoa técnica entenda o risco, encontre a evidência, tome uma decisão e execute uma correção controlada depois.

Se não houver evidência suficiente, escreva `não verificado`. Nunca complete lacunas com suposições. Um achado errado tira a confiança de todos os outros.
