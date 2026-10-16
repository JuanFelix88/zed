# Paralelismo do build release

## Pedido e limites

Documentar como acelerar o build pelo uso de CPU e pelo paralelismo, sem executar builds ou aplicar ajustes de compilação nesta etapa. Por solicitação posterior, exigir confirmação do paralelismo antes de cada compilação.

Após esclarecimento do usuário, ficam fora do escopo: trocar para `release-fast`, alterar LTO, otimização, símbolos, `codegen-units`, flags Rust ou toolchain. Preservar o perfil release atual e seus artefatos reutilizáveis. Não executar `cargo clean`.

## Situação observada

- Máquina: aproximadamente **16 GB de RAM** e **12 processadores lógicos**.
- Build concluído: `cargo build --release --locked -j 2`, em **2h33min12s**, após corrigir a seleção da toolchain.
- A compilação do executável principal permaneceu por bastante tempo com apenas um processo `rustc`. Suas consultas de CPU mostraram atividade durante essa espera.
- O campo `CPU` de `Get-Process` é tempo acumulado de CPU, não porcentagem instantânea nem progresso da compilação.
- Não foram coletados timings detalhados nem uma série de uso global de CPU, memória comprometida e paginação. Portanto, ainda não é possível atribuir toda a demora ao limite `-j 2`.

O histórico do build está em [explorer-type-ahead.md](explorer-type-ahead.md).

## O que ajustar: `-j`

`-j 2` limita a concorrência administrada pelo Cargo. Não significa usar todos os processadores disponíveis. O jobserver também participa do orçamento de paralelismo de partes do backend do compilador; isso não torna todas as fases paralelizáveis.

Duas threads ocupadas equivalem a aproximadamente **17% de 12 processadores lógicos**. Esse cálculo ajuda a interpretar uma CPU aparentemente ociosa, mas não é uma medição do build anterior.

Aumentar `-j` pode permitir mais crates em paralelo e mais concorrência interna onde suportada. Um único processo `rustc` pode usar várias threads: contar processos não basta para medir aproveitamento da CPU. Algumas fases do executável principal, LTO e linker ainda podem ter limites próprios.

| Limite | Uso proposto | Cuidado |
| --- | --- | --- |
| `-j 2` | Referência já utilizada; alternativa se faltar memória | Pode deixar capacidade de CPU sem uso |
| `-j 4` | Primeiro candidato para o próximo build autorizado | Observar RAM e paginação; ainda não validado |
| `-j 6` | Segundo experimento, apenas se `-j 4` deixar CPU livre e houver memória | Vários crates grandes podem exceder a memória disponível |
| `-j 8`, `-j 12` ou automático | Não adotar diretamente nesta máquina | Mais concorrência não garante menos tempo; pode causar paginação ou falha por memória |

**Recomendação inicial: avaliar `-j 4`, sem mudar mais nada.** Não foi aplicado como novo padrão.

## Preservação do cache

Mudar apenas `-j` normalmente não invalida os artefatos Rust já compilados: o parâmetro controla agendamento, não o nível de otimização ou as flags de geração de código. Scripts nativos podem ter comportamento próprio relacionado à concorrência.

Manter:

- O mesmo `target/`, perfil `release`, código/lockfile e toolchain do build necessário.
- As opções existentes em `Cargo.toml` e `.cargo/config.toml`.
- A seleção consistente do Rust 1.98.1 em todos os subprocessos, evitando a mistura de compiladores encontrada anteriormente.

Reutilizar crates já prontas não é o mesmo que a opção de compilação incremental interna do Rust. Esta proposta não altera essa opção. Uma mudança em código pode continuar exigindo recompilar dependentes e refazer a otimização do executável principal.

Não substituir `RUSTFLAGS`: isso pode invalidar o cache e remover flags obrigatórias do Windows. Não mudar a prioridade do processo para alta/tempo real, nem restringir afinidade da CPU como tentativa de acelerar o build.

## Confirmação obrigatória antes de compilar

Regra registrada em `.rules`, na seção **Build guidelines**:

