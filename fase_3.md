Excelente! Vamos agora detalhar ao máximo a Fase 3 do projeto "LlamaFlow", que é a fase de **Desenvolvimento do Pipeline Nextflow**. Nesta fase, vamos criar o pipeline que receberá os dados do n8n, fará o fine-tuning do LLM e enviará o modelo ajustado de volta ao armazenamento.

**Fase 3: Desenvolvimento do Pipeline Nextflow (Detalhado)**

**Objetivo:** Criar um pipeline Nextflow que receba o caminho do arquivo de dados no S3, realize o fine-tuning de um LLM e envie o modelo ajustado para o S3.

**Atividade 1: Criar um Arquivo `nextflow.config` (Detalhado)**

*   **1.1 Criar o Arquivo `nextflow.config`:**
    *   **Ação:** No seu editor de código/IDE, criar um novo arquivo chamado `nextflow.config` dentro do diretório do Nextflow (ex: `n8n-nextflow-project/nextflow`).
*   **1.2 Configurar as Credenciais do S3/MinIO:**
    *   **Ação:** Adicionar as seguintes configurações ao arquivo `nextflow.config`:
        ```groovy
        aws {
          accessKey = System.getenv('AWS_ACCESS_KEY') ?: "<sua_access_key>"
          secretKey = System.getenv('AWS_SECRET_KEY') ?: "<sua_secret_key>"
          region = 'us-east-1'  // Ajustar a região se necessário
          endpoint = 'http://localhost:9000' // Ajustar para o seu endpoint MinIO ou S3
          force_path_style = true // Adicionar se estiver usando MinIO
        }
        ```
        *   **Explicação:**
            *   `accessKey`: Obtém a access key do S3 ou MinIO de uma variável de ambiente, caso não exista, usa o valor default.
            *   `secretKey`: Obtém a secret key do S3 ou MinIO de uma variável de ambiente, caso não exista, usa o valor default.
            *   `region`: Define a região do seu bucket S3.
            *   `endpoint`: Define o endpoint do S3 ou MinIO.
            *   `force_path_style`: Necessário para MinIO, força o uso do estilo path para URLs.
*   **1.3 Configurar o Executor (Local/Docker/Kubernetes):**
    *   **Ação:** Adicionar a seguinte configuração para executar o Nextflow localmente:
        ```groovy
        executor {
          name = 'local'
        }
        ```
    *   **Opção Docker:** Se quiser usar Docker, adicione:
        ```groovy
        executor {
           name = 'docker'
           docker {
              enabled = true
              runOptions = '--rm'
           }
        }
       ```
       * Necessário instalar o Docker.
    *   **Opção Kubernetes/K3s:** Se quiser usar Kubernetes/K3s, adicione:
         ```groovy
         executor {
          name = 'kubernetes'
          k8s {
              namespace = 'default' // Ajustar se necessário
           }
          }
         ```
         * Necessário ter o K3s instalado e o `kubectl` configurado
*   **1.4 Configurar Logs (Opcional):**
    *   **Ação:** Configurar o log do Nextflow para um nível mais detalhado:
        ```groovy
        log {
          level = 'DEBUG'
        }
        ```
*   **1.5 Configurar Cache (Opcional):**
    *   **Ação:** Para acelerar execuções futuras, configurar o cache:
        ```groovy
        cache {
          enabled = true
          dir = '.nextflow_cache'
        }
        ```
* **1.6 Configurar variáveis de ambiente (Opcional):**
   * **Ação:** caso precise usar variáveis de ambiente no seu projeto, configure-as nesta seção:
    ```groovy
    env {
      YOUR_VAR = System.getenv('YOUR_VAR') ?: 'your_default_value'
    }
    ```

**Atividade 2: Criar um Arquivo `workflow.nf` (Detalhado)**

*   **2.1 Criar o Arquivo `workflow.nf`:**
    *   **Ação:** No seu editor de código/IDE, criar um novo arquivo chamado `workflow.nf` dentro do diretório do Nextflow (ex: `n8n-nextflow-project/nextflow`).
