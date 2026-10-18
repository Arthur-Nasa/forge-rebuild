# Caderno de Estudos e Roadmap de Melhorias — Forge Rebuild

Este documento é o registro vivo de todas as descobertas arquiteturais, pontos fracos identificados no código herdado do [forge-ext/forge](https://github.com/forge-ext/forge) e as propostas de engenharia desenhadas para a modernização completa da extensão no **GNOME Shell 50**.

---

## Índice das Descobertas
1. [Detecção de Orientação de Tela (Retrato vs Paisagem & Conversíveis 2-em-1)](#1-detecção-de-orientação-de-tela)
2. [Workspaces Dinâmicos e o Fim do `reloadTree()` Destrutivo](#2-workspaces-dinâmicos-e-o-fim-do-reloadtree-destrutivo)
3. [Segundo Monitor e Respeito a `workspaces-only-on-primary`](#3-segundo-monitor-e-respeito-a-workspaces-only-on-primary)
4. [Hot Swap Inteligente com Workspace Offloading](#4-hot-swap-inteligente-com-workspace-offloading)
5. [Identificação Única por EDID e Perfis por Monitor](#5-identificação-única-por-edid-e-perfis-por-monitor)
6. [Modernizações de APIs GJS, Libadwaita e EGO](#6-modernizações-de-apis-gjs-libadwaita-e-ego)
7. [Otimização de Energia no Focus on Hover (Fim do Polling de 16ms)](#7-otimização-de-energia-no-focus-on-hover)
8. [Plano de Modularização de `window.js` (Decomposição do Monólito de 2.900 Linhas)](#8-plano-de-modularização-de-windowjs)

---

## 1. Detecção de Orientação de Tela
*Inspirado pelo setup: Laptop conversível 2-em-1 (MSI Summit Flip Evo 13) + Monitor Ultrawide*

### Diagnóstico no Código Herdado:
- **Bug do Monitor Cego na Inicialização** ([lib/extension/tree.js:660](file:///home/arthur-nasa/Forge%20refactor/lib/extension/tree.js#L660)): Ao iterar sobre os monitores no método `addMonitor()`, o Forge chama `determineSplitLayout()` sem passar o índice `mi`. Como `determineSplitLayout()` consulta `global.display.get_current_monitor()`, ele atribui a todos os monitores a orientação daquele onde o ponteiro do mouse estava no momento em que a extensão iniciou.
- **Surdez a Rotações Físicas**: O sinal `monitors-changed` não é escutado. Quando o laptop é girado em modo retrato (via acelerômetro e `iio-sensor-proxy`), o Forge não inverte o layout do container pai de `HSPLIT` para `VSPLIT`.
- **Ausência de Aspect Ratio Threshold**: Telas ultrawide ($21:9$) e telas em retrato ($9:16$) recebem o mesmo tratamento genérico.

### Proposta para o Forge Rebuild:
- Passar o índice do monitor em `determineSplitLayout(monitorIndex)` usando `global.display.get_monitor_geometry(monitorIndex)`.
- Conectar `monitors-changed` e disparar reorientação dinâmica da raiz do monitor afetado.
- Adicionar lógica de auto-split por densidade: telas com aspect ratio $> 2.0$ priorizam até 3 colunas `HSPLIT` antes de permitir divisão vertical.

---

## 2. Workspaces Dinâmicos e o Fim do `reloadTree()` Destrutivo

### Diagnóstico no Código Herdado:
- **Indexação Estática por Strings** ([lib/extension/tree.js:676](file:///home/arthur-nasa/Forge%20refactor/lib/extension/tree.js#L676)): As workspaces são nomeadas como `"ws0"`, `"ws1"`, `"ws2"`. Quando o GNOME fecha uma workspace intermediária vazia (ex: fecha a `1`), o Mutter reindexa as restantes para `0, 1, 2`. O Forge apenas remove o nó `"ws1"` e **não reindexa** os nós `"ws2"` e `"ws3"`.
- **O Hack Destrutivo** ([lib/extension/window.js:1200](file:///home/arthur-nasa/Forge%20refactor/lib/extension/window.js#L1200)): Como as strings quebram, o autor inseriu o método `reloadTree()`, que faz `this.tree.childNodes.length = 0`. Ele deleta toda a árvore de janelas em memória, destrói os containers, divisões personalizadas e proporções calculadas, tentando reescanear tudo do zero.

### Proposta para o Forge Rebuild:
- Implementar **Reindexação Reativa**: ao receber `workspace-removed`, percorrer os nós existentes ajustando seus índices (`node.index -= 1`) sem tocar nos containers ou nós de janela.
- Deletar o `reloadTree()` destrutivo e adotar mutações pontuais e atômicas na árvore.

---

## 3. Segundo Monitor e Respeito a `workspaces-only-on-primary`

### Diagnóstico no Código Herdado:
- **Incompatibilidade com o Padrão do GNOME**: O GNOME vem configurado por padrão com `org.gnome.mutter.workspaces-only-on-primary = true`. O monitor secundário não alterna workspaces; suas janelas ficam visíveis o tempo todo.
- O Forge cria uma matriz cega combinada `mo{M}ws{W}` para todos os monitores. Ao alternar a workspace no monitor principal, ele ativa uma sub-árvore vazia no monitor secundário, perdendo o controle das janelas visíveis daquele monitor.
- **Risco de Crash por Workspace Nula** ([lib/extension/window.js:418](file:///home/arthur-nasa/Forge%20refactor/lib/extension/window.js#L418)): Invocar `metaWindow.get_workspace().index()` quando a janela está em todos os monitores ou sem workspace associada dispara `TypeError: get_workspace(...) is null`.

### Proposta para o Forge Rebuild:
- Consultar a chave `workspaces-only-on-primary` do Mutter.
- Se ativada, criar para monitores secundários um container raiz estático independente (`mo1`), imune à troca de workspaces do laptop.
- Proteger todas as consultas de workspace com verificação formal de `metaWindow.is_on_all_workspaces()` e fallbacks para `null`.

---

## 4. Hot Swap Inteligente com Workspace Offloading
*Evitando o "Efeito Avalanche" ao desconectar cabos HDMI/DisplayPort*

### Diagnóstico no Código Herdado:
- Ao desconectar o monitor secundário, o Mutter joga todas as janelas do monitor externo na workspace ativa do laptop.
- O Forge tenta comprimir todas as janelas na mesma tela pequena, espremendo janelas em colunas ilegíveis.
- Ao reconectar o monitor, o Forge não sabe quais janelas pertenciam àquela tela e o monitor externo liga vazio.

### Proposta para o Forge Rebuild:
- **Intercepção de Desconexão**: Ao detectar diminuição no número de monitores em `monitors-changed`, identificar as janelas pertencentes ao nó do monitor desconectado.
- **Migração para Próxima Workspace**: Em vez de permitir o amontoamento na tela atual, mover essas janelas em bloco para uma workspace dedicada (`currentWorkspace + 1`) usando `metaWindow.change_workspace()`.
- **Preservação da Sub-árvore**: Transferir o container inteiro para o novo nó da árvore, mantendo a organização de splits intacta.
- **Restauração Automática**: Marcar as janelas com tag de afinidade de monitor (`_preferredMonitor`) para que, ao plugar o cabo de volta, o Forge pergunte ou restaure automaticamente as janelas para a tela externa.

---

## 5. Identificação Única por EDID e Perfis por Monitor

### Diagnóstico no Código Herdado:
- Monitores são identificados apenas por índices numéricos voláteis (`mo0`, `mo1`). Se a ordem de conexão dos cabos mudar ou você trocar de monitor na mesma porta, o Forge aplica as mesmas proporções geométricas.

### Descoberta nos Protótipos Nativos do Mutter (`Meta.Monitor`):
O Mutter expõe dados de hardware via EDID:
- `monitor.get_vendor()` (ex: `"GSM"` para LG, `"AUS"` para ASUS)
- `monitor.get_product()` (ex: `"LG ULTRAWIDE"`, `"VG248QG"`)
- `monitor.get_serial()` (número de série gravado no hardware)
- `monitor.is_builtin()` (`true` para tela de laptop, `false` para monitores externos)

### Proposta para o Forge Rebuild:
- Criar a função `getMonitorFingerprint(monitor)` gerando chaves como:
  `GSM_LG_ULTRAWIDE_34_908AZX` ou `AUS_ASUS_VG248_1080P_441B`.
- Salvar perfis persistentes em `~/.config/forge/monitor_profiles.json` com:
  - Layout padrão por monitor (ex: 3 colunas para Ultrawide vs 2 colunas para 1080p).
  - Gaps configurados independentemente por tamanho físico de tela.
  - Posições memorizadas específicas para cada modelo de monitor.

---

## 6. Modernizações de APIs GJS, Libadwaita e EGO
*(Já iniciadas na Fase 1)*

- [x] **Substituição de `imports.byteArray`**: Migrado para `new TextDecoder().decode()` em [lib/shared/settings.js](file:///home/arthur-nasa/Forge%20refactor/lib/shared/settings.js).
- [x] **Migração para `Adw.AboutDialog`**: Atualizado o diálogo de Sobre depreciado em [lib/prefs/settings.js](file:///home/arthur-nasa/Forge%20refactor/lib/prefs/settings.js).
- [x] **Remoção de Checagens Legadas**: Eliminado `Utils.isGnomeGTE(45)` em [lib/extension/indicator.js](file:///home/arthur-nasa/Forge%20refactor/lib/extension/indicator.js).
- [x] **Documentação de Transparência de IA**: Criado [AI_TRANSPARENCY.md](file:///home/arthur-nasa/Forge%20refactor/AI_TRANSPARENCY.md) alinhado às diretrizes de revisão do EGO.
- [ ] **Próximo**: Substituir wrappers legados de [lib/prefs/widgets.js](file:///home/arthur-nasa/Forge%20refactor/lib/prefs/widgets.js) pelos componentes oficiais nativos do Libadwaita (`Adw.SwitchRow`, `Adw.ComboRow`).

---

## 7. Otimização de Energia no Focus on Hover (Fim do Polling de 16ms)
*Eliminando desperdício de CPU e dreno de bateria em laptops e conversíveis*

### Diagnóstico no Código Herdado:
- **Loop Contínuo de 60Hz** ([lib/extension/window.js:87-97](file:///home/arthur-nasa/Forge%20refactor/lib/extension/window.js#L87-L97) e [2353](file:///home/arthur-nasa/Forge%20refactor/lib/extension/window.js#L2353)): Quando `focus-on-hover` está ativado, o Forge registra um `GLib.timeout_add(16, ...)` ininterrupto.
- A cada **16 milissegundos**, mesmo se o usuário não encostar no mouse:
  1. Chama `global.get_pointer()` para ler as coordenadas da tela.
  2. Invoca `global.get_window_actors()`.
  3. Itera por todas as janelas do compositor calculando intersecções retangulares.
  4. Dispara `metaWindow.focus(...)` e `metaWindow.raise()`.
- **Impacto Real em Hardware Móvel**: Esse polling impede a CPU de entrar em estados de repouso profundo (C-states), esquentando a máquina e drenando a bateria consideravelmente (especialmente sensível em laptops finos como o MSI Summit Flip).

### Proposta para o Forge Rebuild:
1. **Integração com o Recurso Nativo do GNOME/Mutter**:
   - O Mutter já possui gerenciamento de foco por mouse nativo em C (`org.gnome.desktop.wm.preferences focus-mode 'sloppy'` ou `'mouse'`). A extensão deve preferencialmente sincronizar com essa preferência nativa sem necessidade de loops em JavaScript.
2. **Abordagem Reativa a Eventos de Movimento (se controlado pela extensão)**:
   - Substituir o timeout contínuo por escuta reativa no palco do Clutter: `global.stage.connect('captured-event', ...)` filtrando estritamente `Clutter.EventType.MOTION`.
   - Adicionar verificação de delta mínimo de movimento ($> 2$ pixels) e *throttling* com `GLib.idle_add` apenas enquanto o ponteiro estiver fisicamente em trânsito.

---

## 8. Plano de Modularização de `window.js` (Decomposição do Monólito de 2.900 Linhas)

### Diagnóstico no Código Herdado:
O arquivo [lib/extension/window.js](file:///home/arthur-nasa/Forge%20refactor/lib/extension/window.js) atingiu **2.900 linhas** e viola o Princípio da Responsabilidade Única (SRP), acumulando 6 domínios completamente distintos:
1. **Ciclo de Vida de Sinais**: Escuta mais de 10 sinais de `Meta.Display`, `Meta.WorkspaceManager`, e instâncias de `Meta.Window`.
2. **Rastreamento de Foco e Mouse**: Loop de polling de 16ms, detecção de janela sob o cursor e ativação de foco.
3. **Mutter Grabs (Live Resize e Drag & Drop)**: Intercepta `grab-op-begin`/`grab-op-end`, recalcula porcentagens em tempo real e desenha retângulo de drop.
4. **Decorações Visuais e Bordas**: Gerencia atores `St.Bin` anexados a `global.window_group` para bordas ativas e indicadores de split.
5. **Regras de Janelas Flutuantes**: Lógica de exceções, checagem de `wm_class` e persistência do `windows.json`.
6. **Orquestração de Comandos de Layout**: Ponte entre keybindings, indicator de quick settings e a árvore (`Tree`).

### Problemas Adicionais Identificados:
- **Monkey-Patching de Objetos do Compositor**: O Forge injeta propriedades diretamente nos atores do Mutter (ex: `windowActor.border = splitBorder`, `node.initRect`, `focusNodeWindow.previewHint`).
  - *Risco EGO*: As diretrizes de revisão recomendam não poluir protótipos e objetos do Mutter, pois retêm referências que impedem o Garbage Collector (GC) de desalocar janelas destruídas, gerando vazamento progressivo de memória RAM.

### Proposta de Arquitetura Modular (`lib/extension/wm/`):
Decompor o monólito em submódulos especialistas de baixo acoplamento:

```
lib/extension/
├── window.js (ou wm/manager.js)   <-- Coordenador enxuto (< 350 linhas)
└── wm/
    ├── signals.js                 <-- Registro, desconexão limpa e guardas de ciclo de vida
    ├── grabs.js                   <-- Resize interativo, Drag & Drop e cálculo de deltas
    ├── decorations.js             <-- Atores St.Bin, bordas ativas e hints visuais sem monkey-patching
    ├── focus.js                   <-- Gestão de foco reativo (sem polling)
    └── float.js                   <-- Regras de janelas flutuantes, exemptions e persistência
```

#### Responsabilidades dos Novos Módulos:
- **`wm/signals.js`**:
  Centraliza todos os `connect` e `disconnect`. Mantém uma lista estrita de `signalIds` com método `destroy()` atômico que garante que **nenhum sinal órfão** permaneça ativo se a extensão for desabilitada (requisito obrigatório de revisão do EGO).
- **`wm/grabs.js`**:
  Isola a máquina de estados de redimensionamento e movimentação do Mutter (`Meta.GrabOp`). Calcula as redistribuições de `percent` dos nós irmãos apenas quando a operação é confirmada.
- **`wm/decorations.js`**:
  Substitui o monkey-patching (`windowActor.border`) por uma coleção desacoplada (`Map<Meta.Window, DecorationActors>`). Ao detectar o fechamento da janela (`destroy`), remove os atores filhos de `global.window_group` imediatamente.
- **`wm/focus.js`**:
  Implementa o foco reativo por movimento de ponteiro, eliminando a dependência do timer de 16ms.
- **`wm/float.js`**:
  Separa a persistência de regras de classes (`windows.json`), diálogos modais e exceções de floating, expondo um método limpo `shouldFloat(metaWindow): boolean`.
- **`window.js` (Coordenador)**:
  Apenas recebe chamadas de alto nível, conecta os eventos aos métodos da árvore (`Tree`) e comanda a re-renderização (`renderTree`).
