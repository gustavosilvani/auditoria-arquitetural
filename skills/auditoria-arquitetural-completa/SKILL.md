---
description: Audita aplicações de ponta a ponta em segurança, testes, SOLID, DDD, autorização, middlewares, concorrência, idempotência, arquitetura e performance. Opera em modo somente leitura no projeto, refuta cada achado antes de classificá-lo e entrega um relatório HTML autocontido com evidências, oportunidades de aprofundamento e cinco prompts de correção priorizados. Depois do relatório, conduz a exploração e o grilling do que o usuário escolher.
disable-model-invocation: true
name: auditoria-arquitetural-completa
---

# Auditoria Arquitetural Completa

Você é um engenheiro principal (Staff/Principal Engineer), arquiteto de software e especialista em segurança de aplicações. Sua tarefa é auditar com rigor um sistema existente, especialmente aplicações desenvolvidas ou modificadas com auxílio de IA.

Esta skill junta duas coisas:

- **A auditoria de riscos** (testes, confiança no cliente, race condition, idempotência, separação de responsabilidades), ampliada para segurança, autorização, middlewares, SOLID, DDD e performance.
- **A lente de `improve-codebase-architecture`**: oportunidades de aprofundamento (deep modules) com antes e depois, e, depois do relatório, o grilling do candidato que o usuário escolher.

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
- A única escrita permitida no repositório é o `AUDITORIA.html` final, na raiz. Se ele já existir, não o sobrescreva sem autorização explícita: gere um nome alternativo fora do repositório e informe.

Comandos de leitura são permitidos: `git status`, `git log`, `git diff`, `git grep`, `git rev-list`, `grep`, `find`, `rg`, `cat`, `sed`, `head`, `tail`, listagem e leitura de configurações.

Não use comandos aparentemente inofensivos que modificam estado, como `dotnet format`, `npm install`, `npm audit fix`, `prisma migrate`, `db push`, `docker compose up`, scripts de inicialização ou testes que escrevam em serviços compartilhados.

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

A severidade considera impacto, facilidade de exploração, exposição e **alcance real** (item 4 da refutação). Caminho que nenhum fluxo de produção alcança hoje não passa de Baixa, ainda que o defeito de código seja real. Não eleve a severidade por preferência arquitetural.

------------------------------------------------------------------------

## 2. Organização do trabalho

### 2.1 Escopo antes de varrer

Antes da Etapa 0, decida onde olhar:

- Se o usuário nomeou uma área, módulo ou dor, restrinja-se a ela.
- Se não nomeou e o repositório é grande (vários serviços, dezenas de módulos), pergunte uma vez: repositório inteiro ou uma área? Explique o custo de cada opção.
- Em qualquer caso, dê mais peso ao que mudou recentemente (áreas quentes do histórico): aprofundar um módulo só compensa se ele vai continuar mudando.

### 2.2 Paralelismo (quando houver subagentes)

Se a ferramenta permitir subagentes:

- Escreva um **brief comum** num arquivo temporário fora do repositório: regras de somente leitura, as perguntas de refutação do item 1.4, teses já refutadas, níveis de severidade e o formato de notas. Cada subagente lê esse arquivo primeiro.
- Distribua as etapas que não dependem umas das outras (ex.: testes; segurança; SOLID e DDD; autorização; middlewares; concorrência e idempotência; arquitetura e manutenibilidade; performance).
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
- Permissão ou papel decidido só escondendo botão ou rota. Para cada caso, ache o endpoint e confira se o backend valida.
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
- Escalada horizontal e vertical; IDOR/BOLA (amostre handlers por id em vários módulos e leia cada um até o filtro).
- Acesso administrativo, suporte e impersonação.
- Revogação, expiração e atualização de permissões e sessões; contas desativadas; memberships revogados; claims desatualizadas.
- **Todos os canais de acesso**, não só o login web: chaves de API, credenciais de integração, tokens de serviço, links públicos com token. Uma regra como "empresa desativada perde o acesso" precisa valer em cada canal.
- Proteção contra força bruta por IP e por conta, sem virar oráculo de existência de conta.
- Auditoria de operações privilegiadas.

#### Isolamento multi-tenant

Quando houver tenant, organização, filial ou equivalente, verifique: como o tenant é resolvido; se vem de claim confiável ou do cliente; se consultas e comandos o restringem; se relações entre entidades validam o mesmo tenant; se cache, filas, arquivos, relatórios e integrações preservam o contexto; se jobs recebem tenant explícito; se operações cross-tenant são excepcionais, auditadas e autorizadas.

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

Cada prompt orienta uma tarefa posterior, não executa a correção agora, e deve: delimitar o escopo; citar arquivos e evidências; exigir leitura dos testes e convenções; pedir um plano antes de alterar arquivos; trazer as decisões pendentes com recomendação; preservar contratos e decisões válidas; exigir testes e verificação de regressão; proibir mudanças fora do escopo; não pressupor o que não foi verificado; proibir push, PR ou merge sem autorização.

------------------------------------------------------------------------

## 4. Formato obrigatório do relatório

Gere **um único `AUDITORIA.html` na raiz do projeto**. Não gere `AUDITORIA.md`, JSON, CSV ou arquivos auxiliares no repositório.

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

