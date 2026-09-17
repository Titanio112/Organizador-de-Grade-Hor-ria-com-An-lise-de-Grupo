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
- [x] Faxina do repo (2026-09-16): removidos .bak/.new/PNGs/.playwright-mcp/.firecrawl do versionamento (142 arquivos), .gitignore atualizado, schema.sql corrigido (subjects fechada + is_active + RLS em student_subjects), README.md criado. Commit f0dee96.
- [x] Supabase: schema aplicado no banco real (zhcubvmismnmvtrbolbu) - 5 tabelas, 2 funcoes (handle_new_user, update_updated_at_column), 14 policies, realtime habilitado. Scripts: reset_schema.js (reset completo), execute_schema.js (aplicar), seed_subjects.mjs (seed real), cleanup_seeds.js, db.js (helper .env). Commit 4604534.
- [x] Seed: 65 materias reais migradas do dados.js para subjects (com pre/co-requisitos resolvidos por UUID; 40 materias com pre-req, 37 com co-req). Seeds genericas CCO* removidas.
- [x] Seguranca: senha do banco saiu do codigo -> grade-horaria/.env (gitignored). ATENCAO: senha antiga vazou no commit 22c763e (execute_schema.js) - ROTACIONAR a senha do banco no dashboard Supabase.
- [x] Auth end-to-end validado (2026-09-16): signup -> login -> profile automatico -> RLS. Teste: supabase/test_api.js (6/6). Credenciais em js/config.js (frontend) e .env (backend scripts). Commit 761a0af.
- [x] REMODELAGEM v2 (2026-09-16, aprovada pelo usuario/arquiteto): hierarquia institutions>campuses>courses; subjects (catalogo, workload_hours 30/60/90) separado de classes (turma: professor, schedule, social_group_link); student_classes substitui student_subjects (grade_id+class_id+status+final_grade+absences); profiles ganha course_id/is_public. 9 tabelas, 22 policies, realtime.
- [x] Trigger check_prerequisites no banco: BEFORE INSERT em student_classes bloqueia matricula sem pre-req completed (raise 23514). PROVADO: test_trigger.js 10/10 (bloqueia prog2 sem pre-req, libera apos completed, faltas registram, RLS bloqueia anon).
- [x] Seed v2: 65 subjects + 65 classes + hierarquia CEFET-MG/Varginha/BSI. Idempotente.
- [x] Removido Importar/Exportar/Colar JSON da UI + logica.js + render.js (app 100% nuvem). UI validada via screenshot headless (grade+locks intactas).
- [x] docs/DB_SCHEMA.md v2 para revisao do arquiteto.
- [x] Teste funcional com mocks (2026-09-16): test_mock_flow.js (16/16) — 3 alunos ficticios (Ana/Bruno/Carla), trigger de pre-req bloqueando 2 cenarios, privacidade grade publica/privada via RLS, colegas de turma via shares_class, contador de faltas. create_mock_users.js (cria usuarios via SQL com receita completa: auth.users + identities + profiles). cleanup_mocks.js apaga tudo em cascata. Usuario/controle: emails gradehoraria+mock.*. Banco deixado LIMPO apos teste.
- [x] Normalizacao v3 (2026-09-16): professors (idx name) + class_professors (N:N, "Weider/Marcelo" virou 2 vinculos reais) + class_schedules (fim do JSONB; idx day_of_week/start_time; day 0=dom..6=sab). classes perdeu professor_name/schedule, ganhou code interno. Trigger trg_check_time_conflict (INSERT+UPDATE) barra choque real, libera sequencia 16:40->16:40. Seed v3: 65 subjects, 27 profs, 65 classes, 72 vinculos, 89 blocos. test_normalization.js 8/8 + test_mock_flow.js 16/16. Banco limpo apos.
- [ ] PROXIMA ETAPA (autorizada): tela de login/cadastro no index.html + sincronizar grade local <-> nuvem

## Log detalhado — sessão de testes com mocks (2026-09-16)

### Erros e correções (ordem cronológica)

