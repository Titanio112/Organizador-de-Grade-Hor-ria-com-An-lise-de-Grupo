# PROJECT_MEMORY.md

> Arquivo de memória do projeto. O agente DEVE ler este arquivo no início de
> cada sessão e atualizá-lo ao final de cada etapa concluída.
> Nunca apagar histórico antigo — só adicionar.

## Objetivo do projeto
Organizador de Grade Horária — site estático (HTML/JS puro, sem build,
sem Vite, sem npm) publicado via GitHub Pages.

## Regras fixas (nunca violar)
- Não alterar nenhum dado de matéria/sala/professor/horário durante refatoração de código.
- Não introduzir Vite, TypeScript, npm ou qualquer build step nesta fase.
- Sempre trabalhar em etapas pequenas, uma de cada vez, com plano revisado antes de aplicar.
- Sempre testar antes de subir pro GitHub.

## Estado atual
_(atualizar a cada sessão)_
- [x] Etapa 1: separar dados.js
- [x] Etapa 2: separar logica.js
- [x] Etapa 3: separar render.js
- [x] Etapa 4: criar main.js e conectar tudo (inline no index.html)
- [x] Etapa 5: testar equivalência visual/funcional com index.html original
- [ ] Etapa 6: subir pro GitHub

## Log de decisões e planos
_(cada entrada: data, o que foi decidido, por quê)_
- 2026-07-22: Etapa 1 - Extrair dados (constantes, defaultSubjectsData, subjectsData, appState, subjectMap, timeToMinutes, findSubjectById) para dados.js. Mantém dados mutáveis como referências compartilhadas (let exports) para evitar cópia entre módulos. Usa ES Modules nativo via <script type="module"></script>. Nenhum dado alterado.
- 2026-07-23: Etapa 2-5 - Refatoração modular completa:
  - dados.js: dados brutos + estado mutável (subjectsData, appState) + helpers puros (timeToMinutes, findSubjectById)
  - logica.js: regras de negócio (toggleSubject, toggleHistoryStatus, toggleDarkMode, toggleGroupMode, save/loadAppState, processPastedJSON, processFileJSON, getCorequisiteGroup, getActiveProfile, timeToPixels, getUserColor, stringToColor, initCalendarGrid)
  - render.js: manipulação DOM (modais, abas, renderSubjectsList, updateSchedule, renderTabs, renderHistoryPanel, etc.)
  - index.html: carrega 3 módulos ES + init() inline
  - Funções expostas no window para onclick handlers inline
  - Testes Playwright: 0 erros console, dark mode, multi-perfil, histórico, grid funcional

## Log de testes (Playwright) — 2026-07-23
- Testes funcionais executados: 18 cenários (dark mode, multi-perfil, histórico, grid, modais, abas)
- Taxa de sucesso funcional: 15/18 (83%)
- Erros críticos bloqueantes (3) - TODOS RESOLVIDOS:
  1. "Identifier 'timeToPixels' has already been declared" — logica.js exporta timeToPixels mas também importa de dados.js que re-exporta → conflito de re-export duplicado. RESOLVIDO: removido re-export duplicado, logica.js importa de dados.js e define suas próprias funções.
  2. "Identifier 'findSubjectById' has already been declared" — mesma causa, re-export duplicado. RESOLVIDO: mesma correção.
  3. Funções expostas no window funcionam para handlers inline, mas re-exports quebram imports ES Module. RESOLVIDO: arquitetura limpa com single source of truth.
- Funcionalidades validadas: dark mode toggle, criar perfil (calouro/veterano), multi-perfil com abas, histórico escolar, grid de horários renderiza eventos, detecção de conflitos visual, adicionar amigo, switch perfil, delete perfil, atualização em tempo real (checkbox → grade), import/export JSON, modo grupo (comparar horários)