*   **2.2 Adicionar a Declaração de Parâmetros:**
    *   **Ação:** Adicionar a declaração de parâmetros que serão recebidos do n8n:
         ```groovy
         params.input = ''
         params.model_name = 'meta-llama/Llama-2-7b-hf'
         ```
        *   `params.input`: Recebe o caminho para o arquivo no S3.
        *   `params.model_name`: Recebe o nome do modelo que será usado no fine-tuning (default = `meta-llama/Llama-2-7b-hf`).
*   **2.3 Criar um Processo Básico para Download do S3:**
    *   **Ação:** Adicionar um `process` para fazer o download do arquivo do S3:
        ```groovy
        process DownloadData {
            input:
                path input_file
            output:
               path "data.json"

            script:
            """
                aws s3 cp "$input_file" ./data.json
            """
        }
        ```
       *   `input: path input_file`: Define um input do tipo path, este input vem do parâmetro `input` do seu arquivo nextflow.
       *   `output: path "data.json"`: Define a saída deste processo como um arquivo `data.json` (poderia ser qualquer nome ou formato).
       *   `script`: Executa um script bash onde o arquivo será baixado do S3 com o comando `aws s3 cp`.
*   **2.4 Criar o Workflow e Executar o Processo de Download:**
    *   **Ação:** Criar um workflow que execute o `process` `DownloadData`.
        ```groovy
         workflow {
            DownloadData(params.input)
         }
        ```
*   **2.5 Adicionar um processo de Fine-Tuning:**
    *   **Ação:** Incluir um processo para o fine-tuning do LLM.
    *   **Observação:** Este é um processo complexo e dependerá da biblioteca e técnica utilizada para o fine-tuning (pode usar Transformers + LoRA, etc.). Adapte o código conforme a sua necessidade.
        * **Exemplo usando Llama + LoRA + Transformers:**
          ```groovy
          process FineTuneModel {
              container 'ghcr.io/huggingface/transformers:latest'
              input:
                  path data
              output:
                  path "model/*" // O diretorio model contem o modelo salvo
              script:
              """
                  python finetune.py \
                      --model_name "${params.model_name}" \
                      --data "$data"
              """
          }
          ```
          *   `container:`: Use a imagem do Docker da sua preferência que tenha o ambiente para rodar o fine-tuning (use `ghcr.io/huggingface/transformers:latest` para fine-tuning com transformers).
          *   `input: path data`: Indica que o processo recebe como entrada o arquivo de dados.
          *   `output: path "model/*"`: Define que a saída do processo será um diretório `model` contendo o modelo salvo.
          *  `script:` Script bash com comando para executar o fine-tuning com python, os arquivos `finetune.py` (e outros) devem estar dentro do repositório nextflow,
          *  `--model_name "${params.model_name}"`: Recebe o nome do modelo que será utilizado do parâmetro `model_name` do arquivo `workflow.nf`
          *  `--data "$data"`: Recebe o caminho para o arquivo de dados que será usado no fine-tuning.
       * **Observação:** Aqui você precisa criar um arquivo python `finetune.py` com seu script de fine-tuning e colocar dentro do mesmo diretório do `workflow.nf`, a lógica deste arquivo é de sua responsabilidade, neste projeto utilizaremos como exemplo um script que recebe o arquivo de dados e realiza o fine-tuning, o modelo será salvo no diretório `/model` do processo.
*  **2.6 Incluir processo de upload do modelo**
    * **Ação:** Incluir um processo para enviar o modelo ajustado para o S3:
          ```groovy
           process UploadModel {
                input:
                   path model_dir
                output:
                   path "model_s3.txt"

                script:
                """
                  aws s3 cp --recursive "$model_dir" s3://llamaflow-data/model/
                  echo "modelo enviado para s3://llamaflow-data/model/" > model_s3.txt
                """
           }
           ```
            *   `input: path model_dir`: Recebe como input o diretório com o modelo ajustado.
            *   `output: path "model_s3.txt"`: gera um arquivo como saida para confirmar o envio para o S3.
            *  `script:` script bash que envia o modelo ajustado para o S3 usando `aws s3 cp`.
