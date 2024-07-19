# AWSGlueETL
ETL com AWS Glue

# Projeto de Engenharia de Dados com AWS Glue, S3 e Crawler

Este projeto demonstra um pipeline de engenharia de dados utilizando AWS Glue, S3 e Glue Crawler. O pipeline envolve a extração de dados de um catálogo, a transformação usando PySpark e a gravação dos resultados agregados de volta no S3.

## Índice

- [Introdução](#introdução)
- [Tecnologias Utilizadas](#tecnologias-utilizadas)
- [Configuração](#configuração)
- [Executando o Script](#executando-o-script)
- [Tratamento de Erros](#tratamento-de-erros)
- [Licença](#licença)

## Introdução

Este projeto demonstra um fluxo de trabalho de engenharia de dados que lê dados de um catálogo do AWS Glue, realiza transformações usando PySpark e escreve os resultados agregados em um bucket S3. Os dados são agregados por década, contando o número de filmes e calculando a média de avaliação para cada década.

## Tecnologias Utilizadas

- **AWS Glue**: Serviço de ETL gerenciado para preparar e transformar dados.
- **AWS S3**: Armazenamento de objetos escalável para armazenar dados brutos e processados.
- **PySpark**: API em Python para Apache Spark para processar grandes conjuntos de dados.
- **AWS Glue Crawler**: Descobre e cataloga automaticamente dados no S3.

## Configuração

1. **Configuração da AWS**:
    - Certifique-se de ter uma conta AWS.
    - Crie um banco de dados e uma tabela no Glue.
    - Configure um bucket S3 (por exemplo, `whizlabss`).
    - Verifique se a função de serviço do AWS Glue tem permissões apropriadas para ler do catálogo do Glue e gravar no S3.

2. **Script do Glue**:
    - Salve o script fornecido como `glue_script.py`.

3. **AWS CLI**:
    - Instale e configure o AWS CLI com permissões apropriadas.

## Executando o Script

Para executar o job do Glue, siga estes passos:

1. **Envie o Script**:
    - Envie o `glue_script.py` para o caminho do script do job no S3.

2. **Crie um Job no Glue**:
    - No console do AWS Glue, crie um novo job do Glue.
    - Especifique a localização do script no S3.
    - Configure o job com a função IAM necessária e os recursos.

3. **Execute o Job do Glue**:
    - Inicie o job do Glue a partir do console do AWS Glue.

### Visão Geral do Script

O script realiza os seguintes passos:

1. **Inicializa os Contextos do Glue e Spark**:
    ```python
    from datetime import datetime
    from pyspark.context import SparkContext
    import pyspark.sql.functions as f
    from awsglue.utils import getResolvedOptions
    from pyspark.context import SparkContext
    from awsglue.context import GlueContext
    from awsglue.dynamicframe import DynamicFrame
    from awsglue.job import Job
    import sys

    spark_context = SparkContext.getOrCreate()
    glue_context = GlueContext(spark_context)
    session = glue_context.spark_session
    ```

2. **Define Parâmetros**:
    ```python
    glue_db = "demo"
    glue_tbl = "sample_data_csv"
    s3_write_path = "s3://whizlabss"  # Atualizado para o bucket existente
    ```

3. **Lê os Dados do Catálogo do Glue**:
    ```python
    dynamic_frame_read = glue_context.create_dynamic_frame.from_catalog(database = glue_db, table_name = glue_tbl)
    data_frame = dynamic_frame_read.toDF()
    ```

4. **Transforma os Dados**:
    ```python
    decode_col = f.floor(data_frame["year"]/10)*10
    data_frame = data_frame.withColumn("decade", decode_col)

    data_frame_aggregated = data_frame.groupby("decade").agg(
        f.count(f.col("movie")).alias('movie_count'),
        f.mean(f.col("rating")).alias('rating_mean'),
    )

    data_frame_aggregated = data_frame_aggregated.orderBy(f.desc("movie_count"))
    data_frame_aggregated.show(10)
    ```

5. **Grava os Dados no S3**:
    ```python
    data_frame_aggregated = data_frame_aggregated.repartition(1)
    dynamic_frame_write = DynamicFrame.fromDF(data_frame_aggregated, glue_context, "dynamic_frame_write")

    glue_context.write_dynamic_frame.from_options(
        frame = dynamic_frame_write,
        connection_type = "s3",
        connection_options = {
            "path": s3_write_path,
        },
        format = "csv"
    )
    ```

6. **Registra o Tempo de Início e Fim**:
    ```python
    dt_start = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print("Start time:", dt_start)

    dt_end = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print("End time:", dt_end)
    ```

## Tratamento de Erros

Em caso de erros, verifique:
- Se o bucket S3 especificado existe.
- Se a função IAM possui as permissões necessárias.
- Se os nomes do catálogo e da tabela do Glue estão corretos.


