# Changelog

Alterações neste projeto serão documentadas neste arquivo.

Este é um fork do upstream (v1.4.1), com otimizações específicas. Ao atualizar o módulo a partir do upstream, será necessário fazer cherry-pick seletivo dessas alterações ou aguardar integração no master.

## [v1.4.2-fmees.2] - 2026-06-15

### Correções desta Versão

#### 1. Assistente de IA — travamento do navegador ao citar documento pendente de indexação

- **Arquivo afetado**: `sei/web/modulos/ia/md_ia_chat_js.php`
- **Descrição**: a chamada AJAX de validação de protocolo (`md_ia_consulta_protocolo_assistente_ia_ajax`) era feita com `async: false`, bloqueando a thread JavaScript do navegador enquanto o servidor PHP consultava o Solr e, quando necessário, disparava `gerarIndexacaoProcesso()`. Em cenários com documentos externos recentes (ainda não indexados), essa espera podia ultrapassar vários segundos, tornando a aba completamente irresponsiva — chegando a acionar o diálogo "a página não está respondendo" do Chrome.
- **Alterações**:
  1. `async: false` alterado para `async: true` na chamada AJAX, eliminando o bloqueio da thread.
  2. Adicionada mensagem de status ao usuário durante a espera: *"Verificando o(s) documento(s) citado(s) na sua mensagem. Por favor, aguarde..."* — exibida no `#validacaoMensagem` imediatamente antes do disparo da requisição e limpa ao receber a resposta.
  3. Adicionado `beforeunload` guard (`ativarBloqueioNavegacao` / `desativarBloqueioNavegacao` / `_mdIaBeforeUnloadHandler`): enquanto a validação está em andamento, qualquer tentativa de navegação que recarregue a página principal (menu, links externos, botão voltar, fechar aba) aciona o diálogo nativo de confirmação do navegador — sem bloquear operações que carregam conteúdo em iframes internos do SEI.
  4. Callback `error:` corrigido: anteriormente fazia apenas `console.log`, deixando o widget permanentemente travado em caso de falha de rede ou erro de servidor. Agora exibe mensagem de erro ao usuário e restaura o estado da interface via `estadoDeInteracao()`.
- **Assistência**: GitHub Copilot (Claude Sonnet 4.6)

## [v1.4.2-fmees.1] - 2026-06-09

### Correções desta Versão

1. **Airflow healthchecker — falsos positivos corrigidos**: corrigidas as funções `convert_docker_airflow_output_to_df` e `get_airflow_dag_import_error` em `tests/airflow_tests.py` que geravam alertas de falha mesmo quando os DAGs estavam funcionando normalmente. Os filtros de parsing foram refinados para distinguir corretamente entre linhas de log com pipe (`|`) e avisos de sugestão do importador.

### Evoluções desta Versão

#### 1. Cache com escopo de requisição nas RNs de configuração

- **Arquivos afetados**: `sei/web/modulos/ia/rn/MdIaAdmConfigSimilarRN.php`, `MdIaAdmPesqDocRN.php`, `MdIaAdmOdsOnuRN.php`
- **Descrição**: adicionado `private static $objCache` em cada classe RN. `consultarConectado()` retorna do cache nas chamadas subsequentes dentro da mesma requisição PHP; `cadastrarControlado()` e `alterarControlado()` invalidam o cache para garantir consistência.
- **Impacto**: elimina 1 round-trip Oracle por classe RN por requisição. Em uma página com múltiplos hooks, economia de ~3–4 cursores abertos/fechados.
- **Assistência**: GitHub Copilot (Claude Sonnet 4.6)

#### 2. Batch pre-load + caches por procedimento para hooks de ícone

- **Arquivo afetado**: `sei/web/modulos/ia/IaIntegracao.php`
- **Descrição**: 
  - Adicionado cache estático em nível de classe para `exibeFuncionalidade()`, `exibeFuncionalidadeOdsOnu()`, `consultaUnidadeAlerta()`.
  - Adicionados caches indexados por `$idProcedimento` (`$cacheSugeridos`, `$cacheClassificados`) nos métodos `sugeridoPorUsuarioExtIntArtificial()` e `verificarSeJaFoiClassificadoAlgumaVez()`.
  - Em `montarArrIcone()` (usado por `montarIconeControleProcessos` e `montarIconeAcompanhamentoEspecial`), adicionado pré-carregamento em lote: dois `listar()` com filtro `IN (ids)` carregam todos os procedimentos da página antes do loop, eliminando o padrão N+1.
- **Impacto**: redução de **até 60 consultas para 1** em uma caixa de entrada com 30 procedimentos. `montarIconeProcesso` (chamado por procedimento isolado) cai para 0–1 consulta quando os caches estão populados.
- **Assistência**: GitHub Copilot (Claude Sonnet 4.6)

#### 3. Índices em MD_IA_DOC_INDEXAVEIS

- **Script de migração**: `sei/scripts/sei_atualizar_versao_modulo_ia.php` — método `instalarv141fmees()`
- **Descrição**: criação de índices nas colunas `SIN_VETORIZADO` e `SIN_INDEXADO` via `InfraMetaBD::criarIndice()` (compatível com Oracle, MySQL e PostgreSQL). A migração é executada automaticamente ao rodar o script de atualização a partir da versão `1.4.0` **ou `1.4.1`** do módulo.
- **Impacto**: elimina `TABLE ACCESS STORAGE FULL` nas queries de polling do Airflow (7.7k execuções/dia em `sin_vetorizado`, 2.4k em `sin_indexado`, 1.004 buffers/exec cada).
- **Assistência**: GitHub Copilot (Claude Sonnet 4.6)
- **Observação para Oracle**: a equipe pode avaliar a criação manual como índices bitmap (mais eficientes para colunas flag `'S'`/`'N'` de baixa cardinalidade), substituindo os índices B-tree gerados pelo script.

### Notas de Integração

- Todas as alterações são, em princípio, compatíveis com os demais bancos de dados suportados pelo SEI.
- Os mecanismos de cache são thread-safe dentro do escopo de uma requisição PHP (estáticos ao escopo da classe, não globais).
- Ao fazer merge com upstream, a divergência fica concentrada em 4 arquivos PHP e no script de migração. Usar `git diff upstream/master...local -- sei/web/modulos/ia/ sei/scripts/` para revisar a divergência.

## v1.4.1 (Upstream) e anteriores

Veja [releases de anatelgovbr/mod-sei-ia no GitHub](https://github.com/anatelgovbr/mod-sei-ia/releases) para o histórico completo.
