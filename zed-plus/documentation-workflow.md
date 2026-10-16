# Documentação das alterações

## Pedido

Registrar as features e modificações solicitadas em arquivos `zed-plus/*.md` e adicionar a regra ao `.rules`.

## Fluxo

- Criar ou atualizar um documento diretamente em `zed-plus/` para cada feature ou alteração solicitada.
- Registrar comportamento, implementação, configuração e validações realizadas, distinguindo resultados confirmados de pendências.
- Criar commits somente quando solicitado.
- Após um commit solicitado concluir, registrar seu hash real no documento correspondente. Esse registro ocorre depois do commit e fica pendente para um próximo commit, salvo orientação diferente do usuário; um commit não pode conter seu próprio hash final.
- Não criar um commit adicional apenas para registrar o hash sem autorização.

## Implementação

Regra adicionada à seção **Zed Plus documentation** de `.rules`.

O aviso de revisão no início de `README.md` foi adicionado conforme a regra obrigatória preexistente do repositório; sua remoção é uma confirmação manual do autor.

## Commits

- `6b3500654d4beb2e29707713acd0d4db016eb9f2` — `docs: define Zed Plus change documentation workflow`.
- Hash registrado após a criação do commit, conforme o fluxo acima.
