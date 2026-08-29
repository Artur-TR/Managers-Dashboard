# ALIA · Gestão de Equipes (protótipo)

Protótipo funcional do módulo de **Gestão de Equipes** (técnicos de suporte Sistemas
Domínio) para futura incorporação no portal interno **ALIA**. Arquivo único
(`index.html`), HTML + Tailwind CSS (via CDN) + JavaScript puro, sem build step,
com **Firebase Firestore** como banco de dados em tempo real (técnicos e
planilhas) e Managers/Filas/Treinamentos ainda hardcoded no código.

## Como rodar

```bash
cd alia-gestao-equipes
python -m http.server 8123
```

Depois acesse `http://localhost:8123`. Precisa ser servido por HTTP (não
`file://`), pois o Firebase é carregado como ES module no `<head>`.

**Importante**: o projeto Firestore (`gestao-alia`) precisa ter regras de
segurança que permitam leitura/escrita nas coleções `technicians` e
`spreadsheets` (um projeto novo, por padrão, nega tudo). Para o protótipo,
regras de teste bastam:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /technicians/{doc} { allow read, write: if true; }
    match /spreadsheets/{doc} { allow read, write: if true; }
  }
}
```

Se a leitura/escrita falhar, um toast de erro aparece ("Falha ao sincronizar…")
— é o primeiro lugar a checar.

## Login (simulado)

Não há autenticação real. Ao carregar, o usuário escolhe qual gestor está
"logado" — isso simula a segregação por setor:

- **Luis Antonio Soares** — Coordenador, setor **Área Técnica** (4 técnicos)
- **Jaqueline Mussoi** — Coordenadora, setor **Fiscont** (4 técnicos)

Cada gestor só visualiza os técnicos do próprio setor. O catálogo de filas
contempla também o setor **Folha** (ex.: Fila 13, Fila 81), pronto para um
terceiro gestor ser cadastrado no futuro.

## Funcionalidades

- **Dashboard**: resumo da equipe, alertas automáticos (treinamentos pendentes,
  SLA abaixo de 90%, falta de acompanhamento recente), busca por nome/fila.
- **Perfil do técnico** (modal grande com abas, altura fixa e rolagem interna):
  Visão Geral, Matriz de Filas (toggles binários), Métricas e Metas, Central de
  Treinamentos (concluídos / agendados / pendentes + "Agendar novo"), Linha do
  Tempo (início e fim de cada ação, com duração calculada) e Tarefas (quadro
  Em Andamento / Concluídas).
- **CRUD completo de técnicos**: criar, editar (nome/cargo/tempo de casa) e
  excluir (com modal de confirmação customizado) — tudo gravado no Firestore.
- **Planilhas e Matrizes**: grid de cards (ícone no topo, título na base) com
  busca/filtro, criação e remoção. Duas são **matrizes vivas** (Matriz de
  Filas x Técnicos e Matriz de Treinamentos — toggles refletem direto no
  perfil/dashboard); as demais são planilhas genéricas com células editáveis.
- **Tempo real multiuso**: qualquer alteração (um técnico, uma célula de
  planilha) grava no Firestore e é propagada por `onSnapshot` para **todas as
  abas/usuários conectados**, sem precisar recarregar a página.
- **Resetar dados demo**: apaga tudo no Firestore e recria a carga inicial (2
  gestores hardcoded, 8 técnicos, 4 planilhas).

## Arquitetura

- **Firebase** é carregado via CDN como ES module num `<script type="module">`
  no `<head>` (só `firebase-app` + `firebase-firestore`, sem Analytics). Ele
  inicializa `app`/`db` e expõe as funções do Firestore em
  `window.__firestore`, disparando o evento `firebase-ready`.
- O restante da aplicação (fim do `<body>`) é um script clássico (não-module),
  por depender de escopo compartilhado entre dezenas de funções. Ele escuta
  `firebase-ready` (ou usa `window.__firestore` direto, se já disponível) antes
  de ligar os listeners em tempo real — ver `bootAfterFirebase()`.
- **Camada de dados — Técnicos** (coleção `technicians`): cada função
  (`addTechnician`, `updateTechnician`, `toggleFila`, `addTimelineEntry` etc.)
  grava no Firestore via `addDoc`/`updateDoc`/`deleteDoc` e devolve uma
  `Promise<boolean>` (erros já tratados e reportados via toast). Nenhuma delas
  atualiza a UI diretamente — quem faz isso é sempre o listener `onSnapshot`.
- **Camada de dados — Planilhas** (coleção `spreadsheets`): mesmo padrão. O
  Firestore não aceita *arrays aninhados*, então a grade 2D das planilhas
  genéricas (`dados: [[...]]`, conveniente no código) é achatada para
  `{ linhas, colunas, celulas: {"r-c": valor} }` só no momento de gravar
  (`gridToFirestoreFields`) e reconstruída como array 2D ao ler
  (`firestoreFieldsToGrid`) — o resto do app nunca vê o formato achatado.
- **Tempo real**: `initFirestoreListeners()` registra um `onSnapshot` por
  coleção. Cada callback atualiza o espelho local (`STATE.technicians` /
  `STATE.spreadsheets`) e chama `handleTechniciansUpdate()` /
  `handleSpreadsheetsUpdate()`, que re-renderizam só o que está visível
  (dashboard, aba do perfil aberto, planilha aberta) — nunca o modal inteiro,
  para não re-disparar a animação de entrada nem perder a posição de rolagem.
- **Seed automático**: se uma coleção estiver vazia no primeiro snapshot
  recebido, ela é populada com os dados de demonstração (`seedTechnicians`/
  `seedSpreadsheets`). Um guard (`seedGuard`) evita reseed duplicado quando o
  próprio "Resetar dados demo" esvazia a coleção momentaneamente.
- **Managers, Filas e Treinamentos continuam hardcoded** no início do
  `<script>` (`MANAGERS`, `QUEUES`, `TRAINING_NAMES`), como pedido — não há
  coleção do Firestore para eles.
- **Nota para produção/embedding**: o Tailwind é carregado via Play CDN, que
  injeta um reset global (Preflight). Ao embutir este módulo dentro do ALIA de
  verdade, gere o CSS em build-time com prefixo (ou `corePlugins.preflight:
  false`) para não sobrescrever estilos da página host.

## Filas pré-cadastradas

Catálogo completo no início do `<script>` (`QUEUES`), categorias **FUNCIONAL**
e **ÁREA TÉCNICA**, cada uma com código, categoria e descrição.