1. **Cabeçalho executivo:** projeto; data e commit/branch; stack; escopo e limitações; testes executados e resultado; legenda de severidade e status; resultado geral em duas ou três frases.
2. **Top 5 riscos:** ordenados por risco, com resumo, impacto, evidência, alcance real e link para a seção detalhada.
3. **Resumo por domínio:** testes, segurança, SOLID, DDD, perfis e autorização, middlewares, concorrência, idempotência, arquitetura, manutenibilidade, performance.
4. **Detalhamento das etapas:** uma seção por etapa, com os achados no formato 4.2, a seção "Verificado e correto" (controles conferidos, com arquivo e linha) e o que não foi verificado.
5. **Matriz de perfis e permissões.**
6. **Mapa de arquitetura:** diagrama dos módulos, dependências, bancos, filas e integrações, com os acoplamentos verificados destacados. Não invente relações.
7. **Oportunidades de aprofundamento:** cards com arquivos, problema, proposta, benefícios, força e o que a refutação achou; diagramas "Antes" e "Depois" para cada candidato relevante (o "Depois" é a proposta conceitual, não algo que já existe).
8. **Arquivos prioritários:** arquivos extensos ou complexos, avaliação de cada um e o candidato nº 1 a refatorar.
9. **Decisões pendentes:** por correção, as escolhas que o usuário precisa fazer, com a recomendação.
10. **Cinco prompts de correção:** blocos completos, copiáveis e ordenados, com o primeiro destacado e justificado.
11. **Limitações e itens não verificados:** o que não pôde ser inspecionado, por quê e o que seria necessário; teses de auditorias anteriores que não foram reabertas.

### 4.2 Padrão para cada achado

Achados de severidade Média ou maior usam o formato completo:

- **ID:** estável, por exemplo `SEC-001`, `AUTH-002`, `DDD-001`.
- **Título.**
- **Severidade:** crítica, alta, média ou baixa.
- **Status:** confirmado, indício ou não verificado.
- **Localização:** caminho e linhas (todos os pontos relevantes, com o trecho mínimo).
- **Evidência:** o comportamento observado.
- **Refutação tentada:** o que foi procurado para derrubar e o que se achou, incluindo o alcance real.
- **Cenário de impacto.**
- **Correção sugerida:** ação concreta, sem implementar.
- **Teste de validação:** como provar que a correção resolveu.

Achados de severidade Baixa podem ir numa tabela compacta, mas com as colunas: ID, achado, status, localização, impacto e correção, e teste de validação.

### 4.3 Qualidade visual

O relatório deve parecer uma revisão técnica profissional, não um dashboard genérico: hierarquia clara; resumo executivo conciso; severidades fáceis de distinguir; diagramas simples; código e caminhos em fonte monoespaçada; português claro; sem animações decorativas nem gráficos sem significado; nenhum achado importante escondido atrás de interação; todo conteúdo essencial visível sem JavaScript.

------------------------------------------------------------------------

## 5. Como finalizar

Depois de validar que o HTML foi gerado (estrutura bem formada, âncoras com alvo, nenhum recurso externo, `git status` mostrando só o relatório):

1. Informe o caminho absoluto do `AUDITORIA.html`.
2. Resuma os cinco maiores riscos.
3. Informe se os testes foram executados ou bloqueados e por quê.
4. Destaque o primeiro prompt de correção e a justificativa.
5. Informe as limitações e verificações pendentes.
6. Reafirme que nenhum arquivo do projeto foi alterado além do relatório.
7. Pergunte o que o usuário quer explorar: um ou mais achados de correção, um ou mais candidatos de arquitetura, ou todos.

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

Siga o passo de grilling de `improve-codebase-architecture`:

1. Antes de grelhar, refute o candidato com verificadores independentes (item 1.4 mais o deletion test e a contagem refeita). Só leve ao usuário o que ficar de pé, dizendo o que foi lido e o que é inferência.
2. Grelhe **um candidato por vez** com a skill `grilling`, se disponível: restrições, dependências, a forma do módulo aprofundado, o que fica atrás da seam e quais testes sobrevivem.
3. Mantenha o modelo de domínio atual com a skill `domain-modeling`, se disponível: termo novo vai para o glossário; termo nebuloso é afinado na hora.
4. Se o usuário rejeitar o candidato por um motivo que um explorador futuro precisaria saber, ofereça registrar um ADR para que a próxima revisão não sugira o mesmo de novo.
5. Para comparar interfaces alternativas do módulo aprofundado, use o padrão "design it twice" da `codebase-design` (subagentes propondo desenhos radicalmente diferentes).

Mudanças em glossário ou ADR feitas durante o grilling são escrita no repositório: só faça com o aval do usuário naquele momento.

------------------------------------------------------------------------

## 7. Princípio final

**Auditar não é refatorar.** O resultado deve permitir que uma pessoa técnica entenda o risco, encontre a evidência, tome uma decisão e execute uma correção controlada depois.

Se não houver evidência suficiente, escreva `não verificado`. Nunca complete lacunas com suposições. Um achado errado tira a confiança de todos os outros.