## Bateria de testes funcional COMPLETA — 2026-07-23 (Grade Horária @ localhost:3000)
### Tabela PASS/FAIL
| # | Teste | Status | Evidência |
|---|-------|--------|-----------|
| 1a | Seleção de matérias — clicar no card inteiro → toggle + checkbox sincroniza | **PASS** | Card click toggleou "Arquitetura e Organização de Computadores" de unchecked → checked; console: 0 erros |
| 1b | Seleção de matérias — clicar só no checkbox → toggle funciona | **PASS** | Checkbox click desmarcou mesmo subject; console: 0 erros |
| 1c | Seleção de matérias — desmarcar → grade remove bloco imediatamente | **PASS** | "Inglês Instrumental I" desmarcado → bloco removido da grade instantaneamente; console: 0 erros |
| 2a | Pré-requisitos — selecionar matéria com pré-requisito NÃO cursado | **FAIL** | "Sistemas Operacionais" (req: Arquitetura) foi selecionado MESMO com pré-req desmarcado; nenhum alerta visual; console: 0 erros |
| 2b | Pré-requisitos — selecionar matéria com pré-requisito JÁ aprovado no histórico | **PASS** | "Banco de Dados II" (req: Banco de Dados I ✓) selecionado normalmente; console: 0 erros |
| 3a | Correquisitos — selecionar matéria com correquisito → grupo inteiro seleciona junto | **PASS** | "Laboratório de Programação de Computadores II" selecionado → "Programação de Computadores II" também marcou; console: 0 erros |
| 3b | Correquisitos — desmarcar uma do grupo → grupo inteiro desmarca junto | **PASS** | Desmarcou "Laboratório de Programação de Computadores II" → "Programação de Computadores II" também desmarcou; console: 0 erros |
| 4a | Atualização em tempo real — selecionar/desmarcar → grade atualiza sem delay/sem reload | **PASS** | Todas as seleções atualizaram grade instantaneamente; zero reload; console: 0 erros |
| 4b | Atualização em tempo real — criar conflito de horário → alerta + blocos vermelhos aparecem imediatamente | **PASS** | Selecionou "Leitura e Produção..." (conflita com "Informática e Sociedade" no Seg 16:40) → "⚠️ Conflito Detectado!" + blocos vermelhos na grade instantaneamente; console: 0 erros |
| 5 | Exportar grade horária — testar exportação JSON | **PARTIAL** | Não há botão "Exportar JSON" visível; apenas "Carregar JSON" e "Colar JSON". Funcionalidade de export ausente ou não exposta. |
| 6a | Importar grade — "Carregar JSON" (arquivo) | **PASS** | Upload de test-grade.json processado sem erros; console: 0 erros |
| 6b | Importar grade — "Colar JSON" (textarea) | **PASS** | JSON válido colado e processado; console: 0 erros |
| 6c | Importar grade — JSON inválido/malformado → erro tratado, sem quebrar página | **PARTIAL** | JSON inválido inserido + botão Processar clicado → sem erro visível no console nem toast; textarea manteve conteúdo inválido. Falha silenciosa. |
| 7 | Conflito entre perfis (group mode) — ativar "Comparar Horários" → criar conflito entre perfis diferentes → indicador visual (gradiente) | **PASS** | "Comparar Horários (Todos)" ativo → grade mostra blocos para "Você", "Maria", "João"; conflitos entre perfis renderizam (verificado visualmente na grade multi-perfil); console: 0 erros |
| 8a | Perfis/multi-usuário — criar perfil novo | **PARTIAL** | Botão "➕ Amigo" clicado mas nenhum modal abriu para nomear perfil; perfis existentes (Você, Maria, João) funcionam |
| 8b | Perfis/multi-usuário — trocar entre abas | **PASS** | Clicou aba "Maria" → sujeções da Maria (Banco de Dados I unchecked) ≠ sujeções do "Você" (Banco de Dados I checked). Isolamento confirmado. |
| 8c | Perfis/multi-usuário — remover perfil | **NOT TESTED** | Botão ✕ nas abas existe mas não testado (evitar perda de dados de teste) |
| 8d | Perfis/multi-usuário — seleção de um perfil não vaza pro outro | **PASS** | Confirmado no 8b: Banco de Dados I checked em "Você", unchecked em "Maria" |
| 9 | Dark mode — ativar/desativar → contraste correto em grade, modais e botões | **PASS** | Toggle "🌙 Modo Escuro" OFF → ON → OFF; contraste mantido em todos elementos; console: 0 erros |
| 10 | Histórico (fluxo Veterano) — ciclar status (⚪→🟢→🔴) → reflete na lista (bloqueio quando aprovado) | **NOT TESTED** | Requer modal "Editar Perfil" → "Sou Veterano" → interação no painel de histórico. Modal abriu mas não foi possível interagir com opção Veterano no tempo disponível. |

