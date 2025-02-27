Ótima abordagem! Vou pesquisar as melhores soluções para processar arquivos tabulados grandes (25 milhões de registros) com Java 17 e Spring Batch 5, focando em:

- Leitura eficiente de arquivos tabulados com layout fixo (inspirado no AWS Athena)
- Uso de bibliotecas como Apache Parquet e Apache Arrow para otimização
- Comparação de dados do arquivo com PostgreSQL sem precisar sobrescrever tudo diariamente
- Estratégias de detecção de alterações (inserção, atualização e exclusão)
- Desempenho e paralelismo para lidar com o grande volume de dados

Assim que tiver as informações, volto com um resumo das melhores abordagens e possíveis implementações para seu cenário.

# Processamento Eficiente de Arquivos Tabulados Massivos com Java 17 e Spring Batch 5

## Desafio: Snapshot Diário de 25 Milhões de Registros  
Temos diariamente um **arquivo tabulado de layout fixo** com ~25 milhões de linhas, representando um _snapshot_ completo de dados. O objetivo é **carregar e comparar esse arquivo com dados armazenados no PostgreSQL** para aplicar apenas as diferenças (inserções, atualizações e remoções), em vez de substituir toda a tabela a cada dia. Isso exige uma solução de **alto desempenho** que aproveite os recursos do Java 17 e Spring Batch 5 para leitura eficiente, detecção de mudanças e escrita otimizada no banco de dados.

## Leitura Eficiente de Arquivos de Layout Fixo  
Para ler um arquivo texto tão grande com eficiência:  

