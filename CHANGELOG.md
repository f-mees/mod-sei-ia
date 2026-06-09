# Histórico de Versões

# v1.4.1 (Upstream)
**Versão estável atual**

## Correções desta Versão
1. Correção de bugs gerais.

## Evoluções desta Versão
1. Suporte a ambientes de produção com Oracle.

---

# v1.4.1-ferabreu (fork)
**Melhorias de desempenho para Oracle; baseada em v1.4.1**

> **Nota:** Esta versão é um ramo de desenvolvimento contendo otimizações específicas para o TCE-RS. Ao atualizar o módulo a partir do upstream, será necessário fazer cherry-pick seletivo dessas alterações ou aguardar integração no master.

## Correções desta Versão
1. **Airflow healthchecker — falsos positivos corrigidos**: corrigidas as funções `convert_docker_airflow_output_to_df` e `get_airflow_dag_import_error` em `tests/airflow_tests.py` que geravam alertas de falha mesmo quando os DAGs estavam funcionando normalmente. Os filtros de parsing foram refinados para distinguir corretamente entre linhas de log com pipe (`|`) e avisos de sugestão do importador.

## Evoluções desta Versão

### 1. Cache com escopo de requisição nas RNs de configuração
- **Arquivos afetados**: `sei/web/modulos/ia/rn/MdIaAdmConfigSimilarRN.php`, `MdIaAdmPesqDocRN.php`, `MdIaAdmOdsOnuRN.php`
- **Descrição**: adicionado `private static $objCache` em cada classe RN. `consultarConectado()` retorna do cache nas chamadas subsequentes dentro da mesma requisição PHP; `cadastrarControlado()` e `alterarControlado()` invalidam o cache para garantir consistência.
- **Impacto**: elimina 1 round-trip Oracle por classe RN por requisição. Em uma página com múltiplos hooks, economia de ~3–4 cursores abertos/fechados.
- **Assistência**: GitHub Copilot (Claude Sonnet 4.6)

### 2. Batch pre-load + caches por procedimento para hooks de ícone
- **Arquivo afetado**: `sei/web/modulos/ia/IaIntegracao.php`
- **Descrição**: 
  - Adicionado cache estático em nível de classe para `exibeFuncionalidade()`, `exibeFuncionalidadeOdsOnu()`, `consultaUnidadeAlerta()`.
  - Adicionados caches indexados por `$idProcedimento` (`$cacheSugeridos`, `$cacheClassificados`) nos métodos `sugeridoPorUsuarioExtIntArtificial()` e `verificarSeJaFoiClassificadoAlgumaVez()`.
  - Em `montarArrIcone()` (usado por `montarIconeControleProcessos` e `montarIconeAcompanhamentoEspecial`), adicionado pré-carregamento em lote: dois `listar()` com filtro `IN (ids)` carregam todos os procedimentos da página antes do loop, eliminando o padrão N+1.
- **Impacto**: redução de **até 60 consultas para 1** em uma caixa de entrada com 30 procedimentos. `montarIconeProcesso` (chamado por procedimento isolado) cai para 0–1 consulta quando os caches estão populados.
- **Assistência**: GitHub Copilot (Claude Sonnet 4.6)

### 3. Índices em MD_IA_DOC_INDEXAVEIS (pendente de integração com banco de dados)
- **Script de migração a implementar**: `sei_atualizar_versao_modulo_ia.php`
- **Descrição**: criação de índices nas colunas `SIN_VETORIZADO` e `SIN_INDEXADO` para eliminar full scans nas consultas de polling do Airflow.
- **Impacto**: melhoria significativa no throughput do pipeline de indexação/vetorização em ambientes com volumes maiores.
- **Status**: requer teste e aprovação do DBA antes do deploy.

## Notas de Integração
- Todas as alterações são, em princípio, compatíveis com os demais bancos de dados suportados pelo SEI.
- Os mecanismos de cache são thread-safe dentro do escopo de uma requisição PHP (estáticos ao escopo da classe, não globais).
- Ao fazer merge com upstream, a divergência fica concentrada em 4 arquivos PHP da camada de integração. Usar `git diff upstream/master...tce-rs -- sei/web/modulos/ia/` para revisar a divergência.

---

# v1.4.0 e anteriores
Veja [releases no GitHub](https://github.com/anatelgovbr/mod-sei-ia/releases) para histórico completo.