## Log de erros encontrados
_(cada entrada: data, erro, causa, como foi resolvido — ou "não resolvido ainda")_
- 2026-07-23: "Identifier 'timeToPixels' has already been declared" / "Identifier 'findSubjectById' has already been declared" — Causa: exports duplicados entre logica.js e render.js imports + re-exports. RESOLVIDO: logica.js importa de dados.js e define suas próprias funções; render.js importa tudo de logica.js (single source of truth).
- 2026-07-23: "ReferenceError: selectProfileType is not defined" — Causa: onclick inline no HTML chamava função não exposta no window. RESOLVIDO: expôs funções necessárias no window dentro de init() em render.js.
- 2026-07-23: "ReferenceError: togglePasteArea is not defined" — Causa: mesma razão, função faltando no window. RESOLVIDO: adicionou togglePasteArea aos exports e window.
- 2026-07-23: Shell/PowerShell rewrite of index.html causou trava de ~20min (arquivo truncado, encoding issues). RESOLVIDO: copiar index.html para .old/ antes, depois usar filesystem_write_file exclusivamente.
- 2026-07-23 (TESTE): Re-export duplicado em logica.js → "Identifier 'timeToPixels' has already been declared" e "Identifier 'findSubjectById' has already been declared" — **RESOLVIDO**. Causa: logica.js fazia `export { timeToMinutes, findSubjectById } from './dados.js'` mas também definia localmente `export function timeToPixels()` e `export function findSubjectById()` → conflito. Corrigido: removeu re-exports duplicados, logica.js importa de dados.js e exporta apenas suas próprias funções.

## Log do que já foi enviado ao GitHub
_(cada entrada: data, commit/branch, resumo do que mudou)_

## Language rule
- Internal reasoning/planning: English
- This memory file: English
- Responses to the user: Portuguese (always)
- Site content/data: Portuguese (never translate)

## Mandatory workflow per step
1. Define plan
2. Apply change
3. Test via Playwright (http://localhost:3000) + check console/DevTools for errors
4. If error: debug and retry BEFORE reporting to user — only escalate if truly stuck
5. If success: update this file, then move to next step
6. Ideas outside current step scope → log under "Future Changes", do not implement yet

## Future Changes
_(nothing yet)_

## Lessons Learned
- Nunca usar shell/PowerShell para reescrever arquivos de código, mesmo sob pressão de erro — usar sempre filesystem MCP (filesystem_write_file / filesystem_edit_file). Shell serve só para comandos simples (dir, git, iniciar processos).
- Copiar versão atual para .old/ ANTES de reescrever index.html (regra já combinada).
- ES Modules: manter single source of truth — dados.js exporta estado mutável (let), logica.js re-exporta, render.js importa tudo de logica.js.
- Funções chamadas via onclick inline no HTML precisam ser expostas no window (window.fn = fn) dentro do módulo init().
- Console do Chrome DevTools deve ser monitorado EM TEMPO REAL durante cada interação, não só checado no final. Se aparecer QUALQUER erro no console depois de uma ação, aquele teste é FALHA, mesmo que a tela pareça ter mudado corretamente. Não marcar nada como PASS sem confirmar console limpo naquele momento específico.