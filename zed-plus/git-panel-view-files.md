# View Files no painel Git

## Pedido e decisões

Adicionar apenas um botão **View Files**, antes de **View Diff**, seguindo os componentes, ícones, cores e tamanhos existentes. Abrir todos os arquivos alterados do repositório ativo em abas permanentes e executar o equivalente a `editor::ExpandAllDiffHunks` em cada editor.

Confirmado pelo usuário:

- Incluir arquivos staged, unstaged e novos, sem repetir um caminho por aparecer em mais de uma seção.
- Ignorar arquivos excluídos.
- Disponibilizar uma action para keybindings.
- Compilar debug incremental com **4 jobs**, preservando o cache, e abrir o `.exe` automaticamente após sucesso, no mesmo fluxo do launcher.
- Preservar os padrões de IU: sem legendas extras, novos cartões, textos grandes ou controles além do botão pedido.

## Implementação

Arquivo: `crates/git_ui/src/git_panel.rs`.

- Action `git_panel::ViewFiles`, registrada no workspace, permitindo atalhos também com foco no editor.
- Botão com `ButtonLike`, ícone `IconName::File`, `LabelSize::Small`, `IconSize::Small` e cores `Muted`, como o botão vizinho. Tooltip associado à action; botão desabilitado quando não há entradas.
- Os dois botões ficam no mesmo grupo à esquerda; o cabeçalho mantém quebra de linha para painéis estreitos. **View Diff**, suas estatísticas e as ações de staging mantêm o comportamento original.
- A lista vem de `Repository::cached_status()`, não da seleção ou das linhas atualmente expandidas da árvore. Assim, pastas recolhidas não excluem seus arquivos da operação.
- Filtra `status.is_deleted()`, converte os caminhos para o projeto e remove duplicatas. Caminhos fora dos worktrees abertos não são abertos.
- Usa `Workspace::open_path` sem preview, preservando o reaproveitamento de abas conforme o comportamento nativo do workspace.
- Para cada item `Editor`, aguarda `wait_for_diff_to_load()` e só depois chama `expand_all_diff_hunks(&ExpandAllDiffHunks, ...)` diretamente naquela entidade. Não depende de qual aba está ativa quando o diff termina de carregar.
- Cada abertura tem sua tarefa e tratamento de erros com `detach_and_notify_err`; uma falha não cancela as demais aberturas.
- Itens que usam visualizadores não textuais seguem o comportamento nativo; não recebem uma action de editor.
- Não altera o arquivo no disco, o staging, a base de comparação configurada nem o comportamento normal de expandir/recolher hunks.
- Sem novas dependências, configurações ou atalhos padrão.

## Keybinding opcional

Adicionar às configurações pessoais de keybindings, escolhendo uma combinação livre. Exemplo:

```json
[
  {
    "context": "Workspace",
    "bindings": {
      "ctrl-alt-shift-o": "git_panel::ViewFiles"
    }
  }
]
```

O exemplo não foi instalado automaticamente. A action opera sobre o repositório ativo do painel Git, não sobre todos os repositórios do workspace.

## Build debug e abertura automática

- Paralelismo explicitamente autorizado: `-j 4`. Posteriormente, o usuário autorizou **4 jobs por padrão nesta conversa**, sem repetir a pergunta; essa autorização não altera a regra para sessões futuras.
- Comando iniciado: `cargo build --locked -j 4`, perfil `dev` existente, com `incremental = true`.
- Reutilizado `target/debug`; sem `cargo clean`, alteração de perfil ou flags Rust. A mudança de `-j` controla concorrência, não exige um cache novo.
- Fixados `RUSTUP_TOOLCHAIN=1.98.1-x86_64-pc-windows-msvc` e o executável `RUSTC` correspondente no launcher temporário, evitando a mistura de toolchains diagnosticada anteriormente.
- Após código de saída `0`, o próprio launcher inicia `target/debug/zed.exe` com `--user-data-dir %TEMP%/zed-plus-data`, o repositório e este documento.
- A pasta de dados separada evita encaminhar a abertura para o release já em execução. Não fechar, substituir ou modificar essa instância nem o Preview instalado.
- Arquivos temporários de acompanhamento: `%TEMP%/zed-plus-build.mjs`, `.log`, `.json` e `.exit`. Fora do repositório.

## Validação