*   **2.7 Adicionar a lógica do workflow:**
   * **Ação:**  Adicionar ao seu workflow a lógica de processamento, utilizando os processos de download, fine-tuning e upload, conectando-os através de um canal, adicione o seguinte ao seu arquivo `workflow.nf`:
        ```groovy
        workflow {
            Channel.fromPath(params.input)
                | DownloadData
                | FineTuneModel
                | UploadModel
        }
        ```
        *  `Channel.fromPath(params.input)`: cria um canal com o caminho para o arquivo no S3, esse arquivo será o `input` do processo `DownloadData`.
        *  O resultado do processo `DownloadData` vira o input do processo `FineTuneModel`,
        *  O resultado do processo `FineTuneModel` vira o input do processo `UploadModel`.
        *  Isso faz um encadeamento dos processos para criação do seu workflow.

**Atividade 3: Testar o Pipeline Nextflow (Detalhado)**

*   **3.1 Executar o Pipeline Localmente:**
    *   **Ação:** Abrir um terminal ou prompt de comando no diretório `nextflow` do projeto (ex: `n8n-nextflow-project/nextflow`).
    *   **Comando:** Digitar o seguinte comando (ajustar conforme o caso):
        ```bash
        export AWS_ACCESS_KEY=<sua_access_key>; \
        export AWS_SECRET_KEY=<sua_secret_key>; \
        nextflow run workflow.nf -params.input='s3://llamaflow-data/data-1708816668362.json'
        ```
        *   **Explicação:**
            *   `export AWS_ACCESS_KEY=<sua_access_key>`: Define a variável de ambiente com sua access key.
            *   `export AWS_SECRET_KEY=<sua_secret_key>`: Define a variável de ambiente com sua secret key.
            *   `nextflow run workflow.nf`: Executa o pipeline definido no arquivo `workflow.nf`.
            *  `-params.input='s3://llamaflow-data/data-1708816668362.json'`: Define o valor do parâmetro `input` (ajuste para o caminho do seu arquivo no S3).
    *   **Resultado Esperado:** O Nextflow irá baixar o arquivo, executar o `process` de fine-tuning e upload do modelo,
       * Verifique os logs para ver se o processo está executando corretamente.
*   **3.2 Verificar os Resultados:**
    *   **Ação:** Verificar se o arquivo foi baixado corretamente (verifique o log do processo `DownloadData`).
    *   **Ação:** Verificar se o fine-tuning foi executado corretamente (verifique o log do processo `FineTuneModel` e os arquivos salvos).
    *   **Ação:** Verificar se o modelo foi enviado para o S3 (verifique o log do processo `UploadModel`).

**Atividade 4: Implementar o `Process` de Fine-Tuning (Detalhado)**

