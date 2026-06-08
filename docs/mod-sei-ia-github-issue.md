*Assisted-by: GitHub Copilot:claude-4.6-sonnet*

Título: [MELHORIA]: [Oracle] Ausência de cache com escopo de requisição em RNs de configuração; padrão de consulta N+1 em hooks de ícone; índices ausentes em MD_IA_DOC_INDEXAVEIS

Descrição da melhoria:

---

### Qual funcionalidade você quer melhorar?

O desempenho do módulo durante a renderização de páginas do SEI (caixa de entrada, visualização de processo) em ambientes com banco de dados Oracle, e a eficiência das consultas de background realizadas pelos DAGs do Airflow na tabela `MD_IA_DOC_INDEXAVEIS`.

---

### Como ela funciona hoje?

Foram identificados três problemas de desempenho distintos, todos específicos do comportamento do Oracle (os ambientes PostgreSQL não são afetados da mesma forma).

#### Problema 1 — Ausência de cache nas RNs de configuração

A cada renderização de página do SEI, o módulo invoca os hooks registrados em `IaIntegracao.php` (`montarBotaoProcesso`, `montarBotaoDocumento`, `montarIconeProcesso`, `montarIconeControleProcessos`). Cada um desses hooks chama os métodos `exibeFuncionalidade()` e/ou `exibeFuncionalidadeOdsOnu()`, que por sua vez executam consultas ao banco de dados:

- `exibeFuncionalidade()` (arquivo: `IaIntegracao.php`, linha ~853) executa:
  - `MdIaAdmConfigSimilarRN::consultar()` → consulta `MD_IA_ADM_CONFIG_SIMILAR` (1 linha)
  - `MdIaAdmPesqDocRN::consultar()` → consulta `MD_IA_ADM_PESQ_DOC` (1 linha)
- `exibeFuncionalidadeOdsOnu()` (linha ~831) executa:
  - `MdIaAdmOdsOnuRN::consultar()` → consulta `MD_IA_ADM_ODS_ONU` (1 linha)
- `consultaUnidadeAlerta()` (linha ~117) executa:
  - `MdIaAdmUnidadeAlertaRN::consultar()` → consulta `MD_IA_ADM_UNIDADE_ALERTA` por `id_unidade`

Essas tabelas armazenam uma única linha de configuração que **não muda entre requisições**. Não há nenhum mecanismo de cache na camada RN: cada chamada ao método resulta em um round-trip ao Oracle, incluindo abertura e fechamento de cursor.

#### Problema 2 — Padrão N+1 nos hooks de ícone

Os métodos `montarIconeProcesso()` e o método interno `montarArrIcone()` (usado por `montarIconeControleProcessos`) iteram sobre cada procedimento da lista/caixa de entrada e, para cada um, executam individualmente:

1. `verificaAcessoOdsOnu()` → `exibeFuncionalidadeOdsOnu()` → consulta a `MD_IA_ADM_ODS_ONU`
2. `sugeridoPorUsuarioExtIntArtificial($idProcedimento)` → consulta a `MD_IA_CLASS_META_ODS` filtrando por `id_procedimento`
3. `verificarSeJaFoiClassificadoAlgumaVez($idProcedimento)` → outra consulta a `MD_IA_CLASS_META_ODS` filtrando por `id_procedimento`
4. `consultaUnidadeAlerta()` → consulta a `MD_IA_ADM_UNIDADE_ALERTA`

Isso resulta em **4 consultas Oracle por procedimento na lista**. Uma caixa de entrada com 30 itens gera, portanto, até 120 round-trips adicionais ao banco — por carregamento de página, por usuário simultâneo.

No PostgreSQL, o custo de cada round-trip é menor e o connection pooling (PgBouncer etc.) atenua o problema. No Oracle com `oci_connect()` por requisição (sem pool), cada round-trip extra tem custo mais elevado.

#### Problema 3 — Índices ausentes em MD_IA_DOC_INDEXAVEIS

A tabela `MD_IA_DOC_INDEXAVEIS` possui apenas índice primário em `ID_DOCUMENTO`. As colunas `SIN_VETORIZADO` e `SIN_INDEXADO`, usadas como predicado de filtro nas consultas mais frequentes executadas pelos DAGs do Airflow (polling de documentos pendentes de indexação/vetorização), não possuem índice.

Evidência extraída de `V$SQL` em ambiente de teste com 108.978 linhas:

| Execuções | ms/exec | Buffers/exec | Consulta |
|---|---|---|---|
| 7.733 | 3 ms | 1.004 | `SELECT COUNT(*) FROM md_ia_doc_indexaveis WHERE sin_vetorizado = :v1` |
| 3.228 | 11 ms | 1.004 | `SELECT * FROM md_ia_doc_indexaveis WHERE ...` (diversos filtros) |
| 2.403 | 2 ms | 1.004 | `SELECT COUNT(*) FROM md_ia_doc_indexaveis WHERE sin_indexado = :v1` |

O `EXPLAIN PLAN` confirma `TABLE ACCESS STORAGE FULL` em ambas as colunas. Em produção, com volumes maiores, o impacto é proporcionalmente maior.