- Antes de iniciar qualquer compilação — debug, release, checks, testes ou comandos/scripts que compilem — perguntar via `ask_questions` quantos jobs Cargo (`-j`) o usuário quer usar e aguardar a resposta.
- Informar o número de processadores lógicos e a RAM disponível ao sugerir opções. Explicar que jobs são concorrência, não um limite rígido de CPUs ou afinidade.
- Não reutilizar silenciosamente o `-j 2` anterior nem adotar `-j 4` só porque aparece como candidato neste documento.
- Se o pedido atual já informar explicitamente a quantidade para aquela compilação, a escolha já está confirmada. Uma autorização antiga não vale automaticamente para outra execução.
- Passar o `-j` aprovado explicitamente ao Cargo ou ao wrapper responsável pela compilação.

Exemplo de pergunta: “Sua máquina tem 12 processadores lógicos e X GB de RAM disponíveis. Quantos jobs quer usar: 2 (conservador), 4 (candidato), 6 (maior consumo) ou outro valor?”

Os comandos abaixo são exemplos para futura avaliação, não autorizações para executá-los.

## Comandos para futura avaliação — não executados

Usar o shell com MSVC e CMake já configurados. `cargo +1.98.1` pressupõe o shim do rustup no PATH e fixa a versão também para os subprocessos; se a versão do projeto mudar, acompanhar `rust-toolchain.toml`.

Primeiro candidato:

```powershell
cargo +1.98.1 build --release --locked -j 4 --timings
```

Somente após analisar o primeiro resultado e autorizar nova compilação:

```powershell
cargo +1.98.1 build --release --locked -j 6 --timings
```

`--timings` acrescenta observabilidade, não uma otimização. O relatório HTML fica em `target/cargo-timings/`. Nenhum desses comandos foi executado nesta etapa.

## Como avaliar sem confundir cache com paralelismo

1. Aproveitar próximos builds realmente necessários, autorizados pelo usuário. Não limpar caches para fazer a comparação.
2. Registrar limite de jobs, revisão, tipo de alteração, tempo total e estado do cache. Um build sem alterações mede principalmente a reutilização do cache, não o ganho de CPU.
3. Comparar trabalhos equivalentes; não tratar um release frio de 2h33 como comparação direta com um rebuild pequeno e aquecido.
4. Acompanhar CPU global e do compilador, memória disponível/comprometida e paginação. Manter a carga de outros aplicativos comparável. O objetivo é reduzir tempo sem tornar a máquina impraticável, não alcançar 100% de CPU a qualquer custo.
5. Se `-j 4` causar pressão de memória, voltar a `-j 2`. Só experimentar `-j 6` se houver folga; não aumentar durante um build em andamento, pois exigiria reiniciá-lo.
6. Se as dependências acelerarem, mas o executável principal continuar longo e pouco paralelo, registrar esse limite. Mais jobs não garantem resolver uma fase serial; mudanças de perfil continuam fora do escopo.

Builds release longos ficam para a noite/madrugada, mediante solicitação; durante o dia, manter o fluxo debug existente. Nenhum agendamento automático foi criado. Não incluir testes nem abertura automática do executável sem autorização.

## Validação e estado

- Conferidos os perfis existentes, flags Windows, documentação do build anterior e dados básicos da máquina.
- Criada esta documentação e adicionada a exigência de confirmação em `.rules`, conforme solicitado.
- Nenhum build, teste ou benchmark foi executado; nenhum perfil, flag ou cache de compilação foi alterado.
- Não há redução de tempo medida nem promessa de ganho percentual. `-j 4` e `-j 6` são candidatos pendentes de avaliação.

## Referências locais

- `Cargo.toml`: perfis `dev`, `release` e `release-fast`; mantidos sem alterações.
- `.cargo/config.toml`: flags Windows preservadas.
- `docs/src/development/windows.md`: ambiente MSVC/CMake e cuidado ao sobrescrever `RUSTFLAGS`.
- `zed-plus/explorer-type-ahead.md`: resultado e diagnóstico do release anterior.

## Commits

Commit e push solicitados pelo usuário. O hash real será registrado após a criação do commit.
