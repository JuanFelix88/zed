# Copy Reference

## Pedido

Adicionar **Copy Reference** ao menu de contexto do editor, entre **Copy and Trim** e **Paste**, com uma ação configurável no keymap.

## Comportamento

- Cursor ou seleção em uma linha: `src/arquivo.ts:L2`.
- Seleção de várias linhas: `src/arquivo.ts:L2-L3`.
- Caminho relativo à raiz do worktree do arquivo, sempre com `/`, inclusive no Windows.
- Linhas numeradas a partir de 1; a coluna não é incluída.
- Uma seleção terminando na coluna zero da linha seguinte não inclui essa linha.
- O clique direito fora da seleção move o cursor; dentro dela, preserva a seleção, seguindo o comportamento existente do editor.
- Com múltiplos cursores, usa a seleção mais recente.
- Em multibuffers, usa os números de linha do arquivo original. Uma seleção entre arquivos diferentes usa a posição do cursor da seleção mais recente.
- Sem arquivo associado, não altera a área de transferência; o item do menu fica desabilitado.

## Configuração

Ação: `editor::CopyReference`. Também disponível na paleta de comandos como **editor: copy reference**.

Não há atalho padrão nem nova configuração de formato ou visibilidade. Exemplo opcional no `keymap.json`:

```json
[
  {
    "context": "Editor",
    "bindings": {
      "ctrl-alt-r": "editor::CopyReference"
    }
  }
]
```

O menu usa o componente padrão do Zed, que exibe o atalho configurado para a ação.

## Implementação

- `crates/editor/src/actions.rs`: declaração da ação.
- `crates/editor/src/editor.rs`: cópia e formatação, reutilizando a resolução de seleção de **Copy File Location**, sem alterar seu formato original.
- `crates/editor/src/element.rs`: registro da ação do editor.
- `crates/editor/src/mouse_context_menu.rs`: posição no menu e disponibilidade.
- `crates/editor/src/editor_tests.rs`: testes de atalho, seleção, ausência de arquivo e multibuffer.

## Validação

- `rustfmt --check --edition 2024 crates/editor/src/editor.rs`: passou (inclui os submódulos modificados).
- `git diff --check`: passou.
- `cargo test --locked -p editor --lib -j 2 copy_reference -- --nocapture`: compilação interrompida a pedido do usuário para priorizar o aplicativo; testes não executados, cache preservado.
- `cargo test --locked -p editor --lib -j 2 copy_file_location -- --nocapture`: regressão pendente.
- `cargo build --locked -j 2`: passou em 10min05s, perfil `dev`.
- `target/debug/zed.exe --dump-all-actions`: confirmou `editor::CopyReference` e o nome **editor: copy reference**.
- Aplicativo atualizado iniciado de `target/debug/zed.exe`, com dados separados em `%TEMP%/zed-plus-data`; janela **zed - copy-reference.md** confirmada.
- O lançamento exibiu aviso de ausência de `zed-cli` (`bin/zed.exe` ou `./cli.exe`), mas a janela do editor abriu.
- Validação manual do clique no menu e do conteúdo copiado: pendente.

Testes específicos:

- `test_copy_reference_keybinding`
- `test_copy_reference_without_file`
- `test_copy_reference_context_menu`
- `test_copy_file_location_from_multibuffer` (inclui a nova referência)
- Regressão: demais testes `test_copy_file_location_*`.

## Commits

- `f42328f8f47cc927336f0e50e03f928717fe31af` — `feat(editor): add Copy Reference action to the context menu`.
- Commit autorizado pelo usuário com os testes automatizados e a validação manual ainda pendentes.