*   **4.1 Criar o Script de Fine-Tuning `finetune.py`:**
    *   **Ação:** Criar um arquivo python chamado `finetune.py` no diretório do nextflow
    *   **Ação:** Adicionar o seguinte código (ajuste conforme sua necessidade):
        ```python
        import argparse
        import json
        from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments, Trainer
        from datasets import Dataset

        parser = argparse.ArgumentParser()
        parser.add_argument('--model_name', type=str, default='meta-llama/Llama-2-7b-hf', help="Nome do modelo no Hugging Face")
        parser.add_argument('--data', type=str, help="Caminho para o arquivo de dados")
        args = parser.parse_args()

        model_name = args.model_name
        data_path = args.data

        print(f"Fine-tuning model: {model_name} with data {data_path}")

        tokenizer = AutoTokenizer.from_pretrained(model_name)
        model = AutoModelForCausalLM.from_pretrained(model_name)


        def format_text(example):
            text = f"Texto: {example['text']}\nFonte: {example['source']}"
            return {'text': text}
          
        # Load the JSON file
        with open(data_path, 'r') as f:
          data = [json.loads(line.strip()) for line in f]
          
        dataset = Dataset.from_list(data)
        dataset = dataset.map(format_text)
        dataset = dataset.map(lambda examples: tokenizer(examples["text"], truncation=True, padding='max_length', max_length=128))
        dataset = dataset.remove_columns(["text", "source"])

        training_args = TrainingArguments(
            output_dir='./model',
            num_train_epochs=1,
            per_device_train_batch_size=8,
            save_steps=100,
            save_total_limit=2,
            learning_rate=2e-5,
            logging_dir='./logs'
        )
        trainer = Trainer(
          model=model,
          args=training_args,
          train_dataset=dataset
        )

        trainer.train()
        trainer.save_model("./model")
        ```
    *  **Explicação:**
          *   O script usa a biblioteca `transformers` para baixar e ajustar o modelo.
          *   Recebe como argumento o modelo a ser utilizado e o caminho para o arquivo JSON,
          *   Lê o arquivo JSON com os dados de fine-tuning, formata o texto e transforma em um Dataset.
          *   Realiza o fine-tuning do modelo,
          *  Salva o modelo ajustado no diretório `/model`
* **4.2 Incluir dependências no `requirements.txt`**
    * **Ação:** Crie um arquivo `requirements.txt` no seu diretório `nextflow`.
    * **Ação:** Adicione as dependências necessárias para o python `finetune.py`
        ```
        transformers
        datasets
        torch
        ```
    * **Ação:** Altere o processo `FineTuneModel` para instalar as dependências:
    ```groovy
          process FineTuneModel {
              container 'ghcr.io/huggingface/transformers:latest'
              input:
                  path data
              output:
                  path "model/*" // O diretório model contem o modelo salvo
              script:
              """
                  pip install -r requirements.txt
                  python finetune.py \
                      --model_name "${params.model_name}" \
                      --data "$data"
              """
          }
    ```
*   **4.3 Ajustar o Processo `FineTuneModel`:**
    *   **Ação:** Ajustar o `script` do processo `FineTuneModel` para usar o script `finetune.py` e passar os parâmetros necessários.
    * **Ação:** Adicionar as dependências necessárias para o python.
* **4.4 Testar novamente o pipeline:**
     * Repita os passos do **3.1 Executar o Pipeline Localmente**, verifique agora que o script de fine-tuning está sendo executado com os dados, o modelo ajustado será salvo em um diretório `/model` dentro do processo.

**Atividade 5: Implementar o Upload do Modelo Ajustado (Detalhado)**

*   **5.1 Ajustar o Processo `UploadModel`:**
    *   **Ação:** Ajustar o script do processo `UploadModel` para fazer o upload do diretório com o modelo ajustado para o S3.
     * **Observação:** Se o modelo for muito grande, você poderá criar um script adicional para compactar o diretório e enviar apenas um arquivo para o S3 e no n8n fazer a descompactação do arquivo.
*   **5.2 Testar novamente o pipeline:**
     * Repita os passos do **3.1 Executar o Pipeline Localmente**, verifique agora que o modelo está sendo enviado para o S3.

**Resultado Final da Fase 3:**

Ao final da Fase 3, você terá:

*   Um arquivo `nextflow.config` configurado para o seu ambiente.
*   Um arquivo `workflow.nf` que define um pipeline completo:
    *   Baixa os dados do S3.
    *   Faz o fine-tuning do LLM com `finetune.py`.
    *   Envia o modelo ajustado para o S3.
*   O pipeline testado e funcionando localmente.

Com este pipeline desenvolvido, você estará pronto para fazer a integração completa com o n8n. Lembre-se de testar cada etapa e o pipeline completo.

Se tiver alguma dúvida, me diga! 😉