- `rustfmt --edition 2024 --check crates/git_ui/src/git_panel.rs` e `git diff --check` passaram.
- A primeira tentativa debug, com 4 jobs, falhou em aproximadamente 1min12s (saída `101`) por dois usos incorretos da API: importação privada de `ExpandAllDiffHunks` e um `?` aplicado ao `Option` retornado por `Entity::read_with`.
- Corrigidos para `editor::actions::ExpandAllDiffHunks` e leitura direta do `Option`. Formatação e `git diff --check` passaram novamente.
- O launcher não abriu o executável antigo após a falha. Primeira tentativa arquivada em `%TEMP%/zed-plus-build.view-files-attempt1.log`, `.json` e `.exit`.
- Segunda tentativa concluída com sucesso em **4min19s**, perfil `dev` (`unoptimized + debuginfo`), saída `0`, com 4 jobs e cache preservado.
- Executável confirmado: `C:\www\Juan\zed\target\debug\zed.exe`, **495.831.552 bytes**.
- O próprio launcher abriu o debug após sucesso, PID `20440`. Confirmados o caminho do processo, `Responding = true` e a janela `zed - git-panel-view-files.md`.
- A instância release já em execução permaneceu aberta separadamente; não foi substituída ou fechada.
- O log de abertura registrou `could not find zed-cli from any of: bin/zed.exe, ./cli.exe`, mensagem já observada no debug anterior. Não impediu a janela de abrir; não foi corrigida nesta feature.
- Nenhum teste automatizado adicionado, compilado ou executado nesta etapa. O usuário reforçou que deseja pular os testes e fazer a validação visual pessoalmente.
- O usuário aprovou a feature após receber o debug e solicitou o build release seguido de commit. Não foram relatados resultados individuais de todos os cenários abaixo; a confirmação da janela não substitui essa cobertura.

### Verificação manual sugerida

1. Com alterações em vários arquivos, pressionar **View Files** e verificar que todos abrem em abas permanentes com os hunks inline expandidos.
2. Usar um arquivo parcialmente staged e conferir que ele não aparece duas vezes por causa das seções.
3. Recolher pastas na árvore do Git e verificar que seus arquivos também são abertos.
4. Incluir um arquivo excluído e confirmar que não é aberto como um buffer vazio.
5. Repetir a operação com abas já abertas e conferir o comportamento nativo de reaproveitamento.
6. Associar `git_panel::ViewFiles` a um atalho pessoal e acioná-lo com foco no painel e no editor.
7. Conferir **View Diff**, estatísticas, staging e o cabeçalho em um painel estreito.
8. Recolher um hunk manualmente após a abertura, confirmando que a expansão não foi tornada permanente.

## Release solicitado após aprovação

- Comando autorizado: `cargo build --release --locked -j 6`. Essa escolha explícita vale para este build; o padrão de 4 jobs desta conversa não foi aplicado ao release.
- Reaproveitado `target/release`, mantendo o perfil otimizado existente, incluindo `lto = "thin"`. Sem `cargo clean`, alteração de perfil ou flags Rust; debug preservado.
- Precisão sobre “incremental”: o cache de crates release é reutilizado, mas o perfil release padrão **não ativa a compilação incremental interna do Rust**, ao contrário do debug. Confirmado no processo `rustc` do `zed`: `lto=thin`, sem `-C incremental`. A otimização/LTO e a linkedição do executável ainda precisam ser refeitas; não foi habilitada uma opção nova para tentar evitá-las.
- Launcher temporário `%TEMP%/zed-plus-release-build.mjs` atualizado para abrir `target/release/zed.exe` somente após sucesso do Cargo. Falha na compilação não abre o executável antigo.
- Abertura autorizada posteriormente pelo usuário, com `--user-data-dir %TEMP%/zed-plus-release-data`, repositório e este documento, sem encaminhar para uma eventual instância debug. Preview instalado não é modificado.
- Rust 1.98.1 fixado no launcher para Cargo e subprocessos. Logs e estado em `%TEMP%/zed-plus-release-build.log`, `.json` e `.exit`; registros anteriores preservados com sufixo `.previous`.
- Build concluído com sucesso em **54min21s**, perfil `release` (`optimized + debuginfo`), código de saída `0`, com 6 jobs. Somente `cli`, `git_ui` e `zed` foram marcados para recompilação; as demais crates aproveitaram o cache.
- Executável confirmado: `C:\www\Juan\zed\target\release\zed.exe`, **363.230.720 bytes**. O próprio launcher iniciou o release após sucesso, PID `11032`; confirmados o caminho do processo, `Responding = true` e a janela `zed - git-panel-view-files.md`.
- Avisos não fatais do build: cópias de `conpty.dll` e `OpenConsole.exe` não foram efetuadas porque os arquivos estavam em uso (`os error 32`). Não foram encerrados processos à força; o funcionamento do terminal não foi validado nesta etapa.
- A abertura registrou a mesma mensagem de CLI ausente já observada no debug: `could not find zed-cli from any of: bin/zed.exe, ./cli.exe`. Não impediu a janela de abrir.
- Nenhum teste automatizado adicionado, compilado ou executado. A confirmação da janela não equivale a validar todos os cenários funcionais da feature no release.
- O tempo não é uma comparação A/B com o primeiro build frio de 2h33: cache e trabalho recompilado diferem. Não é possível atribuir a diferença somente aos 6 jobs.
- Compilação realizada antes do commit, conforme pedido: `ZED_COMMIT_SHA` e a base registrada pelo launcher são `f103c74f069748e77272397d1948a18c9c58922b`, com a feature ainda não commitada. Nenhuma fonte foi alterada durante o build; o hash do novo commit não está embutido neste executável.
- O commit solicitado será criado após esta compilação bem-sucedida; o launcher não executa commits Git.

## Commits

Commit solicitado pelo usuário para depois do build release. O hash real será registrado após sua criação, conforme o fluxo de documentação.
