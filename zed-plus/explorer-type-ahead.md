# Navegação por digitação no explorer

## Pedido

Adicionar ao painel de projeto a navegação por nome de pastas e arquivos, semelhante à descrita pelo usuário no VS Code, mantendo a interface existente.

## Comportamento

- Com o foco na árvore de arquivos, digitar seleciona o próximo item cujo nome exibido começa com o texto digitado, sem diferenciar maiúsculas de minúsculas.
- Letras digitadas com menos de **1 segundo** entre elas formam um prefixo. O refinamento mantém a seleção atual se ela ainda corresponder ao prefixo.
- Repetir uma mesma letra percorre os itens que começam com ela. Depois do último match, a busca volta ao começo da árvore.
- A busca considera pastas e arquivos em toda a árvore expandida, inclusive itens fora da tela. Não expande pastas recolhidas, não revela itens filtrados e não abre arquivos.
- A seleção usa a rolagem existente do painel e limpa marcações de seleção múltipla quando encontra um match.
- Sem match, a seleção não muda. Após 1 segundo sem digitar, a próxima letra começa uma nova busca.
- Perder o foco, usar as setas para cima/baixo ou cancelar reinicia o prefixo. Mudar a seleção por outra forma também impede continuar o prefixo anterior.
- A busca não intercepta a edição de nomes nem a digitação em menus. Atalhos existentes têm prioridade, inclusive atalhos Vim e personalizados. Teclas com Ctrl, Alt, Cmd/Windows ou Fn não iniciam buscas.
- A busca usa o mesmo nome exibido pelo painel, inclusive caminhos de pastas compactadas e nomes das raízes.

## Implementação

Em `crates/project_panel/src/project_panel.rs`:

- Estado temporário do prefixo, horário da última letra e seleção associada.
- Listener de teclado no painel, após a resolução normal dos atalhos pelo GPUI, limitado ao foco direto na árvore.
- Busca apenas em `state.visible_entries`, na ordem da árvore, com retorno ao início.
- Reutilização da rolagem existente e extração do cálculo do nome exibido para compartilhar entre renderização e busca.
- Relógio do executor GPUI para o intervalo entre letras, compatível com validações determinísticas futuras.

Sem novas dependências, configurações, controles visuais, alterações de tipografia ou keybindings.

## Validação

- `rustfmt --check --edition 2024 crates/project_panel/src/project_panel.rs`: passou.
- `git diff --check`: passou.
- Revisão estática confirmou que a busca usa os itens da árvore expandida e o mesmo nome da renderização; o listener atua somente com foco direto na árvore, após os atalhos existentes.
- `cargo build --locked -j 2`: passou em **4min39s**, perfil `dev`. Executável atualizado em `target/debug/zed.exe`.
- Aplicativo atualizado aberto com o repositório e este documento, usando `--user-data-dir %TEMP%/zed-plus-data`. Processo, janela e estado responsivo confirmados; o Preview instalado não foi alterado.
- Validação manual do comportamento ainda pendente.

Conforme solicitado, o executável teve prioridade: nenhum teste automatizado foi adicionado, compilado ou executado nesta etapa. A cobertura de testes fica para depois.

### Build release solicitado

Após o commit desta feature, o usuário solicitou compilar o perfil `release`, sem testes. Compilação otimizada ainda pendente; o resultado será registrado após a execução.

### Validação manual sugerida

1. Focar a árvore e pressionar uma letra com vários matches; repetir a letra para avançar e voltar ao começo.
2. Digitar rapidamente um prefixo como `painel`; esperar 1 segundo e buscar outro nome.
3. Expandir uma pasta com vários arquivos e confirmar a seleção e rolagem para um match fora da tela.
4. Confirmar que itens dentro de pastas recolhidas não são alcançados e que arquivos não são abertos só pela digitação.
5. Renomear/criar um item, usar menus e executar atalhos existentes para confirmar que a busca não interfere.
6. Repetir com maiúsculas, pastas compactadas, múltiplas raízes e prefixo sem match.

## Commits

Commit solicitado pelo usuário. O hash será registrado após a criação do commit.