---

### Como você sugere que ela funcione?

#### Sugestão 1 — Cache com escopo de requisição nas RNs de configuração e em IaIntegracao

Foram aplicados dois níveis complementares de cache:

**Nível 1 — Camada RN** (`MdIaAdmConfigSimilarRN.php`, `MdIaAdmPesqDocRN.php`, `MdIaAdmOdsOnuRN.php`): adicionado `private static $objCache = null;` em cada classe. `consultarConectado()` retorna o valor em cache nas chamadas subsequentes dentro da mesma requisição PHP. `cadastrarControlado()` e `alterarControlado()` invalidam o cache (`self::$objCache = null`) para garantir que uma escrita seja sempre seguida de uma leitura atualizada.

```php
class MdIaAdmConfigSimilarRN extends InfraRN
{
    private static $objCache = null;

    // ...

    protected function cadastrarControlado(MdIaAdmConfigSimilarDTO $objMdIaAdmConfigSimilarDTO)
    {
        self::$objCache = null;
        // ...
    }

    protected function alterarControlado(MdIaAdmConfigSimilarDTO $objMdIaAdmConfigSimilarDTO)
    {
        self::$objCache = null;
        // ...
    }

    protected function consultarConectado(MdIaAdmConfigSimilarDTO $objMdIaAdmConfigSimilarDTO)
    {
        if (self::$objCache !== null) {
            return self::$objCache;
        }
        // ...consulta ao banco...
        self::$objCache = $ret;
        return $ret;
    }
}
```

O mesmo padrão foi aplicado a `MdIaAdmPesqDocRN` e `MdIaAdmOdsOnuRN`.

**Nível 2 — IaIntegracao.php**: adicionado cache estático para os resultados computados de `exibeFuncionalidade()`, `exibeFuncionalidadeOdsOnu()` e `consultaUnidadeAlerta()`. Isso elimina também a reavaliação da chamada a `verificarPermissao()` nas chamadas repetidas. As três early returns originais foram substituídos por atribuição ao cache seguida de `return`, para garantir que o valor seja sempre armazenado independente do caminho de execução.

```php
class IaIntegracao extends SeiIntegracao
{
    private static $cacheExibeFuncionalidade = null;
    private static $cacheExibeFuncionalidadeOdsOnu = null;
    private static $cacheConsultaUnidadeAlerta = null;

    public function exibeFuncionalidade()
    {
        if (self::$cacheExibeFuncionalidade !== null) {
            return self::$cacheExibeFuncionalidade;
        }
        // ...lógica existente, sem early returns...
        self::$cacheExibeFuncionalidade = $bolExibirFuncionalidade;
        return $bolExibirFuncionalidade;
    }

    // idem para exibeFuncionalidadeOdsOnu() e consultaUnidadeAlerta()
}
```

#### Sugestão 2 — Substituir N+1 por consulta em lote em MD_IA_CLASS_META_ODS

A solução usa dois mecanismos complementares:

**Caches por procedure ID** nos métodos `sugeridoPorUsuarioExtIntArtificial` e `verificarSeJaFoiClassificadoAlgumaVez` (estáticos, indexados por `$idProcedimento`). Qualquer código que chame esses métodos mais de uma vez para o mesmo procedimento (incluindo `montarIconeProcesso`, que é invocado pela framework para cada procedimento individualmente) obtém o resultado do cache sem tocar o banco.

**Pré-carregamento em lote** em `montarArrIcone` (usado por `montarIconeControleProcessos` e `montarIconeAcompanhamentoEspecial`): antes do loop, dois `listar()` com `IN (ids)` populam os caches estáticos para todos os procedimentos da página. O loop interno continua chamando os mesmos métodos privados — que agora retornam do cache O(1).