- **Streaming e Mapeamento Direto:** Utilize o Spring Batch _FlatFileItemReader_ com um _LineMapper_ adequado para layout fixo. Por exemplo, o _FixedLengthTokenizer_ do Spring Batch permite definir os intervalos de colunas, evitando uso manual de substrings ([Processing files with fixed line length using Spring Batch - Craftsmen](https://craftsmen.nl/processing-files-with-fixed-line-length-using-spring-batch/#:~:text=The%20advantage%20of%20using%20a,much%20clearer%20for%20the%20reader)) ([Processing files with fixed line length using Spring Batch - Craftsmen](https://craftsmen.nl/processing-files-with-fixed-line-length-using-spring-batch/#:~:text=public%20FixedLengthTokenizer%20createLineTokenizer%28%29%20,getColumnRanges)). Isso parseia cada linha em campos de maneira eficiente.  
- **Bufferização e NIO:** Use leitura em fluxo (_streaming_) para não carregar todo o arquivo em memória. Em Java 17, classes NIO (como FileChannel e MappedByteBuffer) podem mapear o arquivo em memória ou ler em blocos grandes, reduzindo chamadas de I/O. Ler chunks maiores de bytes e então separar em linhas amortiza o custo do disco.  
- **Processamento por **chunks****: Configure o Spring Batch em modo *chunk-oriented* com um tamanho de chunk elevado (por exemplo, 5.000 ou 10.000 registros por transação) para reduzir overhead de transações frequentes. Cada chunk lido será processado e escrito em lote, equilibrando uso de memória e throughput.  
- **Parallelização da Leitura:** Considere paralelizar a leitura usando **partitioning** do Spring Batch. Divida o arquivo em partes e processe em threads separadas. Como o layout é fixo (tamanho constante por linha), é viável calcular faixas de linhas ou offsets de byte para cada partição. Por exemplo, 5 threads processando ~5 milhões de linhas cada. O Spring Batch permite usar um *Partitioner* customizado ou pré-dividir o arquivo em vários arquivos menores e usar *MultiResourceItemReader*. Dividir um arquivo grande em partes menores é “a forma indicada” para arquivos massivos ([Spring batch for large files(1billion to 5billion flat file data) - Stack Overflow](https://stackoverflow.com/questions/47933599/spring-batch-for-large-files1billion-to-5billion-flat-file-data#:~:text=Secondly%2C%20you%20are%20correct%20in,can%20be%20easily%20grouped%20later)) – isso habilita leitura e parsing em paralelo aproveitando CPUs multicore (lembrando de limitar o número de partições para não sobrecarregar o framework).  

**Exemplo:** Configurando um leitor para layout fixo no Spring Batch (definindo colunas por posição e mapeando para um objeto):  

```java
@Bean
public FlatFileItemReader<Cliente> clienteItemReader() {
    FlatFileItemReader<Cliente> reader = new FlatFileItemReader<>();
    reader.setResource(new FileSystemResource("dados/snapshot.txt"));
    reader.setEncoding("UTF-8");
    reader.setLineMapper(new DefaultLineMapper<>() {{
        setLineTokenizer(new FixedLengthTokenizer() {{
            setNames("id", "nome", "endereco", "saldo", /* ... */);
            setColumns(new Range[]{
                new Range(1, 10),    // id: colunas 1-10
                new Range(11, 40),   // nome: colunas 11-40
                new Range(41, 80),   // endereco: colunas 41-80
                new Range(81, 90)    // saldo: colunas 81-90
                // ... demais colunas conforme layout fixo
            });
        }});
        setFieldSetMapper(new BeanWrapperFieldSetMapper<>() {{
            setTargetType(Cliente.class);
        }});
    }});
    return reader;
}
```  

Esse leitor streama o arquivo linha a linha, convertendo para instâncias da classe `Cliente`. Para aumentar a performance, podemos envolver este reader com um **SynchronizedItemStreamReader** (se usar múltiplas threads no step) ou criar partições com diferentes recursos/offsets para leitura concorrente.

## Otimizando com Apache Parquet e Arrow (Formato Colunar)  
Uma forma de acelerar a leitura é adotar formatos **colunares** inspirados no Athena (AWS). O Athena recomenda armazenar dados em Parquet ou ORC por serem formatos otimizados para leitura em massa. **Apache Parquet** é um formato colunar comprimido que minimiza I/O e melhora o throughput de leitura ([Apache Parquet: How to be a hero with the open-source columnar data format | Openbridge](https://blog.openbridge.com/how-to-be-a-hero-with-powerful-parquet-google-and-amazon-f2ae0f35ee04#:~:text=optimized%20for%20query%20performance%20and,minimizing%20I%2FO)) ([Apache Parquet: How to be a hero with the open-source columnar data format | Openbridge](https://blog.openbridge.com/how-to-be-a-hero-with-powerful-parquet-google-and-amazon-f2ae0f35ee04#:~:text=Parquet%20performance%2C%20when%20compared%20to,CSV)). Em comparação a CSV/texto, o Parquet traz ganhos significativos em eficiência e custo, pois reduz o volume de dados lido e escrito e suporta compressão efetiva ([Apache Parquet: How to be a hero with the open-source columnar data format | Openbridge](https://blog.openbridge.com/how-to-be-a-hero-with-powerful-parquet-google-and-amazon-f2ae0f35ee04#:~:text=Parquet%20performance%2C%20when%20compared%20to,CSV)). Em aplicações reais, é recomendado usar um formato colunar como Parquet para persistência otimizada – **ler/gravar arquivos Parquet é mais rápido** que em formato texto bruto ([Apache Arrow e Java: Transferência de Big Data na velocidade da luz - InfoQ](https://www.infoq.com/br/articles/apache-arrow-java#:~:text=Observem%20que%20o%20Apache%20Arrow,%C3%A9%20puramente%20por%20motivos%20did%C3%A1ticos)).  

- **Conversão para Parquet:** Se o snapshot diário puder ser pré-processado, converta-o para Parquet. O arquivo Parquet incorporará o esquema tabular e permite leitura seletiva de colunas. Por exemplo, se apenas algumas colunas são necessárias para identificar mudanças (como chave primária e um hash das demais colunas), o leitor Parquet pode projetar apenas essas colunas, reduzindo drasticamente o I/O. O Apache Parquet no Java (biblioteca *parquet-mr*) fornece APIs para ler e escrever Parquet; você pode integrá-las em um ItemReader customizado.  
- **Leitura Vetorizada com Arrow:** O **Apache Arrow** fornece um formato de dados em memória colunar que permite processamento vetorizado e zero-cópia entre sistemas. Para cargas analíticas, um layout colunar em memória explora instruções SIMD da CPU e acessa dados de cada coluna de forma contígua, acelerando filtros e comparações ([Apache Arrow e Java: Transferência de Big Data na velocidade da luz - InfoQ](https://www.infoq.com/br/articles/apache-arrow-java#:~:text=instru%C3%A7%C3%B5es%20SIMD%20das%20CPUs,orientado%20por%20linhas%20do%20FlatBuffers)). Em outras palavras, armazenar os dados lidos em estruturas Arrow (como `VectorSchemaRoot` com `FieldVector`s por coluna) possibilita aplicar operações em lote nas colunas com alta performance. Por exemplo, poderia-se usar Arrow para carregar 10.000 registros em um batch e calcular hashes ou comparar campos de forma vetorizada ao invés de linha a linha.  
- **Integração Parquet ↔ Arrow:** Arrow e Parquet se complementam – pode-se ler um arquivo Parquet diretamente em vetores Arrow. O Arrow Java fornece classes como `ArrowFileReader` ou integração com Parquet (ex.: `ParquetReader` do Arrow) que entregam **RecordBatches** de rows materializados em memória colunar. Assim, podemos ler grandes blocos do snapshot com custo amortizado e processar em memória de forma eficiente antes de persistir no PostgreSQL. *(Observação:* O próprio artigo do Arrow ressalta que Parquet, por incluir compressão e índices de bloco, é superior para armazenamento em disco, enquanto Arrow é voltado a processamento em memória ([Apache Arrow e Java: Transferência de Big Data na velocidade da luz - InfoQ](https://www.infoq.com/br/articles/apache-arrow-java#:~:text=Observem%20que%20o%20Apache%20Arrow,%C3%A9%20puramente%20por%20motivos%20did%C3%A1ticos)).)*  

Em resumo, sempre que possível: **use formato colunar no armazenamento e processamento**. Converta dados de entrada fixo/CSV para Parquet para diminuir o tamanho e acelerar leituras, e internamente use estruturas colunares (Arrow) para manipular os dados em lotes maiores, tirando proveito máximo do hardware.

## Detecção de Inserções, Atualizações e Exclusões (Delta de Dados)  
Com os dados do arquivo sendo lidos eficientemente, o próximo foco é **comparar com o PostgreSQL sem regravar tudo**. Ou seja, implementar um *incremental load* (carga incremental) ao invés de um *full load*. A ideia é carregar **apenas dados novos ou modificados** e identificar quais registros do banco precisam ser removidos. Diferente de uma carga completa que sobrescreve todo o conjunto, uma carga incremental “foca em transferir somente as mudanças” – isso é mais rápido, consome menos recursos e preserva dados já existentes ([Incremental Data Load vs Full Load ETL: Key Differences](https://hevodata.com/learn/incremental-data-load-vs-full-load/#:~:text=Incremental%20loading%20is%20the%20process,resources%2C%20and%20preserves%20historical%20data)). Para conseguir isso:  

- **Chave Primária e Hash:** Assegure que exista uma **chave única** (ou combinação de colunas) para identificar cada registro. Use essa chave para comparação. Uma técnica útil é calcular um *hash* (MD5, CRC32 ou outro) dos campos relevantes de cada registro para detectar mudanças de conteúdo rapidamente. O snapshot diário completo representa o estado final; podemos comparar com o estado atual no PostgreSQL calculando hashes e comparando chaves.  
- **Carga da Base Atual (opcional):** Uma abordagem é extrair do PostgreSQL um mapa (ou conjunto) de chaves existentes e suas hashes. Com 25 milhões de registros, isso exige memória significativa, mas em Java pode ser viável se houver recursos (25 milhões de chaves inteiras, por exemplo, podem ocupar algumas centenas de MB). Supondo memória suficiente, carregar esses dados em uma estrutura como `HashMap<Chave, Hash>` fornece busca O(1) para cada registro do arquivo.  
- **Comparação Registro a Registro:** Conforme iteramos cada registro do novo snapshot:  
  - Se a chave **não existir** no mapa da base atual, marcamos como **inserção** (novo registro).  
  - Se a chave existe, comparamos o hash (ou os campos) para ver se mudou. Se **diferente**, marcamos como **atualização**; se igual, é um registro **inalterado** (podemos ignorá-lo, não requer ação).  
  - Em ambos os casos de inserção/atualização, removemos essa chave do mapa de referência (para no fim sabermos quais sobram).  
- **Coletando Diferenças:** Após processar todo o arquivo, quaisquer chaves que **permaneceram no mapa** de dados antigos representam registros que existiam no banco mas **não estavam no novo snapshot** – ou seja, foram **removidos** na fonte e devem ser excluídos do PostgreSQL.  

Essa lógica extrai somente as diferenças para aplicar no banco. Podemos implementá-la dentro de um *ItemProcessor* do Spring Batch: o processor recebe cada objeto lido e decide se gera um item de saída (por exemplo, um objeto envolto que indica tipo de operação: inserção, update, deleção) ou talvez envia diretamente a ação ao writer correspondente. Alternativamente, podemos separar em dois passos: primeiro inserir/atualizar e coletar keys novas, depois excluir as ausentes.

**Estratégia Alternativa:** Caso não seja viável carregar 25M chaves em memória, podemos usar o próprio **banco de dados** para auxiliar na comparação:  
- Carregue o snapshot em uma **tabela de estágio** (staging) no PostgreSQL, usando por exemplo a função de *COPY* do Postgres para carga em massa (muito rápida) ou um writer batch do Spring. Em seguida, execute comandos SQL para identificar diferenças:  
  - Inserções: `INSERT INTO tabela_destino (...) SELECT ... FROM staging s LEFT JOIN destino d ON s.id=d.id WHERE d.id IS NULL` (novos ids).  
  - Atualizações: `UPDATE destino SET ... FROM staging s WHERE destino.id=s.id AND (campos diferirem)`  
  - Remoções: `DELETE FROM destino d WHERE NOT EXISTS (SELECT 1 FROM staging s WHERE s.id = d.id)` (excluir ids que sumiram).  
- Essa abordagem aproveita operações set-oriented do SGBD, que são bastante otimizadas. Porém, requer espaço extra e passos adicionais (limpar tabela de estágio etc.). Dependendo dos recursos do banco, pode ser tão rápida quanto ou mais que processar em Java. Em um cenário Spring Batch puro, a lógica Java descrita inicialmente (mapa em memória + diff) é comum para aplicar deltas.

## Escrita Otimizada no PostgreSQL (Upserts e Exclusões)  
Após identificar o que inserir, atualizar ou deletar, é preciso aplicar essas mudanças **rapidamente** no PostgreSQL, respeitando a volumetria:  

- **Batch e JDBC Direto:** Evite operações linha a linha ou uso ingênuo de JPA para cada registro – isso seria extremamente lento para milhões de itens. Em vez disso, use JDBC batch via Spring Batch. O **JdbcBatchItemWriter** permite agrupar diversas instruções SQL em uma única ida ao banco a cada chunk. Podemos usá-lo com uma instrução SQL parametrizada para **upsert** (insert/update):  
  - O PostgreSQL suporta nativamente *UPSERT* via sintaxe `INSERT ... ON CONFLICT (chave) DO UPDATE ...`. Essa única query cuida de inserir novos registros e atualizar os existentes em conflito de chave. É muito eficiente: testes mostram que operações de upsert podem ser em média **30 vezes mais rápidas** que usar métodos padrão do JPA para salvar registros individualmente ([Spring and PostgreSQL: Make Your Database Inserts 30 Times Faster | HackerNoon](https://hackernoon.com/spring-and-postgresql-make-your-database-inserts-30-times-faster#:~:text=Tests%20indicate%20that%20upsert%20operations,upsert%20queries%20for%20any%20object)).  
  - Exemplo de configuração do writer com upsert:  

    ```java
    @Bean
    public JdbcBatchItemWriter<Cliente> clienteItemWriter(DataSource dataSource) {
        JdbcBatchItemWriter<Cliente> writer = new JdbcBatchItemWriter<>();
        writer.setDataSource(dataSource);
        writer.setSql("INSERT INTO cliente (id, nome, endereco, saldo) " +
                      "VALUES (:id, :nome, :endereco, :saldo) " +
                      "ON CONFLICT (id) DO UPDATE SET " +
                      "nome = EXCLUDED.nome, " +
                      "endereco = EXCLUDED.endereco, " +
                      "saldo = EXCLUDED.saldo");
        writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
        return writer;
    }
    ```  

  Nesse exemplo, para cada `Cliente` a ser gravado, o Spring Batch fará um batch de inserts. O `ON CONFLICT (id) DO UPDATE` garante que se o `id` já existe, os campos serão atualizados em vez de violar a unicidade. Desse modo, inserções e atualizações são tratadas juntas de forma idempotente.  

- **Tamanho do Lote e Performance:** Utilize chunk sizes grandes o suficiente (por ex. 1.000 ou 5.000) para que cada batch de JDBC envie muitos registros de uma vez. Menos viagens de rede = maior throughput. Além disso, habilite a opção JDBC do driver PostgreSQL `reWriteBatchedInserts=true` na string de conexão. Essa opção faz com que o driver **re-escreva inserts batch em uma única instrução multi-valores** transparentemente ([PostgreSQL reWriteBatchedInserts configuration property - Vlad Mihalcea](https://vladmihalcea.com/postgresql-multi-row-insert-rewritebatchedinserts-property/#:~:text=This%20time%2C%20we%20only%20have,processing%20on%20the%20database%20side)). Por exemplo, em vez de 1000 comandos INSERT separados, ele envia um único INSERT com 1000 linhas `VALUES(...)`. Isso **reduz drasticamente** o tempo de execução no lado do banco, como observado por Vladimir Sitnikov e outros: o número de execuções cai e o processamento do lote fica bem mais rápido ([PostgreSQL reWriteBatchedInserts configuration property - Vlad Mihalcea](https://vladmihalcea.com/postgresql-multi-row-insert-rewritebatchedinserts-property/#:~:text=This%20time%2C%20we%20only%20have,processing%20on%20the%20database%20side)). Em testes, habilitar `reWriteBatchedInserts` levou a bem menos comandos executados e menor tempo total de inserção.  
- **Deletes em Lote:** As exclusões de registros antigos podem ser feitas após processar as inserções/atualizações. Duas abordagens: (1) Coletar as chaves a excluir em memória durante a leitura (como citado, as chaves que sobraram no mapa de antigos) e então usar um **JdbcTemplate.batchUpdate** para emitir deletes em lote para essas chaves, ou (2) mais eficientemente, executar uma única query SQL que remova todos de uma vez. Por exemplo, se inserimos todas as chaves novas em uma tabela temporária `novas_chaves`, podemos executar `DELETE FROM cliente c WHERE NOT EXISTS (SELECT 1 FROM novas_chaves n WHERE n.id = c.id)`. Assim, o banco elimina em bloco todas as linhas que não apareceram no snapshot novo. Essa operação deve usar o índice primário para buscar as não correspondências.  
- **Considerações de Índice:** Certifique-se de que a tabela destino no PostgreSQL tenha índices adequados na chave primária (e eventualmente em colunas de junção usadas na comparação). As operações de upsert e de delete por chave irão efetuar _lookups_ por chave; um índice bem construído garante que essas buscas sejam O(log N) e mantenham a performance mesmo com dezenas de milhões de registros.  

Em Spring Batch, podemos orquestrar isso em **duas etapas (steps)** do Job:  
1. Step de *chunk* lendo o arquivo e gravando (via upsert) no PostgreSQL os novos/alterados.  
2. Step seguinte para deletar os removidos.  

Por exemplo, o Step 2 poderia ser um Tasklet simples que roda uma query de deleção com `jdbcTemplate` ou chama uma SP no banco. Separar em steps mantém a lógica clara e permite commit intermediário, evitando transação gigantesca englobando tudo.

## Paralelismo e Melhores Práticas de Desempenho  
Lidar com 25 milhões de registros é intensivo – tirar proveito de paralelismo e otimizações de JVM é crucial:  

- **Partitioning e Multithreading:** Conforme mencionado na leitura, paralelize o processamento sempre que possível. No Spring Batch, além de particionar a leitura, você pode usar um **TaskExecutor** no step para processar chunks em múltiplas threads. Por exemplo, `stepBuilderFactory.get("step1").<Tipo,Tipo>chunk(10000).reader(reader).processor(proc).writer(writer).taskExecutor(new ThreadPoolTaskExecutor()/*...*/).build();` irá permitir que múltiplos threads consumam o reader e processem chunks simultaneamente. *Nota:* O FlatFileItemReader padrão não é thread-safe, então para usar `taskExecutor` diretamente talvez seja preciso um `SynchronizedItemStreamReader` envolvendo o reader ou garantir partições isoladas por thread. A abordagem de partitioner com vários readers (um por partição) costuma ser mais escalável.  
- **Async Processing:** Outra técnica suportada (via *spring-batch-integration*) é usar **AsyncItemProcessor** e **AsyncItemWriter** para sobrepor etapas – enquanto um thread escreve um chunk no banco, o próximo chunk já pode estar sendo processado em paralelo ([Processing files with fixed line length using Spring Batch - Craftsmen](https://craftsmen.nl/processing-files-with-fixed-line-length-using-spring-batch/#:~:text=,for%20writing)). Isso aumenta o aproveitamento de CPU vs I/O. No caso de upsert no banco, o gargalo pode ser o tempo de espera do DB confirmar a transação; com escrita assíncrona, podemos preparar o próximo lote durante esse tempo.  
- **Recursos do Java 17:** Aproveite as melhorias da JVM moderna. Por exemplo, o G1 GC em Java 17 lida bem com grandes montantes de objetos de curta duração (como criar milhões de objetos de linha) – monitore a GC pause para ajustar _heap_ se necessário. Você pode também experimentar **Threads Virtuais (Project Loom)** introduzidas em Java 19 (em anteprojeto), mas o Spring Batch 5 já é compatível com Java 17 e pode tirar proveito de APIs modernas de concorrência. Certifique-se de usar tipos de dados eficientes (por exemplo, preferir tipos primitivos ou _streams_ de bytes para cálculo de hash em vez de objetos pesados).  
- **Logging e I/O extra:** Desabilite ou minimize logs no loop principal de processamento – escrever logs para milhões de itens irá degradar a performance. Use logging resumido (por exemplo, log a cada 1 milhão de registros processados). Similarmente, evite qualquer operação complexa dentro do processor – mantenha-o o mais leve possível (idealmente apenas cálculo de hash e comparação, ou formatação de dados para output).  
- **Teste de Performance e Tuning:** Com esse volume, é importante testar em ambiente próximo ao de produção. Meça tempos de leitura pura, processamento e escrita separadamente se puder. Ajuste o grau de paralelismo (número de threads ou partitioners) de acordo com os **recursos da máquina e do banco** – há um ponto ótimo em que mais threads não aumentam velocidade devido a saturação de disco ou CPU. Garanta também que o PostgreSQL esteja configurado para receber cargas pesadas (tune de `max_wal_size`, `checkpoint_timeout`, etc., se for muita inserção, bem como autovacuum para evitar bloat nas updates).  

## Exemplo de Fluxo com Spring Batch 5  
Finalmente, combinando as peças, um esboço simplificado do **Job Spring Batch** para esse caso:  

```java
@Bean
public Job importarSnapshotJob(JobBuilderFactory jobFactory, Step upsertStep, Step deleteStep) {
    return jobFactory.get("importarSnapshotJob")
        .start(upsertStep)
        .next(deleteStep)
        .build();
}

@Bean
public Step upsertStep(StepBuilderFactory stepFactory, ItemReader<Cliente> reader,
                       ItemProcessor<Cliente, Cliente> processor, ItemWriter<Cliente> writer,
                       TaskExecutor taskExecutor) {
    return stepFactory.get("upsertStep")
        .<Cliente, Cliente>chunk(5000)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .taskExecutor(taskExecutor)  // paraleliza chunks
        .build();
}

@Bean
public Step deleteStep(StepBuilderFactory stepFactory, DataSource dataSource) {
    // Tasklet que remove registros ausentes no snapshot
    return stepFactory.get("deleteStep")
        .tasklet((contrib, context) -> {
            JdbcTemplate jdbc = new JdbcTemplate(dataSource);
            String sqlDelete = "DELETE FROM cliente c " + 
                               "WHERE NOT EXISTS (SELECT 1 FROM novas_chaves n WHERE n.id = c.id)";
            jdbc.update(sqlDelete);
            return RepeatStatus.FINISHED;
        }).build();
}
```  

No exemplo, o `upsertStep` lê e aplica inserts/updates em paralelo (usando chunks de 5000). O `ItemProcessor` aqui poderia ser responsável por coletar as chaves em uma tabela `novas_chaves` (por exemplo, inserindo cada chave lida em uma tabela temporária ou lista compartilhada) – ou poderíamos ter feito isso dentro do writer mesmo. Após o upsertStep, o `deleteStep` executa uma única query para remover do destino quaisquer IDs não presentes na tabela de novas_chaves (que representa o snapshot atual). Assim, realizamos as **inserções e atualizações incrementalmente**, depois limpamos as **exclusões** em bloco.

*(Observação:* Poderíamos alternadamente usar um `ItemWriter` customizado no step de upsert que, além de escrever no destino, armazena cada chave processada numa coleção; então no método `afterStep` (listener) remover os faltantes comparando com um mapa inicial de chaves do banco. Escolher a estratégia depende do trade-off memória vs. carga DB vs. complexidade.)*

## Conclusão  
A solução combinando **Spring Batch 5** e bibliotecas colunares atinge o objetivo: ler 25 milhões de registros eficientemente, detectar diferenças e aplicar no PostgreSQL somente as mudanças. A chave do desempenho está em *streaming* otimizado (usando formatos de arquivo eficientes como Parquet ([Apache Arrow e Java: Transferência de Big Data na velocidade da luz - InfoQ](https://www.infoq.com/br/articles/apache-arrow-java#:~:text=Observem%20que%20o%20Apache%20Arrow,%C3%A9%20puramente%20por%20motivos%20did%C3%A1ticos)), leitura em lote e possivelmente estrutura Arrow in-memory), *algoritmos de comparação* que evitam retrabalho (carga incremental em vez de total ([Incremental Data Load vs Full Load ETL: Key Differences](https://hevodata.com/learn/incremental-data-load-vs-full-load/#:~:text=Incremental%20loading%20is%20the%20process,resources%2C%20and%20preserves%20historical%20data))), e *operações em lote* otimizadas no banco (aproveitando upsert nativo ([Spring and PostgreSQL: Make Your Database Inserts 30 Times Faster | HackerNoon](https://hackernoon.com/spring-and-postgresql-make-your-database-inserts-30-times-faster#:~:text=Tests%20indicate%20that%20upsert%20operations,upsert%20queries%20for%20any%20object)) e batched inserts com mínima sobrecarga ([PostgreSQL reWriteBatchedInserts configuration property - Vlad Mihalcea](https://vladmihalcea.com/postgresql-multi-row-insert-rewritebatchedinserts-property/#:~:text=This%20time%2C%20we%20only%20have,processing%20on%20the%20database%20side))). Ao parallelizar tarefas e sintonizar parâmetros (tamanho de chunk, threads, uso de AsyncItemProcessor, etc.), conseguimos throughput alto para completar o processamento diário dentro da janela necessária. Em suma, use as melhores práticas de ETL incremental e as facilidades do Spring Batch para escalar: **ler, comparar e atualizar em massa** de forma confiável e rápida. Cada componente – leitura, processamento de diffs e escrita – foi ajustado para lidar com volume, garantindo a atualização consistente do PostgreSQL com desempenho. 

**Referências Utilizadas:** Spring Batch documentação e exemplos; conceitos de incremental load ([Incremental Data Load vs Full Load ETL: Key Differences](https://hevodata.com/learn/incremental-data-load-vs-full-load/#:~:text=Incremental%20loading%20is%20the%20process,resources%2C%20and%20preserves%20historical%20data)); vantagens de formatos colunares Parquet/Arrow ([Apache Arrow e Java: Transferência de Big Data na velocidade da luz - InfoQ](https://www.infoq.com/br/articles/apache-arrow-java#:~:text=Observem%20que%20o%20Apache%20Arrow,%C3%A9%20puramente%20por%20motivos%20did%C3%A1ticos)) ([Apache Parquet: How to be a hero with the open-source columnar data format | Openbridge](https://blog.openbridge.com/how-to-be-a-hero-with-powerful-parquet-google-and-amazon-f2ae0f35ee04#:~:text=Parquet%20performance%2C%20when%20compared%20to,CSV)); práticas de paralelismo em batch ([Spring batch for large files(1billion to 5billion flat file data) - Stack Overflow](https://stackoverflow.com/questions/47933599/spring-batch-for-large-files1billion-to-5billion-flat-file-data#:~:text=Secondly%2C%20you%20are%20correct%20in,can%20be%20easily%20grouped%20later)); otimização de upsert e batch no PostgreSQL ([Spring and PostgreSQL: Make Your Database Inserts 30 Times Faster | HackerNoon](https://hackernoon.com/spring-and-postgresql-make-your-database-inserts-30-times-faster#:~:text=Tests%20indicate%20that%20upsert%20operations,upsert%20queries%20for%20any%20object)) ([PostgreSQL reWriteBatchedInserts configuration property - Vlad Mihalcea](https://vladmihalcea.com/postgresql-multi-row-insert-rewritebatchedinserts-property/#:~:text=This%20time%2C%20we%20only%20have,processing%20on%20the%20database%20side)); entre outros.
