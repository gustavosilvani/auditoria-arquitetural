# Auditoria arquitetural completa

Skill para o [Claude Code](https://claude.com/claude-code) que audita uma aplicação de ponta a ponta, em modo somente leitura, e entrega um relatório HTML autocontido.

Foi pensada para sistemas construídos ou alterados com ajuda de IA, onde é comum o código parecer pronto e esconder regra só no front, race condition, webhook sem idempotência ou teste que não bate na rota de verdade.

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

## Instalação

Com a CLI do [skills.sh](https://skills.sh):

```bash
npx skills add gustavosilvani/auditoria-arquitetural
```

Para instalar no nível do usuário (todos os projetos), acrescente `-g`. Para atualizar depois, `npx skills update`.

Sem a CLI, copie a pasta da skill para o diretório de skills do Claude Code:

```bash
git clone https://github.com/gustavosilvani/auditoria-arquitetural.git
cp -r auditoria-arquitetural/skills/auditoria-arquitetural-completa ~/.claude/skills/
```

Para usar só num projeto, copie para `.claude/skills/` dentro dele.

## Uso

A skill tem `disable-model-invocation: true`: ela só roda quando você a chama.

```text
/auditoria-arquitetural-completa
```

Para restringir o escopo, diga a área junto: `/auditoria-arquitetural-completa só o módulo de pagamentos`. Em repositório grande, sem área definida, ela pergunta antes de varrer tudo.

## Skills opcionais

A etapa de arquitetura e a exploração depois do relatório aproveitam estas skills, se estiverem instaladas, e seguem sem elas:

- `codebase-design`: vocabulário de deep modules e o padrão "design it twice".
- `grilling`: entrevista estruturada sobre um candidato.
- `domain-modeling`: manutenção do glossário e dos ADRs durante o grilling.

## Créditos

A lente de arquitetura (deletion test, locality, leverage, seams e o grilling depois do relatório) vem da skill `improve-codebase-architecture`, de [mattpocock/skills](https://github.com/mattpocock/skills).

## Licença

[MIT](LICENSE)