```php
class IaIntegracao extends SeiIntegracao
{
    // ...caches existentes...
    private static $cacheSugeridos = [];    // indexed by id_procedimento
    private static $cacheClassificados = []; // indexed by id_procedimento

    private function montarArrIcone($arrObjProcedimentoDTO)
    {
        if ($this->verificaAcessoOdsOnu(NULL)) {

            $arrIdsProcedimentos = array_map(
                function($p) { return $p->getIdProcedimento(); },
                $arrObjProcedimentoDTO
            );

            if (!empty($arrIdsProcedimentos)) {
                // Query 1: procedures with a suggestion from IA or external user
                $objDTOSugeridos = new MdIaClassMetaOdsDTO();
                $objDTOSugeridos->setDblIdProcedimento($arrIdsProcedimentos, InfraDTO::$OPER_IN);
                $objDTOSugeridos->setStrStaTipoUsuario(
                    array(MdIaClassMetaOdsRN::$USUARIO_IA, MdIaClassMetaOdsRN::$USUARIO_EXTERNO),
                    InfraDTO::$OPER_IN
                );
                $objDTOSugeridos->retDblIdProcedimento();
                $arrResultSugeridos = (new MdIaClassMetaOdsRN())->listar($objDTOSugeridos);
                $mapSugeridos = [];
                if ($arrResultSugeridos) {
                    foreach ($arrResultSugeridos as $c) {
                        $mapSugeridos[$c->getDblIdProcedimento()] = true;
                    }
                }
                foreach ($arrIdsProcedimentos as $id) {
                    if (!array_key_exists($id, self::$cacheSugeridos)) {
                        self::$cacheSugeridos[$id] = isset($mapSugeridos[$id]);
                    }
                }

                // Query 2: procedures already classified by standard user or scheduler
                $objDTOClassificados = new MdIaClassMetaOdsDTO();
                $objDTOClassificados->setDblIdProcedimento($arrIdsProcedimentos, InfraDTO::$OPER_IN);
                $objDTOClassificados->setStrStaTipoUsuario(
                    array(MdIaClassMetaOdsRN::$USUARIO_PADRAO, MdIaClassMetaOdsRN::$USUARIO_AGENDAMENTO),
                    InfraDTO::$OPER_IN
                );
                $objDTOClassificados->retDblIdProcedimento();
                $arrResultClassificados = (new MdIaClassMetaOdsRN())->listar($objDTOClassificados);
                $mapClassificados = [];
                if ($arrResultClassificados) {
                    foreach ($arrResultClassificados as $c) {
                        $mapClassificados[$c->getDblIdProcedimento()] = true;
                    }
                }
                foreach ($arrIdsProcedimentos as $id) {
                    if (!array_key_exists($id, self::$cacheClassificados)) {
                        self::$cacheClassificados[$id] = isset($mapClassificados[$id]);
                    }
                }
            }

            foreach ($arrObjProcedimentoDTO as $objProcedimentoDTO) {
                // These now hit the static cache — no DB round-trip
                if ($this->sugeridoPorUsuarioExtIntArtificial(...)) { ... }
                if (!$this->verificarSeJaFoiClassificadoAlgumaVez(...) && ...) { ... }
            }
        }
    }

    private function sugeridoPorUsuarioExtIntArtificial($idProcedimento)
    {
        if (array_key_exists($idProcedimento, self::$cacheSugeridos)) {
            return self::$cacheSugeridos[$idProcedimento];
        }
        // ...individual query as fallback (used by montarIconeProcesso)...
        $ret = (bool)(new MdIaClassMetaOdsRN())->consultar($objMdIaClassMetaOdsDTO);
        self::$cacheSugeridos[$idProcedimento] = $ret;
        return $ret;
    }

    // idem para verificarSeJaFoiClassificadoAlgumaVez
}
```

**Resultado:** `montarIconeControleProcessos` / `montarIconeAcompanhamentoEspecial` passam de 2N queries para 2 queries por carregamento de lista. `montarIconeProcesso` (hook por procedimento individual) cai para 0 queries adicionais quando os caches já foram populados pelo lote, ou 1 query por procedimento único quando chamado isoladamente.

#### Sugestão 3 — Adicionar índices em MD_IA_DOC_INDEXAVEIS no script de atualização

Incluir a criação dos índices no script `sei_atualizar_versao_modulo_ia.php`, no bloco de migração da próxima versão, usando a API `InfraMetaBD::criarIndice()` já adotada no restante do script (compatível com Oracle, MySQL e PostgreSQL):

```php
$this->logar('CRIANDO ÍNDICES EM md_ia_doc_indexaveis');
$objInfraMetaBD->criarIndice('md_ia_doc_indexaveis', 'ix_md_ia_doc_idx_sin_vetorizado', array('sin_vetorizado'));
$objInfraMetaBD->criarIndice('md_ia_doc_indexaveis', 'ix_md_ia_doc_idx_sin_indexado',   array('sin_indexado'));
```

Esses índices eliminam o full scan nas consultas de polling do Airflow. Para Oracle, a equipe pode avaliar a criação como índices bitmap (mais eficientes para colunas de baixa cardinalidade como flags `'S'`/`'N'`), mas o método `criarIndice()` já resolve o problema para todos os bancos suportados.

---

### Que benefícios isso traria?

- **Cache (Sugestão 1):** elimina 3–4 round-trips Oracle por hook invocado, por carregamento de página. Em uma caixa de entrada com múltiplos usuários simultâneos, o ganho é imediato e proporcional à carga.
- **Consulta em lote (Sugestão 2):** reduz de N consultas para 1 por carregamento de lista/caixa de entrada para a tabela `MD_IA_CLASS_META_ODS`. Uma caixa com 30 itens passa de até 60 consultas para 1.
- **Índices (Sugestão 3):** elimina full scans repetidos em `MD_IA_DOC_INDEXAVEIS` durante o processamento de background pelo Airflow, reduzindo pressão no buffer cache do Oracle e melhorando o throughput do pipeline de indexação/vetorização.

As três melhorias são ortogonais e podem ser implementadas independentemente.

---

Nome do órgão: TCE-RS

Versão do SEI: 5.0.4

Versão do módulo: 1.4.0

Tipo de banco de dados do SEI: Oracle

Perfil do usuário: administrador do sistema
