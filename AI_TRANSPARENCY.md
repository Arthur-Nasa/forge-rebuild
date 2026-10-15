# Declaração de Uso de IA e Transparência de Modificações (AI Transparency & Disclosure)

> **Aviso Oficial de Conformidade com o GNOME Extensions (EGO)**  
> Este fork foi modificado com assistência de Inteligência Artificial para fins de estudo, refatoração e uso pessoal/educacional voltado ao **GNOME Shell 50**.  
> Em conformidade estrita com as diretrizes de revisão publicadas no blog oficial do GNOME por Javad Rahmatzadeh:  
> [An Open Letter to AI for Generating GNOME Shell Extensions](https://blogs.gnome.org/jrahmatzadeh/2026/07/27/ego-ai-reference/)  
> **NENHUM arquivo gerado ou alterado por IA deve ser submetido ao EGO sem revisão humana completa, domínio técnico do código e compromisso de manutenibilidade contínua.**

---

## 1. Contexto do Projeto: "Forge Rebuild"

O projeto original [forge-ext/forge](https://github.com/forge-ext/forge) é um dos mais populares gerenciadores de janelas em árvore (tiling) para o GNOME Shell, mas atualmente encontra-se sem mantenedor ativo oficial.

A iniciativa **Forge Rebuild** tem como objetivos:
1. Portar e estabilizar a extensão para o **GNOME Shell 50** com o mínimo de modificações invasivas no núcleo matemático de janelas.
2. Eliminar APIs obsoletas do **GJS** e do **Libadwaita**, adequando o código aos padrões modernos de qualidade do ecossistema GNOME.
3. Servir como base de estudo e mentoria estruturada de engenharia reversa para permitir o entendimento e posterior refatoração modular completa do gerenciador de janelas.

---

## 2. Inventário Explicativo das Alterações Realizadas por IA

Abaixo está o registro cirúrgico de cada alteração aplicada por IA nesta branch (`gnome-50`):

| Arquivo Modificado | Tipo de Alteração | Descrição e Racional Técnico |
| :--- | :--- | :--- |
| `metadata.json` | Compatibilidade & Identidade | Alterado o nome para `"Forge Rebuild"`, UUID para `"forge-rebuild@jmmaranan.com"`, e explicitada a compatibilidade com `"shell-version": ["50"]`. |
| `Makefile` | Build & Instalação | Atualizado `UUID = "forge-rebuild@jmmaranan.com"` para que a instalação local em `~/.local/share/gnome-shell/extensions/` ocorra em pasta isolada, evitando conflitos com versões upstream do Forge. |
| `lib/shared/settings.js` | Modernização GJS | Substituição de `imports.byteArray.toString(contents)` por `new TextDecoder().decode(contents)`. `imports.byteArray` é uma API legada do GJS que gera advertências no console e viola as boas práticas atuais do SpiderMonkey. |
| `lib/prefs/settings.js` | GNOME HIG & Libadwaita | Migração de `Adw.AboutWindow` (depreciado no Libadwaita moderno) para `Adw.AboutDialog`, apresentado via `abt.present(parent)`. Nome da aplicação atualizado para `"Forge Rebuild"`. |
| `lib/extension/indicator.js` | Limpeza de Código EGO | Remoção de checagem condicional desnecessária (`Utils.isGnomeGTE(45)`), já que o alvo é o GNOME 50. Nome do menu do Quick Settings atualizado para `"Forge Rebuild"`. |
| `lib/extension/window.js` | Consistência de UI | Atualizado título interno de janela de configurações para `"Forge Rebuild"`. |
| `lib/prefs/appearance.js` | Consistência de UI | Atualizados títulos e descrições do interruptor de Quick Settings para `"Forge Rebuild"`. |

---

## 3. Diretrizes Adotadas

Durante todas as intervenções de IA, os seguintes princípios foram rigorosamente respeitados:
- **Sem Blocos `try/catch` Silenciosos**: Não foram inseridos invólucros defensivos cegos em métodos de desalocação ou conexão de sinais.
- **Isolamento de Processos Preservado**: Nenhuma biblioteca gráfica de desktop (`St`, `Clutter`, `Meta`) foi misturada com bibliotecas de preferências (`Gtk`, `Adw`), respeitando a divisão de processos entre `gnome-shell` e `gnome-extensions-app`.
- **Sem Emojis Unicode na Interface**: A interface gráfica utiliza exclusivamente ícones simbólicos nativos (`-symbolic`).
- **Nenhum Código Incompleto ou Stub**: Todas as alterações são funcionais, testáveis e compiláveis localmente.

---

## 4. Responsabilidade e Manutenibilidade

As alterações neste repositório foram geradas via pair programming entre o desenvolvedor humano (**Arthur-Nasa**) e o assistente de IA (**Antigravity**). O desenvolvedor humano está ativamente revisando o código através de um plano estruturado de estudo da arquitetura da extensão antes de qualquer publicação ou distribuição externa.
