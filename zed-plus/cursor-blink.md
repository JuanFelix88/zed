# Tempos do cursor blink

## Pedido

Incorporar nativamente o comportamento de `C:/www/patch-zed/` ao Zed deste repositório, sem patch binário e sem novas opções no `settings.json`.

## Comportamento

- Intervalo entre alternâncias de visibilidade do cursor: **130 ms**.
- Atraso para retomar o blink após atividade: **180 ms**.
- Aplicação no editor e no terminal integrado.
- Valores fixos no código, conforme confirmado pelo usuário.
- As opções existentes `cursor_blink` e `terminal.blinking` continuam controlando se o cursor pisca. A feature não força a ativação do blink.

O intervalo é por alternância: um ciclo completo visível/invisível corresponde a 260 ms.

## Implementação

- `crates/editor/src/editor.rs`: intervalo do editor de 500 para 130 ms.
- `crates/terminal_view/src/terminal_view.rs`: intervalo do terminal de 500 para 130 ms.
- `crates/editor/src/blink_manager.rs`: atraso compartilhado após atividade de 500 para 180 ms.

Sem alterações na API do `BlinkManager`, em dependências ou em configurações. Os scripts externos e o Zed Preview instalado não foram modificados; esta implementação substitui a necessidade do patch apenas no aplicativo compilado deste repositório.

## Validação

- `rustfmt --check --edition 2024 crates/editor/src/editor.rs crates/terminal_view/src/terminal_view.rs`: passou, incluindo o submódulo `blink_manager`.
- `git diff --check`: passou.
- Verificação estática confirmou que editor e terminal passam 130 ms ao `BlinkManager` e que a pausa compartilhada usa um timer de 180 ms.

- `cargo build --locked -j 2`: passou em **4min54s**, perfil `dev`, após solicitação do usuário.
- Executável atualizado: `target/debug/zed.exe`.
- Testes não foram compilados nem executados, conforme solicitado. Validação visual do blink ainda pendente.
- Aplicativo atualizado aberto a pedido do usuário com `--user-data-dir %TEMP%/zed-plus-data`; processo e janela da versão local confirmados. Isso não substitui a validação visual dos tempos do blink.

## Commits

- `2cf5ebe61952148f87760b1534dbc974126f0df4` — `feat(editor): use 130 ms cursor blink intervals and 180 ms activity delay`.
- Commit autorizado sem compilar ou executar os testes.
- Hash registrado após a criação do commit, conforme o fluxo de documentação.
