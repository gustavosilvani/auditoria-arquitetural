# Auditoria arquitetural completa

Skill para o [Claude Code](https://claude.com/claude-code) que audita uma aplicação de ponta a ponta, em modo somente leitura, e entrega um relatório HTML que você abre no navegador.

Foi pensada para sistemas construídos ou alterados com ajuda de IA, onde é comum o código parecer pronto e esconder regra só no front, race condition, webhook sem idempotência ou teste que não bate na rota de verdade.

## Começando do zero

Nunca usou uma skill? Uma skill é um conjunto de instruções que o Claude Code carrega quando você chama um comando com `/`. Esta aqui ensina o Claude a fazer uma auditoria com método fixo, em vez de uma revisão improvisada.

**1. Tenha o Claude Code instalado.** Siga as instruções em [claude.com/claude-code](https://claude.com/claude-code). Para instalar a skill pelo comando abaixo você também precisa de uma versão recente do [Node.js](https://nodejs.org), que traz o `npx`.

**2. Instale a skill.** No terminal, dentro da pasta do projeto que você quer auditar:

```bash
npx skills add gustavosilvani/auditoria-arquitetural
```

Isso coloca a skill em `.claude/skills/` do projeto. Para ter a skill em todos os projetos, acrescente `-g`.

**3. Abra uma sessão nova do Claude Code** na pasta do projeto (no terminal, `claude`). Uma sessão que já estava aberta antes da instalação não enxerga a skill.

**4. Chame a skill:**

```text
/auditoria-arquitetural-completa
```

Para auditar só uma parte, diga qual:

```text
/auditoria-arquitetural-completa só o módulo de pagamentos
```

**5. Espere o relatório.** A skill avisa o que vai fazer, pergunta o escopo se o projeto for grande e trabalha sozinha. No fim, aparece um `AUDITORIA.html` na raiz do projeto. Abra no navegador: o topo traz os cinco maiores riscos e o fim traz cinco prompts de correção prontos para copiar.

**6. Decida o próximo passo.** A skill termina sugerindo frases para você mandar, como `explorar AUTH-01` (aprofunda um achado e traz as decisões), `explorar todos` ou `implementar o prompt 1`.

### Quanto tempo e quanto custa

Depende do tamanho do projeto. Um projeto pequeno leva minutos. Um monorepo grande pode levar uma hora ou mais e consumir uma parte grande do limite de uso do seu plano, porque a skill lê muito código e confere cada achado. Restringir o escopo a uma área é o jeito mais simples de reduzir as duas coisas. Use o modelo mais capaz que o seu plano oferece: auditoria é trabalho de raciocínio.

## Perguntas frequentes

**Ela altera o meu código?** Não. A única coisa que ela escreve no projeto é o `AUDITORIA.html`. Notas de trabalho ficam numa pasta temporária fora do projeto. Se já existir um `AUDITORIA.html`, ela não sobrescreve sem perguntar.

**Ela roda os meus testes?** Só se conseguir provar, lendo a configuração, que os testes não tocam banco de produção, filas ou serviços externos reais. Se não conseguir provar, não roda e registra o motivo no relatório.

**Ela envia o meu código para algum lugar?** O código é lido pelo Claude Code como em qualquer outra tarefa. A skill não publica nada, não faz commit nem push e não chama APIs de escrita. A única consulta externa prevista é a auditoria de dependências (`npm audit` e parecidos), que manda nomes e versões de pacotes para a base pública de vulnerabilidades, e não o código.

**E se ela errar um achado?** Cada achado traz arquivo e linha e passa por uma tentativa de derrubá-lo antes de entrar no relatório. Mesmo assim, confira os principais antes de agir. O que não deu para provar aparece como "não verificado".

**Posso rodar de novo depois de corrigir?** Pode. Apague ou renomeie o `AUDITORIA.html` antigo antes, ou ela vai perguntar o que fazer com ele.

**A skill não aparece quando digito `/`.** Confira se a pasta `auditoria-arquitetural-completa` com o `SKILL.md` está em `.claude/skills/` do projeto ou em `~/.claude/skills/`, e abra uma sessão nova.

## O que ela faz

Percorre doze etapas, na ordem:

| Etapa | Assunto |
|---|---|
| 0 | Reconhecimento: stack, módulos, histórico Git, áreas quentes, ADRs |
| 1 | Testes: o que existe, o que bate na rota real, o que roda no CI, banco de teste isolado |
| 2 | Segurança: regra só no front, chaves no bundle, RLS e policies, segredos, dependências vulneráveis e cadeia de suprimento, funcionalidades com IA/LLM, upload de arquivos, containers |
| 3 | SOLID |
| 4 | DDD e integridade do domínio |
| 5 | Perfis, autenticação, autorização e isolamento multi-tenant (com matriz de acesso) |
| 6 | Middlewares e pipeline de requisição, incluindo o proxy na frente da aplicação |
| 7 | Concorrência, transações e race conditions |
| 8 | Idempotência, webhooks e processamento assíncrono |
| 9 | Arquitetura: oportunidades de aprofundamento (deep modules), com diagramas antes e depois |
| 10 | Separação de responsabilidades e o candidato nº 1 a refatorar |
| 11 | Performance e resiliência |
| 12 | Consolidação, Top 5 riscos, decisões pendentes e cinco prompts de correção |

Regras que guiam o trabalho:

- **Somente leitura.** A única escrita no projeto é o `AUDITORIA.html` na raiz. Testes só rodam com isolamento comprovado, e a saída de build vai para fora do repositório.
- **Refutação obrigatória.** Todo achado médio ou maior passa por uma tentativa de derrubá-lo antes de ser classificado: o controle existe em outro ponto? É deliberado (ADR, teste que afirma)? Quem chama o código de verdade? O caminho é usado em produção?
- **Severidade pelo alcance real.** Defeito num caminho que nenhum fluxo de produção usa não passa de baixa.
- **Sem chute.** O que não deu para provar aparece como `não verificado`, com o motivo.

Depois do relatório, se você pedir para explorar um achado ou um candidato de arquitetura, a skill aprofunda a refutação, mapeia o desenho da correção, leva as decisões em lote com recomendação e, para arquitetura, conduz o grilling de um candidato por vez. Nada é alterado até você pedir a implementação.

## Outras formas de instalar

Atualizar para a versão mais nova:

```bash
npx skills update
```

Sem o `npx`, copie a pasta à mão:

```bash
git clone https://github.com/gustavosilvani/auditoria-arquitetural.git
cp -r auditoria-arquitetural/skills/auditoria-arquitetural-completa ~/.claude/skills/
```

Para usar só num projeto, copie para `.claude/skills/` dentro dele.

## Outros agentes além do Claude Code

O `SKILL.md` segue o formato aberto de Agent Skills, então a skill funciona em outros agentes que leem esse formato. Com a CLI do skills.sh, escolha o agente na instalação:

```bash
npx skills add gustavosilvani/auditoria-arquitetural -a <agente>
```

Use `-a '*'` para instalar em todos os agentes que a CLI detectar. Fora do Claude Code:

- a forma de chamar muda (cada agente tem a sua: comando com `/`, menção pelo nome ou pedido em texto, como "faça a auditoria arquitetural completa");
- o campo `disable-model-invocation` é do Claude Code e os outros agentes o ignoram, então a skill pode ser acionada sozinha quando você pedir uma auditoria;
- onde a skill fala em subagentes, o agente usa o equivalente que tiver ou faz as etapas em sequência, o que leva mais tempo.

## Skills opcionais

A skill é autossuficiente: não precisa da `improve-codebase-architecture` nem de nenhuma outra. O método das duas está escrito dentro dela. Se estas estiverem instaladas, ela as aproveita; se não, segue o plano B descrito no próprio `SKILL.md`:

| Skill | Para quê | Sem ela |
|---|---|---|
| `codebase-design` | vocabulário de deep modules e o padrão "design it twice" | usa as definições e o método de comparação de desenhos escritos na skill |
| `grilling` | entrevista sobre um candidato, uma pergunta por vez | conduz o grilling pelo roteiro da seção 6.2 |
| `domain-modeling` | glossário e ADRs durante o grilling | atualiza o glossário do projeto direto, com o seu aval |

Para instalar as três, todas de [mattpocock/skills](https://github.com/mattpocock/skills):

```bash
npx skills add mattpocock/skills -s codebase-design -s grilling -s domain-modeling
```

## Créditos

A lente de arquitetura (deletion test, locality, leverage, seams e o grilling depois do relatório) vem da skill `improve-codebase-architecture`, de [mattpocock/skills](https://github.com/mattpocock/skills).

## Licença

[MIT](LICENSE)