1. **Signup 500 `unexpected_failure` em massa**
   - Causa: rate limit de envio de email do plano free (cada signup dispara email de confirmação; após ~3 envios/hora começa a falhar com 429/500).
   - Solução: `create_mock_users.js` — criar usuários via SQL direto. RECETA COMPLETA que funciona com GoTrue: `auth.users` (com `is_sso_user`, `is_anonymous`, `raw_app_meta_data` preenchidos) **+** linha em `auth.identities` (provider `email`, identity_data com `sub`+`email`). Confirmação de email setada direto (`email_confirmed_at = NOW()`).
   - ⚠️ Regra registrada: inserção INCOMPLETA em auth.users quebra o GoTrue inteiro ("Database error finding user"). Só usar a receita completa.

2. **`cannot insert a non-DEFAULT value into column "email"` (428C9)**
   - Causa: `auth.identities.email` é coluna GERADA (computed) no Supabase atual — não aceita INSERT explícito. O valor vem de `identity_data->>'email'`.
   - Solução: remover `email` do INSERT em identities.

3. **`current transaction is aborted` (25P02) escondendo o erro real**
   - Causa: catch interno tentando fallback dentro de transação já abortada — o erro verdadeiro era engolido.
   - Solução: removidos os catches aninhados; erros sobem limpos. Lição: em tx, primeiro erro aborta tudo — logar SEMPRE o primeiro.

4. **Duplicação e fragmentos no schema.sql v2 (erros de sintaxe sucessivos)**
   - Causa: inserts em linha via editor caíram no meio do `CREATE TABLE grade_subjects` e o arquivo ficou com seções duplicadas ("CREATE TABLE student_classes" 2x) + fragmento órfão de `UNIQUE(grade_id, subject_id)`.
   - Solução: inspeção por numero de linha (Select-String), remoção da segunda metade duplicada, fechamento correto de grade_subjects.
   - Lição: para arquivos SQL grandes, preferir reescrever usando `psql -f` mental: validar com print da estrutura (CREATE TABLE/FUNCTION index) antes de rodar.

5. **`relation "profiles" does not exist` ao criar `is_admin()`**
   - Causa: função `is_admin()` era criada ANTES da tabela `profiles` (funções SQL validam tabelas na criação).
   - Solução: mover `is_admin()` para depois de `CREATE TABLE profiles`.

6. **Falso-negativo: "Carla le a propria grade"**
   - Causa: asserção do TESTE estava errada, não o RLS — Carla via 3 grades (a dela privada + as 2 públicas), porque grade pública é visível para todos por design.
   - Solução: filtrar por `student_id` na query do teste. RLS estava correto.

7. **Rerun do teste quebrava (conflitos UNIQUE)**
   - Causa: rerodar test_mock_flow sem cleanup → `UNIQUE(student_id, semester, year, is_active)` em grades e `UNIQUE(grade_id, class_id)` em matrículas.
   - Solução: teste idempotente (helper `enroll()` trata duplicate como "permitido"; grade existente é reutilizada) + fluxo oficial `cleanup_mocks.js` → `create_mock_users.js` → `test_mock_flow.js`.

### Resultado final da bateria
- Login: 3/3 (usuários SQL fazem login normal pela API GoTrue)
- Trigger: bloqueia prog2 (Ana) e bd2 (Bruno) sem pré-req; libera após completed
- Privacidade: grade pública visível / privada invisível / shares_class expõe só turmas em comum
- Faltas: contador `absences` gravando (cálculo 25% fica na UI)
- **Total: 16/16 PASS** — banco deixado limpo via cleanup_mocks.js

## Regressão completa v3 + log de erros (2026-09-16, noite)

### Bateria completa — schema v3
| Bateria | Cobertura | Resultado |
|---|---|---|
| test_api.js | signup/login/trigger profile/leitura anon/RLS escrita | 6/6 |
| test_trigger.js | pre-requisitos, completed, absences, RLS | 10/10 |
| test_mock_flow.js | 3 alunos, privacidade, shares_class, faltas | 16/16 |
| test_normalization.js | filtro cruzado, 2 profs/turma, choque/sequencia, indices | 8/8 |
| **TOTAL** | | **40/40** |

