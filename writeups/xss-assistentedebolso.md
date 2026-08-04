---
layout: writeup
title: "Stored XSS em Assistente de Bolso"
---

# Stored XSS em "Assistente de Bolso" — Exfiltração de Dados Financeiros via localStorage

**Data:** Agosto 2026
**Severidade:** Alta (CVSS 3.1: 6.1 — Medium)
**Tipo:** Stored Cross-Site Scripting (XSS)
**Status:** Corrigido (projeto próprio, ambiente de desenvolvimento/teste)

---

## Resumo Executivo

Durante testes no meu próprio projeto **Assistente de Bolso** (app de controle financeiro pessoal), identifiquei uma vulnerabilidade de **Stored XSS** no campo de descrição de gastos. O campo não realiza sanitização de input, permitindo a injeção de código JavaScript que é executado toda vez que a página carrega o gasto salvo.

Além da falha de sanitização, o app apresenta um problema de design mais amplo: dados financeiros sensíveis (saldo da conta, histórico de gastos, categorias) são armazenados em texto claro no **localStorage** do navegador, sob a chave `assistente-financeiro-state` — um mecanismo acessível por qualquer script executado no contexto da página.

## CVSS 3.1 Score

**Vetor:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:N/A:N`
**Score base:** 6.1 (Medium)

| Métrica | Valor | Justificativa |
|---|---|---|
| Attack Vector (AV) | Network (N) | Exploração não depende de acesso físico ou de rede local |
| Attack Complexity (AC) | Low (L) | Não exige condições especiais além de inserir o payload |
| Privileges Required (PR) | None (N) | App single-user, sem barreira de autenticação para injetar o payload |
| User Interaction (UI) | Required (R) | A vítima precisa recarregar a página / acessar o histórico |
| Scope (S) | Unchanged (U) | Impacto restrito ao mesmo domínio/aplicação comprometida |
| Confidentiality (C) | High (H) | Todo o conteúdo do localStorage (saldo, histórico) é exposto |
| Integrity (I) | None (N) | O payload apenas lê dados, não os altera |
| Availability (A) | None (N) | Não há impacto na disponibilidade do serviço |

## Disclosure Timeline

Como o "Assistente de Bolso" é um projeto pessoal em ambiente de desenvolvimento, não houve processo de disclosure para terceiros — a timeline abaixo documenta o ciclo de identificação e correção, seguindo a estrutura que projetos reais usam para reportar vulnerabilidades:

- **Descoberto em:** Agosto 2026 — durante testes manuais no campo de descrição de gastos
- **PoC desenvolvida em:** Agosto 2026 — payload evoluído em 4 etapas para demonstrar impacto real
- **Publicado em:** Agosto 2026
- **Corrigido em:** Agosto 2026 — sanitização de input implementada no campo de descrição

## Passos para Reproduzir

1. Acessar a tela de registro de novo gasto no app
2. No campo de descrição, inserir um payload JavaScript ao invés de texto normal (ex: `gastei 100 no <payload>`)
3. Salvar o gasto normalmente
4. Recarregar a página (ou navegar para a tela que lista os gastos / histórico do chat)
5. O payload é executado automaticamente, sem qualquer interação adicional do usuário

## Prova de Conceito (PoC)

O payload evoluiu em etapas progressivas, cada uma demonstrando um nível maior de impacto — abordagem intencional para construir o caso de forma metodológica, do "prova que executa" até o "prova que rouba dado real":

**1. Confirmação de execução (prova que o Stored XSS funciona):**
```html
<img src=x onerror=alert('XSS-Costa')>
```

**2. Exfiltração parcial visível (prova que dá pra ler o localStorage):**
```html
<img src=x onerror="alert('Dados que vazariam: ' + localStorage.getItem('assistente-financeiro-state').substring(0,150))">
```

![PoC - Alert exibindo dados do localStorage vazando](xss-poc2.png)

**3. Extração direcionada do dado crítico (saldo da conta):**
```html
<img src=x onerror="alert('Saldo exposto: ' + JSON.parse(localStorage.getItem('assistente-financeiro-state')).initialBalance)">
```

![PoC - Saldo exposto](xss-poc.png)

**4. Simulação de exfiltração real (como um atacante faria de verdade):**
```html
<img src=x onerror="console.log('%c[SIMULAÇÃO DE EXFILTRAÇÃO]', 'color:red;font-weight:bold', 'Um atacante enviaria isto para o servidor dele:', localStorage.getItem('assistente-financeiro-state'))">
```

Em um cenário de ataque real, o `alert()` / `console.log()` seria substituído por um `fetch()` enviando o conteúdo do `localStorage` para um servidor controlado pelo atacante — sem qualquer indício visível para a vítima.

## Causa Raiz

- O campo de descrição do gasto é salvo diretamente no localStorage sem nenhuma sanitização (não remove/escapa tags HTML)
- Ao renderizar a lista de gastos, o valor é injetado no DOM sem escape (provavelmente via `innerHTML` ou equivalente), permitindo que o navegador interprete e execute o HTML/JS malicioso
- **Problema adicional de design:** dados financeiros sensíveis (saldo, histórico) ficam armazenados em `localStorage`, que é acessível por qualquer script rodando no mesmo domínio — incluindo extensões de navegador maliciosas, mesmo sem depender do XSS

## Impacto

Um atacante capaz de injetar esse payload (por exemplo, se o app fosse multiusuário e o campo fosse compartilhado/importado de outra fonte) conseguiria:

- Ler o saldo da conta e histórico completo de gastos
- Exfiltrar esses dados para um servidor externo (substituindo o `alert()`/`console.log()` por um `fetch()` para um endpoint controlado pelo atacante)
- Potencialmente sequestrar a sessão do usuário, dependendo de como a autenticação é gerenciada

## Remediação

1. **Sanitizar todo input do usuário** antes de salvar — usar bibliotecas como `DOMPurify` para remover tags/scripts maliciosos
2. **Escapar output ao renderizar** — nunca usar `innerHTML` com dados não confiáveis; preferir `textContent` ou frameworks que escapam por padrão (React, por exemplo, escapa automaticamente)
3. **Não armazenar dados financeiros sensíveis em localStorage** — considerar armazenamento server-side com sessão autenticada, ou ao menos criptografia no client caso o armazenamento local seja necessário
4. Implementar **Content Security Policy (CSP)** para limitar execução de scripts inline

## Lições Aprendidas

Esse achado reforça um princípio central em segurança de aplicações: **nunca confiar em input do usuário**, mesmo em campos aparentemente inofensivos como "descrição de gasto". A vulnerabilidade de design (dados sensíveis em localStorage) mostra também que XSS raramente é o problema isolado — geralmente expõe uma cadeia de decisões arquiteturais que amplificam o impacto.