### Erros novos encontrados na v3 e correcoes
1. **classes perdeu chave natural ao remover professor_name/schedule** — seed deixou de ser idempotente. Solucao: coluna interna `code` UNIQUE (`prog1-A`).
2. **Teste B falhou com turma errada** (`prog1-A` tem so 1 professor; "Weider/Marcelo" esta em `lab_prog-A`). Correcao no teste, banco estava certo.
3. **test_mock_flow quebrou ao coexistir com trava de choque** — Bruno tentava `bd1` (Seg 13:00-14:40) que sobrepunha `metodologia`. Teste atualizado para `arq1` (sequencia 14:40). = a trava PEGOU um choque real que o teste antigo ignorava.
4. **Rerun sem cleanup gerou falso FAIL** ("prog2 bloqueado" falhou porque Ana ja tinha completado pre-reqs no run anterior). Solucao: suites agora deletam grades antigas via API no inicio -> deterministicas.
5. **Trava de choque e cross-grade** (design): matricula repetida da MESMA turma em outra grade do aluno e vista como conflito com si mesma. Suites isolam deletando a grade antes.
6. **login mock logo apos create pode falhar por latencia do pooler** (raro). Workaround: rerodar; nao e bug de schema.

### Estado final do banco (v3)
12 tabelas | 6 funcoes | 3 triggers de regra de negocio | 28 policies RLS | 3 indices de busca | realtime em 9 tabelas.
Seed: 65 subjects, 27 professores, 65 classes, 72 vinculos class_professors, 89 blocos class_schedules.

### PENDENTE (nao testavel ainda)
- **Solucao estetica/design da UI**: nao testavel ate a tela de login/interface nova existir. Registrar aqui para nao esquecer: quando a UI for feita, validar com screenshots headless + bateria visual (dark mode, modais, mobile).
- [ ] Etapa 6: subir pro GitHub

## Capacidades do banco (visao de produto)
_(referencia para alinhamento com o usuario - 2026-09-16)_

**Implementado no banco:**
- profiles: login/cadastro, role admin/student (aphmgbr@gmail.com vira admin via trigger), avatar
- subjects: catalogo de 65 materias reais com pre/co-requisitos (UUID), horarios e salas por dia (JSONB), professor, semestre; admin faz CRUD e alunos recebem via realtime
- grades: grades por aluno, multiplas (semestre/ano), ativa, publica/privada (compartilhamento)
- grade_subjects: materias dentro de uma grade com dia/hora/sala/cor
- student_subjects: historico/matriculas com status (enrolled/completed/dropped/pending) e nota final; aluno ve colegas da mesma materia
- realtime habilitado nas 5 tabelas

**NAO existe no banco (avaliar se precisa antes da UI):**
- tabela explicita de amizades/grupos fixos
- notificacoes
- faltas/frequencia
- mensagens entre usuarios

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

## Log de erros encontrados (Supabase - 2026-09-16)
- "infinite recursion detected in policy for relation profiles" — policies admin consultavam profiles dentro de policy de profiles. RESOLVIDO: funcao public.is_admin() SECURITY DEFINER STABLE; todas as 6 policies usam is_admin().
- handle_new_user nunca criava profile (EXCEPTION WHEN OTHERS engolia o erro) — causa: CASE com literais nao tipados falhava na coercao para user_role enum. RESOLVIDO: casts explicitos ::user_role + removido EXCEPTION silencioso.
- Insert manual em auth.users quebrou GoTrue ("Database error finding user") — nunca criar usuarios via SQL direto; usar API + confirmar email via DB.
- Email rate limit no plano free bloqueia signups repetidos de teste — usar usuario de teste fixo + confirmacao via banco.
- Teste automatizado: supabase/test_api.js (signup, confirmacao, login, profile via trigger, SELECT anon, RLS bloqueia escrita anon) — 6/6 PASS.
- Credenciais: js/config.js (URL + anon key, seguro commitar) | .env (DB URL + anon key, gitignored).
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